---
title: "Chapter 6 Physical Page Allocation"
date: 2024-05-18T02:10:54+09:00
draft: false
author: "Minho Kim"
categories: ["Gorman's book Translation"]
---

이 글은 Gorman 책 "Understanding the Linux Virtual Memory Manager"의 [Chapter 6](https://www.kernel.org/doc/gorman/html/understand/understand009.html)를 번역한 글입니다.

---

이 챕터에서는 리눅스에서 물리 페이지들이 어떻게 관리되고 할당되는지를 설명한다. 이에 관련하여 사용되는 주요 알고리즘은 Binary Buddy Allocator로, Knowlton이 고안했고, Knuth가 발전시켰다. 이 알고리즘은 다른 할당자들과 비교하였을 때 굉장히 빠른 성능을 보여준다.


