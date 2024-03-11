---
layout: post
title: 제곱근 연산은 정말로 비싼 연산일까?
published: true
category: programming
tags: [programming, c++, csharp, math]
author: altair
toc: false
comments: false
---

제곱근 연산은 게임 프로그래밍에 있어서 자주 쓰이는 연산 중 하나라고 생각합니다.
***백터의 정규화***, ***벡터의 길이***를 구하는 등, 여러 방면에서 유용하게 쓰입니다.

하지만 간혹 최적화에 대한 이야기가 나오면, ***제곱근 연산은 비싼 연산이다***라는 이야기도 함께 나옵니다.  
여러 블로그 글이나, 유니티의 [`Vector3.magnitude`에 대한 문서](https://docs.unity3d.com/ScriptReference/Vector3-magnitude.html)에서도 `Vector3.sqrMagnitude`가 더 빠르니 벡터끼리 길이를 비교하려면 `Vector3.sqrMagnitude`를 써라는 말도 적혀있습니다.  

> `Vector3.sqrMagnitude`는 `Vector3.x * Vector3.x + Vector3.y * Vector3.y + Vector3.z * Vector3.z`이고, `Vector3.magnitude`는 `Mathf.Sqrt(Vector3.sqrMagnitude)`와 같습니다.  
> 단순히 `Vector3.magnitude`가 `Vector3.sqrMagnitude`의 제곱근과 같기 때문에, 후처리 연산이 빠지게 되어 속도의 차이가 난다고 생각해도 될 것 같습니다.

지금까지 그냥 비싸구나라고 생각하고 따로 찾아보지는 않았지만 실제로 왜그런지 궁금해서 몇가지 실험을 해보았습니다.

## 제곱근의 연산

일단 제곱근은 *"어떤 수에 대해서 제곱했을 때 그 수가 되는 모든 수"*라고 정의합니다: 실수 및 복소수 $x$에서 제곱한 수 $x$의 뿌리가 되는 모든 수를 의미합니다.  

실수의 범위 내에서 제곱근은 $1 ... |x|$ 사이의 값입니다. 그렇다면 단순하게 범위 내의 수들을 직접 계산해보면서 제곱했을 때, $x$가 되는 수를 찾으면 됩니다.  
그 방법중 하나가 *이진 탐색*을 통한 방법입니다.  

```cpp
// find square root of given real number `value`.
float sqrt(float value)
{
    float lowest = 0f, highest = value;
    float mid = 0f, sqrMid = 0f;

    while (value > (100.0f * lowest * lowest)) { lowest *= 10.0f; }
    while (value < (0.01f * highest * highest)) { highest *= 0.1f; }

    for (int i = 0; i < 100; ++i) {
        mid = (lowest + highest) * 0.5f;
        sqrMid = mid * mid;

        if (fabs(sqrMid - value) < 0.0001f) { return mid; }

        if (sqrMid > value) highest = mid;
        else lowest = mid;
    }

    return mid;
}
```

위 코드에서는 최대 100번까지 수들을 탐색하면서 제곱근에 가까운 수를 찾아냅니다.  
`sqrt(3)`에 대해서는 *1.732056*까지 계산이 가능했습니다. 만약 더 정확한 수를 계산하고 싶다면 탐색 수를 더 늘리면 됩니다.

*이진탐색*이 아닌 **뉴턴의 방법(뉴턴-랩슨 메소드, Newton-Raphson method)**라는 제곱근 연산 알고리즘도 알 수 있었습니다:

```cpp
// returns the square root of `x` with a precision of up to 4 decimal places
double sqrt(double x)
{
    double bisection = (x * x * x) - (x * x) + 2;
    double derivation = (3 * x * x) - (2 * x);
    double delta = bisection / derivation;
    
    while (fabs(delta) >= 0.0001f)
    {
        delta = (x * x * x) - (x * x) + 2;
        derivation = (3 * x * x) - (2 * x);

        x = x - h;
    }

    return x;
}
```

위 구현들에서 볼 수 있듯, 제곱근 연산은 반복 연산을 이용합니다. 더 정확한 수를 얻기 위해서는 연산을 더 많이 반복하게되고, 그에 따라 비싼 연산이라고 알려진게 아닌가 싶습니다.  

## 그렇다면 삼각함수는 어떨까?

회사 선배분들과 이야기 할 때, ***삼각함수도 생각보다 비싼 연산이다***라는 이야기가 나왔습니다.  
들었을 당시에는 이정도 수준의 성능 최적화를 생각하지는 못해서 그런가보다 하고 넘어갔습니다.  

일단 `sin`, `cos`, `tan` 같은 삼각함수들은 계산하는 방식을 정확하게 생각하기는 어려웠습니다.  
찾아보니 ***보간을 통한 방법***, ***테일러 급수를 이용해 추측하는 방법***을 찾을 수 있었습니다.

첫번째로 보간을 통한 방법은 미리 계산된 삼각함수의 값들을 룩업테이블(Lookup Table, LUT)에 저장해놓고, 주어진 값을 저장된 값들에 대해 보간해서 근사치를 계산하는 방식입니다.

```cpp
float SIN_TABLE[SIZE];

float sin(float x)
{
    float i = map_to_sin_range(x);
    int i1 = floor(i);
    int i2 = i1 + 1;
    float t = i - i1;
    return lerp(SIN_TABLE[i1], SIN_TABLE[i2], t);
}
```

이 방식은 idsoftware에서 만든 *울펜슈타인3D*에서 이용된 방식이며, 미리 계산된 삼각함수의 결과값이 많을수록 정확합니다.  
이 방식은 뒤에 살펴볼 *테일러 급수를 이용한 추측*에 비하면 연산 속도는 빠르지만 삼각함수 값을 캐싱할 메모리 공간을 추가로 요구한다는 단점이 있습니다.

두번째 방법으로는 *테일러 급수*를 이용한 방법입니다.  
이 방법은 크게 *1. 값 범위 축소*, *2. 근사치 계산*, *3. 값의 재구성* 세 단계로 이루어집니다.  

기본적으로 `sin` 함수는 아래와 같은 주기를 가집니다:

<img src="https://drive.google.com/uc?id=1OQSmlhIadggOOS3cOumdcq6XGG91PTsH" alt="Sine Wave" style="height:300px" />

이 주기에서 $0 ... \frac{\pi}{2}$ 범위의 주기만 가지고도 반복되는 주기 내의 값으로 보간할 수 있습니다.  
반복되는 주기를 $0 ... \frac{\pi}{2}$, $\frac{\pi}{2} ... \pi$, $\pi ... -\frac{\pi}{2}$, $-\frac{\pi}{2}...2\pi$ 네 범위로 구분하고, 주어진 값을 각 범위의 값으로 재구성해 `sin(x)`값을 유추해낼 수 있습니다.

```cpp
float sin(float x)
{
    auto x2 = x * x;
    auto x3 = x2 * x;
    auto x5 = x3 * x2;
    return x - (x2 / 6.0f) + (x5 / 120.0f);
}

float sin_approximation(float x)
{
    int k = floor(x * 2.0f / M_PI);
    float y = x - (k * M_PI * 0.5f);
    int quadrant = k % 4;
    switch (quadrant)
    {
        case 0: return sin(y);
        case 1: return sin(M_PI * 0.5f - y);
        case 2: return sin(y) * -1;
        case 3: return sin(M_PI * 0.5f - y) * -1;
    }

    return sin(y);
}
```

> [링크(dbl-64 s_sin.c)](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/ieee754/dbl-64/s_sin.c;hb=HEAD#l194)에서 범용 *x86-64 linux*에서 사용되던 `sin()` 함수의 구현부를 확인 할 수 있습니다.  
> 정확한 원리는 이해하기 어려웠지만, 대략적으로는 미리 정의된 범위에 따라 값을 보간해 `sin` 값을 구하는 방식입니다.

## 테스트

이번에는 실제로 자주 사용하는 연산들끼리의 실행시간을 비교해 성능에서 차이가 있는지 확인을 해보겠습니다.  
사용된 유니티 엔진 버전은 `2022.3.12f1 LTS(csharp 9.0)`이며, 여러 연산들을 반복해서 테스트 해봅니다.

```csharp
private void TestUnityProvidedFunctions()
{
    System.Diagnostics.Stopwatch stopwatch = new();
    System.Text.StringBuilder logBuilder = new();
    
    // Test 1: Multiplication
    stopwatch.Restart();
    for (var i = 0; i < iterations; ++i)
    {
        var num = (float)i * i;
    }
    stopwatch.Stop();
    logBuilder.AppendLine($"Test 1: Multiplication {iterations} times took {stopwatch.ElapsedMilliseconds}ms");
    
    // Test 2: Division
    stopwatch.Restart();
    for (var i = 0; i < iterations; ++i)
    {
        var num = i / 3f;
    }
    stopwatch.Stop();
    logBuilder.AppendLine($"Test 2: Division {iterations} times took {stopwatch.ElapsedMilliseconds}ms");
    
    // Test 3: Mathf.Sqrt()
    stopwatch.Restart();
    for (var i = 0; i < iterations; ++i)
    {
        var num = Mathf.Sqrt(i);
    }
    stopwatch.Stop();
    logBuilder.AppendLine($"Test 3: Mathf.Sqrt() {iterations} times took {stopwatch.ElapsedMilliseconds}ms");
    
    // Test 4: Mathf.Sin() (or other trigonometric functions)
    stopwatch.Restart();
    for (var i = 0; i < iterations; ++i)
    {
        var num = Mathf.Sin(i);
    }
    stopwatch.Stop();
    logBuilder.AppendLine($"Test 4: Mathf.Sin() {iterations} times took {stopwatch.ElapsedMilliseconds}ms");
    
    // Test 5: Vector3.magnitude
    stopwatch.Restart();
    for (var i = 0; i < iterations; ++i)
    {
        var num = new Vector3(i, i, i).magnitude;
    }
    stopwatch.Stop();
    logBuilder.AppendLine($"Test 5: Vector3.magnitude {iterations} times took {stopwatch.ElapsedMilliseconds}ms");
    
    // Test 6: Vector3.sqrMagnitude
    stopwatch.Restart();
    for (var i = 0; i < iterations; ++i)
    {
        var num = new Vector3(i, i, i).sqrMagnitude;
    }
    stopwatch.Stop();
    logBuilder.AppendLine($"Test 6: Vector3.sqrMagnitude {iterations} times took {stopwatch.ElapsedMilliseconds}ms");
    
    // Test 7: Vector3.Normalize()
    stopwatch.Restart();
    for (var i = 0; i < iterations; ++i)
    {
        new Vector3(i, i, i).Normalize();
    }
    stopwatch.Stop();
    logBuilder.AppendLine($"Test 7: Vector3.Normalize() {iterations} times took {stopwatch.ElapsedMilliseconds}ms");
    
    // Test 8: Dot Product
    stopwatch.Restart();
    for (var i = 0; i < iterations; ++i)
    {
        var num = Vector3.Dot(new Vector3(i, i, i), new Vector3(i, i, i));
    }
    stopwatch.Stop();
    logBuilder.AppendLine($"Test 8: Dot Product {iterations} times took {stopwatch.ElapsedMilliseconds}ms");
    
    // Test 9: Cross Product
    stopwatch.Restart();
    for (var i = 0; i < iterations; ++i)
    {
        var num = Vector3.Cross(new Vector3(i, i, i), new Vector3(i, i, i));
    }
    stopwatch.Stop();
    logBuilder.AppendLine($"Test 9: Cross Product {iterations} times took {stopwatch.ElapsedMilliseconds}ms");
    
    Debug.Log(logBuilder.ToString());
}
```

```log
Test 1: Multiplication 1000000 times took 2ms
Test 2: Division 1000000 times took 4ms
Test 3: Mathf.Sqrt() 1000000 times took 22ms
Test 4: Mathf.Sin() 1000000 times took 31ms
Test 5: Vector3.magnitude 1000000 times took 29ms
Test 6: Vector3.sqrMagnitude 1000000 times took 20ms
Test 7: Vector3.Normalize() 1000000 times took 57ms
Test 8: Dot Product 1000000 times took 26ms
Test 9: Cross Product 1000000 times took 25ms
```

> GC와 같은 요인도 배제하려고 3번정도 동일한 테스트를 반복했습니다.  
> 하지만 많이 차이나도 *2ms* 정도 차이만 나서 단일 결과만 나열했습니다.  

> 단순히 유니티 엔진 내에서 이용할 수 있는 벡터/실수 연산들을 이용한것이라 비교가 정확하지 않을 수 있습니다.  
> 최신 버전의 dotnet 환경에서는 이 연산들이 더 최적화된 상태로 이용할 수도 있을거라 생각됩니다.

테스트를 통해 다음과 같은 점을 알 수 있었습니다:  
1. `Vector3.sqrMagnitude`는 `Vector3.magnitude`보다는 확연히 빠릅니다.
2. `Vector3.Normalize()`가 오히려 다른 벡터연산보다 오래 걸립니다.
3. 삼각함수가 생각보다는 빠르게 연산됩니다.

위 결과값이 절대적인 지표가 되지는 않지만, 그래도 대략적인 가늠은 할 수 있다고 생각이 되었습니다.  
과거의 CPU 또는 개발 환경에서는 앞서 언급된 연산들이 비싸다고 생각 될 수는 있겠지만, 현대에 와서는 하드웨어의 발전이나 각종 최적화를 통해 비교적 가벼워졌다고 봅니다.  

## 결론

제곱근 연산이나 삼각함수가 생각했던것 만큼 복잡하고 비싼 연산은 아니라는 것을 알 수 있었습니다.  
하지만 이건 어디까지나 연산 자체의 성능이 비싸지 않다는것이지, 실제로 연산을 이용한 구현부에서 최적화 가능한 부분은 최적화 해야한다는 생각이듭니다.

## references

- [How does C compute sin() and other math functions? - stackoverflow](https://stackoverflow.com/questions/2284860/how-does-c-compute-sin-and-other-math-functions)
- [Fater math functions by Robin Green](https://basesandframes.wordpress.com/2016/05/17/faster-math-functions/)
- [So how does your computer actually compute sine? - SimonDev on Youtube](https://www.youtube.com/watch?v=kkMt4lrJzs8)
- [List of Maclaurin series of some common functions : Trigonometric Functions - wikipedia](https://en.wikipedia.org/wiki/Taylor_series#Trigonometric_functions)
- [Code Challenge : Implement a square root function](https://matt.coneybeare.me/coding-challenge-implement-a-square-root-function/)
- [The Computation of Transcendental Functions on the IA-64 Architecture](https://www.cl.cam.ac.uk/~jrh13/papers/itj.pdf)
- [Newton's Method - wikipedia](https://en.wikipedia.org/wiki/Newton's_method)
- [Sysdeps](https://sourceware.org/git/?p=glibc.git;a=tree;f=sysdeps;hb=HEAD)
- [게임 엔진 블랙 북 : 울펜슈타인 3D](https://www.yes24.com/Product/Goods/94082334)
- [In computer programming, why do you avoid the square root? - Quora](https://qr.ae/pstWyE)