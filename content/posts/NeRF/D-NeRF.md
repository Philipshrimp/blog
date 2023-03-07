---
title: "D-NeRF: Neural Radiance Fields for Dynamic Scenes"
date: 2023-03-07T18:59:58+09:00
author:
  name: Sunho Kim
menu:
  sidebar:
    name: D-NeRF 논문 리뷰
    identifier: dnerf-review
    parent: neural-radiance-fields
    weight: 8
draft: true
---

## Intro

- D-NeRF는 임의의 시점에서의 novel view synthesis를 가능하게 한다
- 근데 이 대상체가 non-rigid geometry를 가짐
- 즉, dynamic object로 확장을 한 nerf 기법
- Deformable vomumetric function 최적화
    - 단안 view에 대한 sparse set이 입력
    - Multi-view image나 ground-truth geometry 불필요가 핵심
- 특이점
    - time이 추가적인 입력으로 들어간다.
    - learning system이 two stage로 나뉜다.
        - canonical space에 scene을 encode하는 network
        - canonical representation을 특정 시점의 deformed scene에 mapping하는 network
    - Fully connected network를 통해 이 두 스테이지가 동시에 학습된다.
- 궁극적으로는 camera view, time variable, object movement를 컨트롤할 수 있다는 것이 핵심

## Problem definition

- Set of images가 입력으로 주어짐
    - 많은 양은 아니고 sparse함
    - 단안 카메라로 찍힘
    - dynamic scene, moving non-rigidly
    - 물론 tf도 포함된 입력
- 본 논문의 목적은 이런 입력을 기반으로 scene encoding하기
- 이를 통해 임의 시점에 대한 novel view synthesis도 가능하게 하자
- Formulation
    - Mapping에 대한 M: (x, d, t) → (c, sigma)로 보통 표현이 되는데
    - 이 맵핑을 두개 표현(?)으로 나누는 게 더 좋은 결과를 보였다고 한다.
    - 아래 첨자로 보아하니 x에 대한 것과 t에 대한 것으로 보인다.
    - Ψx represents the scene in canonical configuration
    - Ψt a mapping between the scene at time instant t and the canonical one
    - 가장먼저 이 Ψt 에 대한 configuration으로 transform을 수행
        - t=0일때는 0이라고 한다. (w.o/ loss of geometry)

## Method

- Canonical network: vanilla nerf같은…
- Deformation network: delta x를 입력 x, t를 기반으로 추정하는 네트워크. 이것을 deformation field라고 이야기하는 중 (time t에서의 transformation을 알아보는..)
    - tranformation: time t에서의 scene ↔ 이것의 canonical configuration 내에 있는 scene?(무슨말..)

## Canonical network

- canonical configuration을 사용한다고 계속 언급 중 - 어떤 공식적인? form에 대한 config를 말하는 것 같은데.. 어렴풋이..
- 특정 viewpoint에서의 잃어버린 정보를 이 canonical configuration을 통해 추정할 수 있다는 주장
- 입력 3차원 포인트가 256차원 벡터로 인코딩 된다..
- 카메라 viewing direction에 따라 concatenated
- 최종적으로 이게 fc layer를 통과한다.
- positional encoding과는 다른 무언가같은 느낌
- 출력값은 밀도와 컬러인 것은 동일

## Deformation network

- deformation field를 추정하게끔 최적화되어 있다.
    - deformation field: 특정 시점의 scene과 canonical space에서의 scene 간의 deformation 정의하는 것으로 보임
- x, t가 입력으로 들어가서 delta x를 내뱉어준다
    - delta x를 얻게 되면 원래 입력이었던 x는 x+delta x가 되는 것 (canonical space 안에서)
    - 일단 초기 t=0인 시점에서는 delta t는 0
- 일단 positional encoding 자체는 두 네트워크 모두에서 사용하긴 함
    - 동일한 인코딩을 한다고 한다
    - L = 10 (x), L = 4(t, d)

## Volume rendering

- camera ray: x(h) = o+hd
- near - far bound는 h로 정의
- 기본 너프 공식하고 비교하면 t라는 인자가 추가됨 (당연하겠지만 time)
    - T(h,t): 동일하게 accumulated probability
    - density, color 인자에 r(t)가 아닌 p(h,t)가 들어가있다.
    - p: ray + deformation field 값
        - p(h,t): 3D point라고 정의하는 중. camera ray 상에서의 3D point!
        - 단, deformation field 정의에 따라 transformed point 사용
    - []로 묶인 부분: 색과 밀도가 canonical network를 통해 추정됨을 의미
- 물론 이 식 역시 discretize

## Learning the model

- 두 네트워크가 MSE를 최소화하는 쪽으로 동시에 학습됨
- MSE: T개의 RGB image에 대한거라고 하는걸 보니 GT to rendered인듯?
- 근데 추가로 상응하는 cam pose에 대한 것도 mse로 최적화하나보다.
- 매 training batch마다
    - 처음에 pixel의 random sets을 sampling
    - 이 샘플링된 픽셀들에 대해 추정 결과와 GT를 비교
    - 일부 카메라 포지션 T로부터 출발한 ray
    - 그니까 t번짜 카메라 포지션(t:1~T)에 대해 랜덤 샘플링된 픽셀이 i번째야(i: 1~Ns)
    - 해당 포지션과 이에 상응하는 이;미지가 있을거고, 거기서의 픽셀 i위치에서 추정된 컬러 값이 GT와 일치하느냐!

## Implementation detatils

- 두 네트워크 모두 8개 layer의 MLP (with ReLU)로 구성되어 있다.
- 단, canonical에는 sigmoid non-linearity가 적용됨
- curriculum learning 전략 사용
    - 우선 이미지를 timestamp 순으로 sorting했다. (낮은 것부터 높은 것까지)
    - 그 다음 sorting 순서로 이미지를 학습 때 점차 늘려간다.
- Iteration 수가 무려 80만회
- GTX 1080기준 이틀걸림