-xx系列命令分为2种  
<font color=red>**①boolean型**</font>：-XX:+PrintFlagsInitial  布尔类型用加减表示是否开启  

-XX:+PrintFlagsInitial  查看jvm默认的配置  

-XX:+PrintFlagsFinal  查看用户或者jvm修改后的配置（列表，包括改的和没改的）  

-XX:+PrintCommandLineFlags  查看用户或者jvm修改后的配置（只列举改的）

-XX:+PrintGCDetails 打印GC详细信息  

-XX:+HeapDumpOnOutOfMemoryError 发生oom异常时候，自动生成dump文件  

<font color=red>**②kv类型**</font>：-XX:MaxHeapSize=123
（温馨提示：Xmx和Xms他们只是缩写MaxHeapSize、InitailHeapSize）

-Xmx：最大堆内存(-XX:MaxHeapSize) 默认物理机的1/4  
-Xms：初始内存(-XX:InitailHeapSize) 默认物理机的1/64  
一般这两个值设置一样的数值，避免超过Xms后，内存重新整理。  

-Xmn：新生代内存大小(-XX:MaxNewSize)    
一般为Xmx的1/3。新生代包括Edgen和Survivor区，Survivor还被平均分为了两块 from space和to space，默认情况Edgen和2个Survivor大小比例8：2

-Xss：每个线程栈大小(-XX:ThreadStackSize)  

-XX:MetaSpaceSize 元空间初始大小，元空间使用本地物理内存即内存条限制

-XX:MaxTenuringThreshold 设置新生代到老年代最大的GC次数，默认15次