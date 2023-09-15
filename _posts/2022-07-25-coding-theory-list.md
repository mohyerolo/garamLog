---
title: List, ArrayList, LinkedList
layout: post
categories: coding
tags: theory
---
## List
Interface는 다른 클래스를 작성할 때 기본이 되는 틀을 제공하면서, 다른 클래스 사이의 중간 매개 역할까지 담당하는 일종의 추상 클래스이다.    
List는 인터페이스로 리스트를 구현한  <mark style='background-color: #fff5b1'>ArrayList, LinkedList, Vector, Stack</mark>이 있다.

보통 리스트와 배열의 차이점을 물어보는 것 같다.    
<hr>

### [배열]
배열은 자료형의 집합을 의미한다. String, int, Integer와 같은 타입들의 묶음을 의미하며 배열을 만들기위해서는 배열의 길이를 알아야한다. 따라서 초기값 없이 배열 변수를 만들때에는 반드시 __길이에 대한 숫자값__ 이 필요하다.

```java
int[] arr = new int[5];
int[] arr = {1, 2, 3, 4};
int[] arr = new int[];
```
즉, 마지막 방법은 불가능한 방법으로 'Variable must provide either dimension expressions or an array initializer'이라는 오류가 뜬다.    

이렇게 선언한 배열의 크기는 변경할 수 없고 이것은 정적 할당(static allocation)이라고 한다. 배열의 원소는 인덱스 값으로 접근이 가능하고 메모리에 연속적으로 나열되어 할당되므로 인덱스에 위치한 원소를 삭제하더라고 해당 인덱스는 빈공간으로 남아있는다. 따라서 수정, 삭제, 중간에 추가와 같은 작없은 해당 인덱스뿐만이 아닌 그 뒤의 인덱스마저 추가로 수정이 필요하다.    

<요약>   
- 배열의 크기 변경 불가
- 인덱스로 접근하므로 탐색이 빠름
- 데이터를 수정, 삭제, 추가하는 과정에서는 추가적인 작업 필요
- 하나의 데이터를 삭제하더라도 해당 인덱스는 빈공간으로 남아있음

### [리스트]
리스트는 배열과 달리 선언할때 크기, 길이가 필요하지 않다. 가변적이므로 이를 동적 할당(dynamic allocation)이라고 한다.    
또한 메모리에 나열되는 것이 아닌 각 데이터들은 주소로 연결되어 있고 원소들 사이에 빈 공간을 허용하지 않는다. 

그러나 적은 양의 데이터를 사용할 경우에는 주소 연결에 필요한 추가적인 자리가 들어가는 리스트보다 메모리상으로 배열이 훨씬 효과적일 수 있고 수정과 삽입은 배열보다 간단해도 탐색은 배열보다 성능이 떨어진다.    

그렇다면 List, ArrayList, LinkedList중에서 우리는 어떤 List를 사용해야되는걸까.    

### [ArrayList]
ArrayList는 내부적으로 데이터를 배열에서 관리하며 데이터의 추가, 삭제를 위해 임시 배열을 생성해 데이터를 복사하는 방법을 사용하고 있다.    
어떻게 배열과 리스트가 함께 있는지 의문이 들었다.    

<https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/ArrayList.html>    
_Each ArrayList instance has a capacity. The capacity is the size of the array used to store the elements in the list. It is always at least as large as the list size. As elements are added to an ArrayList, its capacity grows automatically. The details of the growth policy are not specified beyond the fact that adding an element has constant amortized time cost._   
각 ArrayList 인스턴스에는 정해진 용량이 있고 이는 배열의 크기라고 한다. 또한 항상 목록의 크기보다 크게 책정되고 새로운 원소가 추가되면 자동으로 증가한다고 한다.    

일반 배열과 동일하게 연속된 메모리 공간을 사용하며 인덱스는 0부터 시작하지만 배열과 다른 점은 크기가 가변적으로 변한다는 점이다. 즉,  위의 설명처럼 새로운 원소가 추가되면 자동으로 증가한다는 것이다.    
ArrayList.class를 살펴보면 
```java
private Object[] grow(int minCapacity) {
    return elementData = Arrays.copyOf(elementData,
        newCapacity(minCapacity));
}

private Object[] grow() {
    return grow(size + 1);
}

private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
}
```
위와 같은 작업을 통해 ArrayList에 원소를 추가하는 작업이 진행된다.    
크기가 가변적이라는 것은 결국 크기를 늘릴 수 있다는 것이고 그 과정은 Arrays.copyOf로 진행되는 걸 볼 수 있다. 즉, 용량이 꽉 찼을 경우 더 큰 용량의 배열을 만들어 옮긴다는 것이다.    

복사를 통해 데이터의 추가와 삭제가 이루어지므로 대량의 자료의 경우에는 그만큼의 데이터 복사가 많이 일어나게 되어 성능 저하를 일으킬 수 있다. 그러나 각 데이터가 인덱스를 가지고 있다는 점에서 데이터의 검색에는 유리하다고 볼 수 있다.

### [LinkedList]
LinkedList는 내부적으로 양방향 연결 리스트로 구성되어 참조하려는 원소에 따라 처음부터 순방향으로 또는 역순으로 순회할 수 있다.

LinkedList는 인덱스를 가지고 있지 않아 데이터의 검색에는 성능상 불리하나 ArrayList처럼 데이터의 복사는 없으므로 데이터의 추가, 삭제에는 유리하다.    

<hr>
이처럼 처리하고자 하는 데이터에 따라서 어떤 자료구조를 선택할지를 잘 판단하는 게 중요하다. 

그렇다면 List로 객체를 선언하는 것과 ArrayList로 선언하는 것의 차이점은 무엇이 있을까?    

```java
List<Object> list = new ArrayList<>();
ArrayList<Object> list = new ArrayList<>();
```    
위의 두 코드의 차이점은 유연성에서 나온다. 특정 타입을 미리 지정해주는 것이 아닌 필요에 의해 지정할 수 있어지므로 내부 디테일과 메모리 함축에서 이점과 성능을 개선시킬 수 있고 다형성에도 이점을 갖는다.
