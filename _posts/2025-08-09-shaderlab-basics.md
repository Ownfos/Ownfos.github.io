---
layout: post
title: ShaderLab Basics
date: 2025-08-09 05:42 +0000
category: [Unity, Shader]
---

# 유니티 ShaderLab 기초 지식
## 전체적인 구조
```
Shader "Lit/CustomLitShader"    <-- 셰이더의 이름
{
    Properties              <-- _MainTex처럼 외부에서 들어오는 파라미터 목록
    {
        ...
    }
    SubShader               <-- 기기가 지원하는 것 중에서 제일 위에 있는 SubShader를 사용
    {
        Tags                <-- 해당 SubShader를 언제 사용할지 결정하는 조건 ex) "RenderPipeline" = "UniversalPipeline"
        {
            ...
        }

        Pass
        {
            Tags            <-- 해당 Pass를 언제 사용할지 결정하는 조건 ex) "LightMode" = "UniversalForward"
            {
                ...
            }

        	...
        }
    }
    SubShader               <-- 위에 있는 SubShader가 모두 지원되지 않는 경우 다음으로 시도
    {
        ...
    }

    Fallback "Unlit/Color"  <-- 모든 SubShader가 실패할 경우 사용하는 기본값
}
```
## [Properties Block](https://docs.unity3d.com/2022.3/Documentation/Manual/SL-Properties.html)
오브젝트의 메인 텍스처와 노말맵 등 입력 파라미터를  ```[속성] 변수이름("인스펙터에 노출될 이름", 타입) = 기본값```형태로 나열하는 구역이다.  
흔히 사용되는 프로퍼티는 다음과 같다:

```
Properties
{
  // 2D 텍스처
  _MainTex("Main Texture", 2D) = "white" {}
  _NormalMap("Normal Map", 2D) = "bump" {}

  // 색상 (HDR attribute를 붙이면 인스펙터에서 intensity까지 조절 가능해짐)
  _SimpleColor("Simple Color", Color) = (1,1,1,1)
  [HDR] _HDRColor("HDR Color", Color) = (1,1,1,1)

  // 실수
  _SomeFloat1("Name1", Float) = 0.1
  _SomeFloat2("Name2", Range(0.0, 1.0)) = 0.1

  // 정수
  _SomeInt("Name3", Integer) = 1

  // float4 (float2와 float3은 없음)
  _SomeVector("Name4", Vector) = (1,1,1,1)
}
```

## SubShader
SubShader는 하드웨어 성능이나 렌더 파이프라인의 종류 등에 맞춰서 각 상황에 호환 가능한 셰이더를 제공하기 위해 나온 개념이다.  
대체로 렌더링 퀄리티와 연산량은 비례한다는 것을 염두하고 다음 상황을 상상해보자.  

> 당신은 하이엔드 데스크탑과 사무용 노트북에서 모두 돌아가는 게임을 개발하는 중이며  
> 게임의 핵심적인 시각 효과를 담당하는 셰이더는 고사양 버전과 저사양 버전이 준비되어있다.  
> 만약 셰이더가 기기 사양을 전혀 고려하지 않는다면 어떻게 해야할까?

만약 하드웨어 성능을 고려해 적절한 셰이더를 골라주는 기능이 없다면  
당신은 노트북용 빌드를 만들 때 모든 셰이더를 저사양 버전으로 교체한 뒤 빌드하고  
다시 모든 셰이더를 고사양 버전으로 교체한 뒤 빌드하는 번거로운 상황에 처할 것이다.  

정확히 이런 일을 해주는 기능이 바로 SubShader의 ```LOD Block```이다.  

### [LOD Block](https://docs.unity3d.com/2022.3/Documentation/Manual/SL-ShaderLOD.html)
기기가 감당 가능한 연산량에 따라 다른 셰이더 버전을 사용하는 가장 간단한 방법은 LOD 값이 다른 SubShader를 여럿 준비하는 것이다.  

작동 방식도 매우 단순하다!  
가장 위에 있는 SubShader부터 ```LOD Block```을 살펴보면서  
이 값이 허용된 최대 LOD를 넘지 않는다면 해당 SubShader를 선택한다.

> 주의: 탐색 순서는 항상 위에서 아래이므로 LOD가 높은 SubShader를 위에 배치해야 한다!

아쉽게도 얼마만큼의 LOD까지 허용할 것인지를 하드웨어에 따라 자동으로 결정해주지는 않는다.    
다만, 설정창에 "그래픽 퀄리티"같은 옵션을 추가하는 것으로 LOD 최대치를 조절 가능하게 만들 수는 있을 것이다.

### [SubShader Tags](https://docs.unity3d.com/2022.3/Documentation/Manual/SL-SubShaderTags.html)
LOD 이외에도 SubShader의 특성을 결정하는 태그들이 있다.  
태그를 여럿 부여하는 경우 ```Tags { "name1" = "value1" "name"2 = "value2" }```처럼 쉼표 없이 나열하면 된다.
#### RenderPipeline
URP 또는 HDRP 파이프라인에만 호환 가능하도록 제한하는 태그이다.  
```
Tags { "RenderPipeline" = "UniversalPipeline" } // URP에만 호환 가능
Tags { "RenderPipeline" = "HDRenderPipeline" } // HDRP에만 호환 가능
```

