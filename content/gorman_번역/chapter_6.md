---
title: "Chapter 6 Physical Page Allocation"
date: 2024-05-18T02:10:54+09:00
draft: true
author: "Minho Kim"
categories: ["Gorman Book Translation"]
categories_weight: 6
---

이 글은 Gorman 책 "Understanding the Linux Virtual Memory Manager"의 [Chapter 6](https://www.kernel.org/doc/gorman/html/understand/understand009.html)를 번역한 글입니다.

---

이 챕터에서는 리눅스에서 물리 페이지들이 어떻게 관리되고 할당되는지를 설명한다. 이에 관련하여 사용되는 주요 알고리즘은 Binary Buddy Allocator로, Knowlton이 고안했고, Knuth가 발전시켰다. 이 알고리즘은 다른 할당자들과 비교하였을 때 굉장히 빠른 성능을 보여준다.

이는 일반적인 2의 거듭제곱 할당자에 free 버퍼 합체 기술을 결합한 할당 scheme이며, 기본 컨셉은 꽤 간단하다. 메모리는 2의 거듭제곱의 페이지로 구성된 큰 페이지 블록들로 나누어진다. 만약 원하는 크기의 블록이 없는 경우, 큰 블록은 서로에게 buddy인 두 개의 블록으로 반으로 쪼개어진다. 이 중 하나는 할당에 사용되고, 나머지는 free 상태가 된다. 필요한 경우, 원하는 크기의 블록이 생길 때까지 이를 반복한다. 블록이 해제되는 경우, 버디를 
