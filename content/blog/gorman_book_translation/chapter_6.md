---
title: "Chapter 6 Physical Page Allocation"
date: 2024-05-23T02:10:54+09:00
draft: false
author: "Minho Kim"
categories: ["Gorman Book Translation"]
categories_weight: 6
---

이 글은 Gorman 책 "Understanding the Linux Virtual Memory Manager"의 [Chapter 6](https://www.kernel.org/doc/gorman/html/understand/understand009.html)를 번역한 글입니다.

---

이 챕터에서는 리눅스에서 물리 페이지들이 어떻게 관리되고 할당되는지를 설명한다. 이에 관련하여 사용되는 주요 알고리즘은 Binary Buddy Allocator로, Knowlton이 고안했고, Knuth가 발전시켰다. 이 알고리즘은 다른 할당자들과 비교하였을 때 굉장히 빠른 성능을 보여준다.

이는 일반적인 2의 거듭제곱 할당자에 free 버퍼 합체 기술을 결합한 할당 scheme이며, 기본 컨셉은 꽤 간단하다. 메모리는 2의 거듭제곱의 페이지로 구성된 큰 페이지 블록들로 나누어진다. 만약 원하는 크기의 블록이 없는 경우, 큰 블록은 서로에게 buddy인 두 개의 블록으로 반으로 쪼개어진다. 이 중 하나는 할당에 사용되고, 나머지는 free 상태가 된다. 필요한 경우, 원하는 크기의 블록이 생길 때까지 이를 반복한다. 블록이 해제되는 경우, 버디를 체크하여 free하면 둘을 합체하여 큰 블록으로 만든다.

이 챕터는 리눅스가 어떤 메모리 블록이 free한지를 기억하는 방법을 설명한다. 그 후 페이지를 할당하고 해제하는 방법에 대해 자세히 설명한다. 그런 다음 할장자의 동작에 영향을 주는 플래그들에 대해서 다루고, 마지막으로 단편화 문제에 대해 설명하고, 할당자가 이를 어떻게 처리하는 지를 설명한다. 

## 6.1 Managing Free Blocks

앞서 말했듯이, 할당자는 2의 거듭제곱개의 페이지로 구성된 free 페이지 블록을 관리한다. 블록의 2의 지수값을 order라고 한다. Figure 6.1에서 확인할 수 있듯이, free_area_t 구조체의 배열이 관리되며, 각 요소는 인덱스에 해당하는 order의 페이지 블록들로 구성된 linked list를 가리킨다.

---
![Figure 6.1](/images/gorman_번역/figure6.1.png "Figure 6.1: Free page block management")

---

그러므로, 배열의 0번째 요소는 2^0, 즉 1개의 페이지로 구성된 free 페이지 블록의 리스트를 가리킬 것이며, 1번째는 2^1(=2)개의 페이지를, MAX_ORDER-1 번째는 2^(MAX_ORDER-1)개의 페이지들의 리스트를 가리킬 것이다 (MAX_ORDER는 현재에는 10으로 정의되어있다). 이는 작은 블록이 충분함에도 큰 블록을 쪼개는 상황이 발생할 확률을 제거한다. 페이지 블록들은 page->list를 통해 linear linked list로 관리된다

각 존은 free_area_t 구조체의 배열인 free_area[MAX_ORDER]을 가진다. 이는 <linux/mm.h>에 다음과 같이 선언된다:
```c
22 typedef struct free_area_struct {
23         struct list_head        free_list;
24         unsigned long           *map;
25 } free_area_t; 
```
각 필드의 의미는 다음과 같다:
- free_list: free page block의 linked list이다.
- map: 버디 쌍의 상태를 표현하는 비트맵이다.

리눅스는 각 버디 쌍마다 두 비트 대신 하나의 비트만 사용함으로써 메모리를 절약한다. 버디가 할당되거나 해제될 때마다, 버디 쌍에 해당하는 비트가 전환되는데, 버디가 모두 free하거나 할당된 경우 0으로 설정되고, 둘 중 하나만 할당된 경우에 1로 설정된다. 비트의 전환은 page_alloc.c에 존재하는 MARK_USED() 매크로가 수행하며, 이는 다음과 같이 선언되어있다:
```c
164 #define MARK_USED(index, order, area) \
165         __change_bit((index) >> (1+(order)), (area)->map)
```
Index는 페이지의 전역 mem_map 배열 내에서의 위치를 의미한다. 이것을 오른쪽으로 1+order 만큼 쉬프트하면, 배열 내에서 버디쌍의 위치를 계산할 수 있다.

## 6.2 Allocating Pages

리눅스는 페이지 프레임의 할당을 위해 꽤 많은 API들을 제공한다. 그들은 모두 gfp_mask를 인자로 받는데, 이는 할당자가 어떻게 동작할 지를 결정하기 위한 플래그들의 집합이다. 플래그들은 섹션 6.4에서 설명한다.

할당 API들은 모두 핵심 함수인 __alloc_pages()를 사용하지만, 정확한 노드와 존을 선택하기 위해 API가 존재한다. 유저는 서로 다른 존을 필요로 할 것이다. 예를 들면, 장치 드라이버는 ZONE_DMA를 요구할 것이고, 디스크 버퍼는 ZONE_NORMAL을 요구할 것이다. 또한 호출자는 어떤 노드가 사용되는 지에 대해 알 필요는 없을 것이다. 페이지 할당 API의 전체 리스트는 Table 6.1에서 확인할 수 있다.

---
{{< table title="Table 6.1: Physical Pages Allocation API" >}}
| |
|---|
| struct page * alloc_page(unsigned int gfp_mask) |
| 페이지 1개를 할당하고 struct page의 주소를 반환한다. |
| struct page * alloc_pages(unsigned int gfp_mask, unsigned int order) |
| 2^order 개의 페이지들을 할당하고 struct page의 주소를 반환한다. |
| unsigned long get_free_page(unsigned int gfp_mask) |
| 페이지 1개를 할당하고, 0으로 초기화한 후, 이것의 가상 주소를 반환한다. |
| unsigned long __get_free_page(unsigned int gfp_mask) |
| 페이지 1개를 할당하고, 가상 주소를 반환한다. |
| unsigned long __get_free_pages(unsigned int gfp_mask, unsigned int order) |
| 2^order개의 페이지들을 할당하고 가상 주소를 반환한다. |
| struct page * __get_dma_pages(unsigned int gfp_mask, unsigned int order) |
| 2^order개의 페이지들을 DMA 존에서 할당하고 struct page의 주소를 반환한다. |
{{< /table >}}

---
할당할 때에는 항상 특정 order을 인자로 전달해야한다. 단일 페이지의 경우에는 0을 전달해야한다. 만약 해당 order에 대한 free 블록을 찾지 못한 경우, 더 높은 order의 블록을 두 개의 버디로 쪼갠다. 이 중 하나는 할당되고 나머지는 낮은 order의 free list에 삽입된다. Figure 6.2는 2^4 블록이 프로세스가 요구하는 크기의 블록으로 쪼개지면서 free list에 추가되는 과정을 나타낸다.

---
![](/images/gorman_번역/figure6.2.png "Figure 6.2: Allocating physical pages")

---

블록이 해제되면, 버디를 체크할 것이다. 만약 두 블록이 모두 해제된 상태라면 그들은 더 높은 order의 블록으로 병합되고 해당 order의 free list에 삽입되며, 다시 해당 order에서의 버디를 체크하는 과정을 반복한다. 만약 버디가 할당된 상태라면, 해제된 블록은 현재 order의 free list에 삽입된다. free list를 조작하는 동안에는 인터럽트를 비활성화하여, 프로세스가 free list를 수정하는 도중에 인터럽트 핸들러가 리스트를 조작하는 일이 없도록 해야한다. 이를 위해 interrupt safe spinlock을 사용한다.

order를 결정했다면, 두 번째로는 어떤 메모리 노드 또는 pg_data_t를 사용할지를 결정해야한다. 리눅스는 node-local 할당 정책을 사용하며, 이는 페이지를 할당하는 프로세스가 돌고있는 CPU에 가까운 메모리 뱅크를 우선적으로 사용하는 정책을 말한다. 이와 관련해서 _alloc_pages() 함수가 중요하며, 이 함수는 UMA인지 NUMA인지에 따라 다른 코드로 컴파일이 된다.

어떤 API가 사용되는지에 상관없이, mm/page_alloc.c의 __alloc_pages() 함수가 할당의 심장 역할을 한다. 이 함수는 절대 직접 호출되지않으며, 선택된 존을 조사하고 필요한 페이지 수를 할당하기에 적합한지를 검사한다. 만약 존이 적합하지않다면, 할당자는 다른 존으로 폴백(fall back)할 것이다. 폴백하는 존의 순서는 build_zonelists()에 의해 부트 타임에 결정되지만 일반적으로 ZONE_HIGHMEM, ZONE_NORMAL, 그리고 ZONE_DMA 순으로 폴백한다. 만약 free 페이지의 수가 pages_low 워터마크에 도달한다면, 이는 **kswapd**를 깨워 존들로부터 페이지들을 해제하도록 할 것이다. 하지만 만약 메모리가 극단적으로 부족한 상황이라면, 할당자는 **kswapd**의 일을 직접 할 것이다.

---
![](/images/gorman_번역/figure6.3.png "Figure 6.3: Call Graph: alloc_pages()")

---
할당할 존이 결정되고나면, rmqueue()가 호출되어 페이지 블록을 할당하거나, 만약 적절한 크기의 블록이 없다면 큰 블록을 쪼갤 것이다.

## 6.3 Free Pages

페이지의 해제를 위한 API는 매우 간단하고, 버디 할당자의 한가지 단점이 호출자가 할당했던 크기를 기억하고있어야 한다는 것이기 때문에, 이를 기억하도록 돕기위해 API가 존재한다. 해제를 위한 API는 Table 6.2에 나와있다.

---
{{< table title="Table 6.2: Physical Pages Free API" >}}
| |
|---|
| void __free_pages(struct page *page, unsigned int order) |
| 주어진 page로부터 2^order 개의 페이지를 해제한다. |
| void __free_page(struct page *page) |
| 단일 페이지를 해제한다. |
| void free_page(void *addr) |
| 주어진 가상 주소에 해당하는 페이지를 해제한다. |
{{< /table >}}

---
페이지 해제의 주요 함수는 __free_pages_ok()이며, 이는 직접 호출되서는 안된다. 대신에 __free_pages()가 제공되어, Figure 6.4에 나와있듯이 __free_pages_ok()를 호출하기 전에 간단한 검사들을 수행한다.

---
![](/images/gorman_번역/figure6.4.png "Figure 6.4: Call Graph: __free_pages()")

---
버디가 해제될 때, 리눅스는 버디 쌍을 가능하다면 즉시 병합하고자 시도한다. 최악의 경우에는 합쳐진 많은 버디들이 즉시 쪼개질 수 있기 때문에, 이러한 정책이 최적은 아니다.

버디들을 병합할 수 있는지를 판단하기위해, 리눅스는 free_area->map에서 버디 쌍에 해당하는 비트를 확인한다. 이 함수에 의해 하나의 버디가 해제된 상태기 때문에, 최소 하나의 버디가 free하다는 것은 명백히 알 수 있다. 만약 map의 비트가 0이라면, 우리는 다른 버디또한 free하다는 것을 알 수 있다. map의 비트가 0이라면 두 버디가 모두 할당되었거나 해제되었다는 것을 의미하기 때문이다. 따라서 비트가 0이면, 버디를 병합할 수 있다.

버디의 주소를 계산하는 방법은 잘 알려져있는 개념이다. 할당이 항상 2의 거듭제곱의 크기로 이루어지기 때문에, 블록의 주소 또는 최소한 이것의 zone_mem_map에서의 offset도 2의 거듭제곱의 배수가 될 것이다. 최종 결과로 주소 비트의 오른쪽에는 항상 최소 k개의 0이 존재할 것이다. 버디의 주소를 얻기 위해서, 오른쪽에서 k번째의 비트가 검사된다. 만약 0이라면, 버디는 해당 비트가 1일 것이다. 이 비트를 얻기위해 리눅스는 mask를 다음과 같이 계산한다:
```c
  mask = (0 << k)
```
우리가 관심있는 mask는 다음과 같다:
```c
  imask = 1 + mask
```
리눅스는 이를 다음과 같이 이를 간단히 계산한다:
```c
  imask = -mask = 1 + mask
```
버디가 병합되고나면, 기존의 free list에서 제거되고 새로 생긴 블록에 대해 다음 order로 넘어가 다시 병합될 수 있는 지를 확인하게된다.

## 6.4 Get Free Page (GFP) Flags

전체 VM에 대해 항상 적용되는 개념은 Get Free Page (GFP) flag이다. 이 플래그들은 할당자와 **kswapd**가 페이지를 할당 또는 해제할 때 어떻게 동작할 지를 결정한다. 예를 들어, __GFP_WAIT 플래그는 호출자가 sleep 상태를 허용한다는 것을 의미하기 때문에, 인터럽트 핸들러는 이를 설정하지 않을 것이다. 세 집합의 GFP 플래그가 있으며, 이들은 모두 <linux/mm.h>에 정의되어있다.

셋 중 첫 번째는 zone modifiers이며 Table 6.3에 나와있다. 이 플래그들은 호출자가 특정 존에서 할당을 시도해야한다는 것을 나타낸다. 눈치빠른 독자들은 ZONE_NORMAL을 위한 플래그가 없다는 것을 알아챘을 것이다. 이는 해당 플래그들이 배열의 오프셋으로 사용되며, 0인 경우에는 암시적으로 ZONE_NORMAL에서의 할당을 의미하기 때문이다.

---
{{< table title="Table 6.3: Low Level GFP Flags Affecting Zone Allocation" >}}
| Flag | Description |
| --- | --- |
| __GFP_DMA | 가능하면 ZONE_DMA에서 할당한다. |
| __GFP_HIGHMEM | 가능하면 ZONE_HIGHMEM에서 할당한다. |
| GFP_DMA | __GFP_DMA의 alias이다. |
{{< /table >}}

---
다음 플래그들은 action modifiers로 Table 6.4에 나와있다. 이들은 VM의 동작과 호출 프로세스가 무엇을 할 것인지를 결정한다. 이들에 대한 low level 플래그들은 쉽게 사용하기에는 너무 원시적이다.

---
{{< table title="Table 6.4: Low Level GFP Flags Affecting Affecting Allocator behaviour" >}}
| Flag | Description |
| --- | --- |
| __GFP_WAIT | 호출자가 높은 우선순위가 아니며 sleep하거나 reschedule해도된다는 것을 나타낸다. |
| __GFP_HIGH | 높은 우선순위 또는 커널 프로세스가 사용한다. Kernel 2.2.x는 이것을 프로세스가 메모리의 긴급 pool에 접근가능한 지를 결정하기위해 사용했다. 2.4.x 커널에서는 이것이 사용되지않는다. |
| __GFP_IO | 호출자가 저수준 IO를 수행할 수 있는지를 나타낸다. 2.4.x에서 이것은 주로 try_to_free_buffers()가 버퍼를 flush 할지 말지를 결정하는데 사용된다. 이는 최소한 하나의 journaled filesystem에 의해 사용되고있다. |
| __GFP_HIGHIO | High memory에 매핑된 페이지에 대해 IO를 수행할 수 있는지를 결정한다. try_to_free_buffers()에서만 사용된다. |
| __GFP_FS | 호출자가 파일시스템 계층에 대한 호출을 수행할 수 있는지를 나타낸다. 호출자가 파일시스템과 관련이 있고 재귀적으로 호출되는 것을 피하기위해 사용되며, buffer cache를 예로 들 수 있다. |
{{< /table >}}

---
각 인스턴스의 정확한 조합이 무엇인지를 아는 것은 어렵기 때문에, 몇개의 high level 조합이 정의되어 있으며, Table 6.5에서 확인할 수 있다. 편의를 위해 __GFP_를 생략했기 때문에, __GFP_HIGH 플래그는 아래에서 HIGH로 쓰여져있을 것이다. 각 조합에 대한 설명은 Table 6.6에 나와있다. 이해를 돕기위해, GFP_ATOMIC을 예로들어 설명하겠다. 이것은 오직 __GFP_HIGH 플래그를 set한다. 이는 이것이 높은 우선순위임을 의미하며, emergency pool을 사용할 수 있고, sleep하지않으며, IO를 수행하거나 파일시스템에 접근하지 않을 것이다. 이 플래그는 인터럽트 핸들러가 사용할 것이다.

---
{{< table title="Table 6.5: Low Level GFP Flag Combinations For High Level Use" >}}
| Flag | Low Level Flag Combination |
| --- | --- |
| GFP_ATOMIC | HIGH |
| GFP_NOIO | HIGH \| WAIT |
| GFP_NOHIGHIO | HIGH \| WAIT \| IO |
| GFP_NOFS | HIGH \| WAIT \| IO \| HIGHIO |
| GFP_KERNEL | HIGH \| WAIT \| IO \| HIGHIO \| FS |
| GFP_NFS | HIGH \| WAIT \| IO \| HIGHIO \| FS |
| GFP_USER | WAIT \| IO \| HIGHIO \| FS |
| GFP_HIGHUSER | WAIT \| IO \| HIGHIO \| FS \| HIGHMEM |
| GFP_KSWAPD | WAIT \| IO \| HIGHIO \| FS |
{{< /table >}}

---

---
{{< table title="Table 6.6: High Level GFP Flags Affecting Allocator Behaviour" >}}
| Flag | Description |
| --- | --- |
| GFP_ATOMIC | 이 플래그는 호출자가 sleep할 수 없고 가능하면 반드시 성공해야하는 상황에 사용된다. 인터럽트 핸들러가 메모리를 필요로 한다면, sleep과 IO 수행을 방지하기 위해 이 플래그를 사용해야한다. buffer_init() 또는 inode_init() 과 같은 많은 서스시스템의 초기화 과정은 이것을 사용한다. |
| GFP_NOIO | 호출자가 이미 IO와 관련된 작업을 수행중인 경우 이 플래그를 사용한다. 예를 들어, loop back 장치가 buffer head를 위해 페이지를 할당하고자하는 경우, 이것은 추가적인 IO를 야기할 수 있는 일부 동작을 수행하지 않도록 하기위해 이 플래그를 사용해야한다. 사실 이 플래그는 loopback 장치에서의 데드락을 피하기위해 소개되었다. |
| GFP_NOHIGHIO | 이 플래그는 오로지 alloc_bounce_page()에서 high memory에 IO를 위한 bounce buffer를 생성하는동안 사용된다. |
| GFP_NOFS | 이것은 오로지 buffer cache와 파일시스템들에 의해서만 사용되며, 그들 스스로를 재귀적으로 호출하지 않도록 하기위해 사용된다. |
| GFP_KERNEL | 고수준 GFP 플래그들 중에서 가장 자유로운 플래그이다. 이것은 호출자가 무엇이든 허용함을 의미한다. GFP_USER와 오로지 다른 점은 emergency pool을 사용할 수 있다는 점이지만, 2.4.x 커널에서는 이것이 no-op이다. |
| GFP_USER | 역사적 중요성을 지닌 또 다른 플래그이다. 2.2.x 시리즈에서는, 할당이 LOW, MEDIUM, 또는 HIGH 우선순위를 가졌었다. 만약 메모리가 거의 없다면, GFP_USER(low) 플래그가 설정된 요청은 실패하며, 다른 할당은 계속 시도할 것이다. 이제 이것은 중요하지 않으며, GFP_KERNEL과 구별되지 않는다. |
| GFP_HIGHUSER | 이 플래그는 할당자가 가능하면 ZONE_HIGHMEM에서 할당해야함을 의미한다. 유저 프로세스 대신 페이지를 할당하는 경우 이것을 사용한다. |
| GFP_NFS | 이것은 이제 사용하지않는다. 2.0.x 시리즈에서 이 플래그는 예약된 페이지 크기를 결정했었다. 보통은 20개의 free page가 예약되었고, 이 플래그가 설정된 경우, 5개의 페이지만 예약됬었다. |
| GFP_KSWAPD | 역사적인 의미가 있는 플래그이다. 하지만 더이상 GFP_KERNEL과 구별되지않는다. |
{{< /table >}}

---

### 6.4.1 Process Flags

프로세스는 task_struct에도 할당자 동작에 영향을 주는 플래그를 설정할 수 있다. 프로세스 플래그의 전체 리스트는 <linux/sched.h>에 정의되어있다. 이 중 VM 동작에 영향을 주는 플래그들을 Table 6.7에서 확인할 수 있다.

---
{{< table title="Table 6.7: Process Flags Affecting Allocator behaviour" >}}
| Flag | Description |
| --- | --- |
| PF_MEMALLOC | 프로세스가 메모리 할당자임을 표시한다. **kswapd**와 Out of Memory (OOM, 13장 참조) killer가 kill하기로 결정한 프로세스들에게 이 플래그가 설정된다. 이것은 버디 할당자에게 존 워터마크를 무시하고 가능하다면 최대한 페이지를 할당하도록 한다. |
| PF_MEMDIE | 이것은 OOM killer에 의해 설정되며, PF_MEMALLOC과 마찬가지로 페이지 할당자에게 프로세스가 곧 죽기 때문에 가능하다면 페이지를 할당하도록 말해준다. |
| PF_FREE_PAGES | 버디 할당자가 try_to_free_pages()를 호출할 때 자기 자신에게 설정하여, __free_pages_ok()가 해제한 페이지를 free list에 넣는 대신 예약된 상태로 만들도록 지시한다. |
{{< /table >}}

---

## 6.5 Avoiding Fragmentation

내부 및 외부 단편화 문제는 어떤 할당자에서든 고려해야할 중요한 문제이다. 외부 단편화는 원하는 크기보다 작은 블록밖에 없어서 할당을 못하는 문제를 말한다. 내부 단편화는 실제 할당하는 블록 크기보다 작은 메모리를 사용하여 공간을 낭비하는 문제를 말한다. 리눅스에서 크고 연속된 페이지에 대한 할당 요청은 드물기 때문에 외부 단편화는 심각한 문제가 아니며, 대개 vmalloc() (7장 참조)으로 해당 요청을 충분히 수행할 수 있다. free 블록을 order별 리스트로 관리하기 때문에 큰 블록을 불필요하게 쪼갤 필요가 없다.

내부 단편화는 이진 버디 시스템의 가장 심각한 단 하나의 단점이다. 내부 단편화는 28%의 영역을 낭비할 것으로 기대되지만, 이것이 60%의 영역까지도 낭비할 수도 있다는 것이 보여졌으며, first fit 할당자는 단지 1%만 낭비하는 것과 대비된다. 버디 시스템의 다양한 변형들을 사용하는 것 또한 도움이 별로 되지않는다는 것이 보여졌다. 이 문제를 다루기위해, 리눅스는 slab allocator를 사용하여 페이지를 더 작은 블록들로 조각을 내서 할당에 사용하며, 이는 8장에서 더 자세히 논의한다. 이러한 할당자들의 조합을 통해, 커널은 내부 단편화로 인해 낭비되는 메모리의 양을 최소한으로 유지하도록 보장한다.

## 6.6 What's New In 2.6

#### Allocating Pages

주목할만한 첫 번째 변화는 겉표면에서만 이루어진 것으로 보인다. alloc_pages() 함수는 원래 <linux/mm.h>에 정의된 함수였으나, 이제는 <linux/gfp.h>에 정의된 매크로이다. 새로운 레이아웃은 여전히 매우 알아보기 쉬우며, 주요 차이점은 미묘하지만 중요하다. 2.4에서는 스케줄링된 CPU를 기반하여 할당할 정확한 노드를 고르기위한 전용 코드가 있었지만, 2.6에서는 NUMA와 UMA 아키텍처간의 이러한 구분이 제거되었다.

2.6에서, alloc_pages()는 numa_node_id()를 호출하여 현재 실행중인 CPU와 연관된 노드의 논리적 ID를 알아낸다. 이 NID는 _alloc_pages()에게 전달되고, 이 함수는 NID를 인자로 사용하여 NODE_DATA()를 호출한다. UMA 아키텍처에서 이것은 항상 contig_page_data를 반환하지만, NUMA에서는 NODE_DATA()가 NID를 통해 offset 정보를 알 수 있도록하는 배열을 준비한다. 즉, 아키텍처는 CPU ID와 NUMA 메모리 노드간의 매핑을 설정할 책임이 있다. 정책은 2.4와 동일하게 node-local allocation을 따르지만, 구현이 훨씬 더 명확해졌다.

#### Per-CPU Page Lists

페이지 할당에 추가된 가장 중요한 것은 per-cpu lists의 추가이며, 2.6 섹션에서 논의됐었다.

2.4에서, 페이지 할당은 interrupt safe spinlock을 할당하는 동안 잡고있어야한다. 2.6에서는 페이지들이 buffered_rmqueue()를 통해 struct per_cpu_pageset에서 할당된다. low 워터마크 (per_cpu_pageset->low)가 도달하지 않았다면, 페이지들을 스핀락을 잡지않고 pageset에서 할당할 수 있다. low 워터마크에 도달하면, 많은 페이지를 interrupt safe spinlock을 잡고 한번에 할당하여 per-cpu list에 추가한 후, 하나를 caller에 반환한다.

상대적으로 드문 높은 order의 할당은 여전히 interrupt safe spinlock을 잡아야하며 블록을 쪼개는 것과 병합하는 것을 즉시 처리한다. order 0의 할당은 low 워터마크에 도달하기 전까지 쪼개는 것을 미룰 것이고, high 워터마크에 도달하기 전까지 병합하는 것을 미룰 것이다.

그러나, 엄격히 말해서, 이것은 lazy buddy 알고리즘이 아니다. pageset이 order-0의 할당의 merging delay를 도입하기는 했지만, 이것은 의도된 특징이 아니라 부작용이며, pageset을 비우고 버디들을 병합할 수 있는 이용가능한 method는 없다. 즉, per-cpu와 새로운 accounting 관련하여 많은 양의 코드가 mm/page_alloc.c에 추가되었지만, 버디 알고리즘의 핵심은 여전히 2.4 버전과 동일하다.

이러한 변화의 영향은 직관적이다. 버디 리스트들을 보호하는 스핀락을 잡아야하는 횟수가 줄어든다. 리눅스에서 높은 order에 대한 할당은 드물기 때문에 이 최적화는 대부분의 경우에 적용된다. 이 변화는 단일 CPU 머신에 대해서는 차이점이 거의 없겠지만, 많은 수의 CPU를 가진 머신에서 그 효과가 드러날 것이다. pageset과 관련된 몇가지 이슈가 존재하지만, 그들은 심각한 문제는 아니다. 첫 번째 문제는 pageset이 order-0 페이지를 캐싱해두고 있기때문에 높은 order의 할당이 실패할 수도 있다는 것이다. 두 번째는 memory가 적고, 현재 CPU의 pageset이 비어있고, 다른 CPU의 pageset이 꽉 차있는 경우, "원격" pageset의 페이지를 회수하기위한 메커니즘이 없기때문에 order-0 할당이 실패할 수도 있다는 것이다. 마지막 잠재적 문제는 새롭게 해제된 페이지들의 버디쌍이 서로 다른 pageset에 속해있는 경우, 단편화 문제를 야기할 수도 있다는 것이다.

#### Freeing Pages

free_hot_page()와 free_cold_page()라는 새로운 free API 함수가 도입되었다. 이것은 해제된 페이지가 per-cpu pageset의 hot 또는 cold 리스트 중 어떤 리스트에 삽입될 지를 결정한다. 그러나, free_cold_page()도 export되고 사용가능함에도 불구하고, 실제로는 누구도 이것을 호출하지 않는다.

__free_pages()를 통해 해제되었거나 __page_cache_release()를 통해 방출된 page cache로 인해 해제된 order-0 페이지들은 hot list에 삽입되며, 높은 order의 페이지의 경우 __free_pages_ok()를 통해 즉시 해제된다. Order-0는 대개 유저공간과 관련되며 할당과 해제에 쓰이는 가장 흔한 타입이다. 이들을 local CPU에 캐싱하면 대부분의 할당이 order-0이므로, lock contention이 줄어들 것이다.

결국, 페이지들의 리스트가 free_pages_bulk()로 전달되거나, pageset 리스트가 모든 free 페이지들을 캐싱하게된다. free_pages_bulk() 함수는 페이지 블록 할당의 리스트, 각 블록의 order, 그리고 리스트에서 해제할 블록의 수인 count를 인자로 받는다. 첫 번째는 __free_pages_ok()에 전달된 high order의 free 페이지이다. 이 경우, 페이지 블록은 지정된 순서와 count 1로 연결 리스트에 배치된다. 두 번째 경우는 실행 중인 CPU의 pageset에서 high 워터마크에 도달한 경우이다. 이 경우, pageset이 order 0과 pageset->batch의 카운트로 전달된다.

핵심 함수 __free_pages_bulk()에 도달하면, 페이지를 버디 리스트로 해제하는 메커니즘은 버전 2.4와 매우 유사하다.

#### GFP Flags

여전히 세 개의 영역만 있기 때문에 zone modifier들은 동일하게 유지되지만, 요청을 만족시키기 위해 VM이 얼마나 열심히 작동할지, 또는 작동하지 않을지에 영향을 미치는 세 개의 새로운 GFP 플래그가 추가되었다. 플래그는 다음과 같다:

- __GFP_NOFAIL: 이 플래그는 할당이 절대 실패하지 않아야 하며 할당자가 무기한으로 할당을 시도해야 한다는 것을 호출자가 나타낼 때 사용된다.
- __GFP_REPEAT: 이 플래그는 요청이 실패할 경우 할당을 반복 시도해야 한다는 것을 호출자가 나타낼 때 사용된다. 현재 구현에서는 __GFP_NOFAIL과 동일하게 동작하지만, 나중에는 일정 시간이 지나면 실패하도록 결정할 수 있다.
- __GFP_NORETRY: 이 플래그는 거의 __GFP_NOFAIL의 반대이다. 할당이 실패할 경우 즉시 반환해야 한다는 것을 나타낸다.

이 글을 쓰는 시점에서는 많이 사용되지 않지만, 도입된 지 얼마 되지 않아 시간이 지나면 더 많이 사용될 가능성이 높다. 특히 __GFP_REPEAT 플래그는 커널 전반에 이 플래그의 동작을 구현하는 코드 블록이 존재하기 때문에 많이 사용될 가능성이 높다.

다음으로 도입된 GFP 플래그는 __GFP_COLD라는 할당 수정자로, 이는 per-cpu 리스트에서 cold 페이지가 할당되도록 보장하는 데 사용된다. VM의 관점에서 이 플래그의 유일한 사용자는 주로 IO readahead 시 사용되는 함수 page_cache_alloc_cold()이다. 일반적으로 페이지 할당은 hot 페이지 리스트에서 가져오게 된다.

마지막 새로운 플래그는 __GFP_NO_GROW이다. 이는 8장에서 논의되는 슬랩 할당자에 의해서만 사용되는 내부 플래그로, SLAB_NO_GROW 플래그와 동일하게 사용된다. 이는 특정 캐시에 대해 새로운 슬랩을 절대 할당해서는 안 되는 경우를 나타내기 위해 사용된다. 실제로 이 GFP 플래그는 현재 메인 커널에서 사용되지 않는 기존 SLAB_NO_GROW 플래그를 보완하기 위해 도입된 것이다.
