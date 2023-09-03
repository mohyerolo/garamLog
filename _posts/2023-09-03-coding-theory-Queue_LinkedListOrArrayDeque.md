---
title: Queue 구현 시 LinkedList, ArrayDeque
layout: post
categories: coding
tags: theory
---
백준에서 bfs 문제를 풀 때 처음 알게 된 풀이는 Queue를 LinkedList로 구현하는 것이었다. 무지하게도 그런 이유 하나로 당연하게 큐는 LinkedList로 구현하게 됐고 백준의 1697 숨바꼭질 또한 LinkedList로 큐를 구현해서 풀었다.    
```java
private static Queue<Temp> queue = new LinkedList<Temp>();
```

그런데 다른 풀이를 보니 LinkedList가 아닌 ArrayDeque를 이용해서 큐를 구현한 것을 발견했다.    

ArrayDeque라는 거를 처음 봐서 그거에 대해 찾아봤고 ArrayDeque와 LinkedList의 차이점을 알아봤다.

<https://tecoble.techcourse.co.kr/post/2021-05-10-stack-vs-deque/>

<https://velog.io/@newdana01/Java-%ED%81%90-%EA%B5%AC%ED%98%84%EC%8B%9C-ArrayDeque%EC%99%80-LinkedList-%EC%84%B1%EB%8A%A5-%EC%B0%A8%EC%9D%B4-Deque-Queue-%EC%9D%B8%ED%84%B0%ED%8E%98%EC%9D%B4%EC%8A%A4>

<img src="https://github.com/mohyerolo/mohyerolo.github.io/assets/68698007/a7d0c698-51ed-42b5-918d-25c7b64a40cd">

그 후 같은 문제를 ArrayDeque로 큐를 구현해 풀어봤고 시간은 똑같으나 메모리 사용이 현저히 줄어든 것을 확인했다.     
```java
private static Queue<Temp> queue = new ArrayDeque<Temp>();
```
<img src="https://github.com/mohyerolo/mohyerolo.github.io/assets/68698007/53206c3e-c201-4247-9c17-d7def744c061">
