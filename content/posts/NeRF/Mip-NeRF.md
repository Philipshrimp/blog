---
title: "Mip NeRF"
date: 2023-03-07T19:01:06+09:00
draft: true
---

## Abstract

- 고전적인 nerf는 픽셀당 하나의 ray만을 sampling하기 때문에 blurring 현상이나 aliasing 현상이 발생하기 매우 쉽다.
- 이런 현상은 특히나 상이한 해상도에 대해서 수행할 때 유독 발생한다.
- 직접적으로 픽셀당 다수의 ray를 이용해 렌더링을 하는 supersampling 방법을 NeRF에 바로 사용하는 것은 실용적이지 못해
- ray 하나당 MLP 모델 하나를 학습시킬 필요가 있기 때문이다.
- 이러면 시간이 몇백배로 소모가 될 것이야
- 그렇기 때문에 Mip-NeRF가 제안됨
- 콘 형태의 frustum을 사용해서 ray 사용하는 방법을 대체 → anti-aliased rendering 잘 하기가 목표
    - 이를 통해 Mip-NeRF는 objectionable aliasing artifact를 줄였고
    - Fine detail도 물론 잘 잡았다.
    - 심지어 속도 또한 오리지널 nerf에 비해 7% 상승했다고 한다.
    - 사이즈는 절반..? 모델 사이즈?
- 논문의 그림 1
    - 오리지널 NeRF는 x라는 point 하나를 ray로부터 샘플링해서 각 pixel에 투영되는 형식
    - Mip-NeRF는 3차원의 conical frustum(=원뿔 형태의 view frustum)을 사용
    - 해당 frustum은 카메라 픽셀로부터 정의된다고 한다(?)
    - 논문에선 추가로 Integrated positional encoding을 제안했나 봄
        - Multivariate Gaussian을 기반으로 근사화한다..?
        - Integral?
        - 뭔지는 자세한 내용 뒤에서 보자
- 논문의 그림 2
    - 원본 데이터셋에 대해 1/8로 리사이즈 해서 테스트한 결과인가보다.

## NeRF 선수 지식

- 뭔가 어렵게 써있어
- 기존 발표자료 참고하자…

## Method overview

- 기본적으로 NeRF에서 렌더링된 픽셀의 색상: 모든 incoming radiance(pixel의 frustum 내)의 Integration
- 그림 3번
    - NeRF는 point-sampled features
    - point-sampled feature는 각 ray에서 보이게 되는 volume의 size와 shape를 무시하는 경향이 있음
    - 그래서 만약 두 개의 서로 다른 view를 보는 카메라가 물체의 동일한 한 점을 본다라고 했을 때,
    - 심지어 스케일이 다르다고 하면
    - 모호한 형태의 두 point-sampled feature를 동일하게 만들어낼 것이다.
    - 아마 이러면 스케일링같은 크기 정보 고려되지 않은 채로 성능 저하를 유발할 것
    - 그래서 mip-nerf는 포인트가 아니라 그림처럼 영역 기반으로 본다
    - 그게 어떤 식으로 영역을 정의할지는 내용을 봐야할 듯

## Cone tracing

- 설명은 pixel 단위로 이루어짐ㅁ
- ray처럼 카메라 센터 포지션 o와 direction d로 이루어진 것은 동일
- 그리고 이 direction은 pixel의 center를 향한다.
- 다시 말해서, 원뿔의 꼭짓점은 o에 있다.
- image plane o+d 상에 있는 cone의 radius는 r_dot으로 파라미터화
- 여기서 r_dot은 pixel width로 설정한다고 한다.
    - 이 width는 world coordinates 상에 있고, 2/root12만큼 scaled(왜?)
    - 원뿔 밑넓이 공식같은거랑 연관있나?
- position 집합 x는 식 5와 같이 정의할 수 있다고 한다.
    - 여기서 영역의 범위는 t_0와 t_1 사이 기준
    - 1: indicator function (x가 frustum 안에 있으면 1이라는 뜻)
    - 공식 겁나 어렵게 생겼는디…

## Integrated Positional Encoding

- 도형안에 유니폼 샘플링하는 확률 함수가 있다고 생각
- 점 하나하나에 대해서 그게 1/볼륨 크기를 갖는다고 하면
- 그럼 전체 합은 1일거야
- 그럼 pdf로 볼 수 있겠지?

- 가우시안 근사 덕분에 positional encoing 구ㅏ하기는 편해졌다고 한다.

- 원통이 크다의 의미…?
    - 저해상도에서 mpinerf를 하면 원통 크기가 크다 (픽셀 사이즈가 크다)
    - 원통의 사이즈보다 작은 고주파 특징은 학습에 안쓴다라..
    - 

## Architecture

- 입력 feature에 scale을 명시적으로 encode하는 것이 가능하다?!
    - 이를 통ㅇㅇㅇ해 MLP가 multiscale 표현력을 가질 수 있게 한다.
    - MLP는 하나만 있대 그래서
    - 모델 사이즈 반으로 줄일 수 있다.
    - 렌더링 정확도 상승
    - 샘플링이 효율적
    - 전체 알고리즘이 단순해짐
- 최적화 문제는 공식 17번에 따라 푼다고 한다.
    - 각각 t_c, t_f에 대한 렌더링 퀄리티의 MSE를 계산
    - coarse loss는 하이퍼 파라미터로 가중치가 조정되고
    - fine loss에 좀더 타겟을 둔 것 같다.
- .