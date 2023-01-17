---
title: "ArrayList"
date: 2018-12-20T09:49:03+10:00
layout: post
categories: ["ArrayList", "Java11", "STL"]
description: Java11의 ArrayList의 내부 구현을 코드를 보며 직접 분석해봅시다. ArrayList에 사용된 최적화 기법과 설계에 대해서 코드를 보며 분석합니다.
---

# ArrayList

ArrayList는 동적 배열 자료구조로 랜덤 액세스 가능한 리스트 데이터 구조를 구현화한 Java의 대표적인 STL 중 하나이다.

동적 배열의 경우 정적 배열과 다르게, 배열의 크기가 동적으로 조정되기에 동적 배열 사용자는 배열의 크기를 신경쓰지 않으며,배열을 유연하게 사용하는 것이 가능합니다.

## 동적 배열과 capacity

동적 배열은 사용하고자 하는 배열보다 큰 고정된 배열을 사용하여 구현할 수 있습니다. 동적 배열의 원소들은 순차적으로 저장 되며, 아직 사용되지 않은 공간은 이후 들어올 수 원소를 위해 비워둡니다. 배열에 원소를 append 하는 동작은 기존 배열이 완전히 차기 전까지 상수 시간으로 작동합니다. 만약 기존의 배열이 완전하게 차버린다면, 기존 배열의 크기를 적절하게 증가시킵니다. 배열의 크기를 증가시키는 동작은 새로운 배열을 생성하고 기존 배열 내용을 복사시켜야하므로 많은 시간 자원을 필요로 합니다. 배열의 크기를 증가시키는 동작을 resizing 라고 합니다.

> 실제 배열 내부에 들어있는 원소의 갯수를 size라고 표현하며 <br>
> 현재 할당되어 있는 고정된 배열의 크기를 capacity라고 합니다.

## resizing과 Amortized 시간 복잡도

여러번의 resizing 동작을 호출하지 않기 위하여, 동적 배열은 resizing 동작에서 배열 크기를 한번에 매우 크게 증가시킵니다.

```
function insertEnd(dynarray a, element e)
    if (a.size == a.capacity)
        // resize a to twice its current capacity:
        a.capacity ← a.capacity * 2
        // (copy the contents to the new memory location here)
    a[a.size] ← e
    a.size ← a.size + 1
```