#### Queue
오브젝트의 렌더링 순서를 결정하는 속성이다.  
Background -> Geometry -> AlphaTest -> Transparent -> Overlay 순서로 렌더링 된다고 하는데  
가장 많이 쓰이는 값은 Geometry와 Transparent이다.  

투명한 오브젝트는 자신보다 뒤에 있는 물체의 색을 참고해야 하므로  
반드시 불투명한 오브젝트보다 나중에 렌더링되어야 한다!

이럴 때 불투명한 오브젝트에는 ```Tags { "Queue" = "Geometry" }```를,  
투명한 오브젝트에는 ```Tags { "Queue" = "Transparent" }```를 지정하면 된다.

보다 세밀한 순서 조정을 위해 ```Tags { "Queue" = "Geometry+1" }```처럼 오프셋을 줄 수도 있다고 한다.  

#### RenderType
RenderType은 다른 태그와 다르게 혼자서는 아무 영향도 주지 못한다.  
카메라에 [SetReplacementShader](https://docs.unity3d.com/2022.3/Documentation/Manual/SL-ShaderReplacement.html)를 하는 경우 유용하게 사용할 수 있는데,  
이 태그를 사용해 하나의 셰이더를 다른 셰이더로 잠시 대체할 때 어떤 SubShader가 대응될지 결정할 수 있다.

예를 들어, LitShader를 깊이감을 표현하는 DepthShader로 바꾸고 싶은 경우  
LitShader의 ```Tags { "RenderType" = "Transparent" }```가 달린 SubShader를 DepthShader의 ```Tags { "RenderType" = "Transparent" }```가 달린 SubShader로,  
LitShader의 ```Tags { "RenderType" = "Opaque" }```가 달린 SubShader를 DepthShader의 ```Tags { "RenderType" = "Opaque" }```가 달린 SubShader로 대응시켜서  
투명한 오브젝트와 불투명한 오브젝트가 서로 다른 SubShader로 대체되도록 만들 수 있다.

그리 직관적인 태그는 아니다보니 더 자세히 설명해주는 [자료](https://youtu.be/Tjl8jP5Nuvc?feature=shared)들을 찾아보는 것을 추천한다.  

## Pass
어떤 SubShader가 사용 가능하다고 판단되면 렌더링 과정에서 그 안에 있는 하나 이상의 Pass들이 각자의 타이밍에 맞게 실행된다.
SubShader의 역할이 호환 가능한 셰이더 버전을 선택하는 것이라면 Pass는 셰이더 로직을 순서와 목적에 맞게 구분하는 역할이라고 보면 된다.  

아직까지는 왜 Pass라는 개념이 필요한지 별로 와닿지 않을 수 있다.
하지만 LightMode 태그와 2D URP Renderer의 노말맵 처리 방식을 보면 왜 여러개의 Pass가 필요한지 금방 납득하게 될 것이다.  

### [LightMode 태그](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/urp-shaders/urp-shaderlab-pass-tags.html)

```LightMode``` 태그는 렌더링 과정의 특정 시점에 ```Pass```가 실행되도록 결정한다.

| LightMode        | 시점                                                          | 비고                                                                             |
| ---------------- | ------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| NormalsRendering | 2D Light 렌더링이 일어나기 전                                 | Normal Maps - Quality 옵션이 None이 아닌 2D Light가 하나 이상 존재할 때만 실행됨 |
| Universal2D      | 2D URP Renderer가 실행되는 순간 (2D Light 렌더링이 끝난 이후) |                                                                                  |
| UniversalForward | 3D URP Renderer가 실행되는 순간                               | 에디터의 Scene View에는 해당 Pass가 렌더링되지 않음                              |
| SRPDefaultUnlit  | 기본적인 2D 혹은 3D 렌더링이 끝난 뒤                          | LightMode 태그를 지정하지 않으면 기본으로 이 값이 사용됨                         |

#### 2D URP Renderer와 노말맵
2D URP Renderer는 빛을 처리하는 방식이 아주 독특하다.  
흔히들 알고있는 3D 셰이더의 경우 오브젝트의 fragment shader에서 광원과 노말맵 정보를 활용해 빛을 계산하는 반면  
1. 오브젝트의 vertex shader에서 위치와 노말맵 계산
2. 오브젝트의 fragment shader에서 광원 정보를 활용해 빛의 영향 계산

2D URP는 노말맵과 광원 연산의 순서가 뒤집혀있다:

1. 화면을 기준으로 모든 오브젝트의 노말맵 렌더링
2. 노말맵 정보를 바탕으로 모든 2D Light 렌더링
3. 스프라이트 렌더링 시점에 노말맵 반영까지 끝난 2D Light Texture를 색상에 반영

이 과정이 시사하는 바는 다음과 같다:
- 2D 셰이더는 빛의 목록 등 광원 정보를 알 수 없으며 최종적인 Light Texture만 제공받는다
- 2D Light에 노말맵을 적용하고 싶다면 셰이더에서 NormalsRendering 시점에 노말맵을 렌더링하는 Pass를 추가로 제공해야 한다
