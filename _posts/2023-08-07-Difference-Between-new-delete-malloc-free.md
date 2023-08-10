---
layout: post
title: C++ new, delete와 malloc, free의 차이
published: true
date: 2023-08-07 23:09:00
category: programming
tags: [programming, c, c++]
author: altair
toc: false
comments: false
---

C++에서는 `malloc`, `new`, `calloc`을 통해 동적 메모리 할당이 가능합니다.
반대로 해제할 때에는 `free`, `delete`를 통해 해제가 가능하고요.

동적 메모리 할당은 이름 그대로 프로그램이 실행되어 있는 시간, 즉 런타임 동안 사용할 메모리 공간을 할당하는 것입니다.

`malloc`, `free`, `new`, `delete`의 차이점은 다음과 같습니다:

| malloc(calloc), free 	| new, delete 	|
|---	|---	|
| 함수(function) 	| 연산자(operator) 	|
| C/C++에서 사용 가능(라이브러리에서 제공) 	| C++에서만 사용 가능(언어의 기능으로서 제공됨) 	|
| 지정한 크기의 메모리가 할당됨 	| 객체 생성을 위한 메모리가 할당됨 	|
| 반환값 형변환 필요 	| 해당 타입의 포인터를 반환(형변환 필요 없음) 	|
| 에러 발생시 null 반환됨 	| 에러 발생시 exception throw 발생 (null 반환 없음) 	|
| 함수이기 때문에 오버로딩 불가능 	| 연산자라 오버로딩이 가능함 	|
| 별도의 코드 호출 없음 	| 생성자, 소멸자 호출됨 	|

위 표는 대략적인 차이점을 정리한 내용입니다. 이번에는 조금 더 자세히 살펴봅시다.

### `malloc`과 `new`의 호출 방식

앞서 `malloc`은 필요한 크기의 메모리를 매개변수로, `new`는 할당할 타입을 지정하고 지정한 타입의 포인터를 반환합니다.

``` cpp
#include <stdlib.h>
#include <stdio.h>

int main()
{
    // int 4개 만큼의 메모리 할당
    int *indices = (int*)malloc(sizeof(int) * 4);

    // 배열 요소들 초기화
    indices[0] = 0;
    indices[1] = 1;
    indices[2] = 2;
    indices[3] = 3;
    
    for (int i = 0; i < sizeof(indices); i++)
    {
        printf("%d\n", indices[i]);
    }

    // indices 메모리 해제
    free(indices);
}
```

위 코드는 `malloc`/`free`를 사용해 길이가 4인 배열을 할당 및 해제하는 예제입니다.

보이듯이 배열 선언에 필요한 만큼의 메모리 크기를 매개변수로 넘겨주고 있습니다.

``` cpp
#include <stdlib.h>
#include <stdio.h>

int main()
{
    // int 4개 만큼의 메모리 할당
    int *indices = new int[4];

    indices[0] = 0;
    indices[1] = 1;
    indices[2] = 2;
    indices[3] = 3;
    
    for (int i = 0; i < sizeof(indices); i++)
    {
        printf("%d\n", indices[i]);
    }
    
    delete [] indices;
}
```

이번에는 `new`/`delete`를 이용해 배열을 할당/해제하는 예제입니다.

메모리를 할당해줄 때, 별도의 지정한 타입의 포인터가 반환되어 별도의 형변환이 필요없다는것을 확인할 수 있습니다.

## 생성자/소멸자의 호출 여부 차이

`new`는 `malloc`과 다르게, 호출과 동시에 초기화가 가능합니다.
``` cpp
string* str = new string("Test");
```

생성자와 소멸자가 호출이 된다는걸 좀 더 명시적으로 확인해볼까요:
``` cpp
#include <iostream>

class Vector
{
public:
    Vector(float x, float y) : m_X(x), m_Y(y)
    {
        std::cout << "constructed: Vector(" << m_X << ", " << m_Y << ")" << std::endl;
    }
    
    ~Vector()
    {
        std::cout << "destructed: Vector(" << m_X << ", " << m_Y << ")" << std::endl;
    }
    
private:
    float m_X;
    float m_Y;
};

int main()
{
    Vector* vector = new Vector(5, 4);
    delete vector;
}

// Output:
//  constructed: Vector(5, 4)
//  destructed: Vector(5, 4)
```

## 할당된 메모리 크기의 조정 가능 여부

위의 표에서는 언급하지 않았지만, `malloc`으로 할당된 메모리는 `realloc`을 통해 할당된 메모리의 크기를 변경할 수 있습니다.

반면 `new`로 할당된 메모리는 처음 할당한 이후에는 크기 변경이 안됩니다. 새로 메모리를 할당하고, 값을 복사해주고 원래의 할당된 메모리를 해제시켜줘야합니다.

메모리 할당/해제의 빈도에 따라 `malloc`을 사용할지, `new`를 사용할지 판단은 필요합니다.

## 마치며

C#에서는 Garbage Collector를 통해 프로그래머가 따로 메모리를 관리해주지 않아도 알아서 메모리를 관리해주니 크게 문제가 없습니다.

하지만 C/C++에서는 프로그래머가 직접 할당을 해제해주어야 메모리 누수 같은 메모리 관련 문제를 예방할 수 있습니다.
