---
layout: post
title: Awake, Start, and OnEnable
date: 2025-07-06 04:19 +0000
category: [Unity, Basics]
---

# 유니티 스크립트의 세 가지 초기화 이벤트
## 서론
유니티를 배울 때 가장 헷갈리는 것 중에 하나가 바로 초기화용 이벤트가 세 가지나 있다는 점이다.
공식 문서의 도표와 설명을 봐도 굳이 셋이나 존재할 이유가 있는지는 불명확하다.  
![Unity Docs](https://docs.unity3d.com/kr/2022.1/uploads/Main/monobehaviour_flowchart.svg)
*유니티 2022.1 Documentation - 이벤트 함수의 실행 순서*

일단 Awake -> OnEnable -> Start 순서로 호출된다는 것은 알겠는데,    
이들 사이에 어떤 차이가 있길래 구분해둔것일까?

## 이벤트별 특성과 용도
### 선요약

| 이벤트   | 특성                                                      | 용도                                         | 예시                                     |
| -------- | --------------------------------------------------------- | -------------------------------------------- | ---------------------------------------- |
| Awake    | 최초 활성화 시점에 실행                                   | 독립적인 초기화 로직                         | GetComponent 결과 캐싱                   |
| OnEnable | 스크립트가 활성화 될때마다 실행 <br>(최초 생성 시점 포함) | 다시 활성화 될때마다 필요한 초기화 로직      | 오브젝트 풀에서 재사용되는 오브젝트      |
| Start    | 최초 활성화 시점에 실행                                   | 다른 스크립트의 Awake에 의존하는 초기화 로직 | GetComponent 결과를 사용하는 함수를 호출 |

> 주의사항:  
> 모든 초기화 이벤트는 <b>최초 활성화 시점</b>에 실행되기 때문에  
> 비활성화 상태로 시작하는 스크립트는 Awake조차 실행되지 않은 상태로 남아있음!

### 초기화 로직 특성에 따른 분류
크게 보면 초기화 로직을 둘로 구분할 수 있다:
1. 생명 주기를 통틀어서 한 번만 초기화하면 되는 경우
2. 최초의 초기화 이후에도 다시 상태 리셋이 일어날 수 있는 경우

#### 1. 일회성 초기화
예를 들어서 살펴보자.  
주인공의 체력바를 관리하는 스크립트 ```PlayerHealth```가 있다고 치자.  
이 스크립트는 ```Slider``` 컴포넌트를 활용해 UI를 업데이트하는 함수 ```void UpdateHealth(float value)```를 제공한다.  
누군가가 이 로직을 아래와 같이 구현하였다고 생각해보자.  
```c#
public class PlayerHealth : MonoBehaviour
{
    public void UpdateHealth(float value)
    {
        Slider slider = GetComponent<Slider>();
        slider.value = value;
    }
}
```
물론 이 코드도 작동은 하겠지만, 매 프레임 호출될 함수에 ```GetComponent```를 사용하자니 느린 속도가 마음에 걸린다.  
그렇다면 어떻게 바꾸는게 좋을까?  
체력바 UI의 ```Slider``` 컴포넌트가 게임 도중에 바뀌지는 않을테니 ```GetComponent```를 한 번만 실행하도록 해보자.  
```c#
public class PlayerHealth : MonoBehaviour
{
    private Slider slider;

    public void UpdateHealth(float value)
    {
        if (slider == null)
        {
            slider = GetComponent<Slider>();
        }
        slider.value = value;
    }
}
```
아까보다는 게임이 빨라지겠지만 두 가지 단점이 남아있다.
1. 매번 널체크가 필요하다
2. ```slider```를 사용하는 다른 함수가 생기면 같은 초기화 로직을 넣어줘야한다

Awake와 Start를 사용하면 이 문제를 모두 해결할 수 있다.  
스크립트가 처음 활성화될 때 1회에 한해 실행되기 때문!
```c#
public class PlayerHealth : MonoBehaviour
{
    private Slider slider;

    private void Awake()
    {
        slider = GetComponent<Slider>();

        // precondition: 같은 오브젝트에 Slider 컴포넌트가 항상 존재한다
        // class invariant: 스크립트가 활성화된 뒤로는 slider가 항상 null이 아니다
        Assert.IsNotNull(slider);
    }

    public void UpdateHealth(float value)
    {
        slider.value = value;
    }
}
```

이제 플레이어가 생성될 때 체력바를 현재 체력에 맞게 바꿔주는 로직을 생각해보자.  
이전 맵에서 넘어오느라 체력이 반 정도 깎인 상황일 수도 있기 때문에 최대치로 시작할 수는 없다.  
```c#
public class Player : MonoBehaviour
{
    [SerializeField] private PlayerHealth playerHealth;
    [SerializeField, Range(0f, 1f)] private float hp;

    private void Awake()
    {
        playerHealth.UpdateHealth(hp);
    }
}
```

위 코드는 겉보기에 별 문제 없어보이지만 치명적인 문제가 숨어있다.  
> 서로 다른 스크립트 사이의 Awake는 고정된 호출 순서가 존재하지 않는다

즉, 운 좋게 ```PlayerHealth```의 Awake가 먼저 호출된다면 문제 없이 실행되겠지만,  
```Player```의 Awake가 먼저 호출되어서 <b>아직 초기화되지 않은</b> ```PlayerHealth```를 사용할 가능성도 있는 것이다!

이 경우 Player의 초기화 로직을 Start로 옮겨주면 ```PlayerHeatlh```의 Awake가 먼저 호출되는 것을 보장할 수 있다.  

> 참고:  
> 초기화 의존성은 대부분 ```Player```와 ```PlayerHealth```처럼 두 단계 정도에서 끝나기 때문에  
> Awake와 Start만으로도 원하는 초기화 순서를 보장할 수 있다.  
> 
> 만약 세 단계 이상의 초기화 순서가 존재한다면 ```Initialize```같은 초기화 함수를 제공하고  
> 외부에서 이를 호출하도록 만드는 방식으로 임의의 초기화 과정을 구성할 수 있다.

#### 2. 반복적인 초기화
어떤 오브젝트는 파괴되기 전까지 활성화/비활성화를 반복한다.  
오브젝트 풀 패턴이 구현되는 방식을 보면 반복적인 초기화가 왜 필요한지 직관적으로 알 수 있다.  

1. 오브젝트를 생성한다
2. 더 이상 필요하지 않게 되면 <b>오브젝트를 비활성화</b>하고 대기 목록에 넣어둔다
3. 다시 필요해지면 대기 목록에서 꺼내고 <b>오브젝트를 활성화</b>한다

플레이어가 대쉬를 할 때마다 뒤에 잔상이 남게 만들고 싶다고 하자.  

단순히 생각하면, 매 프레임 플레이어의 위치에 같은 스프라이트를 가진 잔상 오브젝트를 생성하고  
페이드 아웃을 걸어서 완전히 투명해진 뒤 잔상 오브젝트를 파괴하는 방식으로 구현할 수 있을 것이다.  
하지만 오브젝트의 생성과 파괴를 매 프레임 수행하는 것은 가비지 컬렉터에 큰 부담을 준다.  

동시에 존재할 수 있는 잔상 오브젝트의 수는 한정적일테니 투명해진 잔상 오브젝트를  
남겨뒀다가 나중에 투명도만 복원해서 재사용한다면 훨씬 효율적으로 만들 수 있다.

이 때 <b>투명도 복원</b>이라는 초기화 로직은 오브젝트가 다시 활성화될 때마다 실행되어야 한다.  
이 역할을 해주는 것이 바로 OnEnable 이벤트이다.

## Instantiate로 오브젝트를 생성할 때 초기화 순서 주의사항
지금까지 어떤 이벤트에 무슨 초기화 로직을 넣어야 하는지 알아보았다.  
이제 총알처럼 오브젝트를 동적으로 생성하는 경우의 전체적인 실행 순서를 알아보자.

```c#
public class Bullet : MonoBehaviour
{
    private void Awake()
    {
        Debug.Log("Awake");
    }

    private void Start()
    {
        Debug.Log("Start");
    }

    private void OnEnable()
    {
        Debug.Log("OnEnable");
    }
}

public class Test : MonoBehaviour
{
    // 가정: bulletPrefab에는 Bullet 스크립트가 달려있음
    [SerializeField] private GameObject bulletPrefab;

    private void SpawnBullet()
    {
        Debug.Log("Before Instantiate");
        GameObject bullet = Instantiate(bulletPrefab);
        Debug.Log("After Instantiate");
    }
}
```

위와 같은 코드가 있을 때 ```SpawnBullet``` 함수가 호출되면 로그가 어떤 순서로 남을까?  
유니티에서 돌려보면 콘솔창에서 이런 로그 순서를 볼 수 있을 것이다:
> Before Instantiate  
> Awake  
> OnEnable  
> After Instantiate  
> Start

여기서는 두 가지 파트에 주목해야 한다:
1. Awake와 OnEnable은 ```Instantiate``` 시점에 즉시 호출된다
2. 하지만 Start는 ```Instantiate```가 있는 코드가 모두 끝난 뒤에 호출된다

즉, 스크립트의 초기화가 Start에서 일어나는 경우 "After Instantiate" 시점에 아직 <b>초기화가 끝나지 않은 상태</b>일지도 모른다!  
이런 상황에서 초기화된 객체를 즉시 사용해야 하는 경우 "1프레임 기다린 뒤 사용"하는 로직을 추가해야 하므로 상당히 번거롭다.

따라서 ```Player```처럼 초기화 의존성 때문에 어쩔 수 없이 Start를 써야 하는게 아닌 이상  
최대한 초기화를 Awake와 OnEnable에서 끝내는 것이 좋다.  
