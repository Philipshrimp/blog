---
title: "NerfingMVS"
date: 2023-03-07T19:10:47+09:00
draft: true
---

## Scene-specific adaptation

- 각 scene별로 depth network를 동작시켜 scene에 특화된 결과를 얻으려고 시도 - final depth output의 퀄리티 상승을 위함
- test scene에 COLMAP을 돌려 각 view에 대한 sparse depth를 얻음
- 얻는 방법: 3D sparse point를 각 view에 projection
- 이 결과가 scene-specific depth prior 뽑을 때 가이드로 사용됨
- Scale invariant loss
    - scale의 모호성을 없애기 위한 방법
    - 근데 알파 값이 무슨 도움이 돼 대체..

## Guided optimization

- per-view depth를 뽑는 기본적인 방법은 렌더링 공식과 유사
- shape-radiance ambiguity
    - Incorrect geometry 결과가 나오기도 한다는 점
    - 이런 문제가 poorly textured area에서 특히 발생하는데
    - 벽이 있거나 한 indoor scene에서는 이게 정말 취약해
- Error map
    - geometry consistency check를 통해 취득
    - T: relative pose
    - p: 2d coord
    - D: depth
    - 각 view에 대해 e를 정의 (error value?)
- Error value를 기반으로 adaptive sampling range를 정의함
    - alpha: relative lower / higher bounds of range (하이퍼파라미터)
    - clamp (e, alphas): error value가 나왔을 때, 해당 alpha range 안으로 조정시켜준다고 함
        - 즉, 알파 범위를 벗어나면 해당 알파 값을 리턴해주고
        - 알파 범위 안에 e가 있으면 해당 e 값을 리턴
        - 픽셀 샘플링 결과가 적은 에러를 가지고 있으면 adapted depth prior에 좀 더 집중
        - 큰 에러를 가지고 있으면 오리지널 너프 결과에 초점

- 이쯤되면 궁금한 점
    - loss는? - 코드 보는 중
    - Depth prior
        - .
    - NeRF
        - MSE loss - rendering quality

## Inference & View synthesis

- Adaptive range 구했던 걸 직접적으로 inference에 적용
- per-pixel confidence 사용
    - 렌더링 결과를 기반으로 비교
    - 이를 기반으로 필터링 수행?
    - Plane bilateral filtering (Google ARCore에 사용됨)
    - ARCore에서는 planar bilateral solver라고 하고 있다.
    
    https://github.com/tvandenzegel/fast_bilateral_space_stereo
    

## Implementation detail

- Mannequin Challenge 논문에서 소개된 구조를 기반으로 depth prior를 얻음
    - pretrained weight 사용
    - 15회의 finetuning epochs: 목적은 scene-specific adaptation
- NeRF 구조도 사용
    - neural radiance field 네트워크 최적화를 위해 별도의 coarse-to-fine 없이 바닐라 너프씀
    - 네트워크 regularization: Gaussian random noise 사용 (??) - 이를 밀도에 사용
    -