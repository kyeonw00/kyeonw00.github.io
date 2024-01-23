---
layout: post
title: Unity에서의 Null에 대한 고찰
published: true
category: programming
tags: [programming, unity, csharp]
author: altair
toc: false
comments: false
---

`Null`은 기본적으로 비어있는 값을 의미합니다.
참조 형식 변수들의 기본 값이 되는게 `Null`이죠.

C#에서는 이 `Null`과 관련된 여러가지 연산들을 지원합니다.
`??=`, `?.`, `?[]` 등의 연산자들을 지원하죠:
- `??=`: 피연산자가 `Null`이 아닌경우 왼쪽 피연산자의 값을 반환하는 연산자
- `?.`: `a?.x`와 같이 사용할 경우, `a`가 `Null`이라고 판단 될 때 `a?.x`는 `Null`을 반환합니다.

Unity에서도 물론 `Null`과 관련된 연산자들을 사용할 수 있습니다.
```csharp
var collider = GetComponent<BoxCollider>();
if (collider != null) { /* do something... */ }
```

그런데 Unity에서 `UnityEngine.Object` 객체에 대해 `Null` 비교연산을 이용하게 될 경우 조심해서 사용해야 합니다.

일례로 다음의 코드를 살펴봅시다:
```csharp
public GameObject testGameObject;

private IEnumerator Start()
{
    Destroy(testGameObject);
    yield return null;
    
    Debug.Assert(testGameObject == null, $"{nameof(testGameObject)} == null ⇒ assert failure");
    Debug.Assert(testGameObject is null, $"{nameof(testGameObject)} is null ⇒ assert failure");
}
```
단순히 생각해본다면, `Start()` 메소드 내부의 `Debug.Assert()` 함수들은 모두 성공할 것으로 예상이 됩니다.  
하지만 실제로 코드를 실행해본다면 결과는 조금 다릅니다.

![code-result-1](/assets/img/post-resources/2024-01-22-null-in-unity-1.png)  
실제로 실행했을 때에는 `testGameObject is null` 조건문에 대해서만 *assertion failure*가 발생합니다.

`Start()` 메소드 내부의 코드들에 대해서 디버깅을 해본다면 다음과 같이 확인할 수 있습니다:
![code-result-2](/assets/img/post-resources/2024-01-22-null-in-unity-2.png)  
`Destroy()`를 통해 파괴되어서 `testGameObject`는 `"null"`이 되었다고 합니다: 단순히 생각해보면 *dangling pointer* 또는 `null`이 맞는데 말이죠.

## `UnityEngine.Object`와 *C++ Native Object*

유니티 프로젝트에서 작성한 대부분의 클래스들과 유니티 엔진에 포함된 대부분의 컴포넌트들은 `UnityEngine.Object`를 상속합니다.  
여기서 `UnityEngine.Object`는 C++ Native Object에 대한 Wrapper 객체이죠.  

`Object.Destroy()`를 통해 유니티 오브젝트를 파괴한다면, 이 C++ 네이티브 객체는 할당 해제되는게 맞습니다.  
다만, 이 객체를 랩핑한 유니티 오브젝트는 바로 해제되지 않습니다. **가비지 컬렉터(Garbage Collector, GC)**가 메모리 영역을 해제할 때까지 남아있게 되죠.  
이 상태를 유니티에서는 **Fake Null**이라고 부릅니다.  

이전의 코드에서 봤던 것처럼 `==`, `!=` 연산자에 대해서는 작동은 하는것으로 보입니다.  
`UnityEngine.Object`에 이 두 연산자를 오버로딩 해두어서 네이티브 객체 존재 여부에 대해서도 판단 할 수 있습니다.  

다시 본론으로 돌아와 앞선 코드에서 `testGameObject is null`이라는 조건문에서 *assertion failure*가 발생한 이유는 `testGameObject`의 C++ 개체는 해제된 상태이지만, C# 래퍼 개체는 남아있어 *failure*가 발생한다고 봐도 되겠죠.

