linux内核异步内存回收的另一个思路探索：识别出冷热文件的冷热区域，更精准的回收冷page
-------------------------------------
【问题背景】

线上经过遇到，因pagecache太多且可直接分配的内存太少，导致业务进程内存分配时阻塞，发生性能抖动、卡顿，严重影响业务。并且，网卡软中断里分配page因进入内存分配流程的slow分支而频繁触发dump_stack告警，严重的还会导致内核crash。pagecache很多是谁导致的？测试证实，根源是有不少文件在读写后产生了100M到1G大小的pagecache。但是systemtap或kprobe动态跟踪并打印这些文件被读写文件页page索引，发现总是只打印几个固定的文件页page索引。这说明这些文件的pagecache中只有少部分被经常访问，大部分都很少被访问。这些很少访问的文件的pagecache对应的就是冷文件页page，如果能准确识别出这些冷page并回收掉，将腾出大量内存，还基本不会发生refault。

因此产生了一个想法，内存回收的单位能否是文件？并且要能找出产生pagecache很多但大部分pagecache都很少被访问的文件，专门回收这些文件的pagecache对应的文件页page，内存回收效率将很高。如果能有效规避内存回收时令人都疼的refault问题，会更好。最后，大部分互联网公司没有能力修改定制内核，用的都是红帽、ubuntu发行版原生的内核，而要想有效解决pagecache太多带来的内存回收和分配难题，不从内核里解决根本不行。并且，据了解google对安卓手机厂商修改定制内核的限制也越来越严格！如果能把这个以文件为单位的内存回收方案做成一个内核ko，既能实现对pagecache高效的内存回收，还不用修改编译内核，就更完美了。经过艰难探索，这个异步内存回收方案已经实现了：达到了预期的异步内存回收效果，并且cpu性能损耗还较低！

-------------------------------------

【基本设计思路】

内存回收的单位是一个个文件，再把文件的pagecache分成一个个小区域(或者叫小单元)，一个区域由4个索引连续的文件页page组成。比如把索引是0到3的文件页page组成一个小区域，索引是4到7的文件页page再组成一个小区域，其他区域类推。一个区域内的文件页page冷热属性接近，每个区域分配一个file_area结构，精确统计该区域内的page的访问频次。然后，提前判断出文件的pagecache哪些区域是进程频繁访问的(即热区域，该区域的文件页page频繁被读写)，哪些区域是进程很少访问的(即冷区域，该区域的文件页page很少被读写)。异步内存回收线程工作时，一个个遍历指定数目的文件，再把每个文件pagecache的冷区域找出来，最后回收掉冷区域对应的文件页page。

-------------------------------------

【方案优势】

1：不用修改编译内核，直接编译成内核ko(主要用到struct address_space结构最后的预留字段)，可以单独作为一个内存回收工具使用

2：系统总有较多数目的文件，产生的pagecache很多，但是大部分pagecache都很少访问，这种文件的pagecache中冷区域占比高(称为冷文件)。内存回收时优先找到这种文件，因为能从这种文件找到很多的冷区域，继而高效回收到很多的冷文件页page

3：有些文件的pagecache大部分都被频繁读写(称为热文件)，这种文件的pagecache中热区域占比很高。内存回收时尽量避开这种文件，不回收这种文件的文件页page，因为有较大概率会发生refault

4：针对内存回收后发生refault的文件页page，该文件页page所在区域的file_area数据结构将移入所属文件的refault链表，长时间禁止回收该page，有效避免频繁refault

5：可以精确统计每个文件pagecache的每个区域里的文件页page的访问频次，因此可以精确控制文件页page多久没访问后再回收。比如，设定文件页page 1个小时都没被访问过后，判定为冷文件页，然后才能回收掉。这个功能目前已经实现了，并可以通过proc接口设置这个时间


其他优势

1：针对加载该ko前已产生的pagecache，开发了异步drop_cache功能，可以更平滑的回收这些pagecache

2：每个内存回收周期，被访问的文件页page所在区域的file_area数据结构将移动到的自定义的内存回收链表头(有一定策略，不是每次都移动)，这样链表尾对应的都是冷page对应的file_area。内存回收时先从链表尾扫描这些冷page对应的file_area，内存回收效率比较高

-------------------------------------
【方案详细设计】

查看文章 https://blog.csdn.net/hu1610552336/article/details/132331352

-------------------------------------
【测试效果】

目前主要测试了如下几个场景

1：每隔一段时间读写10个左右的大小在100M~2G的文件，然后cat /proc/meminfo观察系统cache总量。这些文件产生的pagecache总能在规定的时间内(这个时间可通过proc接口调整)不被访问后，被判定为冷page，然后被回收掉，系统的总cache量总能跌落回最初水平。

2：编译内核，把cpu消耗光。然后每隔一段时间读写几个大小在1G左右的文件，这些文件产生的pagecache也能在长时间不被访问后，及时被回收掉。top观察异步内存回收线程的cpu使用率，低于5%，大部分时间接近0。并且，perf top也没发现热点函数。这说明本异步内存回收方案的cpu损耗是很低的。

3：SATA盘，每次drop_caches后，cat 1G大小的文件，启动/禁止该异步内存回收功能，总耗时都在12.5s左右(iostat显示IO读流量在80M/s~100M/s)。这说明该异步内存回收模块并没有带来明显的性能损耗！当然，不同硬盘、不同场景估计测试结果会有差异，后续再多找些场景测试。

-------------------------------------
【使用方法】

红帽8/9系列，centos8(内核版本4.18.0-240)、rocky9(内核版本5.14.0-284.11.1)已经适配，可直接make编译成ko。其他内核发行版也可以直接编译成内核ko使用：

1：安卓手机开源内核(一加)，需要把本源码里 mapping->rh_reserved1 改成 mapping->android_kabi_reserved1，https://github.com/OnePlusOSS/android_kernel_oneplus_sm8550

2: 腾讯opencloud开源内核，需要把本源码里 mapping->rh_reserved1 改成 mapping->kabi_reserved1，https://github.com/OpenCloudOS/OpenCloudOS-Kernel-Stream

3: 阿里龙蜥开源内核，需要把本源码里 mapping->rh_reserved1 改成 mapping->ck_reserved1，https://gitee.com/anolis/cloud-kernel


说明：可能不同的内核编译该ko时，会遇到内核版本不同内存回收有关函数有差异而导致编译失败，此时需要一些修改。后期尽快多适配不同的内核，目前仅适配了红帽的内核。
