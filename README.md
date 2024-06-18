## 祭祖期末project

**以下路徑預設使用在gem5目錄執行**

### (Q1) GEM5 + NVMAIN BUILD-UP

ubuntu-18.04(推薦版本)開不了機，查了一下發現是apple晶片在搞事，後來使用ubuntu-24.04使用docker執行`ubuntu:18.04`。

先在24.04的虛擬機下載gem5的東西，再載入到18.04的虛擬機裡面，剩下的就照eeclass提供的教學做。

### (Q2) Enable L3 last level cache in GEM5 + NVMAIN

基本上都是把L2複製然後稍微改一下

在`./config/common/Caches.py`複製L2Cache的class製作L3Cache的class
```python
class L3Cache(Cache):
    assoc = 64
    tag_latency = 32
    data_latency = 32
    response_latency = 32
    mshrs = 32
    tgts_per_mshr = 24
    write_buffers = 16
```

在`./config/common/CacheConfig.py`找到
```python
dcache_class, icache_class, l2_cache_class, walk_cache_class = (
    core.O3_ARM_v7a_DCache,
    core.O3_ARM_v7a_ICache,
    core.O3_ARM_v7aL2,
    None,
)
```

把剛剛的L3Cache加上去變成
```python
dcache_class, icache_class, l2_cache_class, walk_cache_class,l3_cache_class = (
    core.O3_ARM_v7a_DCache,
    core.O3_ARM_v7a_ICache,
    core.O3_ARM_v7aL2,
    None,
    core.O3_ARM_v7aL3,
)
```

再到底下找到
```python
else:
        dcache_class, icache_class, l2_cache_class, walk_cache_class = (
            L1_DCache,
            L1_ICache,
            L2Cache,
            None,
        )
```

一樣補上L3Cache的部分
```python
else:
        dcache_class, icache_class, l2_cache_class, walk_cache_class,l3_cache_class = (
            L1_DCache,
            L1_ICache,
            L2Cache,
            None,
            L3Cache,
        )
```

再到底下找到
```python
if options.l2cache:
        system.l2 = l2_cache_class(
            clk_domain=system.cpu_clk_domain, **_get_cache_opts("l2", options)
        )

        system.tol2bus = L2XBar(clk_domain=system.cpu_clk_domain)
        system.l2.cpu_side = system.tol2bus.mem_side_ports
        system.l2.mem_side = system.membus.cpu_side_ports
```

把這段改一下變成`if(l2 and l3)... elif(l2)...`的樣子
```python
if options.l2cache and  options.l3cache:
        system.l2 = l2_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l2_size,
                                   assoc=options.l2_assoc)
        system.l3 = l3_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l3_size,
                                   assoc=options.l3_assoc)

        system.tol2bus = L2XBar(clk_domain = system.cpu_clk_domain)
        system.tol3bus = L3XBar(clk_domain = system.cpu_clk_domain)

        system.l2.cpu_side = system.tol2bus.master
        system.l2.mem_side = system.tol3bus.slave

        system.l3.cpu_side = system.tol3bus.master
        system.l3.mem_side = system.membus.slave
        
elif options.l2cache:
        system.l2 = l2_cache_class(
            clk_domain=system.cpu_clk_domain, **_get_cache_opts("l2", options)
        )

        system.tol2bus = L2XBar(clk_domain=system.cpu_clk_domain)
        system.l2.cpu_side = system.tol2bus.mem_side_ports
        system.l2.mem_side = system.membus.cpu_side_ports
```

到`./src/mem/XBar.py`增加L3XBar（一樣是複製L2XBar去改）
```python
class L3XBar(CoherentXBar):
    width = 32
    
    frontend_latency = 1
    forward_latency = 0
    response_latency = 1
    snoop_response_latency = 1
```

到`./src/cpu/BaseCPU.py`補上L3Cache的一些東西

先import剛剛增加的L3XBar
```python
from m5.objects.XBar import L3XBar
```

然後在底下的`addTwoLevelCacheHierarchy`函式底下補上
```python
def addThreeLevelCacheHierarchy(self, ic, dc, l3c, iwc = None, dwc = None):
        self.addPrivateSplitL1Caches(ic, dc, iwc, dwc)
        self.toL3Bus = L3XBar()
        self.connectCachedPorts(self.toL3Bus)
        self.l3cache = l3c
        self.toL2Bus.master = self.l3cache.cpu_side
        self._cached_ports = ['l3cache.mem_side']
```

再到`./config/common/Options.py`找到
```python
parser.add_argument("--caches", action="store_true")
parser.add_argument("--l2cache", action="store_true")
parser.add_argument("--num-dirs", type=int, default=1)
```

