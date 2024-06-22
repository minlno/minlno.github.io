---
title: "Chapter 9  High Memory Management"
date: 2024-06-22T02:10:54+09:00
draft: false
author: "Minho Kim"
categories: ["Gorman Book Translation"]
categories_weight: 9
---

이 글은 Gorman 책 "Understanding the Linux Virtual Memory Manager"의 [Chapter 9](https://www.kernel.org/doc/gorman/html/understand/understand012.html)를 번역한 글입니다.

---

커널은 페이지 테이블 엔트리가 세팅된 메모리 주소에 대해서만 직접 접근할 수 있다. 4.1절에서 설명했듯이, 일반적으로 유저/커널 공간은 3GiB/1GiB로 분리되어 있어서 32비트 머신에서 커널은 최대 896MiB의 메모리만 직접 접근할 수 있다. 64비트 하드웨어에서는 충분한 가상 주소 공간이 있으므로 이것이 문제가 되지 않는다. 2.4 커널이 몇 테라바이트의 RAM이 달린 머신에서 실행될 가능성이 매우 낮기 때문이다.

많은 고성능의 32비트 머신들은 1GiB 이상의 메모리를 탑재하므로 이러한 문제를 무시할 수 없다. 리눅스는 고메모리 (high memory)의 페이지들을 임시적으로 낮은 페이지 테이블에 매핑하여 이 문제를 해결한다. 자세한 내용은 9.2절에서 설명할 것이다.

모든 장치들이 고메모리 또는 CPU가 이용가능한 메모리에 접근할 수 있는 것은 아니기 때문에, IO와 고메모리는 해결해야할 비슷한 문제를 가지고있다. CPU가 PAE 확장 기능을 가지고있고 장치는 signed 32-bit 정수의 크기 (2GiB) 까지만 접근가능하거나, 32비트 장치가 64비트 아키텍처에서 사용되는 경우가 이에 해당된다. 이때 장치에게 메모리 쓰기를 요청할 경우, 이는 실패할 것이고 최악의 경우 커널이 망가질 수도 있다. 이 문제는 "바운스 버퍼 (bounce buffer)"를 사용하여 해결하며, 9.4절에서 설명한다.

이 장은 PKMap (Persistent Kernel Map) 주소 공간이 어떻게 관리되는 지에 대해 먼저 간단히 설명한 후, 페이지들이 고메모리에서 어떻게 매핑되고 매핑 해제되는 지를 설명한다. 다음 절에서는 매핑이 원자적이어야하는 케이스를 설명하고, 바운스 버퍼에 대해 자세히 설명한다. 마지막으로 우리는 메모리가 매우 부족한 경우에 사용되는 긴급 풀 (emergency pool)에 대해 이야기할 것이다.

## 9.1  Managing the PKMap Address Space

커널 페이지 테이블에서 PKMAP_BASE부터 FIXADDR_START까지의 공간이 PKMap을 위한 공간으로 예약된다. 해당 공간의 크기는 약간씩 다르다. x86에서 PKMAP_BASE는 0xFE000000이며 FIXADDR_START는 설정 옵션에 따라 컴파일 타임에 결정되며 보통 선형 주소 공간의 마지막에서 몇개의 페이지만큼만 떨어져있다. 이는 고메모리가 사용가능한 공간으로 매핑될 수 있는 크기가 32MiB보다 약간 작다는 것을 의미한다.

한 페이지만큼의 PTE들이 페이지를 매핑하기위해 PKMap 영역의 시작부분에 저장된다. 이를 이용하여 1024개의 high 페이지들이 저메모리에 kmap()/kunmap()을 통해 매핑/매핑 해제된다. Pool이 매우 작아보이지만, 페이지는 kmap()에 의해 매우 짧은 시간 동안만 매핑될 것이다. 코드의 주석을 보면 contiguous 페이지 테이블 엔트리를 할당하여 이 영역을 확장할 계획이 있음을 알 수 있다.

kmap()이 사용하는 페이지 테이블 엔트리는 pkmap_page_table이라고 불리며, PKMAP_BASE에 위치하고 시스템 초기화 과정에서 세팅된다. x86에서, 이는 pagetable_init() 함수의 마지막에 수행된다. PGD와 PMD 엔트리들을 위한 페이지들은 부트 메모리 할당자에 의해 할당된다.

페이지 테이블 엔트리들의 현재 상태는 pkmap_count라는 이름의 LAST_PKMAP만큼의 요소를 가진 간단한 배열에 의해 관리된다. PAE 기능이 없는 x86 시스템에서 이 값은 1024이고 PAE에서는 512이다. 더 정확히는, 코드에는 표현되어있지는 않지만, LAST_PKMAP 변수는 PTRS_PER_PTE와 동일한 값이다.

각 요소는 정확히 참조 카운트를 의미하지는 않지만 매우 유사하다. 만약 값이 0이라면 페이지는 free하며, 마지막 TLB flush 이후에 한번도 사용된 적이 없음을 의미한다. 만약 1이라면, slot은 사용되지않지만 page가 여전히 매핑된 상태로 TLB flush를 기다리고있음을 의미한다. 전역 페이지 테이블이 수정되면 모든 CPU에 대한 flush가 필요하고 이는 매우 비싼 작업이므로, 모든 슬롯이 사용될 때까지는 flush들이 지연된다. 이보다 더 큰 값은 n-1의 유저가 참조하고 있음을 의미한다.

## 9.2  Mapping High Memory Pages

Table 9.1에 고메모리의 페이지를 매핑할 때 사용하는 API들이 나와있다. 메인으로 사용되는 함수는 kmap()이다. block되기를 원하지 않는 사용자들은 kmap_nonblock()을 사용해야하며, 인터럽트 컨텍스트의 사용자들은 kmap_atomic()을 사용해야한다. kmap pool이 많이 작아 저메모리에 비해 고메모리에 대한 메모리 압력이 훨씬 빠르게 증가하기 때문에, kmap()의 사용자들은 가능한 한 빨리 kunmap()을 호출해야한다.

---
![](/images/gorman_번역/figure9.1.png "Figure 9.1: Call Graph: kmap()")

---

kmap() 함수 자체는 꽤 간단하다. 이것은 먼저 인터럽트 컨텍스트에서 이것을 호출한 것이 아님을 체크하고 (잠들 수도 있기 때문), 만약 그렇다면 out_of_line_bug()를 호출한다. 인터럽트 핸들러에서 BUG()가 호출되면 시스템이 패닉 상태에 빠지므로, out_of_line_bug()는 버그 관련 정보를 출력하고 안전하게 종료한다. 두 번째로는 요청된 페이지가 highmem_start_page보다 낮은 주소에 있는 지 확인하는데, 만약 그렇다면 해당 페이지는 이미 visible하며 매핑될 필요가 없음을 의미한다.

다음으로는 페이지가 이미 low 메모리에 있는 지 확인하며, 만약 그렇다면 해당 주소를 단순히 반환한다. 이를 통해, 함수의 사용자는 요청 페이지가 low 메모리 페이지인지를 인지하지않고도 안전하게 kmap() 함수를 사용할 수 있다. 만약 이것이 high page였다면, kmap_high()를 호출하여 실제 작업을 시작한다.

kmap_high() 함수는 먼저 page->virtual 필드를 체크하여 이미 매핑되었는 지를 확인한다. 만약 NULL이라면, 매핑이 안된 것이므로 map_new_virtual()을 통해 페이지를 매핑한다.

map_new_virtual() 함수는 새로운 가상 매핑을 생성하기위해 단순히 선형적으로 pkmap_count 배열을 스캔한다. 이때 다른 kmap() 호출 때 스캔한 영역과 같은 영역을 중복해서 스캔하는 것을 방지하기위해, 0 대신 last_pkmap_nr부터 스캔을 시작한다. last_pkmap_nr이 다시 0으로 돌아왔을 경우에는 flush_all_zero_pkmaps()를 호출하여 모든 엔트리들을 1에서 0으로 초기화하고 TLB를 flush한다.

만약 스캔을 통해 엔트리를 찾지 못했다면, 프로세스는 pkmap_map_wait 이라는 이름의 wait queue에서 잠든 후, 다음 kunmap() 호출을 통해 깨어나게 될 것이다.

매핑이 생성된 후에는, pkmap_count 배열의 해당 엔트리의 값이 증가하며, low 메모리에서의 가상 주소가 반환된다.

---
{{< table title="Table 9.1: High Memory Mapping API" >}}
| |
| --- |
| void * kmap(struct page *page) |
| high 메모리의 page 구조체를 인자로 받아서 이를 low 메모리에 매핑한다. 매핑된 가상주소가 반환된다. |
| void * kmap_nonblock(struct page *page) |
| 슬롯이 부족할 경우 sleep 하는 대신 NULL을 반환한다는 점에서 kmap()과 다르다. kmap_atomic()의 경우 특별히 예약된 슬롯을 쓴다는 점에서 이와 다르다. |
| void * kmap_atomic(struct page *page, enum km_type type) |
| 인터럽트 컨텍스트에서 원자적으로 사용할 수 있도록 map에 유지되는 슬롯들이 존재한다 (9.3절 참조). 이것의 사용은 매우 권장되지 않으며 이 함수의 호출자는 sleep 또는 schedule하지 않을 수 있다. 이 함수는 특수한 목적으로 high 메모리의 페이지를 매핑할 것이다.  |
{{< /table >}}

---

### 9.2.1  Unmapping Pages

Table 9.2에 high 메모리 매핑을 해제하기위한 API들이 나와있다. kunmap() 함수는 kmap()과 마찬가지로 두가지 체크를 한다. 첫번째는 인터럽트 컨텍스트인지이다. 두번째는 페이지가 highmem_start_page보다 낮은지이다. 만약 그렇다면 페이지는 이미 low 메모리에 존재하여 더이상의 처리가 필요하지않다. 페이지의 매핑을 해제해야한다면, kunmap_high()를 호출하여 해당 작업을 수행한다.

---
![](/images/gorman_번역/figure9.2.png "Figure 9.2: Call Graph: kunmap()")

---

kunmap_high()는 단순한 원리로 동작한다. 이것은 pkmap_count의 해당하는 엔트리를 감소시킨다. 만약 이 값이 1에 도달하면 해당 슬롯이 이용가능해졌음을 의미하므로 pkmap_map_wait에서 대기하고 있는 아무 프로세스를 깨운다. 페이지를 page table에서 unmap하는 것은 TLB flush를 요구하므로, 이 작업은 flush_all_zero_pkmaps()가 호출될 때까지 미뤄둔다.

---
{{< table title="Table 9.2: High Memory Unmapping API" >}}
| |
| --- |
| void * kunmap(struct page *page) |
| low 메모리에서 페이지를 unmap하고 이를 매핑하고 있는 page table entry를 해제한다. |
| void * kunmap_atomic(struct page *page, enum km_type type) |
| 원자적으로 매핑되었던 page를 unmap한다. |
{{< /table >}}

---

## 9.3  Mapping High Memory Pages Atomically

kmap_atomic()의 사용은 권장되지 않지만 필요한 경우를 위해 각 CPU를 위한 슬롯인 bounce buffer가 예약되며, 장치가 인터럽트 처리 과정에서 이를 사용한다. 원자적 high 메모리 매핑에 대해 아키텍처마다 서로 다른 수의 사용 케이스르 가지며, 이는 열거형 타입인 km_type으로 정의된다. 전체 개수는 KM_TYPE_NR으로 정의된다. x86의 경우, atomic kmap의 6가지 사용 케이스가 존재한다.

원자적 매핑을 위해 프로세서마다 KM_TYPE_NR 개의 엔트리가 부트 타임에 FIX_KMAP_BEGIN부터 FIX_KMAP_END 사이의 공간에 예약된다. kmap_atomic()을 호출한 후, 호출자가 sleep하게되면 다음 순서로 스케줄된 프로세스가 같은 엔트리를 사용하려고 시도할 수 있고 이는 실패할 것이기 때문에, kmap_atomic()을 호출한 프로세스는 kunmap_atomic()을 호출하기 전에는 sleep하거나 exit하지 않아야 한다.

kmap_atomic() 함수는 프로세서의 특정 사용 케이스를 위해 페이지 테이블에 미리 설정된 슬롯에 페이지를 매핑하는 아주 간단한 작업을 수행한다. kunmap_atomic() 함수의 흥미로운 점은 디버깅 옵션이 켜졌을 때만 PTE를 pte_clear()로 클리어 한다는 것이다. 이는 다음 kmap_atomic()이 어차피 다시 사용할 매핑이기 때문에 TLB flush가 필요없으므로 불필요한 작업이다.

## 9.4  Bounce Buffers

바운스 버퍼 (Bounce buffers)는 CPU의 메모리의 전체 범위를 모두 접근할 수는 없는 장치에게 필요하다. 이것의 명백한 예시는 64 비트 아키텍처 또는 PAE가 enable된 최근 인텔 프로세서에 장착된 32 비트 장치처럼 주소에 CPU보다 적은 수의 비트를 사용하는 장치이다.

기본 컨셉은 매우 단순하다. 바운스 버퍼는 장치가 데이터를 복사해오거나 할 수 있을만큼 충분이 낮은 주소에 위치한다. 그런 다음 이 데이터는 high 메모리에 있는 유저 페이지에 복사된다. 이 추가적인 복사는 바람직하지 않지만, 피할 수 없다. 장치의 DMA를 위해 사용되는 버퍼 페이지들을 low 메모리에 할당한다. IO 작업이 완료되고나면 커널이 이들을 high 메모리에 있는 버퍼 페이지에 복사한다. 즉, 징검다리와 같은 역할을 하는 것이다. 전체 페이지를 복사해야한다는 점에서 상당한 오버헤드가 존재하지만, low 메모리의 페이지를 스왑 아웃하는 것에 비하면 이는 사소하다.

### 9.4.1  Disk Buffering

일반적으로 1KiB정도 크기인 블록들은 페이지에 담겨 슬랩 할당자로 할당되는 struct buffer_head에 의해 관리된다. 버퍼 헤드의 사용자들은 콜백 함수를 등록할 수 있다. 콜백 함수는 buffer_head->b_end_io()에 저장되며, IO가 완료될 때 호출된다. 이것이 바운스 버퍼의 데이터를 high 메모리에 복사하는 메커니즘이다. 등록된 콜백은 bounce_end_io_write() 함수이다.

버퍼 헤드의 다른 기능이나 그들이 블록 레이어에 의해 어떻게 사용되는 지는 이 문서의 범위를 벗어나며 IO 계층에 더 관련된다.

### 9.4.2  Creating Bounce Buffers

바운스 버퍼를 생성하는 것은 create_bounce() 함수에 의해 간단히 수행된다. 원칙은 매우 간단하다. 제공되는 버퍼 헤드를 템플릿 삼아 새로운 버퍼를 생성한다. 함수는 read/write 인자 (rw)와 사용할 템플릿 버퍼 헤드 (bh_orig)를 인자로 받는다.

---
![](/images/gorman_번역/figure9.3.png "Figure 9.3: Call Graph: create_bounce()")

---

버퍼 자체를 위한 페이지는 alloc_bounce_page() 함수를 통해 할당되며, 이 함수는 alloc_page()에 한가지 중요한 추가 기능을 제공하는 wrapper 함수이다. 만약 할당이 실패했다면, 바운스 버퍼가 사용할 수 있는 페이지와 버퍼 헤드의 긴급 풀 (emergency pool)이 존재한다. 이는 9.5절에서 더 자세히 설명한다.

버퍼 헤드는 alloc_bounce_bh()에 의해 할당되며, 이는 buffer_head를 위한 슬랩 할당자를 호출하며, 할당에 실패한 경우 긴급 풀을 사용한다. 추가적으로, bdflush를 깨워서 dirty한 버퍼를 디스크로 flush하도록 함으로써, 버퍼들이 곧 해제되도록 만든다.

페이지와 buffer_head가 할당된 후에는, 템플릿 버퍼 헤드의 정보를 새로운 버퍼 헤드에 복사한다. 이 작업 일부에서 kmap_atomic()을 사용할 수도 있기 때문에, 바운스 버퍼는 IRQ safe한 io_request_lock을 잡은 상태에서만 생성된다. IO 완료 콜백은 bounce_end_io_write() 또는 bounce_end_io_read()로 설정되며, 이는 이것이 읽기 또는 쓰기 버퍼인지에 따라 결정된다.

할당과 관련하여 알아두어야할 가장 중요한 점은 GFP flag가 high 메모리를 포함하는 IO 작업을 사용할 수 없음을 지정한다는 것이다. 이는 슬랩 할당자에는 SLAB_NOHIGHIO를, 버디 할당자에는 GFP_NOHIGHIO를 설정함으로써 명시된다. 이는 바운스 버퍼가 high 메모리를 포함하는 IO 작업에 사용되기 때문에 중요하다. 만약 할당자가 high 메모리 IO를 수행하게된다면, 이는 재귀 문제를 일으켜 결국에는 crash가 발생할 것이다.

### 9.4.3  Copying via bounce buffers

---
![](/images/gorman_번역/figure9.4.png "Figure 9.3: Call Graph: bounce_end_io_read/write()")

---

데이터는 바운스 버퍼가 읽기 버퍼인지 쓰기 버퍼인지에 따라 다르게 복사된다. 만약 장치에 쓰기 위한 버퍼인 경우, 버퍼는 바운스 버퍼 생성 과정에서 copy_from_high_bh()를 통해 high 메모리의 데이터로 채워진다. 콜백 함수 bounce_end_io_write()가 디바이스가 데이터에 준비가 됐을 때 IO를 완수할 것이다.

만약 버퍼가 장치에서 읽기 위해 사용되는 경우, 장치가 준비될 때까지 데이터 전송은 발생하지 않을 것이다. 준비가 되면, 장치의 인터럽트 핸들러가 콜백 함수인 bounce_end_io_read()를 호출하고, 이는 copy_to_high_bh_irq()를 호출하여 데이터를 high 메모리에 복사한다.

두 경우 모두, IO가 완수된 후에는 bounce_end_io()를 통해 버퍼 헤드와 페이지를 회수하며, 템플릿 버퍼 헤드를 위한 IO 완료 함수가 호출된다. 만약 긴급 풀에 자원이 충분치 않다면, 자원들은 풀에 추가되며, 그렇지 않다면 관련 할당자에게 반환된다.

## 9.5  Emergency Pools

바운스 버퍼가 빠르게 사용할 수 있도록 버퍼 헤드와 페이지의 두 긴급 풀이 관리된다. 메모리가 할당하기에 너무 빠듯하면, low 메모리가 이용가능해질 때까지 high 메모리의 버퍼가 해제될 수 없으므로 IO 요청의 실패가 상황을 복잡하게 만든다. 이는 프로세스들이 중단되도록 만들기 때문에, 그들 자신의 메모리를 해제할 가능성이 줄어들게 된다.

긴급 풀은 init_emergency_pool()에 의해 초기화되며, 현재는 32로 정의되어있는 POOL_SIZE개의 엔트리를 포함하게된다. 페이지들은 emergency_pages 리스트 헤드에 page->list로 연결되어 관리된다. Figure 9.5는 페이지들이 긴급 풀에 어떻게 저장되며, 획득되는 지를 나타낸다.

버퍼 헤드 또한 매우 유사하게 emergency_bhs 리스트헤드에 buffer_head->inode_buffers로 연결된다. 페이지와 버퍼 리스트에 남은 엔트리 수는 nr_emergency_pages와 nr_emergency_bhs의 두 카운터에 각각 기록되며, 두 리스트는 스핀락인 emergency_lock에 의해 보호된다.

---
![](/images/gorman_번역/figure9.5.png "Figure 9.5: Acquiring Pages from Emergency Pools")

---

## 9.6  What's New in 2.6

#### Memory Pools

2.4 버전에서는 high 메모리 매니저만이 페이지의 긴급 풀을 유지했다. 2.6 버전에서는 메모리 풀이 메모리가 부족할 때 최소한의 "자원"을 예약해야 할 때 사용되는 일반적인 개념으로 구현되었다. 여기서 "자원"은 high 메모리 매니저의 경우 페이지일 수도 있고, 더 자주 슬랩 할당자가 관리하는 어떤 객체일 수도 있다. 메모리 풀은 mempool_create()로 초기화되며, 여러 인자를 받습니다. 여기에는 예약해야 하는 최소 객체 수 (min_nr), 객체 타입에 대한 할당 함수 (alloc_fn()), 해제 함수 (free_fn()), 그리고 할당 및 해제 함수에 전달될 선택적 개인 데이터가 포함된다.

메모리 풀 API는 mempool_alloc_slab()과 mempool_free_slab()라는 두 가지 일반적인 할당 및 해제 함수를 제공한다. 이 일반적인 함수들이 사용될 때, 개인 데이터는 객체가 할당되고 해제될 슬랩 캐시이다.

High 메모리 매니저의 경우, 두 개의 페이지 풀이 생성된다. 하나는 일반 용도이고, 다른 하나는 ZONE_DMA에서 할당해야 하는 ISA 장치용이다. 할당 함수는 page_pool_alloc()이고, 전달된 개인 데이터 파라미터는 사용할 GFP 플래그를 나타낸다. 해제 함수는 page_pool_free()이다. 메모리 풀은 2.4 버전에 존재하던 긴급 풀 코드를 대체한다.

메모리 풀에서 객체를 할당하거나 해제하기 위해서는 메모리 풀 API 함수인 mempool_alloc()과 mempool_free()를 사용한다. 메모리 풀은 mempool_destroy()로 파괴된다.

#### Mapping High Memory Pages

2.4 버전에서는 page→virtual 필드를 사용하여 페이지의 주소를 pkmap_count 배열에 저장했다. High 메모리 시스템에서 존재하는 많은 struct 페이지들 때문에, ZONE_NORMAL에 매핑해야 하는 비교적 적은 수의 페이지들에 대해 이는 매우 큰 페널티였다. 2.6 버전은 여전히 pkmap_count 배열을 가지고 있지만, 이를 매우 다르게 관리한다.

2.6 버전에서는 page_address_htable이라는 해시 테이블이 생성된다. 이 테이블은 struct 페이지의 주소를 기반으로 해싱되며, 리스트는 struct page_address_slot을 찾는 데 사용된다. 이 구조체는 두 가지 중요한 필드를 가지고 있는데, 하나는 struct 페이지이고 다른 하나는 가상 주소이다. 커널이 매핑된 페이지에서 사용되는 가상 주소를 찾아야 할 때, 이 해시 버킷을 통해 찾아간다. 페이지가 실제로 low 메모리에 매핑되는 방식은 2.4 버전과 본질적으로 동일하지만, 이제는 page→virtual이 더 이상 필요하지 않다.

#### Performing IO

마지막 주요 변경 사항은 IO를 수행할 때 struct buffer_head 대신 struct bio가 사용된다는 것이다. bio 구조체가 어떻게 작동하는지는 이 책의 범위를 벗어난다. 그러나 bio 구조체가 도입된 주된 이유는 IO가 기본 장치가 지원하는 크기의 블록 단위로 수행될 수 있도록 하기 위함이다. 2.4 버전에서는 모든 IO가 기본 장치의 전송 속도와 관계없이 페이지 크기의 청크로 분할되어야 했다.