`n` 개의 원소가 append 되었을 때, 동적 배열의 capacity는 등비 수열을 이룹니다.
배열의 크기는 일정한 크기로 증가하므로, 현재 capacity가 n인 동적배열에 n개의 원소를 삽입하기 위한 **전체** 시간 복잡도가 `O(n)`이 됩니다. 따라서 각 append 동작의 [amortized](https://ko.wikipedia.org/wiki/분할상환분석)한 시간 복잡도는 `O(1)`입니다. 또한 많은 동적 배열 구현에서는 특정 역치값을 기준으로 필요 없는 잉여 공간을 할당-해제하기도 합니다. 이때 [hysteresis](https://en.wikipedia.org/wiki/Hysteresis)를 방지하기 위해, 할당-해제 역치값을 resizing growth factor `1/a` 보다 작지 않도록 해야합니다.

# Java 11 ArrayList 분석

## Constructor

```java
    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = {};
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    transient Object[] elementData;
    private int size;

    public MyArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    public MyArrayList(int initialCapacity) {
        if(initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if(initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
        }
    }
```

ArrayList에서 동적 배열에 들어있는 원소들은 `elementData` 필드에 저장됩니다.

`EMPTY_ELEMENTDATA`는 capacity가 `0` 인 인스턴스를 나타내기 위한 static 필드입니다.
만약 크기가 `0` 인 동적 배열을 생성하게 되면, `elementData`는 `EMPTY_ELEMENTDATA`를 참조하게 됩니다. 이는 불필요하게 메모리를 낭비하지 않게하기 위해 새로 인스턴스를 생성하지 않고, static 배열을 참조하도록 설계된 것입니다.

`DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 는 `DEFAULT_CAPACITY` 로 생성된 아직 원소가 들어가기 전 동적 배열을 나타내는 static 필드입니다. 이는 메모리 할당을 동적 배열에 실제 원소가 들어오기 전까지 최대한 미룸으로써 메모리를 효율적으로 사용하도록 설계한 것입니다. 따라서 `DEFAULT_CAPACITY`로 선언만하고 아직 원소를 넣지 않은 ArrayList의 `capacity(elementData.length)` 값은 `0`으로 나타납니다(아직 메모리를 할당하기 전이므로 elementData는 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`를 참조함) [참고](https://stackoverflow.com/questions/34250207/in-java-8-why-is-the-default-capacity-of-arraylist-now-zero)

## Resizing

```java
    // 배열의 크기를 max(minCapacity, 현재 배열 크기 * 1.5)로 resize
    private Object[] grow(int minCapacity) {
        return Arrays.copyOf(elementData, newCapacity(minCapacity));
    }

    // 오버플로우와 DEFAULTCAPACITY_EMPTY_ELEMENTDATA를 고려하여 max(minCapacity, 현재 배열 크기 * 1.5) 계산
    private int newCapacity(int minCapacity) {
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if(newCapacity <= minCapacity) {
            if(elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
                return Math.max(DEFAULT_CAPACITY, minCapacity);
            if(minCapacity < 0) // overflow
                throw new OutOfMemoryError();
            return minCapacity;
        }
        return (newCapacity - MAX_ARRAY_SIZE <= 0)
                ? newCapacity
                : hugeCapacity(minCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE)
                ? Integer.MAX_VALUE
                : MAX_ARRAY_SIZE;
    }
```

만약 동적 배열의 크기를 resize할 필요가 생긴다면, `grow` 함수를 사용하여 배열을 resize 할 수 있습니다. `grow` 함수의 매개변수 `minCapacity`는 resize된 새로운 배열의 크기가 적어도 `minCapacity`보다 크게끔 보장하여 줍니다.

기본적으로 현재 배열의 `1.5`배 큰 새로운 크기의 배열로 resize 되므로 ArrayList의 resize growth rate = `1.5`가 됩니다.

## get, add, remove

```java
    private Object[] grow(int minCapacity) {
        return Arrays.copyOf(elementData, newCapacity(minCapacity));
    }

    private Object[] grow() {
        return grow(size + 1);
    }

    private void add(E e, Object[] elementData, int s) {
        if(s == elementData.length)
            elementData = grow();
        elementData[s] = e;
        size += 1;
    }

    public boolean add(E e) {
        add(e, elementData, size);
        return true;
    }

    public E remove(int index) {
        Objects.checkIndex(index, size);
        final Object[] es = elementData;

        @SuppressWarnings("unchecked") E oldValue = (E) es[index];
        fastRemove(es, index);

        return oldValue;
    }

    private void fastRemove(Object[] es, int i) {
        final int newSize;
        if((newSize = size - 1) > i)
            System.arraycopy(es, i + 1, es, i, newSize - i);
        es[size = newSize] = null;
    }
```

`add` 함수는 ArrayList에 맨 뒤에 원소를 하나 추가해줍니다. 특이하게도 ArrayList의 구현 코드를 보게 되면, 함수가 두개로 나뉘어서 작성된 것을 알 수가 있는데, 이는 public으로 공개된 `add(E)` 함수가 c1 compiled-loop 에서 호출 되기 위한 조건(bytecode가 35 이하)을 지키기 위해 분리되어 있는 것입니다. c1-compile은 JVM의 코드 최적화 방법과 관련된 내용인데, 단순하게 설명하자면 자주 사용되는 코드를 캐시에 올려 빠르게 사용될 수 있게끔 하는 것입니다. (자세한 내용은 다른 포스팅에서 다뤄보도록 하겠습니다)

`remove` 함수는 ArrayList에서 특정 인덱스에 위치하는 원소를 삭제하는 동작을 합니다. 이때 해당 동작은 인덱스 이후의 원소들을 한칸씩 앞으로 이동하는 연산을 수행하기 때문에, 해당 함수의 시간복잡도는 원소의 `index`의 값과 상관 없이 `O(n)`으로 동작할 것 같습니다. 하지만 실제 내부 코드를 들여다보면, `newSize > i`일 경우에만 부분 배열을 복사-이동 시키기 때문에, 마지막 원소를 `remove`하기 위한 시간 복잡도는 `O(1)`이 맞습니다.

> ⚠️ Java11 API 주석 내용에 누락되어 가끔 책이나 기술 블로그에 오개념을 적어두는 경우가 많으니 조심하시길 바랍니다.

ArrayList의 경우 원소를 remove 하더라도 별도로 capacity를 줄이는 메소드가 존재하지 않으므로, 메모리가 낭비되게 됩니다. 따라서 개발자가 직접 `trimToSize` 같은 함수를 사용하여 메모리를 **직접 관리**해야합니다.

```java
    public void trimToSize() {
        if (size < elementData.length) {
            elementData = (size == 0)
                    ? EMPTY_ELEMENTDATA
                    : Arrays.copyOf(elementData, size);
        }
    }
```

## 멀티스레딩 환경에서 ArrayList 사용

ArrayList는 멀티-쓰레드 환경에서 안전하지 않습니다.

# 결론

이번 포스팅에서 다뤄진 ArrayList의 구현은 핵심적인 내용만 다뤄 보았습니다. 실제 STL 내부에는 Serializer, AbstractList, Iterator, Collection 등 여러 요소들의 집합으로 이뤄져 있습니다. Bottom-Up 접근 방식으로 Java STL의 핵심적인 자료구조가 어떤식으로 작동하는지 천천히 모두 분석 해보도록 하겠습니다.

감사합니다.

## Reference

- [Oracle JAVA11 Documentation](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/ArrayList.html)
- [Wikipedia Dynamic Array](https://en.wikipedia.org/wiki/Dynamic_array)
