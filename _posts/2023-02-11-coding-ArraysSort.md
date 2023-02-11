---
title: "[Java] Arrays.sort()"
layout: post
categories: coding
tags: theory
comments: false   
---
Arrays.sort()는 배열을 정렬할 수 있는 방법 중 하나로 API 문서를 살펴보면 아래와 같이 여러 타입의 정렬이 가능하다.    

<center><img src="https://user-images.githubusercontent.com/68698007/218243185-515697d6-41e2-410f-8184-669ee9f185ab.png" width="60%" height="60%"></center>

내가 궁금했던거는 Arrays.sort의 효율, 성능이다. 어떤 상황에 써야되는건지를 알고싶었다.    

<br>

### primitive 타입의 배열인 경우
이때 Arrays.sort()는 내부적으로 DualPivotQuickSort를 사용한다. QuickSorts는 최악의 경우 __O(N^2)__ , 평균적으로 __O(NlogN)__ 의 시간 복잡도를 가지고 DualPivotQuickSort의 경우에는 이보다 더 나은 시간 복잡도를 보장하나 __최악의 경우 O(N^2)__ 을 가진다.    

따라서 정렬 문제에서 이와 같은 시간 복잡도를 보장하는 코드가 필요하다면 기본형 타입의 배열을 Arrays.sort()로 정렬해서는 안된다.    

### reference 타입의 배열인 경우
이 경우에는 TimSort를 사용한다.    
삽입 정렬과 병합 정렬의 장점을 가지고 있어 최선의 경우에는 O(N), 최악의 경우 O(NlogN)을 갖는다.