---
layout: post
title: Coordinate Space for Shader
date: 2025-06-07 04:53 +0000
category: [Unity, Shader]
---

# 셰이더를 위한 좌표공간 개념
## 서론
우리에게 ```cup.obj```라는 컵 모양의 3D 모델 파일이 있다고 상상해보자.  
```cup.obj``` 파일을 열어보면 컵의 정점들에 대한 좌표가 들어있을 것이다.  
예를 들어, 컵의 손잡이 좌표는 (0, 1, 0)같은 값을 가질 수 있다.

여기서 (0, 1, 0)는 대체 무슨 의미일까?  
주전자의 바닥을 기준으로 1만큼 위에 있다는 것일까,  
아니면 주전자의 입구를 기준으로 1만큼 앞에 있다는 것일까?

애초에, 방향은 누가 결정하는걸까?  
누군가는 주전자 손잡이를 마주보는 상황을 기준으로 방향을 정의해야 한다고,  
누군가는 180도 회전시켜서 주전자 입구를 마주보는 상황을 기준으로 삼아야한다고 주장할 수 있을 것이다.

여기서 알 수 있듯이 좌표의 의미를 해석하는 과정은 다음과 같다:
> "A 지점을 기준으로 B 방향을 따라 C만큼 떨어진 지점"

| 기호 | 의미                              | 예시                                     |
| ---- | --------------------------------- | ---------------------------------------- |
| A    | 좌표 공간의 <b>원점</b>           | 주전자 뚜껑의 중심                       |
| B    | 해당 축의 방향 (<b>기저 벡터</b>) | 주전자 입구를 정면에서 마주볼 때 앞 방향 |
| C    | 해당 축으로의 거리                | 1                                        |

<b>즉, 좌표는 원점과 기저 벡터에 따라 의미가 결정된다!</b>

## 셰이더에서 자주 등장하는 좌표공간
### 1. Local Space
```cup.obj``` 예시에서 본 것처럼 해당 모델 자신을 기준으로 결정되는 좌표를 의미한다.  
컵 오브젝트가 게임에서 계속 이동하더라도 ```컵 손잡이의 local space coordinate```는 항상 동일하다!

> 유니티 셰이더에서는 ```object space```라는 이름으로 등장함

### 2. World Space
게임 오브젝트의 이동이 일어나는 좌표 공간에서의 좌표를 의미한다.  
유니티를 기준으로 생각해보면 <b>원점</b>은 scene의 (0,0,0) 지점이고  
<b>기저 벡터</b>는 유니티 에디터에서 빨강,파랑,초록 화살표 기즈모로 표현되는 ```scene 기준 x, y, z축```이다.  
```컵 손잡이의 world space coordinate```는 컵 오브젝트의 transform (position, rotation, scale)에 따라 달라진다!

### 3. (Homogeneous) Clip Space
카메라의 시점을 기준으로 표현된 좌표를 의미한다.  
<b>원점</b>은 카메라의 위치이고 <b>기저 벡터</b>는 카메라가 바라보는 방향에 따라 결정된다.  
카메라의 projection 방식이 orthogonal인지 perspective인지에 따라 약간의 차이가 존재하지만,  
기본적으로 모든 카메라는 "카메라가 볼 수 있는 육면체 영역"인 view frustum을 가진다.  

![view frustum](https://learnopengl.com/img/guest/2021/Frustum_culling/VisualCameraFrustum.png)
*view frustum ([learnopengl.com](https://learnopengl.com/Guest-Articles/2021/Scene/Frustum-Culling))*  

```clip space coordinate```는 view frustum의 경계면에서 -1 또는 +1 값을 갖는다.  
좌표가 [-1, +1] 구간에 속하지 않는다면 화면에서 잘리기 때문에 명칭이 <b>"clip"</b> space라고 한다.

단, 실제 셰이더의 좌표 변환 과정에서는 homogeneous coordinate를 사용하므로  
임의의 ```homogenous clip space coordinate``` (x, y, z, w)에 대해  
x, y, z의 값은 [-w, +w] 구간으로 주어진다고 생각해야 한다.

### 4. (Homogeneous) Normalized Device Coordinate Space (NDC)
```clip space coordinate```를 [0, 1] 구간으로 매핑한 결과로, OpenGL 기준으로 (0, 0)이 화면 좌측 하단에 대응된다.  
DirectX의 경우 ```NDC space```의 y축이 OpenGL과 다르게 아래 방향이라 (0, 0)이 화면 좌측 상단에 대응된다고 한다.  

![y axis difference](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjREEMZrCibHsXaeOHUDaNLQ4twX7l9aDFNR0QYupluGHVyYUjAtauWfGnmKcyGUHUOo9c8Be4gyd8xokBHM2dgHMURsyf5IdK_v2SM2u-E5l1Ym6dWXA6SehuKHOH448y8IrgDy1LFnxhc/s1600/20100531_DX_OpenGL.png)
*그래픽 라이브러리에 따른 y축 방향 차이 ([thedev-log](https://thedev-log.blogspot.com/2012/07/texture-coordinates-tutorial-opengl-and.html))*

유니티는 실제 그래픽 라이브러리와 무관하게 셰이더가 동작하도록 모든 좌표계를 OpenGL 기준으로 변환해 사용한다.  
여기서 필요한 ```NDC space```의 y축 방향 정보는 ```_ProjectionParams.x```로 제공된다.  
이 값은 OpenGL일 때 +1이고 DirectX일 때 -1이어서 ```NDC space coordinate```의  
y축 요소에 곱해주면 항상 OpenGL식 방향으로 처리할 수 있게 된다.

## Case Study: 유니티 URP 기본 셰이더의 좌표 변환
[유니티 소스 코드](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.universal/ShaderLibrary/ShaderVariablesFunctions.hlsl)
```hlsl
// positionOS: object space (a.k.a. local space)
// positionWS: world space
// positionCS: clip space
// positionNDC: normalized device coordinate space
VertexPositionInputs GetVertexPositionInputs(float3 positionOS)
{
    VertexPositionInputs input;
    input.positionWS = TransformObjectToWorld(positionOS);
    input.positionCS = TransformWorldToHClip(input.positionWS);

    // 나중에 input.positionNDC.w로 xyz값을 나눈다면
    // (cs.xy * 0.5 + cs.w * 0.5) / cs.w가 나옴.
    // clip space의 xy값이 [-w, +w] 구간으로 나오니까 이걸 [0, 1] 구간으로 변환하는 것.
    // 즉, 여기서 계산하는 NDC는 우리가 생각하는 일반적인 좌표가 아니라 homogenous 좌표임!
    //
    // DirectX는 OpenGL과 y축 방향이 반대라고 함.
    // 중간에 곱해지는 _ProjectionParams.x는 OpenGL 방식으로 생각할 수 있게 부호를 통일해주는 역할.
    // * OpenGL일 때 +1, 방향이 뒤집혔으면 -1이 제공된다고 함
    float4 ndc = input.positionCS * 0.5f;
    input.positionNDC.xy = float2(ndc.x, ndc.y * _ProjectionParams.x) + ndc.w;
    input.positionNDC.zw = input.positionCS.zw;
    
    return input;
}
```
