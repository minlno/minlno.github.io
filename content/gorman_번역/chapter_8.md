---
title: "Chapter 8  Slab Allocator"
draft: false
author: "Minho Kim"
categories: ["Gorman Book Translation"]
categories_weight: 8
---

이 글은 Gorman 책 "Understanding the Linux Virtual Memory Manager"의 [Chapte8 7](https://www.kernel.org/doc/gorman/html/understand/understand110.html)를 번역한 글입니다.

---

이번 장에서는 범용 목적의 할당자에 대해 설명한다. 그것은 바로 슬랩 할당자로, Solaris에서 사용되는 범용 커널 할당자와 굉장히 닮아있다. 리눅스의 구현은 Bonwick의 첫 슬랩 할당자 논문에 기반을 두어 구현되었고, 그의 다음 논문에 설명된 것과 유사한 많은 개선점이 포함된다. 우리는 먼저 할당자에 대한 빠른 오버뷰 설명을 한 후, 여러 구조체들에 대해 설명하고, 할당자의 각 작업들에 대해 깊이있는 설명을 할 것이다.

슬랩 할당자의 기반 아이디어는 자주 사용되는 객체들을 커널이 사용할 수 있게 초기화된 상태로 캐싱해두는 것이다. 객체 기반 할당자가 없다면, 커널은 같은 객체를 할당하고 초기화하고 해제하는데에 많은 시간을 소비하게 될 것이다. 슬랩 할당자는 해제된 객체를 캐싱하여 기본 구조체들을 사용 사이에 보존하는 것을 목표로 한다.

슬랩 할당자는 cache chain이라고 불리는 doubly linked circular list로 서로 연결된 변하는 수의 캐시들로 구성된다. 슬랩 할당자의 컨텍스트에서 캐시는 mm_struct 또는 fs_cache와 같은 특정 타입의 많은 수의 객체에 대한 관리자를 말하며, 이것은 추후에 설명될 struct kmem_cache_s에 의해 관리된다. 캐시들은 cache 구조체의 next 필드를 통해 연결된다.

각 캐시는 slabs라고 불리는 연속된 페이지 블록을 관리하며, 이는 캐시가 관리하는 자료구조 또는 객체를 위한 작은 청크들로 유지된다. 구조체간의 관계도가 Figure 8.1에 나와있다.

---
![](/images/gorman_번역/figure8.1.png "Figure 8.1: Layout of the Slab Allocator")

---

슬랩 할당자는 3개의 원칙을 목표로 한다:
- 작은 메모리 블록을 할당하여 버디 시스템으로 인해 발생할 내부 단편화를 제거하는 것
- 자주 사용되는 객체들을 캐싱하여 시스템이 객체를 할당, 초기화, 삭제하는데에 시간을 낭비하지 않게 하는 것
- 객체를 L1 또는 L2 캐시에 align하여 하드웨어 캐시를 더욱 잘 활용하는 것

이진 버디 할당자로 인해 발생하는 내부 단편화를 제거하기위해, 2^5 바이트부터 2^17 바이트 사이의 작은 메모리 버퍼들의 두 캐시 set을 관리한다. 한 캐시 set은 DMA 장치를 위한 것이다. 이 캐시들은 size-N과 size-N(DMA)라고 불리며, 여기서 N은 할당의 크기이다. 이들을 할당하기위한 함수로 kmalloc()가 제공된다. 이를 통해 low level page allocator의 단 한가지 엄청난 문제가 해결된다. 캐시들의 크기는 추후에 자세히 설명된다. 

슬랩 할당자의 두번째 작업은 자주 사용되는 객체들의 캐시들을 유지하는 것이다. 커널에서 사용되는 많은 구조체들은 할당하는 시간만큼 또는 그 이상의 시간이 객체를 초기화하는데 소요된다. 새로운 슬랩이 생성되면, 많은 수의 객체가 채워지며, constructor가 있다면 이를 초기화하는데 사용된다. 객체가 해제되면, 초기화가 되어있는 상태 그대로 남게되며, 다음 할당 때 재활용된다.

슬랩 할당자의 마지막 작업은 하드웨어 캐시의 효율 개선이다. 객체들을 채운 후에 슬랩에 공간이 남아있다면, 남은 공간은 슬랩을 "컬러링(coloring)"하는데 사용된다. 슬랩 컬러링은 슬랩의 객체들이 서로 다른 캐시라인을 사용하도록 하는 기술이다. 객체들을 슬랩내에서 서로 다른 오프셋에서 시작하도록 함으로써, 객체들이 서로 다른 캐시라인을 사용할 확률을 높이고, 이를 통해 같은 슬랩 캐시내의 객체들이 서로를 flush할 가능성을 낮춘다. 이를 통해, 낭비되는 공간도 최적화에 사용된다. Figure 8.2는 버디 할당자를 통해 할당된 페이지에 L1 CPU 캐시로 객체들을 align하여 저장하는 모습을 보여준다.

---
![](/images/gorman_번역/figure8.2.png "Figure 8.2: Slab page containing Objects Aligned to L1 CPU Cache")

---
리눅스는 페이지 할당을 그들의 물리 주소, order, 또는 code segment 기반으로 컬러링을 시도하지않지만, 현재 방식은 실제로 캐시 라인 사용을 개선한다. 캐시 컬러링에 대해서는 섹션 8.1.5에서 자세히 논의한다. SMP 시스템에서는, 각 CPU에게 객체들의 작은 배열들이 예약되어있어, 캐시 효율성을 높이기위한 추가 작업이 수행된다. 이는 섹션 8.5에서 자세히 설명한다.

슬랩 할당자는 CONFIG_SLAB_DEBUG와 함께 컴파일된 경우, 슬랩 디버깅을 위한 추가 옵션을 제공한다. "red zoning"과 "object poisoning"의 두가지 디버깅 옵션이 제공된다. red zoning은, 각 object의 끝에 마커를 저장한다. 만약 이 마커가 잘못될 경우, 할당자는 버퍼 오버플로우가 발생한 곳을 알 수 있고, 이를 보고한다. object poisoning은 객체를 슬랩 생성과 해제시, 미리 정의된 비트 패턴(mm/slab.c에서는 0x5A로 정의됨)으로 초기화한다. 할당 시, 이 패턴을 검사하며, 만약 값이 다르다면, 할당자는 객체가 할당되기전에 사용되었다는 것을 알 수 있고, 이를 표시한다.

할당자가 export한 작지만 강력한 API들이 Table 8.1에 나열되어있다.

---
{{< table title="Table 8.1: Slab Allocator API for caches" >}}
| |
| --- |
| kmem_cache_t * kmem_cache_create(const char *name, size_t size, size_t offset, unsigned long flags,  void (*ctor)(void*, kmem_cache_t *, unsigned long), void (*dtor)(void*, kmem_cache_t *, unsigned long)) |
| 새로운 캐시를 생성하고 캐시 체인에 추가한다. |
| int kmem_cache_reap(int gfp_mask) |
| 최대 REAP_SCANLEN 개의 캐시를 스캔하고 하나를 골라서 모든 per-cpu object를 회수하고 슬랩을 해제한다. 메모리가 부족한 경우 호출된다. |
| int kmem_cache_shrink(kmem_cache_t *cachep) |
| 이 함수는 캐시와 관련된 모든 per-cpu object들을 삭제하고 slabs_free 리스트에 있는 모든 슬랩들을 삭제한다. 이는 해제된 페이지의 개수를 반환한다. |
| void * kmem_cache_alloc(kmem_cache_t *cachep, int flags)
| 캐시에서 한 객체를 할당하고 반환한다. |
| void kmem_cache_free(kmem_cache_t *cachep, void *objp) |
| 객체를 해제하고 이를 캐시에 반환한다. |
| void * kmalloc(size_t size, int flags) |
| sizes 캐시로부터 메모리 블록을 할당한다. |
| void kfree(const void *objp) |
| kmalloc으로 할당된 메모리를 해제한다. |
| int kmem_cache_destroy(kmem_cache_t * cachep) |
| 모든 슬랩의 모든 객체를 삭제하고 관련된 모든 메모리를 해제한 후, 체인에서 캐시를 제거한다. |
{{< /table >}}

---

## 8.1  Caches

캐싱할 객체 타입마다 하나의 캐시(cache)가 존재한다. `cat /proc/slabinfo`를 실행하면, 시스템의 모든 캐시들의 리스트를 확인할 수 있다. 해당 파일은 캐시의 기본적인 정보를 제공한다. 출력 내용은 다음과 같다;

```bash
slabinfo - version: 1.1 (SMP)
kmem_cache            80     80    248    5    5    1 :  252  126
urb_priv                   0       0     64    0    0    1 :  252  126
tcp_bind_bucket       15    226     32    2    2    1 :  252  126
inode_cache         5714   5992    512  856  856    1 :  124   62
dentry_cache        5160   5160    128  172  172    1 :  252  126
mm_struct            240    240    160   10   10    1 :  252  126
vm_area_struct      3911   4480     96  112  112    1 :  252  126
size-64(DMA)           0      0     64    0    0    1 :  252  126
size-64              432   1357     64   23   23    1 :  252  126
size-32(DMA)          17    113     32    1    1    1 :  252  126
size-32              850   2712     32   24   24    1 :  252  126
```

각 열은 struct kmem_cache_s 구조체의 각 필드를 나타낸다. 위 출력에 나와있는 각 열들의 설명은 다음과 같다:
- cache-name: "tcp_bind_bucket"과 같이 사람이 읽을 수 있는 이름이다.
- num-active-objs: 사용중인 객체 (object)의 개수이다.
- total-objs: 사용중이지 않은 객체까지 포함한 전체 객체 수이다.
- obj-size: 각 객체의 크기. 보통 매우 작다.
- num-active-slabs: active한 객체를 포함하는 슬랩의 개수이다.
- total-slabs: 전체 슬랩 개수이다.
- num-pages-per-slab: 하나의 슬랩을 생성하는데 필요한 페이지 수이며, 일반적으로 1이다.

만약 SMP 시스템이라면, 두 개의 열이 콜론뒤에 추가된다. 이들은 섹션 8.5에서 설명할 per CPU cache를 나타내며, 설명은 다음과 같다:
- limit: 이 pool의 절반이 global free pool에 반환되기전까지 이것이 가질 수 있는 free object의 수이다.
- batchcount: 사용가능한 객체가 없을 때, 블록 내 프로세서에 할당된 객체 수이다.

객체와 슬랩의 할당과 해제를 빠르게 하기위해서, 그들은 세개의 리스트 slabs_full, slabs_partial, 그리고 slabs_free로 관리된다. slabs_full은 모든 객체가 사용중인 슬랩들로 구성된다. slabs_partial은 free object를 포함하며, 객체의 할당에 최우선적으로 사용된다. slabs_free는 할당된 객체가 없으며 slab을 삭제할 때 가장 우선적으로 선택된다.

### 8.1.1  Cache Descriptor

캐시를 나타내는 모든 정보는 mm/slab.c에 선언되는 struct kmem_cache_s에 저장된다. 이는 매우 큰 구조체이므로 부분적으로 설명하겠다.

```c
190 struct kmem_cache_s {
193     struct list_head        slabs_full;
194     struct list_head        slabs_partial;
195     struct list_head        slabs_free;
196     unsigned int            objsize;
197     unsigned int            flags;
198     unsigned int            num;
199     spinlock_t              spinlock;
200 #ifdef CONFIG_SMP
201     unsigned int            batchcount;
202 #endif
203 
```
대부분의 필드는 객체를 할당하거나 해제할 때 사용된다.
- slabs_*: 이전 섹션에서 설명했듯이, 슬랩이 저장되는 3개의 리스트이다.
- objsize: 슬랩을 구성하는 각 객체의 크기이다.
- flags: 캐시를 다룰 때 할당자의 각 파트가 어떻게 행동하는 지를 결정한다. 섹션 8.1.2를 참조하라.
- num: 각 슬랩에 포함되는 객체의 수이다.
- spinlock: 동시 접근으로부터 구조체를 보호하는 스핀락이다.
- batchcount: 이전 섹션에서 설명한 per-cpu cache를 위해 한번에 할당되는 객체의 개수이다.

```c
206     unsigned int            gfporder;
209     unsigned int            gfpflags;
210 
211     size_t                  colour;
212     unsigned int            colour_off;
213     unsigned int            colour_next;
214     kmem_cache_t            *slabp_cache;
215     unsigned int            growing;
216     unsigned int            dflags;
217 
219     void (*ctor)(void *, kmem_cache_t *, unsigned long);
222     void (*dtor)(void *, kmem_cache_t *, unsigned long);
223 
224     unsigned long           failures;
225 
```

이 필드들은 캐시에서 슬랩을 할당하거나 해제할 때 사용된다.
- gfporder: 슬랩에 사용되는 페이지 수를 나타낸다. 슬랩은 버디 할당자에서 2^gfporder 개의 페이지를 할당한다.
- gfpflags: 페이지를 할당하기위해 버디 할당자를 호출할 때 사용하는 GFP flag이다. 자세한건 섹션 6.4를 참조하라.
- colour: 각 슬랩은 가능하다면 객체를 서로다른 캐시라인에 저장한다. 캐시 컬러링은 섹션 8.1.5에서 자세히 설명한다.
- colour_off: 슬랩들이 정렬되는 단위의 크기이다. 예를 들면, size-X 캐시의 슬랩들은 L1 캐시에 대해 정렬된다.
- colour_next: 사용할 다음 컬러 라인이다. 이 값은 colour에 도달하면 다시 0으로 돌아간다.
- growing: 캐시가 증가하는 지를 표시하는 플래그이다. 만약 그렇다면, 메모리가 부족할 때 슬랩을 해제하기 위해서 이 캐시가 선택될 확률이 낮아진다.
- dflags: 캐시의 수명동안 값이 변하는 dynamic flags이다. 섹션 8.1.3을 참조하라.
- ctor: 복잡한 객체는 새로운 객체를 초기화할 때 호출될 constructor 함수를 제공할 수도 있다. 이 필드는 해당 함수에 대한 포인터이며, NULL이 저장될 수도 있다.
- dtor: 객체 destructor 함수 포인터로, NULL일 수 있다.
- failures: 이 필드는 0으로 초기화되며, 다른 코드 어떤 곳에서든 사용되지않는다.

```c
227     char                    name[CACHE_NAMELEN];
228     struct list_head        next;
```
캐시 생성 과정에서 설정된다.
- name: 캐시의 이름이다.
- next: 캐시 체인에서 next cache를 나타낸다.

```c
229 #ifdef CONFIG_SMP
231     cpucache_t              *cpudata[NR_CPUS];
232 #endif
```
- cpudata: 이는 per-cpu data로 섹션 8.5에서 추후에 설명한다.

```c
233 #if STATS
234     unsigned long           num_active;
235     unsigned long           num_allocations;
236     unsigned long           high_mark;
237     unsigned long           grown;
238     unsigned long           reaped;
239     unsigned long           errors;
240 #ifdef CONFIG_SMP
241     atomic_t                allochit;
242     atomic_t                allocmiss;
243     atomic_t                freehit;
244     atomic_t                freemiss;
245 #endif
246 #endif
247 };
```
이 필드들은 오로지 CONFIG_SLAB_DEBUG가 설정되었을 경우에만 이용가능하다. 이들은 모두 beancounter로 일반적으로 관심은 없다. /proc/slabinfo의 통계치들은 위의 필드들에 의존하는 대신, proc entry가 읽혔을 때 각 캐시에 의해 사용되는 모든 슬랩을 검사함으로써 계산된다.
- num_active: 캐시의 active한 객체의 수이다.
- num_allocations: 캐시에 할당된 총 객체 수이다.
- high_mark: 이는 현재까지 num_active가 보유한 제일 높은 값이다.
- grown: kmem_cache_grow()가 호출된 횟수이다.
- reaped: 캐시가 회수된 횟수이다.
- errors: 사용되지않는다.
- allochit: 할당이 per-cpu cache를 사용한 횟수이다.
- allocmiss: per-cpu cache의 미스 횟수이다.
- freehit: per-cpu cache에 free 객체가 반환된 횟수이다.
- freemiss: free 객체가 global pool에 반환된 횟수이다.

### 8.1.2  Cache Static Flags

많은 플래그들은 캐시를 생성할 때 초기화되며, 캐시가 살아있는 동안 값이 바뀌지않는다. 그들은 슬랩이 어떻게 구조화되고 객체들이 어떻게 그 안에 저장되는지에 영향을 준다. 모든 플래그들은 캐시 디스크립터의 flags 필드인 bit mask에 저장된다. 사용가능한 모든 플래그들은 <linux/slab.h>에 선언되어있다.

그들은 세 종류로 나뉜다. 첫 번째는 내부 플래그로, 슬랩 할당자에 의해서만 설정되며 Table 8.2에 나와있다. 해당 종류에서 관련있는 단 하나의 플래그는 CFGS_OFF_SLAB 플래그로, 슬랩 디스크립터가 어디에 저장되는지를 결정한다.

---
{{< table title="Table 8.2: Internal cache static flags" >}}
| Flag | Description |
| --- | --- |
| CFGS_OFF_SLAB | 이 캐시에 대한 슬랩 관리자가 off-slab으로 유지되도록 한다.  이것은 섹션 8.2.1에서 자세히 설명한다. |
| CFLGS_OPTIMIZE | 이 플래그는 설정만 되고 사용되지 않는다. |
{{< /table >}}

---

두 번째는 캐시 생성자에 의해 설정되며, 할당자가 슬랩을 어떻게 대하는지와 객체들이 어떤 방식으로 저장되는지를 결정한다. 이들는 Table 8.3에 나열되어있다.

---
{{< table title="Table 8.3: Cache static flags set by caller" >}}
| Flag | Description |
| --- | --- |
| SLAB_HWCACHE_ALIGN | 객체를 L1 CPU cache에 정렬한다. |
| SLAB_MUST_HWCACHE_ALIGN | 메모리를 낭비하거나 슬랩 디버깅이 활성화되어 있더라도 L1 CPU cache에 대한 정렬을 강제한다. |
| SLAB_NO_REAP | 이 캐시의 슬랩을 절대 회수하지 않도록한다. |
| SLAB_CACHE_DMA | 슬랩을 ZONE_DMA에서 할당하도록 한다. |
{{< /table >}}

---

마지막은 CONFIG_SLAB_DEBUG가 컴파일 타임에 설정된 경우에만 사용된다. 이들은 어떤 추가적인 검사가 슬랩에 수행될 지를 결정하며, 새로운 캐시를 개발하는 동안에 주로 사용된다.

---
{{< table title="Table 8.4: Cache static debug flags" >}}
| Flag | Description |
| --- | --- |
| SLAB_DEBUG_FREE | 해제 시에 오래걸리는 검사를 수행한다. |
| SLAB_DEBUG_INITIAL | 해제 시에 생성자를 호출하여 객체가 정확히 초기화되어있는 상태에 있도록 보장한다. |
| SLAB_RED_ZONE | overflow를 감지할 수 있도록 객체의 끝에 마커를 표시한다. |
| SLAB_POISON | 객체를 정해진 패턴으로 표시하여 할당되지 않았거나 초기화되지않은 객체가 수정되었는 지를 감지한다. |
{{< /table >}}

---

호출자가 잘못된 플래그를 설정하지 못하도록 하기위해, mm/slab.c에는 CREATE_MASK가 정의되어있다. 캐시가 생성되면, 요청된 플래그가 CREATE_MASK와 비교되며, 만약 유효하지않은 플래그가 사용되었다면 버그를 보고한다.

### 8.1.3  Cache Dynamic Flags

dflags는 DFLGS_GROWN이라는 단 하나의 중요한 플래그를 가지고 있다. 플래그는 kmem_cache_grow() 함수가 설정하여, kmem_cache_reap() 함수가 회수하는동안 해당 캐시를 선택하지 않도록한다. 이 함수가 이 플래그가 설정된 캐시를 찾은 경우, 플래그를 제거한 뒤 캐시를 스킵한다.

### 8.1.4  Cache Allocation Flags

이 플래그들은 슬랩을 위한 페이지를 할당할 때 어떤 GFP flag가 사용될지를 결정한다. 호출자가 SLAB_*과 GFP_* 플래그를 함께 사용하는 경우도 있지만, 실제로는 오로지 SLAB_* 플래그만을 사용해야한다. 이들은 섹션 6.4에서 설명된 플래그들과 동일하므로 여기서는 그 의미를 자세히 설명하지 않는다. 이 플래그들은 명확성을 위해 존재하는 것으로 추정되며, 슬랩 할당자가 특정 플래그에 따라 다르게 행동해야할 필요가 있지만, 실제로는 그렇지않다.

---
{{< table title="Table 8.5: Cache Allocation Flags" >}}
| Flag | Description |
| --- | --- |
| SLAB_ATOMIC | GFP_ATOMIC과 같다. |
| SLAB_DMA | GFP_DMA과 같다. |
| SLAB_KERNEL | GFP_KERNEL과 같다. |
| SLAB_NFS | GFP_NFS과 같다. |
| SLAB_NOFS | GFP_NOFS과 같다. |
| SLAB_NOHIGHIO | GFP_NOHIGHIO과 같다. |
| SLAB_NOIO | GFP_NOIO과 같다. |
| SLAB_USER | GFP_USER과 같다. |
{{< /table >}}

---
Table 8.6에 나열된 매우 적은 수의 플래그들이 생성자와 소멸자에게 전달될 수도 있다.

---
{{< table title="Table 8.6: Cache Constructor Flags" >}}
| Flag | Description |
| --- | --- |
| SLAB_CTOR_CONSTRUCTOR | 같은 함수를 생성자와 소멸자로 사용하는 경우, 해당 함수가 생성자로 호출될 때 이 플래그가 설정된다. |
| SLAB_CTOR_ATOMIC | 생성자가 잠들지 않도록 한다. |
| SLAB_CTOR_VERIFY | 생성자가 단지 객체가 정확히 초기화되었는지만 검증하도록 한다. |
{{< /table >}}

---

### 8.1.5  Cache Coloring

하드웨어 캐시를 더 잘 활용할 수 있도록, 슬랩 할당자는 슬랩 내 남은 공간의 양에 따라 다른 슬랩의 객체들을 서로 다른 양만큼 오프셋 시킬 것이다. 오프셋은 SLAB_HWCACHE_ALIGN이 설정되어있다면 L1_CACHE_BYTES의 단위일 것이고, 그렇지 않다면 BYTES_PER_WORD 단위일 것이다.

캐시가 생성되는 동안, 몇 개의 객체가 슬랩에 저장될 수 있는 지와 (섹션 8.2.7 참조) 몇 바이트가 낭비되는지를 계산한다. 낭비되는 양을 기반으로, 두 가지 값이 계산된다:
- colour: 사용가능한 서로다른 오프셋의 개수이다.
- colour_off: 슬랩에서 각 객체의 오프셋에 곱하는 수이다.
객체의 오프셋을 통해 그들은 하드웨어 캐시에서 서로 다른 라인을 사용하게 될 것이다. 그러므로, 슬랩의 객체들은 서로를 메모리에 덮어쓰지않을 가능성이 높다.

이것의 결과는 예제를 통해 가장 잘 설명할 수 있다. 슬랩의 첫 객체의 주소인 s_mem이 0이고, 100 바이트가 낭비되며, 플래티넘 II의 L1 하드웨어 캐시에 32 바이트 단위로 정렬된다고 가정하자.

이 시나리오에서, 첫 슬랩은 자신의 객체가 0에서 시작할 것이다. 두번째는 32에서 시작할 것이며, 세번째는 64, 네번째는 96 그리고 다섯번째는 다시 0에서 시작할 것이다. 이를 통해, 각 슬랩의 객체들은 CPU의 같은 캐시라인에 hit하지 않을 것이다. 이때, colour의 값은 3이며, colour_off은 32이다.

### 8.1.6  Cache Creation

kmem_cache_create() 함수가 새로운 캐시를 생성하고 그들을 캐시 체인에 추가하는 작업을 수행한다. 캐시를 생성하기 위해 다음과 같은 작업들이 수행된다:
- 잘못된 사용을 감지하기 위한 기본적인 위생 검사를 수행한다.
- 만약 CONFIG_SLAB_DEBUG가 설정되어있다면, 디버깅 검사를 수행한다.
- cache_cache 슬랩 캐시로부터 kmem_cache_t를 할당한다.
- word 크기로 객체 크기를 정렬한다.
- 슬랩에 몇 개의 객체가 저장될 수 있는지를 계산한다.
- 객체 크기를 하드웨어 캐시에 정렬한다.
- colour offset을 계산한다.
- 캐시 서술자의 나머지 필드들을 초기화한다.
- 새 캐시를 캐시 체인에 추가한다.

Figure 8.3이 캐시 생성과 관련된 call graph를 보여준다. 각 함수는 Code Commentary에서 자세히 설명한다.

---
![](/images/gorman_번역/figure8.3.png "Figure 8.3: Call Graph: kmem_cache_create()")

---

### 8.1.7  Cache Reaping

슬랩이 해제되면 나중에 사용하기 위해 slabs_free 리스트에 추가된다. 캐시들은 스스로 메모리 사용량을 줄이지 않기때문에, 메모리가 부족해지면 kswapd가 kmem_cache_reap()를 호출해서 일부 메모리를 해제한다. 이 함수는 자신의 메모리 사용량을 줄여야하는 캐시를 선택한다. 캐시 회수는 어떤 메모리 노드 또는 존이 메모리가 부족한지를 고려하지않는다. 이는 NUMA또는 high memory 머신에서, 커널이 메모리 압력이 존재하지 않는 노드에서 메모리를 해제하느라 많은 시간을 소비할 수도 있다는 것을 의미하지만, 오직 하나의 메모리 뱅크만 있는 x86과 같은 아키텍처에서는 문제가 되지않는다.

---
![](/images/gorman_번역/figure8.4.png "Figure 8.4: Call Graph: kmem_cache_reap()")

___

Figure 8.4에 있는 Call Graph는 많이 단순화된 것이고, 실제로는 회수하기에 적절한 캐시를 선택하기 위한 긴 작업이 존재한다. 시스템에 캐시가 많은 경우, 각 호출마다 REAP_SCANLEN (현재는 10으로 정의됨) 만큼의 캐시들이 검사된다. 같은 캐시를 반복해서 검사하지않기위해 마지막으로 스캔한 캐시를 clock_searchp에 저장한다. 캐시를 스캔할 때는 다음과 같은 작업을 수행한다:
- SLAB_NO_REAP를 체크하고, 만약 설정되어있다면 이 캐시는 스킵한다.
- 캐시가 커지는 중이라면 스킵한다.
- 만약 캐시가 최근에 커졌거나, 지금 커지는 중이라면, DFLGS_GROWN이 설정될 것이다. 이 플래그가 설정되어있다면, 스킵하면서 플래그를 클리어하여, 다음 스캔에는 회수 후보가 될 수 있도록한다.
-slabs_free에 있는 free 슬랩 개수를 카운팅하고, 해제될 페이지의 개수를 계산하여 pages 변수에 저장한다.
- 만약 해제될 페이지의 개수가 REAP_PERFECT를 초과한다면, slabs_free에 있는 슬랩의 절반을 해제한다.
- 그렇지않다면 캐시의 나머지를 스캔하여, 슬랩의 절반을 해제하여 가장 많은 페이지를 해제하게될 캐시를 선택한다.

### 8.1.8  Cache Shrinking

캐시가 스스로 줄어들도록 선택되었을 때 수행되는 작업은 간단하고 잔인하다.
- per CPU cache에 있는 모든 객체들을 삭제한다.
- growing flags가 설정되어있지않는한 slabs_free의 모든 슬랩들을 삭제한다.

미묘하지않은 리눅스는 리눅스가 아니다.

---
![](/images/gorman_번역/figure8.5.png "Figure 8.5: Call Graph: kmem_cache_shrink()")

---

이름이 비슷해서 헷갈리는 두 개의 shrink 함수가 제공된다. kmem_cache_shrink()는 slabs_free에서 모든 슬랩을 제거하고 해제된 페이지 수를 반환한다. 이 함수는 슬랩 할당자의 사용을 위해 export되는 주요 함수이다.

---
![](/images/gorman_번역/figure8.6.png "Figure 8.6: Call Graph: __kmem_cache_shrink()")

---

두 번째 함수인 __kmem_cache_shrink()는 slabs_free에서 모든 슬랩을 해제하고 slabs_partial과 slabs_full이 비어있음을 확인한다. 이는 내부적으로만 사용되며, 몇개의 페이지가 해제되었는 지는 중요하지않고, 캐시를 비우는 것만이 중요한 상황에 사용된다.

### 8.1.9  Cache Destroying

모듈이 unload될 때, kmem_cache_destroy()를 통해 모든 캐시를 파괴해야한다. 같은 human-readable 이름을 가진 캐시는 두개 이상 존재할 수 없으므로, 캐시를 적절히 삭제하는 것은 중요하다. 핵심 커널 코드는 시스템 수명 동안 캐시가 지속되므로, 캐시의 삭제를 신경쓰지않는 경우가 많다. 캐시의 삭제는 다음과 같이 수행된다:
- 캐시 체인에서 캐시를 삭제한다.
- 모든 슬랩을 삭제하기 위해 cache를 shrink한다.
- 모든 per CPU cache들을 해제한다 (kfree()).
- cache_cache에서 캐시 서술자를 삭제한다.

---
![](/images/gorman_번역/figure8.7.png "Figure 8.7: Call Graph: kmem_cache_destroy()")

---

## 8.2  Slabs

이번 섹션에서는 슬랩이 어떤 구조이고 어떻게 관리되는지를 설명한다. 슬랩 구조체는 캐시 구조체보다 훨씬 간단하지만, 캐시가 어떻게 구성되는 지는 상당히 더 복잡하다. 이것은 아래와 같이 선언된다:
```c
typedef struct slab_s {
    struct list_head        list;
    unsigned long           colouroff;
    void                    *s_mem;
    unsigned int            inuse;
    kmem_bufctl_t           free;
} slab_t;
```
이 간단한 구조체의 필드들의 설명은 다음과 같다:
- list: 슬랩이 속한 연결 리스트이다. 리스트는 캐시 매니저의 slab_full, slab_partial 또는 slab_free 중 하나일 것이다.
- colouroff: 슬랩의 첫 객체의 base 주소의 colour offset이다. 첫 객체의 주소는 s_mem + colouroff이다.
- s_mem: 슬랩의 첫 객체의 시작 주소이다.
- inuse: 슬랩에서 활성화된 객체의 개수이다.
- free: free 객체의 위치를 저장하는 bufctl 들의 배열이다. 자세한 내용은 섹션 8.2.3을 참조하라.

독자들은 슬랩 관리자 또는 슬랩의 객체를 가지고, 그들이 어떤 슬랩 또는 캐시에 속해있는 지를 알 수 없음에 주목할 것이다. 이것은 캐시를 구성하는 struct page의 list 필드를 사용하여 처리된다. SET_PAGE_CACHE()와 SET_PAGE_SLAB()은 객체가 속한 캐시와 슬랩을 추적하기위해 page->list의 next와 prev 필드를 사용한다. page 서술자는 GET_PAGE_CACHE()와 GET_PAGE_SLAB() 매크로를 사용하여 얻을 수 있다. 이들의 관계는 Figure 8.8에 나타나있다.

---
![](/images/gorman_번역/figure8.8.png "Figure 8.8: Page to Cache and Slab Relationship")

---

마지막 문제는 어디에 슬랩 관리자 구조체를 저장하느냐이다. 슬랩 관리자들은 on 또는 off-slab에 유지된다. 이들이 어디에 위치하는지는 캐시 생성 과정에서 객체의 크기에 의해 결정된다. Figure 8.8에서 유의해야하는 것은, 그림에서는 struct slab_t가 struct page와 별도의 위치에 있는 것처럼 나와있지만, 실제로는 struct page의 내부에 위치할 수도 있다는 것이다.

### 8.2.1  Storing the Slab Descriptor

만약 객체가 어떤 임계값(x86에서는 512바이트)보다 크다면, CFGS_OFF_SLAB이 캐시 플래그에 설정되고, 슬랩 디스크립터는 사이즈 캐시 중 하나인 off-slab으로 관리된다 (섹션 8.4 참조). 선택된 사이즈 캐시는 struct slab_t를 포함할 수 있을만큼 충분히 크며, kmem_cache_slabmgmt()는 필요할 때 여기서 할당한다. 이것은 bufctl을 위한 공간에 제약이 있어, 슬랩에 저장될 수 있는 객체의 수를 제한하지만, 객체가 커서 하나의 슬랩에 많은 수가 저장될 필요가 없기 때문에 이는 별로 중요하지않다.

---
![](/images/gorman_번역/figure8.9.png "Figure 8.9: Slab With Descriptor On-Slab")

---

슬랩의 시작 부분에 슬랩 관리자가 예약된다. on-slab에 저장되는 경우, 슬랩 시작 부분에 충분한 공간이 예약되어 slab_t와 unsigned int의 배열인 kmem_bufctl_t를 저장하는데 사용된다. 이 배열은 사용가능한 다음 free 객체의 인덱스를 추적하는데에 사용되며, 섹션 8.2.3에서 더 논의된다.

Figure 8.9는 on-slab 디스크립터를 가지는 슬랩이 어떻게 생겼는지를 보여주며, Figure 8.10는 디스크립터가 off-slab으로 관리되는 경우 캐시가 사이즈 캐시를 슬랩 디스크립터를 저장하기위해 어떻게 사용하는지를 보여준다.

---
![](/images/gorman_번역/figure8.10.png "Figure 8.10: Slab With Descriptor Off-Slab")

---

### 8.2.2  Slab Creation

---
![](/images/gorman_번역/figure8.11.png "Figure 8.11: Call Graph: kmem_cache_grow()")

---

지금까지 우린 캐시가 어떻게 생성되는 지에 대해 이야기했다. 갓 생성된 캐시는 비어있으며, 이것의 slab_full, slab_partial, 그리고 slabs_free 또한 비어있다. 캐시의 새로운 슬랩은 kmem_cache_grow()를 호출하여 할당한다. 이는 보통 "cache growing"이라고 불리며, slab_partial 리스트에 비어있는 객체가 없고, slabs_free에 슬랩이 존재하지않는 경우 수행된다. 이때 수행되는 작업들은 다음과 같다:
- 잘못된 사용을 방지하기위한 기본적인 위생 검사를 수행
- 이 슬랩의 객체를 위한 colour offset 계산
- 슬랩을 위한 메모리 할당 및 슬랩 디스크립터 획득
- 섹션 8.2에서 설명된 것과 같이, 슬랩을 위해 사용되는 페이지들을 슬랩과 캐시 디스크립터에 연결
- 슬랩의 객체들을 초기화
- 슬랩을 캐시에 추가가

### 8.2.3  Tracking Free Objects

슬랩 할당자는 부분적으로 할당된 슬랩에서 free 객체를 간단하고 빠른 방법으로 추적한다. 이는 kmem_bufctl_t라고 불리는 unsigned int의 배열을 사용하여 구현되며, kmem_bufctl_t는 각 슬랩 매니저에 연관되며, 이를 어떻게 사용하는 지는 슬랩 매니저에게 달려있다.

역사적으로 그리고 슬랩 할당자를 서술하는 논문에 따르면, kmem_bufctl_t는 객체들의 연결리스트였다. Linux 2.2.x버전에서 이 구조체는 next free object를 가리키는 포인터, 슬랩 매니저를 가리키는 포인터, 그리고 객체에 대한 포인터, 이렇게 3개의 필드로 구성된 union이었다. 어떤 필드가 사용되는지는 객체의 상태에 따라 달라진다.

오늘날, 객체가 속한 슬랩과 캐시는 struct page에 의해 결정되며, kmem_bufctl_t는 단순히 객체 인덱스의 정수 배열일뿐이다. 배열의 요소의 개수는 슬랩의 객체 수와 동일하다.

```c
141 typedef unsigned int kmem_bufctl_t;
```

이 배열은 슬랩 디스크립터의 바로 뒤에 저장되기때문에, 첫 배열 원소를 가리키는 별도의 포인터를 저장하지않으며, 대신에 헬퍼 매크로인 slab_bufctl()이 제공된다.

```c
163 #define slab_bufctl(slabp) \
164         ((kmem_bufctl_t *)(((slab_t*)slabp)+1))
```

다소 난해한 이 매크로는 잘 살펴보면 꽤 단순하다. 인자 slabp는 슬랩 매니저에 대한 포인터이다. ((slab_t*)slabp)+1은 slabp를 slab_t 구조체로 캐스팅하고 1을 더한다. 이를 통해 우리는 kmem_bufctl_t 배열의 첫 slab_t 자료형 원소에 대한 포인터를 얻을 수 있다. (kmem_bufctl_t *)는 slab_t 포인터를 요구되는 타입으로 캐스팅한다. 코드는 이를 slab_bufctl(slabp)[i] 와 같은 방식으로 사용할 것이다. 이는 슬랩 디스크립터를 통해 kmem_bufctl_t 배열을 얻고, 해당 배열의 i번째 원소를 반환한다.

슬랩의 다음 free 객체의 인덱스는 slab_t->free에 저장되며, 이는 free 객체들을 연결리스트로 관리할 필요를 없애준다. 객체가 할당되거나 해제될 때, 이 포인터는 kmem_bufctl_t 배열의 정보를 기반으로 업데이트된다.

### 8.2.4  Initialising the kmem_bufctl_t Array

캐시가 자라나면, 슬랩의 모든 객체들과 kmem_bufctl_t 배열이 초기화된다. 배열은 1로 시작하는 각 객체의 인덱스로 채워지며 BUFCTL_END 값으로 끝나게된다.

---
![](/images/gorman_번역/figure8.12.png "Figure 8.12: Initialised kmem_bufctl_t Array")

---

0번째 객체가가 가장 첫번째로 사용할 free object이기 때문에 slab_t->free에 0이 저장된다. kmem_bufctl_t[n]에는 n번째 객체의 다음 free object의 인덱스가 저장된다. 위 그림의 배열을 보면, 0번째 객체 다음은 1인 것을 알 수 있다. 1다음은 2, 그 다음은 3 이런식이다. 배열이 사용되기 때문에, 이러한 배치는 배열이 free 객체의 LIFO로 동작하게 만든다.

### 8.2.5  Finding the Next Free Object

객체를 할당할 때, kmem_cache_alloc()이 kmem_cache_alloc_one_tail()을 호출함으로써 kmem_bufctl_T() 배열을 업데이트한다. slab_t->free 필드가 첫 free 객체의 인덱스를 저장한다. 다음 free 객체의 인덱스는 kmem_bufctl_t[slab_t->free]에 저장되어있다. 코드는 다음과 같을 것이다.

```c
1253     objp = slabp->s_mem + slabp->free*cachep->objsize;
1254     slabp->free=slab_bufctl(slabp)[slabp->free];
```

slabp->s_mem은 슬랩의 첫 객체에 대한 포인터이다. slabp->free는 할당할 객체의 인덱스이며 이는 객체의 크기로 곱해진다.

다음 free 객체의 인덱스는 kmem_bufctl_t[slabp->free]에 저장된다. 해당 배열을 직접 가리키는 포인터 변수는 없기때문에 slab_bufctl() 매크로가 사용된다. kmem_bufctl_t 배열은 할당으로 인해 변하지않지만, 해제된 배열의 원소들은 도달될 수 없음에 주목하라. 예를 들면, 2번의 할당후에, kmem_bufctl_t 배열의 0, 1 인덱스는 어떠한 다른 요소들도 가리키지 않을 것이다.

### 8.2.6  Updating kmem_bufctl_t

kmem_cache_free_one() 함수에서 객체가 해제되면 kmem_bufctl_t 배열이 업데이트된다. 배열의 업데이트는 다음과 같이 수행된다.

```c
1451     unsigned int objnr = (objp - slabp->s_mem)/cachep->objsize;
1452 
1453     slab_bufctl(slabp)[objnr] = slabp->free;
1454     slabp->free = objnr;
```

objp는 해제하고자하는 객체의 주소이며, objnr은 객체의 인덱스이다. kmem_bufctl_t[objnr]은 slabp->free의 현재값으로 업데이트되며, 이를 통해 해제하고자하는 객체가 free를 가리킬 수 있도록한다. slabp->free는 objnr로 수정되어 다음 순서서에 이 객체가 할당되도록 한다.

### 8.2.7  Calculating the Number of Objects on a Slab

캐시를 생성하는 과정에서 kmem_cache_estimate() 함수가 호출되어 하나의 슬랩에 몇 개의 객체가 저장될지를 계산하며, 이는 슬랩 디스크립터를 on-slab, off-slab 중에서 어디에 저장할 지와 객체가 free한지를 추적하는데 필요한 kmem_bufctl_t의 크기를 결정하는 데 반영된다. 남는 바이트의 크기는 캐시 컬러링을 적용 여부를 결정한다.

계산은 꽤 간단하며 아래와 같은 단계를 따른다.
- wastage를 슬랩의 전체 크기로 초기화한다. (i.e. PAGE_SIZE^(gfp_order))
- 슬랩 디스크립터를 저장하기위해 필요한 공간을 뺀다.
- 저장될 객체의 개수를 센다. 만약 슬랩 디스크립터를 슬랩 안에 저장한다면 kmem_bufctl_t의 크기도 포함한다. 슬랩이 다 찰 때까지 i를 증가시킨다.
- 객체의 수와 낭비되는 공간의 크기를 반환한다.

### 8.2.8  Slab Destroying

캐시가 작아지거나 삭제될 때, 슬랩들도 삭제될 것이다. 객체들은 소멸자를 가지고 있으며, 이들은 삭제 과정에서 호출되어야한다. 이 함수의 작업은 다음과 같다:
- 소멸자가 있다면, 이를 슬랩의 모든 객체에 대해 호출한다.
- 디버깅이 켜져있다면, 레드 마크와 poison 패턴을 체크한다.
- 슬랩이 사용하고있는 페이지들을 해제한다.

Figure 8.13을 보면 매우 간단한 Call Graph를 확인할 수 있다.

---
![](/images/gorman_번역/figure8.13.png "Figure 8.13: Call Graph: kmem_slab_destroy()")

---


## 8.3  Objects

이 섹션은 객체들이 어떻게 관리되는지에 대해 설명한다. 이 시점에서는, 대부분의 매우 어려운 작업들이 캐시 또는 슬랩 관리자에 의해 완수되었다.

### 8.3.1  Initialising Objects in a Slab

슬랩이 생성되면, 슬랩 내의 모든 객체들은 초기화된 상태가 된다. 만약 생성자가 있다면 각 객체에 대해 호출되며, 객체는 해제 시 초기화된 상태로 유지된다. 개념적으로 초기화는 매우 간단한데, 모든 객체를 돌면서 생성자를 호출하고 이를 위한 kmem_bufctl을 초기화한다. 객체를 초기화하는 작업은 kmem_cache_init_objs()가 담당한다.

### 8.3.2  Object Allocation

kmem_cache_alloc()을 통해 하나의 객체를 할당할 수 있으며, 이 함수의 동작은 단일 프로세스와 멀티 프로세스 머신에서 약간의 차이가 존재한다. Figure 8.14는 SMP 머신에서 객체를 할당할 때의 기본적인 Call Graph를 보여준다.

---
![](/images/gorman_번역/figure8.14.png "Figure 8.14: Call Graph: kmem_cache_alloc()")

---

4가지의 기본 단계가 존재한다. 첫 번째는 kmem_cache_alloc_head()로, 할당이 허용되는 지를 검사한다. 두 번째 단계에서는 어떤 slabs_partial과 slabs_free 중 어떤 리스트에서 할당할 지를 결정한다. 만약 slabs_free에 슬랩이 없다면, 캐시는 slabs_free에 새로운 슬랩을 생성하기위해 성장할 것이다. 마지막 단계는 선택된 슬랩에서 객체를 할당한다.

SMP에서는 한가지 추가 단계가 존재한다. 객체를 할당하기전에, per-CPU cache에 할당할 수 있는 객체가 있는 지 확인하고, 있으면 해당 객체를 할당한다. 만약 없다면, batchcount 만큼의 객체를 묶음으로 할당하고 그들을 per-cpu cache에 배치한다. per-cpu cache에 대한 내용은 섹션 8.5에서 더 다룬다.

### 8.3.3  Object Freeing

객체의 해제는 kmem_cache_free() 함수를 사용하며, 이는 상대적으로 단순한 작업이다. kmem_cache_alloc()과 마찬가지로, 이는 UP머신과 SMP머신에서 서로 다르게 동작한다. 주요 차이점은 UP에서는 slab에 직접 반환되지만, SMP에서는 per-cpu cache에 반환된다는 것이다. 두 경우 모두, 소멸자가 있다면 호출될 것이다. 소멸자는 객체를 초기화된 상태로 만들어줄 것이다.

---
![](/images/gorman_번역/figure8.15.png "Figure 8.15: Call Graph: kmem_cache_free()")

---

## 8.4  Sizes Cache

리눅스는 물리 페이지 할당자로 할당하기에는 적합하지않은 작은 사이즈의 할당을 위해 두 종류의 캐시를 관리한다. 하나는 DMA를 위한 것이고 나머지는 일반 사용을 위한 것이다. 이들의 이름은 size-N cache과 size-N(DMA) cache로 /proc/slabinfo에서 확인할 수 있다. 각 사이즈 캐시의 정보는 struct cache_sizes에 저장되며 cache_sizes_t로 typedef되며, mm/slab.s에 다음과 같이 정의된다:
```c
331 typedef struct cache_sizes {
332     size_t           cs_size;
333     kmem_cache_t    *cs_cachep;
334     kmem_cache_t    *cs_dmacachep;
335 } cache_sizes_t;
``` 
각 필드에 대한 설명은 아래와 같다:
- cs_size: 메모리 블록의 크기
- cs_cachep: 일반 메모리 사용을 위한 블록들의 캐시
- cs_dmacachep: DMA를 위한 블록들의 캐시

이 캐시들은 제한된 개수만 존재할 수 있기때문에, 이들은 cache_sizes라는 이름의 정적 배열로 초기화되며, 4KiB 머신에서는 32바이트부터, 더 큰 페이지 사이즈의 머신에서는 64바이트부터 시작한다.
```c
337 static cache_sizes_t cache_sizes[] = {
338 #if PAGE_SIZE == 4096
339     {    32,        NULL, NULL},
340 #endif
341     {    64,        NULL, NULL},
342     {   128,        NULL, NULL},
343     {   256,        NULL, NULL},
344     {   512,        NULL, NULL},
345     {  1024,        NULL, NULL},
346     {  2048,        NULL, NULL},
347     {  4096,        NULL, NULL},
348     {  8192,        NULL, NULL},
349     { 16384,        NULL, NULL},
350     { 32768,        NULL, NULL},
351     { 65536,        NULL, NULL},
352     {131072,        NULL, NULL},
353     {     0,        NULL, NULL}
```
보면 알 수 있듯이, 이 배열은 2^5부터 2^17의 크기의 캐시로 구성된다. 이 배열은 각 사이즈 캐시를 설명하며, 각 캐시는 시스템 시작 시에 초기화되어야 한다.

### 8.4.1  kmalloc()

사이즈 캐시 덕분에 슬랩할당자는 작은 메모리 버퍼가 필요할 때 사용하기위한 새로운 할당 함수인 kmalloc()을 제공할 수 있다. 요청이 들어오면, 적절한 사이즈 캐시를 선택하여 해당 객체를 할당한다. Call Graph는 Figure 8.16에서 확인할 수 있으며, 당연하게도 어려운 작업은 캐시 할당 내부에서 수행하기 때문에 kmalloc() 자체는 단순한 것을 볼 수 있다.

---
![](/images/gorman_번역/figure8.16.png "Figure 8.16: Call Graph: kmalloc()")

---

### 8.4.2  kfree()

작은 메모리 객체를 할당하기위한 함수로 kmalloc()이 존재하는 것처럼, 그것을 해제하기위한 함수로 kfree()가 존재한다. kmalloc()과 마찬가지로 실제 작업은 객체의 해제 과정에서 수행되므로 Figure 8.17의 Call Graph는 매우 단순하다.

---
![](/images/gorman_번역/figure8.17.png "Figure 8.17: Call Graph: kfree()")

---

## 8.5  Per-CPU Object Cache

슬랩 할당자의 목표 중 하나는 하드웨어 캐시의 활용도를 개선하는 것이다. 고성능 컴퓨팅의 일반적인 목표는 가능한 한 오랫동한 동일한 CPU에서 데이터를 사용하는 것이다. 리눅스는 객체들을 최대한 cpucache라고 불리는 Per-CPU object cache의 같은 CPU 캐시에서 할당하려고 시도함으로써 이를 달성한다.

객체를 할당 또는 해제할 때, 이들은 cpucache에 배치된다. 만약 free 객체가 없다면 batch 만큼의 객체가 pool에 배치된다. pool이 너무 커졌다면, 그들의 절반은 제거되어 global cache에 배치된다. 이렇게 함으로써 하드웨어 캐시가 동일한 CPU에서 가능한 한 오랫동안 사용될 수 있다.

이 방식의 두 번째 주된 이득은 CPU pool에 접근할 때 다른 CPU에서는 이를 접근하지않으므로 스핀락을 잡을 필요가 없다는 것이다. cpucache가 없다면 매 할당과 해제마다 락을 잡아야하므로 이는 중요하다.

### 8.5.1  Describing the Per-CPU Object Cache

각 캐시 디스크립터는 아래와 같이 cpucache들의 배열에 대한 포인터를 가지고있다.
```c 
231    cpucache_t              *cpudata[NR_CPUS];
```
이 구조체 매우 단순하다
```c
173 typedef struct cpucache_s {
174     unsigned int avail;
175     unsigned int limit;
176 } cpucache_t;
```
필드의 의미는 다음과 같다:
- avail: cpucache의 이용가능한 free 객체의 수
- limit: free 객체의 최대 개수

캐시와 프로세서가 주어졌을 때, 간편하게 cpucache에 접근할 수 있는 헬퍼 매크로 cc_data()가 제공되며, 다음과 같이 정의된다.
```c
180 #define cc_data(cachep) \
181         ((cachep)->cpudata[smp_processor_id()])
```
이는 주어진 cachep(캐시 디스크립터)를 가지고 cpucache의 포인터를 반환한다. 이때 배열의 인덱스를 얻기위해 현재 프로세서의 ID를 반환하는 함수인 smp_processor_id()를 사용한다.

cpucache의 객체들에 대한 포인터는 cpucache_t 구조체 바로 다음에 위치한다. 이는 객체들이 슬랩 디스크립터 뒤에 저장되는 방식과 매우 유사하다.

### 8.5.2  Adding/Removing Objects from the Per-CPU Cache

단편화를 방지하기위해, 객체들은 항상 배열의 마지막에서 추가되거나 제거된다. 객채(obj)를 CPU cache (cc)에 추가하기위해, 다음과 같은 코드가 사용된다.
```c
       cc_entry(cc)[cc->avail++] = obj;
```
제거를 위한 코드는 다음과 같다.
```c
       obj = cc_entry(cc)[--cc->avail];
```
cpucache의 첫 객체의 포인터를 반환하는 헬퍼 매크로인 cc_entry()가 제공된다. 이는 다음과 같이 정의된다.
```c
178 #define cc_entry(cpucache) \
179         ((void **)(((cpucache_t*)(cpucache))+1))
```
이는 cpucache의 포인터를 인자로 받아 cpucache_t의 크기만큼 더하여 캐시의 첫 객체의 포인터를 계산한다.

### 8.5.3  Enabling Per-CPU Caches

캐시가 생성될 때, 해당 CPU 캐시가 활성화되고 kmalloc()을 사용하여 메모리가 할당되어야 한다. enable_cpucache() 함수는 캐시의 크기를 결정하고 kmem_tune_cpucache()를 호출하여 메모리를 할당하는 역할을 한다.

CPU 캐시는 다양한 크기의 캐시들이 활성화된 후에야 존재할 수 있으므로, 전역 변수인 g_cpucache_up을 사용하여 CPU 캐시가 조기에 활성화되는 것을 방지한다. enable_all_cpucaches() 함수는 캐시 체인의 모든 캐시를 순회하며 그들의 cpucache를 활성화한다.

CPU 캐시가 설정된 후에는 락 없이 접근할 수 있다. CPU는 잘못된 cpucache에 접근하지 않으므로 안전한 접근이 보장된다.

### 8.5.4  Updating Per-CPU Information

per-cpu 캐시가 생성되거나 변한 경우, 각 CPU는 IPI를 통해 시그널을 받는다. 캐시 일관성 문제때문에 캐시 디스크립터의 모든 값들을 바꾸는 것으로는 충분하지 않으며, 이러한 방법은 CPU 캐시들을 보호하기위해 스핀락을 사용해야한다. 대신 ccupdate_t 구조체에 각 CPU가 필요로 하는 모든 정보를 채워넣으며, 각 CPU가 캐시 디스크립터의 낡은 정보를 새 데이터로 갈아끼우는 방식을 사용한다. 새로운 cpucache 정보를 저장하기위한 구조체는 다음과 같이 정의되어있다.
```c
868 typedef struct ccupdate_struct_s
869 {
870     kmem_cache_t *cachep;
871     cpucache_t *new[NR_CPUS];
872 } ccupdate_struct_t;
```
cachep는 업데이트된 캐시이고, new는 시스템의 각 CPU들의 cpucache 디스크립터로 구성되는 배열이다. smp_function_all_cpus() 함수를 통해 각 CPU가 do_ccupdate_local()를 호출하도록 하여 ccupdate_struct_t의 정보를 캐시 디스크립터와 스왑하도록 한다.

일단 정보다 스왑되고나면, 낡은 데이터는 삭제될 수 있다.

### 8.5.5  Draining a Per-CPU Cache

캐시가 줄어들 때, 첫 단계는 모든 객체의 cpucache를 비우는 것이며, 이는 drain_cpu_caches() 함수를 호출함으로써 수행된다. 이를 통해 슬랩 할당자는 어떤 슬랩이 해제될 수 있는 지 여부를 더 확실하게 판단할 수 있다. 이러한 작업이 필요한 이유는, 슬랩의 단 하나의 객체라도 per-cpu 캐시에 존재할 경우, 모든 슬랩이 해제될 수 없기 때문이다. 시스템의 메모리가 부족할 때에는 할당 과정에서의 몇 밀리초를 절약하는 것은 그다지 중요하지 않을 것이다.

## 8.6  Slab Allocator Initialisation

여기서 우리는 슬랩 할당자가 자신을 어떻게 초기화하는 지에 대해 설명할 것이다. 슬랩 할당자가 새로운 캐시를 생성할 때, 이것은 cache_cache 또는 kmem_cache에서 kmem_cache_t를 할당한다. 이는 명백히 닭과 달걀 문제이므로 cache_cache는 다음과 같이 정적으로 초기화되어야 한다.
```c
357 static kmem_cache_t cache_cache = {
358     slabs_full:     LIST_HEAD_INIT(cache_cache.slabs_full),
359     slabs_partial:  LIST_HEAD_INIT(cache_cache.slabs_partial),
360     slabs_free:     LIST_HEAD_INIT(cache_cache.slabs_free),
361     objsize:        sizeof(kmem_cache_t),
362     flags:          SLAB_NO_REAP,
363     spinlock:       SPIN_LOCK_UNLOCKED,
364     colour_off:     L1_CACHE_BYTES,
365     name:           "kmem_cache",
366 };
```
이 코드는 다음과 같이 kmem_cache_t 구조체를 정적으로 초기화한다:
- 358-360: 3개의 리스트를 빈 리스트로 초기화한다.
- 361: 각 객체의 크기는 캐시 디스크립터의 크기이다.
- 362: 이 캐시의 생성과 삭제는 매우 드물게 발생하므로 이것은 절대 회수하지 않는다.
- 363: 스핀락을 unlock된 상태로 초기화한다.
- 364: 객체를 L1 캐시로 정렬한다.
- 365: 사람이 읽을 수 있는 이름을 기록한다.

이것은 컴파일 타임에 계산할 수 있는 모든 필드를 정의한다. 구조체의 나머지는 start_kernel()이 kmem_cache_init()을 호출하여 초기화한다.

## 8.7  Interfacing with the Buddy Allocator

슬랩 할당자는 자신만의 페이지를 보유하지 않는다. 따라서 이것은 물리 페이지 할당자로부터 페이지를 요청해야한다. 이러한 작업을 위해서 kmem_getpages()와 kmem_freepages(), 두 함수가 제공된다. 이들은 버디 할당자 API에 대한 wrapper 함수로, 슬랩 플래그가 할당에 적용되도록 하는 기능을 한다. 할당에서는 cachep->gfpflags가 기본 플래그로 사용되며, cachep->gfporder가 order로 사용된다. 여기서 cachep는 페이지를 요청하는 캐시이다. 페이지를 해제하는 경우에는, free_pages()를 호출하기전에 PageClearSlab()을 모든 페이지에 대해 호출한다.

## 8.8  Whats New in 2.6

첫 번째로 눈에 띄는 변화는 /proc/slabinfo 포맷의 버전이 1.1에서 2.0으로 변경되었으며, 훨씬 읽기 쉬워졌다는 것이다. 가장 유용한 변화는 이제 필드에 헤더가 있어 각 열이 의미하는 바를 외울 필요가 없어졌다는 점이다.

주요 알고리즘과 아이디어는 동일하며, 큰 알고리즘 변화는 없지만 구현은 상당히 다르다. 특히, per-cpu 객체의 사용과 락 회피에 더 중점을 두고있다. 또한, 디버깅 코드가 많이 추가되었으므로, 코들르 처음 읽을 때는 #ifdef DEBUG 블록을 무시해도 된다. 마지막으로, 일부 변화는 단순히 함수 이름 변경과 같은 형태적 변화로, 예를 들어, kmem_cache_estimate()는 이제 cache_estimate()라고 불리지만 다른 모든 면에서는 동일하다.

#### Cache Descriptor

kmem_cache_s의 변화는 최소한으로 진행되었다. 첫째, 자주 사용되는 요소(예: per-cpu 관련 데이터)를 구조체의 시작 부분에 배치하도록 요소들이 재정렬되었다(이유는 3.9절 참조). 둘째, 슬랩 리스트(예: slabs_full) 및 관련 통계는 별도의 kmem_list3 구조체로 이동되었다. 주석과 비정상적인 매크로 사용은 구조체를 노드별로 만들려는 계획이 있음을 나타낸다.

#### Cache Static Flags

2.4 버전의 플래그는 여전히 존재하며 그 사용 방법도 동일하다. CFLGS_OPTIMIZE는 더 이상 존재하지 않으며, 2.4에서도 사용되지 않았다. 두 개의 새로운 플래그가 도입되었다:
- SLAB_STORE_USER: 이는 객체를 해제한 함수를 기록하기 위한 디버깅 전용 플래그이다. 객체가 해제된 후 사용되면 포이즌 바이트가 일치하지 않아 커널 오류 메시지가 표시된다. 객체를 마지막으로 사용한 함수가 알려지기 때문에 디버깅이 단순해질 수 있다.
- SLAB_RECLAIM_ACCOUNT: 이는 inode 캐시와 같이 쉽게 회수할 수 있는 객체를 가진 캐시를 위한 플래그이다. slab_reclaim_pages라는 변수에 슬랩에 할당된 페이지 수를 기록하는 카운터가 유지된다. 이 카운터는 나중에 vm_enough_memory()에서 시스템이 실제로 메모리가 부족한지 판단하는 데 사용된다.

#### Cache Reaping

이는 슬랩 할당자의 가장 흥미로운 변화 중 하나이다. kmem_cache_reap()는 더 이상 존재하지 않으며, 이는 캐시 사용자보다 훨씬 우수한 선택을 할 수 있는 경우에도 무차별적으로 캐시를 축소하기 때문이다. 이제 캐시 사용자는 set_shrinker()를 통해 캐시 축소 콜백을 등록하여 슬랩을 지능적으로 에이징 및 축소할 수 있다. 이 간단한 함수는 콜백에 대한 포인터와 객체를 재생성하는 것이 얼마나 어려운 지를 나타내는 "seeks" 가중치를 포함한 struct shrinker를 채워 shrinker_list라는 연결 리스트에 배치한다.

페이지 회수 중에는 shrink_slab() 함수가 호출되어 전체 shrinker_list를 순회하며 각 shrinker 콜백을 두 번 호출한다. 첫 번째 호출에서는 0을 매개변수로 전달하여 콜백이 적절히 호출될 경우 얼마나 많은 페이지를 해제할 수 있는지 반환하게 한다. 기본적인 휴리스틱을 적용하여 콜백을 사용하는 비용이 가치 있는지를 판단한다. 만약 있다면 두 번째 호출에서는 해제할 객체 수를 매개변수로 전달한다.

이 메커니즘이 페이지 수를 계산하는 방법은 약간 까다롭다. 각 task 구조체에는 reclaim_state라는 필드가 있다. 슬랩 할당자가 페이지를 해제할 때 이 필드는 해제된 페이지 수로 업데이트된다. shrink_slab()을 호출하기 전에 이 필드를 0으로 설정하고, shrink_cache가 반환된 후 다시 읽어 얼마나 많은 페이지가 해제되었는지 판단한다.

#### Other changes

나머지 변경 사항은 본질적으로 형태적 변화이다. 예를 들어, 슬랩 디스크립터는 slab_t 대신 struct slab으로 불리며, 이는 typedef 사용을 지양하는 일반적인 추세와 일치한다. per-cpu 캐시는 구조체와 API의 이름이 변경된 것 외에는 본질적으로 동일하다. 동일한 유형의 변화는 2.6 슬랩 할당자 구현의 대부분에 적용된다.