그래서 저는 보통 `Destroy()`를 호출한 뒤에는 명시적으로 `null`을 대입해주는 방식으로 우회를 합니다.
```csharp
Destroy(testGameObject);
testGameObject = null;
```
![code-result-3](/assets/img/post-resources/2024-01-22-null-in-unity-3.png)  

## `UnityEngine.Object`와 `Null` 비교 연산은 비싸다.

앞에서 말했듯 `UnityEngine.Object`의 오버로딩된 `==`, `!=` 비교 연산자는 네이티브 개체의 존재 여부도 판단합니다.  
그래서 `UnityEngine.Object`의 `==`, `!=` 등은 단순히 C# 개체와 `null`을 비교하는게 아닌 또다른 비교 연산이 존재하기에 비싼 연산이라고 할 수 있을것 같습니다.

일례로 *`MonoBehaviour.transform`은 호출 비용이 비싸니까 캐싱해놓는게 좋다*라고 하죠.  
```csharp
private Transform m_CachedTransform;

private void Start()
{
    m_CachedTransform = GetComponent<Transform>();
}
```

프로퍼티로 감싸서 이용하는 경우도 종종있습니다.
```csharp
private Transform m_CachedTransform;

public Transform CachedTransform
{
    get
    {
        if (m_CachedTransform == null)
            m_CachedTransform = GetComponent<Transform>();

        return m_CachedTransform;
    }
}
```
하지만 이 코드는 결국에는 `CachedTransform`을 호출 할 때마다 *null checking*이 수행되기에 결국 `CachedTransform` 프로퍼티는 별다른 성능 개선은 없었다고 봐야합니다.  

만약에 위 코드를 최적화 시키고 싶다면, `m_CachedTransform`을 C# 네이티브 오브젝트로 캐스팅해 비교하는 방법이 있겠죠.
```csharp
private Transform m_CachedTransform;

public Transform CachedTransform
{
    get
    {
        if (m_CachedTransform is null)
            m_CachedTransform = GetComponent<Transform>();

        return m_CachedTransform;
    }
}
```

> [이 블로그 글](https://overworks.github.io/unity/2019/07/22/null-of-unity-object-part-2.html)에서 *ILSpy*를 통해 실제로 오버로드된 연산자들이 어떻게 구현되어있는지 정리되어있으니 궁금하다면 보는것도 좋을듯 합니다.  

## 값이 대입되지 않은 `UnityEngine.Object`는 *fake null* 상태로 직렬화 된다.

이번에는 `testGameObject` 맴버변수에 아무것도 대입하지 않은 상태에서 테스트를 진행해봅시다.  
아무것도 할당되지 않은 상태니 `testGameObject`는 `null`이겠죠?
```csharp
public GameObject testGameObject;

private void Start()
{
    Debug.Assert(testGameObject is null, $"{nameof(testGameObject)} is null ⇒ assert failure");
}
```
![code-result-4](/assets/img/post-resources/2024-01-22-null-in-unity-4.png)  
![code-result-5](/assets/img/post-resources/2024-01-22-null-in-unity-5.png)  
하지만 실제로 실행해보면 *fake null* 상태로 보입니다.  
![code-result-6](/assets/img/post-resources/2024-01-22-null-in-unity-6.png)  

정확히 왜 이렇게 되는건지는 내부 구현을 봐야 알겠지만, 지금까지 고찰해본 것들에 미루어보아 `UnityEngine.Object`가 역직렬화 되는 과정에서 *fake null* 상태로 만드는게 아닌가 싶습니다.

### 참고한 외부 블로그 글
- [유니티 오브젝트의 fake null - 평생 공부 블로그](https://ansohxxn.github.io/unitydocs/fakenull/)
- [유니티 오브젝트의 null 비교 시 유의사항](https://overworks.github.io/unity/2019/07/16/null-of-unity-object.html#fnref:1)