---
layout: post
title: UV Coordinate for Sprite Sheet
date: 2025-06-15 10:44 +0000
category: [Unity, Shader]
---

# 스프라이트 시트를 사용할 때 UV 좌표 구하기
## 무엇이 문제인가?
2D 애니메이션을 만들기 위해 각 프레임의 이미지를 옆으로 이어붙인  
하나의 커다란 이미지를 스프라이트 시트라고 부른다.  
실제로는 sprite editor에서 개별 이미지를 slice해서 사용하므로  
우리 눈에는 한 시점에 전체 시트의 한 부분만 보이게 된다.  
![coins](/assets/img/Coins.png)

하지만 위에 있는 스프라이트 시트에서 동색 동전을 sprite renderer에 넣고  
UV 좌표를 시각화하는 셰이더를 적용해보면 약간 당황스러운 결과가 나온다.
![bronze coin uv](/assets/img//bronze-coin-uv.png)
*표시된 색상은 (u, v, 0, 1)*

UV 좌표는 좌측 하단에서 (0, 0), 우측 상단에서 (1, 1)이어야 할텐데  
이미지의 모든 곳에서 (0, 1)에 가까운 값이 보인다.

자세히 살펴보니 해당 스프라이트도 전체 스프라이트 시트에서 좌측 상단에 있다.  
그럼 금색 동전 이미지를 넣으면 (0, 0)에 가까운 값이 나올까?  
동일한 셰이더를 사용해 렌더링하면 실제로 아래와 같은 색을 볼 수 있다!
![gold coin uv](/assets/img/gold-coin-uv.png)

<b>이는 slice된 스프라이트의 UV 좌표가 전체 스프라이트 시트를 기준으로 주어진다는 뜻이다</b>

## 개별 스프라이트 기준 UV 좌표 계산하기
### 원리
slice된 개별 스프라이트가 전체 시트에서 차지하는 범위를 셰이더에 넘겨줄 수만 있다면  
역으로 전체 시트 기준 UV 좌표를 개별 스프라이트 기준 UV 좌표로 보정할 수 있다.

예를 들어, 금색 동전이 차지하는 범위를 [0, 1] 범위로 표현하면 대략 아래같은 수치가 나올 것이다.

|      | 시작 | 끝   |
| ---- | ---- | ---- |
| 가로 | 0    | 0.25 |
| 세로 | 0    | 0.33 |

이는 셰이더에 주어진 UV 좌표가 (0, 0)일 때 좌측 하단이고 (0.25, 0.33)일 때 우측 상단이라는 뜻이다.  
우리가 [0, 1] 범위로 보정된 UV 좌표를 원한다면 [0, 0.25] 구간을 [0, 1]로 매핑해주면 되는 것이다.

```c
// raw: 셰이더에 주어진 UV 좌표
// mapped: 해당 slice를 기준으로 [0, 1] 범위에 맞게 보정된 UV 좌표
mapped = (raw - min) / (max - min)
```

> ```min```과 ```max``` 정보는 ```Sprite``` 클래스가 제공함!

### 구현

#### 1. 셰이더
##### 1-1. ```min```과 ```max```를 전달받을 property 생성
UV 보정에는 x축과 y축 각각의 ```min```과 ```max```가 필요하므로 총 float 4개가 필요하다.  
셰이더 프로퍼티를 최대한 간소하게 유지하기 위해 네 수치를 Vector 하나로 처리할 것이다.  
Properties 블록에 아래 라인을 추가하면 코드에서 ```"_SpriteRect"```라는 이름으로 접근할 수 있다.
> _SpriteRect ("Sprite Rect", Vector) = (0, 0, 0, 0)


##### 1-2. UV 좌표 보정하기
여기서는 fragment shader에서 보정을 수행했지만  
약간 더 효율적으로 만들고 싶다면 ```v2f``` 구조체에 필드를 하나 추가해  
vertex shader가 기존 UV와 보정된 UV를 모두 전달하도록 하면 된다.  

참고로 기존 UV는 텍스처 샘플링에 필요하므로 여전히 필요하다!

```hlsl
// Property로 추가한 변수를 코드에서 쓸 수 있게 선언
float4 _SpriteRect;

float4 frag (v2f i) : SV_Target
{
    float2 min_uv = _SpriteRect.xy; // 해당 프레임에 좌측 하단 끝에 부여될 "시트 기준 uv 좌표"
    float2 max_uv = _SpriteRect.zw; // 해당 프레임에 우측 상단 끝에 부여될 "시트 기준 uv 좌표"
    float2 real_uv = (i.uv - min_uv) / (max_uv - min_uv);

    return float4(real_uv, 0, 1);
}
```

#### 2. 스크립트

이제 렌더링이 일어나기 직전인 ```LateUpdate()```에서 셰이더로 정보를 전달할 스크립트가 필요하다.

```c#
using UnityEngine;

public class SpriteUVCalculator : MonoBehaviour
{
    SpriteRenderer spriteRenderer;

    void Awake()
    {
        spriteRenderer = GetComponent<SpriteRenderer>();
    }

    // 애니메이터 업데이트가 끝난 시점
    void LateUpdate()
    {
        // 전체 스프라이트 시트 (픽셀 단위 가로세로 길이 정보)
        Texture2D spriteSheet = spriteRenderer.sprite.texture;
        
        // 스프라이트 시트의 특정 부분 (sprite editor에서 slice한 사각형 범위 하나)
        Rect rawRect = spriteRenderer.sprite.textureRect;

        // 픽셀 단위로 주어진 시트 상의 영역을 0~1 범위로 정규화
        var normalizedRect = new Vector4(
            // 좌측 하단 지점의 "시트 기준 uv"
            rawRect.xMin / spriteSheet.width,
            rawRect.yMin / spriteSheet.height,
            // 우측 상단 지점의 "시트 기준 uv"
            rawRect.xMax / spriteSheet.width,
            rawRect.yMax / spriteSheet.height
        );

        // 셰이더에 전달
        spriteRenderer.material.SetVector("_SpriteRect", normalizedRect);
    }
}
```
### 결과

이제 어떤 스프라이트를 넣어도 slice된 위치와 무관하게 일정한 UV 좌표를 얻을 수 있다!

![bronze coin correct uv](/assets/img/bronze-coin-correct-uv.png)
![gold coin correct uv](/assets/img/gold-coin-correct-uv.png)

## 셰이더 최종본

```shader
Shader "Unlit/UV"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _SpriteRect ("Sprite Rect", Vector) = (0, 0, 0, 0)
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            HLSLPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = TransformObjectToHClip(v.vertex.xyz);
                o.uv = v.uv;
                return o;
            }

            float4 _SpriteRect;

            float4 frag (v2f i) : SV_Target
            {
                float2 min_uv = _SpriteRect.xy; // 해당 프레임에 좌측 하단 끝에 부여될 "시트 기준 uv 좌표"
                float2 max_uv = _SpriteRect.zw; // 해당 프레임에 우측 상단 끝에 부여될 "시트 기준 uv 좌표"
                float2 real_uv = (i.uv - min_uv) / (max_uv - min_uv);

                return float4(real_uv, 0, 1);
                // return float4(i.uv, 0, 1); // 기본 uv가 보고 싶다면 사용
            }
            ENDHLSL
        }
    }
}
```

## 에셋 출처
- [골드메탈 2D 플랫포머 에셋](https://assetstore.unity.com/packages/2d/characters/simple-2d-platformer-assets-pack-188518)
