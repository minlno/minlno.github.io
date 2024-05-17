---
title: "Chapter 5 Boot Memory Allocator"
date: 2024-05-17T02:10:54+09:00
draft: false
author: "Minho Kim"
categories: ["Gorman's book Translation"]
---

이 글은 Gorman 책 "Understanding the Linux Virtual Memory Manager"의 [Chapter 5](https://www.kernel.org/doc/gorman/html/understand/understand008.html)를 번역한 글입니다.

---

하드웨어 구성의 조합의 수가 매우 많기 때문에, 컴파일 타임에 정적으로 핵심 커널 메모리 자료구조들을 초기화하는 것은 비현실적이다. 그러나 다음 장에서 설명할 물리 페이지 할당자조차도 자기자신을 초기화하기 위해 메모리를 할당해야한다. 그러나 어떻게 물리 페이지 할당자가 자기자신을 위한 메모리를 할당할 수 있겠는가.

이 문제를 해결하기위해, Boot Memory Allocator라고 불리는 특별한 할당자가 사용된다. 이것은 가장 기초적인 할당 방식인 First Fit 할당에 기반하며, free block의 연결 리스트 대신 비트맵을 사용하여 메모리를 표현한다. 만약 비트가 1이면 해당 페이지는 할당된 것이고 0이면 할당되지않은 것이다. 페이지 크기보다 작은 할당을 수행하기 위해, 할당자는 마지막 할당의 PFN (Physical Frame Number)와 마지막 할당이 끝난 offset을 기록한다. 그 후의 작은 할당들은 병합되어 같은 페이지에 저장된다.

독자들은 이 할당자를 왜 시스템을 수행하는 동안에도 사용하지 않는 것인지 궁금할 수 있다. 한 가지 설득력있는 이유는 비록 first fit 할당자가 fragmentation 문제를 겪지는 않더라도, 할당을 위해 메모리를 자주 선형 탐색해야하는 문제가 있다는 것이다. 이 방식은 비트맵을 이용하기 때문에 매우 오버헤드가 크며, 특히 물리 메모리의 앞부분에 작은 크기의 free block이 많이 생기는 경향이 있는데, 이 때문에 큰 사이즈의 할당을 원할 때에 탐색 오버헤드가 더욱 커지게 된다. 

Table 5.1: Boot Memory Allocator API for UMA Architectures
- unsigned long init_bootmem(unsigned long start, unsigned long page)
  - 이것은 0부터 PFN page까지의 메모리를 초기화한다. 사용가능한 메모리가 PFN start부터 시작한다.
- void reserve_bootmem(unsigned long addr, unsigned long size)
  - addr부터 addr+size 주소 사이의 페이지들을 예약 상태로 표시한다. 페이지의 일부분에 대한 예약 요청은 페이지 단위로 처리되며, 따라서 해당 페이지 전체가 예약 상태로 표시된다.
- void free_bootmem(unsigned long addr, unsigned long size)
  - addr부터 addr+size 사이의 페이지들을 free 상태로 표시한다.
- void \* alloc_bootmem(unsigned long size)
  - size 바이트만큼 ZONE_NORMAL에서 할당한다. 할당은 하드웨어 캐시로부터 최대의 이익을 보기위해 L1 하드웨어 캐시에 align된다.
- void \* alloc_bootmem_low(unsigned long size)
  - ZONE_DMA에서 size 바이트만큼 할당한다. 역시 L1 하드웨어 캐시에 align된다.
- void \* alloc_bootmem_pages(unsigned long size)
  - ZONE_NORMAL에서 size 바이트만큼 할당하며, 페이지 크기에 align된다. 따라서 페이지 전체가 caller에게 반환된다.
- unsigned long bootmem_bootmap_pages(unsigned long pages)
  - pages 개수의 페이지들의 할당 상태를 표현할 수 있는 비트맵을 저장하기 위해 필요한 페이지 수를 계산한다.
- unsigned long free_all_bootmem()
  - boot allocator의 수명이 끝날 때 사용된다. 이것은 비트맵의 모든 페이지를 순환한다. free한 페이지 각각에 대해, flag를 clear하고 physical page allocator에게 반환되어 runtime allocator가 자신의 free list를 초기화할 수 있게된다.

할당자를 위해 비슷하지만 다른 두 개씩의 API가 있다. 하나는 UMA 아키텍처를 위한 것으로 Table 5.1에서 확인할 수 있고, 나머지는 NUMA를 위한 것으로 Table 5.2에서 확인할 수 있다. 주요 차이점은 NUMA API는 해당 작업을 수행할 노드가 인자로 전달되어야 한다는 것인데, 이 API의 caller는 아키텍처 의존적인 코드에 존재하므로, 이것은 문제가 되지 않는다.

이 장은 할당자가 각 노드의 이용가능한 물리 메모리를 설명하기위해 사용하는 구조체들에 대한 설명으로 시작된다. 다음으로 우리는 물리 메모리의 끝과 각 존의 크기를 어떻게 알아내는 지에 대해 설명하며, boot memory allocator 구조체들을 초기화하기위해 해당 정보들이 어떻게 사용되는 지를 설명한다. 할당과 해제 루틴들을 설명할 것이며, 마지막으로는 boot memory allocator가 어떻게 끝을 맺는 지 이야기할 것이다.

Table 5.2: Boot Memory Allocator API for NUMA Architectures
- unsigned long init_bootmem_node(pg_data_t \*pgdat, unsigned long freepfn, unsigned long startpfn, unsigned long endpfn)
  - NUMA 아키텍처에서 사용하기위한 함수이다. freepfn에서 사용가능한 첫번째 PFN을 사용하여 startpfn에서 endpfn 사이의 메모리들을 초기화한다. 초기화가 완료되면, pgdat 노드는 pgdat_list에 삽입된다.
- void reserve_bootmem_node(pg_data_t \*pgdat, unsigned long physaddr, unsigned long size)
  - pgdat 노드의 addr부터 addr+size 사이의 페이지를 예약 상태로 표시한다. 페이지 일부분에 대한 예약 요청은 전체 페이지에 대한 예약으로 수행된다.
- void free_bootmem_node(pg_data_t \*pgdat, unsigned long physaddr, unsigned long size)
  - pgdat 노드의 addr부터 addr+size 사이의 페이지를 free 상태로 표시한다.
- void \* alloc_bootmem_node(pg_data_t \* pgdat, unsigned long size)
  - pgdat의 ZONE_NORMAL에서 size 바이트만큼을 할당한다. 할당은 하드웨어 캐시의 이득을 최대화하기위해 L1 하드웨어 캐시에 align된다.
- void \* alloc_bootmem_pages_node(pg_data_t \*pgdat, unsigned long size)
  - pgdat 노드의 ZONE_NORMAL에서 size 바이트만큼을 할당하되, page 크기로 align한다.
- void \* alloc_bootmem_low_pages_node(pg_data_t \*pgdat, unsigned long size)
  - pgdat 노드의 ZONE_DMA에서 size 바이트만큼을 할당하되, page 크기로 align한다.
- unsigned long free_all_bootmem_node(pg_data_t \*pgdat)
  - boot allocator의 수명이 끝날 때 사용된다. 이것은 특정 노드의 비트맵의 모든 페이지를 순환한다. free한 페이지 각각에 대해, flag를 clear하고 physical page allocator에게 반환되어 runtime allocator가 자신의 free list를 초기화할 수 있게된다.
  
## 5.1 Representing the Boot Map

시스템의 각 메모리 노드마다 bootmem_data라는 구조체가 존재한다. 이는 부트 메모리 할당자가 해당 노드의 메모리를 할당하기위해 필요한 정보를 포함하며, 여기에는 할당된 페이지를 표시하는 비트맵과 어디에 메모리가 위치해있는지 등이 포함된다. <linux/bootmem.h>에 아래와 같이 선언된다:
```c
<linux/bootmem.h>:

 25 typedef struct bootmem_data {
 26     unsigned long node_boot_start;
 27     unsigned long node_low_pfn;
 28     void *node_bootmem_map;
 29     unsigned long last_offset;
 30     unsigned long last_pos;
 31 } bootmem_data_t;
```
각 필드에 대한 설명은 다음과 같다:
- node_boot_start: 이 블록의 시작 물리 주소이다.
- node_low_pfn: 끝 물리 주소, 즉, 이 노드가 표현하는 ZONE_NORMAL의 끝 주소이다.
- node_bootmem_map: 각 페이지의 할당 여부를 표현하는 비트맵을 가리키는 포인터이다.
- last_offset: 마지막으로 할당한 메모리의 페이지 내에서의 오프셋이다. 0이라면 페이지가 전부 할당됨을 의미한다.
- last_pos: 마지막 할당에 사용된 페이지의 PFN을 저장한다. last_offset과 함께 사용하여, 새로운 페이지를 할당하는 대신 이전 페이지를 같이 사용할 수 있을 지를 판단할 수 있다.

## 5.2 Initialising the Boot Memory Allocator

모든 아키텍처들은 setup_arch() 함수에 부트 메모리 할당자를 초기화하는데 필요한 인자값들을 제공할 필요가 있다.

각 아키텍처들은 필요한 인자들을 얻기 위한 자신만의 함수가 있다. x86에서 이는 setup_memory()이며, MIPS 또는 Sparc에서는 bootmem_init(), 그리고 PPC에서는 do_init_bootmem()이다. 아키텍처와 상관없이 하는 일은 본질적으로 같다. 함수들이 계산하는 인자들을 다음과 같다:
- min_low_pfn: 시스템에 이용가능한 가장 낮은 PFN이다.
- max_low_pfn: ZONE_NORMAL에 해당하는 가장 큰 PFN이다.
- highstart_pfn: ZONE_HIGHMEM의 시작 PFN이다.
- highend_pfn: ZONE_HIGHMEM의 마지막 PFN이다.
- max_pfn: 시스템에 이용가능한 가장 마지막 PFN이다.

### 5.2.1 Initialising bootmem_data

이용가능한 물리 메모리의 limit들을 setup_memory()를 통해 알게되면, 두 부트 메모리 초기화 함수 중 하나를 초기화할 노드의 시작과 끝 PFN을 인자로 호출한다. UMA 아키텍처에서는 contig_page_data를 초기화하는 init_bootmem()을 사용하며, NUMA에서는 특정 노드를 초기화하는 init_bootmem_node()를 사용한다. 두 함수는 모두 init_bootmem_core()을 호출하여 실제 중요한 일을 하게된다.

핵심 함수는 먼저 pgdat_data_t를 pgdat_list에 삽입하며, 이 함수가 끝나면 해당 노드는 사용할 준비가 된다. 이 함수는 다음으로 노드와 관련된 bootmem_data_t에 노드의 시작과 끝 주소를 기록하며, 페이지 할당을 표현하는 비트맵을 할당한다. 비트맵의 바이트 크기는 다음과 같이 계산된다:
``` c
mapsize = ((end_pfn - start_pfn) + 7) / 8
```
비트맵이 표현하는 물리 주소는 bootmem_data_t->node_boot_start에 저장되며, 비트맵의 가상주소는 bootmem_data_t->node_bootmem_map에 저장된다. 메모리의 hole을 감지할 수 있는 아키텍처 독립적인 방법이 없기 때문에, 전체 비트맵은 1로 초기화되어 모든 페이지를 할당된 상태로 표시한다. 사용 가능한 페이지의 비트를 0으로 설정하는 것은 아키텍처 의존적인 코드에 달려있지만, 실제로 Sparc 아키텍처만 이 비트맵을 사용한다. x86의 경우 register_bootmem_low_pages() 함수가 e820 맵을 읽고 free_bootmem()을 호출하여 사용 가능한 각 페이지의 비트를 0으로 설정한 후, reserve_bootmem()을 사용하여 실제 비트맵에 필요한 페이지를 예약한다.

## 5.3 Allocating Memory

reserve_bootmem() 또한 사용할 페이지를 예약하기위해 사용될 수 있지만, 이는 범용적인 할당을 위해서는 사용하기 매우 번거롭다. UMA 아키텍처의 경우 쉬운 할당을 위해 alloc_bootmem(), alloc_bootmem_low(), alloc_bootmem_pages() 그리고 alloc_bootmem_low_pages() 함수가 제공되며, Table 5.1에 설명이 나와있다. 이 함수들의 call graph가 Figure 5.1에 나와있다.
![](https://velog.velcdn.com/images/minlno/post/0a5db7e6-078d-400b-bfed-a4c5698dc895/image.png)
<figcaption> Figure 5.1: Call Graph: alloc_bootmem() </figcaption>
NUMA에도 노드를 추가 인자로 받는 비슷한 함수들이 존재하며, Table 5.2에 나와있다. 이들은 alloc_bootmem_node(), alloc_bootmem_pages_node() 그리고 alloc_bootmem_low_pages_node()라고 불리며, 모두 \_\_alloc_bootmem_node()를 서로 다른 인자를 통해 호출하게된다.

\_\_alloc_bootmem()과 \_\_alloc_bootmem_node()가 받는 인자는 본질적으로 동일하다:
- pgdat: 할당할 노드이다. UMA의 경우 생략되며, contig_page_data로 가정된다.
- size: 요청된 할당의 바이트 단위 크기이다.
- align: align되어야할 바이트 크기를 의미한다. 작은 할당들은 SMP_CACHE_BYTES로 align 될 것이며, x86에서는 L1 하드웨어 캐시에 align될 것이다.
- goal: 할당을 위해 선호되는 시작 주소이다. "low" 함수들은 0에서 시작할 것이고, 다른 함수들은 아키텍처의 최대 DMA 주소를 의미하는 MAX_DMA_ADDRESS로부터 시작할 것이다.

모든 할당 API들의 핵심 함수는 \_\_alloc_bootmem_core()이다. 이는 매우 큰 함수이지만 간단한 단계들로 쪼갤 수 있다. 이 함수는 goal 주소로부터 메모리를 선형적으로 스캔하여 할당을 위한 빈 공간을 찾는다. goal 주소는 API에 따라 DMA-frendly한 할당의 경우 0이 될 수도 있고, 아니라면 MAX_DMA_ADDRESS가 될 것이다.

이 함수의 똑똑한 파트이자 주요한 파트에서는 이 할당이 이전 할당과 병합될 수 있는 지를 결정한다. 다음 조건들을 만족하면 병합이 된다:
- 직전에 할당된 페이지 (bootmem_data->pos)가 이번 할당을 위해 탐색된 페이지와 인접한  경우
- 이전 페이지가 free space를 포함하고 있는 경우 (bootmem_data->offset != 0)
- alignment가 PAGE_SIZE보다 작은 경우
할당들이 병합되는지에 상관없이, pos와 offset 필드는 마지막으로 할당된 페이지가 무엇인 지 그리고 얼마나 사용되는 지를 나타내도록 업데이트된다. 마지막 페이지가 전부 사용된다면, offset은 0으로 설정된다.

## 5.4 Freeing Memory

할당 함수와는 달리, free 함수는 UMA에서는 free_bootmem(), NUMA에서는 free_bootmem_node() 이렇게 두 종류만 제공된다. 두 함수 모두 free_bootmem_core()를 호출하며 NUMA에서는 pgdat이 제공된다는 점만 다르다.

core 함수는 할당 함수에 비해 비교적 간단하다. free로 인해 완전히 해제가 되는 페이지에 대하여, 비트맵의 해당 비트를 0으로 설정한다. 만약 이미 0이였다면, BUG()를 호출하여 이중 해제가 발생했음을 알린다. 이것은 실행중인 프로세스를 종료하며 커널 oops를 발생시켜서 개발자가 bug를 고칠 수 있도록 stack trace와 디버깅 정보를 보여준다.

free 함수의 중요한 제약 사항은 오로지 전체 페이지만 해제할 수 있다는 것이다. 페이지가 부분 할당되었다는 것은 기록이 되지 않기 때문에, 만약 부분 해제하는 경우에는 전체 페이지가 할당된 상태로 남게된다. 시스템의 lifetime 동안 할당이 항상 지속되는 것처럼 보이므로, 이것은 major한 문제는 아니다. 그렇지만, 여전히 부트 타임동안 개발자에게 중요한 제약 사항이다.

## 5.5 Retiring the Boot Memory Allocator

Bootstrapping 프로세스의 후반부에, start_kernel() 함수가 호출되는데, 이때는 boot allocator와 관련 자료구조들을 모두 삭제해도 안전하다. 각 아키텍처들은 boot memory allocator와 관련 자료구조들을 제거하는 함수인 mem_init()을 제공할 의무가 있다.
![](https://velog.velcdn.com/images/minlno/post/39977de6-48ce-4ece-b312-a78db65073d1/image.png)
<figcaption> Figure 5.2: Call Graph: mem_init() </figcaption>

이 함수의 목적은 꽤 단순하다. low와 high memory의 dimension들을 계산하고, 관련 정보를 유저에게 출력하며, 필요한 경우 하드웨어의 최종 초기화 작업을 수행한다. x86에서, VM과 관련이 있는 주요 함수는 free_pages_init()이다.

이 함수는 먼저 UMA의 경우 free_all_bootmem(), NUMA의 경우 free_all_bootmem_node()를 호출하여 boot memory allocator가 은퇴하도록 한다. 두 함수 모두 free_all_bootmem_core()를 서로 다른 인자로 호출한다. core 함수는 단순한 원칙으로 동작하며 다음과 같은 작업을 수행한다:
- 이 노드의 할당자에 대하여, 할당되지 않은 모든 페이지에 대해
  - struct page의 PG_reserved 플래그를 clear한다.
  - count를 1로 설정한다.
  - \_\_free_pages()를 호출하여 buddy allocator가 자신의 free list를 구축할 수 있도록한다.
- 비트맵을 위해 사용되고 있던 모든 페이지들을 해제하고 buddy allocator에게 준다.

이 단계에서, buddy allocator는 이제 low memory의 모든 페이지에 대한 제어권을 가진다. free_all_bootmem()이 반환한 이후에는, accounting 목적으로 reserved page들의 수를 먼저 카운팅한다. free_pages_init() 함수의 나머지는 high memory page들에 대한 것이다. 그러나, 이 시점에서 global mem_map 배열이 어떻게 할당되고 초기화되며 main allocator에게 페이지가 주어지는 지를 명백히 알아야 한다. 싱글 노드 시스템에서 low memory의 페이지들을 초기화하는 기본 플로우가 Figure 5.3에 나타나있다.
![](https://velog.velcdn.com/images/minlno/post/9538c743-d256-4fbe-9fcc-70b8f13b5383/image.png)
<figcaption> Figure 5.3: Initialising mem_map and the Main Physical Page Allocator </figcaption>

free_all_bootmem()이 반환하고나면, ZONE_NORMAL의 모든 페이지가 버디 할당자에게 반환된다. high memory 페이지들을 초기화하기위해, free_pages_init()은 highstart_pfn부터 highend_pfn 사이의 모든 페이지에 대해 one_highpage_init() 함수를 호출한다. one_highpage_init()은 단순히 PG_reserved 플래그를 클리어하고, PG_highmem 플래그를 set하며, count를 1로 설정하고, \_\_free_pages()를 호출하여 이 페이지를 free_all_bootmem_core()가 그랬던 것처럼 버디 할당자에게 반환한다.

이 시점에서, 부트 메모리 할당자는 더이상 필요하지 않으며, 버디 할당자가 시스템의 메인 물리 페이지 할당자가 된다. 흥미로운 사실은 부트 할당자를 위한 데이터뿐만 아니라 시스템의 bootstrap에 사용된 코드 또한 모두 삭제된다는 것이다. 시스템 start-up 과정에만 필요한 초기화 함수들은 모두 다음과 같이 \_\_init으로 표시된다:
```c
321 unsigned long __init free_all_bootmem (void)
```

링커는 이러한 함수들을 모두 .init 섹션에 위치시킨다. x86에서 free_initmem() 함수는 \_\_init_begin부터 \_\_init_end 사이의 모든 페이지를 순회하면서 해당 페이지들을 버디 할당자에게 반환한다. 이러한 방법으로 리눅스는 더이상 필요없는 bootstrap 코드에 사용되는 상당량의 메모리를 해제할 수 있다. 예를 들어, 이 문서를 작성하고 있는 머신의 커널을 부팅하는 동안 27개의 페이지가 해제되었다.

## 5.6 What's New in 2.6

부트 메모리 할당자는 2.4 이후로 많이 바뀌지 않았으며, 주로 최적화나 마이너한 NUMA 관련 수정에 집중되었다. 첫 최적화는 bootmem_data_t 구조체에 last_success 필드가 추가된 것이다. 이름에서 알 수 있듯이, 이것은 탐색 시간을 줄이기 위해 마지막으로 성공한 할당의 위치를 추적한다. 만약 last_success 이전의 주소가 해제되면, 이 값은 해제된 위치로 수정될 것이다.

두 번째 최적화 또한 선형 탐색과 관련이 있다. free page를 찾을 때, 2.4는 비싼 작업인 모든 비트를 테스트하는 작업을 수행했다. 2.6은 그 대신 BIT_PER_LONG 만큼의 블록마다 모두 1인지를 테스트한다. 만약 아니라면, 그때서야 블록안의 비트들을 개별적으로 테스트한다. 선형 탐색을 돕기 위해, 노드들은 init_bootmem()에 의해 물리 주소순으로 정렬된다.

마지막 변화는 NUMA와 contiguous 아키텍처와 관련이 있다. Contiguous 아키텍처는 이제 자신만의 init_bootmem() 함수를 정의하며 어떤 아키텍처든지 선택적으로 자신만의 reserve_bootmem() 함수를 정의한다.