補上L3Cache的option
```python
parser.add_argument("--caches", action="store_true")
parser.add_argument("--l2cache", action="store_true")
parser.add_argument("--l3cache", action="store_true")
parser.add_argument("--num-dirs", type=int, default=1)
```

弄完這些後再混合編譯一次
```
scons EXTRAS=../NVmain build/X86/gem5.opt
```

最後執行
```
./build/X86/gem5.opt configs/example/se.py -c tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

### (Q3) Config last level cache to 2-way and full-way associative cache and test performance

使用指令編譯`./quicksort.c`產生`./quicksort`
```
gcc --static quicksort.c -o quicksort
```
但我的虛擬機是arrch64，無法編譯出x86檔案，所以我嘗試使用windows桌機編譯後傳回虛擬機裡的docker虛擬機，但仍不是x86的，最後決定到網路上找別人編譯好的檔案，後面的multuply編譯同理。

先執行2-way的執行指令並將結果存到`./Q3/2-way/log.txt`
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config>./Q3/2-way/log.txt
```

然後將`./m5out`複製到`./Q3/2-way/`
```
cp ./m5out ./Q3/2-way/ -r
```

再執行full-way的(--l3_assoc改成1)，並將結果存到`./Q3/full-way/log.txt`
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=1 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config>./Q3/full-way/log.txt
```

然後將`./m5out`複製到`./Q3/full-way/`
```
cp ./m5out ./Q3/full-way/ -r
```

結果如下：
![](https://github.com/poyen0806/gem5/blob/master/result_image/Q3.png)

**備註：原本的quicksort是100000，較看不出差別，所以改成1000000**

### (Q4) Modify last level cache policy based on frequency based replacement policy

**我這邊是選用multiply**

先執行一次，並存到`./Q4/policy/log.txt`
```
./build/X86/gem5.opt configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config>./Q4/policy/log.txt
```

然後將`./m5out`複製到`./Q4/policy/`
```
cp ./m5out ./Q4/policy/ -r
```

到`./config/common/Caches.py`找到`L3Cache`改成
```python
class L3Cache(Cache):
    assoc = 64
    tag_latency = 32
    data_latency = 32
    response_latency = 32
    mshrs = 32
    tgts_per_mshr = 24
    write_buffers = 16
    
    # 加了這個
    replacement_policy = Param.BaseReplacementPolicy(
        LFURP(), "Replacement policy"
    )
```

然後混合編譯
```
scons EXTRAS=../NVmain build/X86/gem5.opt
```

再執行一次，並存到`./Q4/freq_base_policy/log.txt`
```
./build/X86/gem5.opt configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config>./Q4/freq_base_policy/log.txt
```

然後將`./m5out`複製到`./Q4/freq_base_policy/`
```
cp ./m5out ./Q4/freq_base_policy/ -r
```

結果如下：
![](https://github.com/poyen0806/gem5/blob/master/result_image/Q4.png)

### (Q5) Test the performance of write back and write through policy based on 4-way associative cache with isscc_pcm

**我quicksort跟multiply都有做，這邊以multiply為例**

先執行一次，並存到`./Q5/write_back/multiply/log.txt`
```
./build/X86/gem5.opt configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=4 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config>./Q5/write_back/multiply/log.txt
```

然後將`./m5out`複製到`./Q5/write_back/multiply/`
```
cp ./m5out ./Q5/write_back/multiply/ -r
```

到`./src/mem/cache/base.cc`找到以下片段(大概在1072行左右)
```cc
else if (blk && (pkt->needsWritable() ? blk->isWritable() :
                       blk->isReadable())) {
        // OK to satisfy access
        incHitCount(pkt);
        satisfyRequest(pkt, blk);
        maintainClusivity(pkt->fromCache(), blk);

        return true;
    }
```

把這部分加上一些東西改成
```cc
else if (blk && (pkt->needsWritable() ? blk->isWritable() :
                       blk->isReadable())) {
        // OK to satisfy access
        incHitCount(pkt);
        satisfyRequest(pkt, blk);
        maintainClusivity(pkt->fromCache(), blk);

        // 加上這段
        if (blk->isWritable()) {
            PacketPtr writeclean_pkt = writecleanBlk(
                            blk, pkt->req->getDest(), pkt->id);
            writebacks.push_back(writeclean_pkt);
        }

        return true;
    }
```

然後混合編譯
```
scons EXTRAS=../NVmain build/X86/gem5.opt
```

再執行一次，並存到`./Q5/write_through/multiply/log.txt`
```
./build/X86/gem5.opt configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=4 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config>./Q5/write_through/multiply/log.txt
```

然後將`./m5out`複製到`./Q5/write_through/multiply/`
```
cp ./m5out ./Q5/write_through/multiply/ -r
```

結果如下：
![](https://github.com/poyen0806/gem5/blob/master/result_image/Q5.png)
