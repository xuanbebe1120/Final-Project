# 2025計算機組織期末專題
### (Q1)  GEM5 + NVMAIN BUILD-UP (40%) 
根據簡報上操作
### (Q2) Enable L3 last level cache in GEM5 + NVMAIN (15%)
1. 在 gem5/configs/common/Caches.py 依照L2新增L3 cache
```
class L3Cache(Cache):
    assoc = 8
    tag_latency = 20
    data_latency = 20
    response_latency = 20
    mshrs = 20
    tgts_per_mshr = 12
    write_buffers = 8
```
目的是定義L3 cache
2.在gem5/configs/common/CacheConfig.py修改
```
       dcache_class, icache_class, l2_cache_class, walk_cache_class,l3_cache_class = \
            O3_ARM_v7a_DCache, O3_ARM_v7a_ICache, O3_ARM_v7aL2, \
            O3_ARM_v7aWalkCache,O3_ARM_v7aL3
    else:
        dcache_class, icache_class, l2_cache_class, walk_cache_class,l3_cache_class = \
            L1_DCache, L1_ICache, L2Cache, None,L3Cache
```
目的是指定用甚麼cache
```
    if options.l2cache and options.l3cache:
        # Provide a clock for the L2 and the L1-to-L2 bus here as they
        # are not connected using addTwoLevelCacheHierarchy. Use the
        # same clock as the CPUs.
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
```
`if options.l2cache and options.l3cache`:目的是為了設定只有L2、L3同時存在時，才啟用L3
其餘是設定資料的流向，L2->tol3bus->L3->membus

3.到gem5/src/mem/XBar.py 依照L2XBar新增L3XBar
```
class L3XBar(CoherentXBar):
    # 256-bit crossbar by default
    width = 32

    # Assume that most of this is covered by the cache latencies, with
    # no more than a single pipeline stage for any packet.
    frontend_latency = 1
    forward_latency = 0
    response_latency = 1
    snoop_response_latency = 1

    # Use a snoop-filter by default, and set the latency to zero as
    # the lookup is assumed to overlap with the frontend latency of
    # the crossbar
    snoop_filter = SnoopFilter(lookup_latency = 0)

    # This specialisation of the coherent crossbar is to be considered
    # the point of unification, it connects the dcache and the icache
    # to the first level of unified cache.
    point_of_unification = True

```
目的是定義L3 Cache 的Bus

4.到gem5/src/cpu/BaseCPU.py
在開頭加上，導入剛剛寫好的L3XBar
```
from XBar import L3XBar
```
在addTwoLevelCacheHierarchy下面加上
```
    def addThreeLevelCacheHierarchy(self, ic, dc, l3c, iwc=None, dwc=None):
        self.addPrivateSplitL1Caches(ic, dc, iwc, dwc)
        self.toL3Bus = L3XBar() 
        self.connectCachedPorts(self.toL3Bus)
        self.l3cache = l3c
        self.toL2Bus.master = self.l3cache.cpu_side
        self._cached_ports = ['l3cache.mem_side']
```
目的是在 CPU 模型裡實現三層cache階層

5.到gem5/configs/common/Options.py在# Cache Options新增，用以啟用l3cache這個參數
`parser.add_option("--l3cache", action="store_true")`
### (Q3) Config last level cache to 2-way and full-way associative cache and test performance (15%)
1.先將quicksort使用gcc --static 編譯成執行檔
`gcc --static quicksort.c -o quicksort`
2.將編譯出來的quicksort放到gem5的目錄下
3.使用以下指令執行
(1)2-way
```
./build/X86/gem5.opt configs/example/se.py \
-c .quicksort --cpu-type=TimingSimpleCPU \
--caches --l1i_size=32kB --l1d_size=32kB --l2cache --l2_size=128kB \
--l3cache --l3_size=1MB --l3_assoc=2 --mem-type=NVMainMemory \
--nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config 
```
(2)Full-way
```
Gem5 block size預設是 64 Bytes
block 數 = 1MB / 64B = 16384 blocks
assoc = 16384
```
指令
```
./build/X86/gem5.opt configs/example/se.py \
-c .quicksort --cpu-type=TimingSimpleCPU \
--caches --l1i_size=32kB --l1d_size=32kB --l2cache --l2_size=128kB \
--l3cache --l3_size=1MB --l3_assoc=16384 --mem-type=NVMainMemory \
--nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config 
```



