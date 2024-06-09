---
title: "Chapter 7  Non-Contiguous Memory Allocation"
date: 2024-05-29T02:10:54+09:00
draft: false
author: "Minho Kim"
categories: ["Gorman Book Translation"]
categories_weight: 7
---

이 글은 Gorman 책 "Understanding the Linux Virtual Memory Manager"의 [Chapter 7](https://www.kernel.org/doc/gorman/html/understand/understand010.html)를 번역한 글입니다.

---

많은 양의 메모리를 물리적으로 연속된 페이지로 할당하는 것은 캐시와 메모리 접근 시간 관점에서 더 좋다. 그러나 불행하게도, 버디 할당자의 외부 단편화 문제로 인해, 이것이 항상 가능하지는 않다. 리눅스는 연속적이지 않은 물리 메모리를 연속된 가상 메모리에 사용할 수 있도록하는 메커니즘인 vmalloc()을 제공한다.

VMALLOC_START와 VMALLOC_END 사이의 가상 주소 공간이 예약된다. VMALLOC_START의 위치는 이용가능한 물리 메모리의 양에 따라 결정되지만, 영역의 크기는 최소한 VMALLOC_RESERVE 크기 (x86의 경우 128MiB) 이상으로 설정된다. 해당 영역의 정확한 크기에 대해서는 섹션 4.1에서 설명한다.

이 영역의 페이지 테이블은 필요한 경우 일반 물리 페이지 할당자를 사용하여 할당된 물리 페이지를 가리키도록 조정된다. 이는 할당이 페이지 크기의 배수여야 함을 의미한다. 할당 시 커널 페이지 테이블을 변경해야 하기 때문에, vmalloc()으로 매핑할 수 있는 메모리 양에는 제한이 있다. 이는 VMALLOC_START와 VMALLOC_END 사이의 가상 주소 공간만 사용할 수 있기 때문이다. 결과적으로, 코어 커널에서는 이를 제한적으로 사용한다. 2.4.22 버전에서는 스왑 맵 정보를 저장하고 (11장에서 설명) 커널 모듈을 메모리에 로드하는 데만 사용된다.

이번 장은 내용이 적다. 먼저 커널이 vmalloc 주소 공간의 어떤 영역이 사용중인 지를 어떻게 추적하는지 설명하고, 영역들이 어떻게 할당되고 해제되는 지를 설명한다.

## 7.1 Describing Virtual Memory Areas

vmalloc 주소 공간은 resource map allocator로 관리된다. struct vm_struct가 base, size 쌍들을 저장하는데 사용된다. 이것은 <linux/vmalloc.h>에 다음과 같이 선언되어있다:
```c
 14 struct vm_struct {
 15         unsigned long flags;
 16         void * addr;
 17         unsigned long size;
 18         struct vm_struct * next;
 19 };
```

VMA를 그대로 사용할 수 있겠지만, 이는 vmalloc 영역에 적용되지 않는 추가 정보를 포함하고 있어 낭비가 될 것이다. 여기 이 작은 구조체의 필드에 대한 간략한 설명이 있다:
- flags: 이는 vmalloc()과 함께 사용할 경우 VM_ALLOC으로 설정되고, high memory를 커널 가상 주소 공간에 매핑하기 위해 ioremap이 사용될 때는 VM_IOREMAP으로 설정된다.
- addr: 이는 메모리 블록의 시작 주소이다.
- size: 이는 예측 가능한 대로, 바이트 단위의 크기이다.
- next: 이는 다음 vm_struct에 대한 포인터이다. 이들은 주소에 따라 정렬되며, 리스트는 vmlist_lock 락에 의해 보호된다.

영역들은 next 필드를 통해 서로 연결되며, 간단한 검색을 위해 주소 순서대로 정렬된다. 각 영역은 overrun을 방지하기 위해 최소 한 페이지씩 분리되어 있다. 이는 Figure 7.1의 gap들로 나타나있다.

---
![](/images/gorman_번역/figure7.1.png "Figure 7.1: vmalloc Address Space")

---
커널이 새로운 영역을 할당하고 싶은 경우, vm_struct 리스트를 get_vm_area() 함수에서 선형 탐색한다. 구조체를 위한 공간은 kmalloc()을 통해 할당된다. 가상 영역이 IO를 위한 영역을 재매핑 (보통 ioremapping이라고 함)에 사용되는 경우, 이 함수는 요청된 영역을 매핑하기위해 직접적으로 호출된다.

## 7.2 Allocating A Non-Contiguous Area

---
![](/images/gorman_번역/figure7.2.png "Figure 7.2: Call Graph: vmalloc()")

---

가상 주소 공간에 연속된 메모리 영역을 할당하기위해 vmalloc(), vmalloc_dma(), 그리고 vmalloc_32() 함수가 제공된다. 이들은 모두 하나의 인자로 size를 인자로 받으며, 이것은 다음 페이지 단위로 align되도록 올림된다. 이들은 모두 새로 할당된 영역의 선형 주소를 반환한다.

---
{{< table title="Table 7.1: Non-Contiguous Memory Allocation API" >}}
| |
| --- |
| void * vmalloc(unsigned long size) |
| 요청된 사이즈를 만족할 수 있도록 페이지를 vmalloc 공간에서 할당한다. |
| void * vmalloc_dma(unsigned long size) |
| ZONE_DMA에서 페이지들을 할당한다. |
| void * vmalloc_32(unsigned long size) |
| 32 비트 주소에 적합한 메모리를 할당한다. 32비트 장치를 위해, 물리 페이지 프레임이 ZONE_NORMAL에 있도록 보장한다. |
{{< /table >}}

---
Figure 7.2에서 분명히 알 수 있듯이, 할당에는 두 단계가 있다. 첫 번째 단계는 get_vm_area()로, 요청을 수행하기에 충분히 큰 영역을 찾는다. 이것은 vm_struct들의 선형 연결 리스트를 탐색하여 할당된 영역을 나타내는 새로운 구조체를 반환한다.

두 번째 단계는 필요한 PGD entries, PMD entries, PTE entries, 그리고 page를 각각 vmalloc_area_pages(), alloc_area_pmd(), alloc_area_pte(), 그리고 alloc_page()로 순서대로 할당한다.

vmalloc()은 현재 프로세스가 아니라, init_mm->pgd에 저장되어있는 참조 페이지 테이블을 업데이트한다. 이것은 vmalloc 영역을 프로세스가 접근할 때, 해당 페이지 테이블이 제대로 된 매핑을 저장하고 있지 않기때문에, 페이지 폴트가 발생함을 의미한다. 이러한 경우를 처리하는 페이지 폴트 코드가 있으며, 이는 폴트가 vmalloc area에서 발생했음을 감지하고, master 페이지 테이블의 정보를 이용하여 현재 프로세스의 페이지 테이블을 업데이트한다. vmalloc()이 버디 할당자와 페이지 폴트에 어떻게 연결되는 지가 Figure 7.3에 나와있다.

---
![](/images/gorman_번역/figure7.3.png "Figure 7.3: Relationship between vmalloc(), alloc_page() and Page Faulting")

---

## 7.3 Freeing A Non-Contiguous Area

vfree()가 가상 영역의 해제를 책임진다. 이것은 vm_struct의 리스트를 탐색하여 원하는 영역을 찾은 다음, 해제할 메모리 영역에 대해 vmfree_area_pages()를 호출한다.

---
![](/images/gorman_번역/figure7.4.png "Figure 7.4: Call Graph: vfree()")

---

vmfree_area_pages()는 vmalloc_area_pages()와 정확히 반대된다. 이는 페이지 테이블을 walking하면서 영역과 관련된 페이지 테이블 엔트리들과 페이지들을 해제한다.

---
{{< table title="Table 7.2: Non-Contiguous Memory Free API" >}}
| |
| --- |
| void * vfree(void *addr) |
| vmalloc(), vmalloc_dma(), 또는 vmalloc_32()를 통해 할당된 메모리 영역을 해제한다. |
{{< /table >}}

---

## 7.4 Whats New in 2.6

비연속 메모리 할당은 2.6에서도 본질적으로 동일하다. 주요 차이점은 페이지가 할당되는 시점에 영향을 미치는 약간 다른 내부 API이다. 2.4에서는 vmalloc_area_pages()가 페이지 테이블 워크를 시작하고, alloc_area_pte() 함수에서 PTE에 도달할 때 페이지를 할당한다. 2.6에서는 __vmalloc()이 모든 페이지를 미리 할당하고, 이 페이지들을 배열에 배치하여 map_vm_area()에 전달하여 커널 페이지 테이블에 삽입한다.

get_vm_area() API는 아주 약간 변경되었다. 호출되었을 때, 이전과 동일하게 vmalloc 가상 주소 공간 전체를 검색하여 빈 영역을 찾는다. 그러나 호출자는 __get_vm_area()를 직접 호출하여 범위를 지정함으로써 vmalloc 주소 공간의 하위 집합만 검색할 수 있다. 이는 모듈을 로드할 때 ARM 아키텍처에서만 사용된다.

마지막으로 중요한 변경 사항은 vmap()이라는 새로운 인터페이스의 도입이다. 이는 vmalloc 주소 공간에 페이지 배열을 삽입하는데 사용되며, 오직 사운드 서브시스템 코어에서만 사용된다. 이 인터페이스는 2.4.22로 백포트되었지만 전혀 사용되지 않는다. 이는 우연한 백포트의 결과이거나, vmap()을 필요로 하는 공급업체별 패치의 적용을 용이하게 하기 위해 병합된 것이다.
