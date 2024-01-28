---
layout: post
title: 피보나치 수열과 피사노 주기
published: true
category: math
tags: [math, number-theory]
author: altair
toc: false
comments: false
---

최근 DSA 공부를 위해서 여러가지 문제를 풀어보고 있습니다.

그중에 ***피보나치 수열(Fibonacci Sequence)***에 대한 문제를 풀던 중 **피사노 주기(Pisano Period)**라는 개념이 등장했습니다.

상당히 흥미로운 주제라 정리해볼겸 글을 써봤습니다.

## 피보나치 수열과 피사노 주기

우선 *피보나치 수열*부터 다시 정리해보겠습니다:  
*피보나치 수열*은 $F(n) = F(n-1) + F(n-2)\quad(단, n \geq 2)$를 식으로 가지는 수열입니다.  
$n=15$ 일 때 수열의 형태는 $0, 1, 1, 2, 3, 5, 8, \cdots 144, 233, 377$으로 나타낼 수 있고, $F(10) = 377$입니다.

**피사노 주기**의 개념은 `피보나치 수를 어떤 수 M으로 나눌 때, 그 나머지는 항상 주기를 가지게 된다.`라는 것입니다.  
한번 직접 대입을 해서 확인해 봅시다.  

이번에는 크기를 늘려 $n = 20$ 일 때를 확인해봅시다.
| n | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7  | 8  | 9  | 10 | 11 | 12  | 13  | 14  | 15  | 16  | 17   | 18   | 919  | 20   |
|--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |
| F(n) | 0 | 1 | 1 | 2 | 3 | 5 | 8 | 13 | 21 | 34 | 55 | 89 | 144 | 233 | 377 | 610 | 987 | 1597 | 2584 | 4181 | 6765 |

그리고 어떤 수 $M = 4$를 각 수열 요소에 대해 나누어보면 다음과 같이 나옵니다:
| n    | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7  | 8  | 9  | 10 | 11 | 12  | 13  | 14  | 15  | 16  | 17   | 18   | 919  | 20   |
|--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |--- |
| F(n) | 0 | 1 | 1 | 2 | 3 | 5 | 8 | 13 | 21 | 34 | 55 | 89 | 144 | 233 | 377 | 610 | 987 | 1597 | 2584 | 4181 | 6765 |
| P(n) | 0 | 1 | 1 | 2 | 3 | 1 | 0 | 1  | 1  | 2  | 3  | 1  | 0   | 1   | 1   | 2   | 3   | 1    | 0    | 1    | 1    |

> $P(n) = F(n) \mod M$

$P(n)$을 잘 보면 어떤 패턴을 띄는게 보입니다: $0, 1, 1, 2, 3, 1$이 계속 반복되는게 보이죠.  
이렇게 *피보나치 수열*을 임의의 정수 $M$으로 나눈 나머지로 이루어진 수열은 주기를 띄는게 **피사노 주기**의 성질입니다.  

> *피사노 주기*는 $\pi(n)$으로 표기합니다.

앞에서 정리한 표를 바탕으로 봤을 때, $\pi(4) = 6$이 되겠죠.  

마지막으로 $\pi(M)$을 구하는 방법은 다음과 같습니다:  
$\pi(M) = 15 \times 10^{k - 1}(\text{단, }M = 10^{k}, k > 2)$

## 적용

> [피보나치 수 3 (baekjoon online judge)](https://www.acmicpc.net/problem/2749)

위 문제를 푸는데 *피사노 주기*를 이용했습니다.

*피보나치 수열*을 구현할 때 가장 먼저 생각나는 방법은 *재귀적으로 $n$번째 까지 계산하는 방법*이었습니다.

``` cpp
int getFibonacciNumberAt(int index)
{
    if (index <= 1)
        return index;

    return getFibonacciNumberAt(index - 1) + getFibonacciNumberAt(index - 2);
}
```

하지만 이 방법은 문제에서 주어지는 메모리 제한이나 시간 제한을 충족하지 못합니다.  
$F(n)$의 $n$이 커질수록 실행시간이 길어지니까요($O(2^n)$).

*동적 계획법*으로 풀어보려고해도 메모리 크기 제한 때문에 답이 보이지 않습니다.  
단순히 $F(1000)$만 계산해봐도 대략 $10^{208}$자릿수가 나오니까요.

> `unsigned long long`의 값 표현 법위는 0 ~ 18,446,744,073,709,551,615($2^{64}$, $10^{19}$ 자릿수)입니다.  

그래서 서칭을 하다보니 *피사노 주기*를 찾게되었습니다.  
문제에서 제시하는 수 $M = 1000000$에 대해서 $\pi({1000000}) = 1500000$을 대입해 문제를 해결 할 수 있었습니다:
```cpp
typedef unsigned long long ull;

ull fibonacci(ull n, const ull MOD)
{
    vector<ull> fibonacciSequence = { 0, 1 };
    const ull fisanoPeriodSize = 1500000;

    for (ull i = 2; i < fisanoPeriodSize; ++i)
    {
        fibonacciSequence.push_back((fibonacciSequence[i - 1] + fibonacciSequence[i - 2]) % MOD);
    }

    return fibonacciSequence[n % fisanoPeriodSize];
}
```

## 마치며

물론 앞의 *피보나치 수열 3*에 대한 코드 내용에는 언제나 $M = 100000$이라는 가정하에 구현된 내용입니다.  
$M$은 언제나 10의 거듭제곱이라는 조건이 주어진다면 `getPisanoPeriod(unsigned long long modulo)` 같은 함수를 통해 좀 더 확장 할 수도 있을 것 같습니다.  
```cpp
typedef unsigned long long ull;

// returns pisoano period of modulo.
// This function takes adventage of the property that the start of a Pisano Cycle always starts with 0 and 1.
ull getPisanoPeriod(ull modulo)
{
    ull a = 0, b = 1, c = a + b;
    for (ull i = 0; i < modulo * modulo; ++i) {
        c = (a + b) % m;
        a = b;
        b = c;

        if (a == 0 && b == 1) {
            return i + 1;
        }
    }
}
```

수학과 관련된 내용들은 언제나 배울게 많네요.

### references

- [Fibonacci Sequence(wikipedia)](https://en.wikipedia.org/wiki/Fibonacci_sequence)
- [Pisano Period(wikipeida)](https://en.wikipedia.org/wiki/Pisano_period)