---
title: "Attention Concatenation Volume for Accurate and Efficient Stereo Matching (ACVNet)"
date: 2022-07-07T17:47:06+09:00
author:
  name: Sunho Kim
menu:
  sidebar:
    name: ACVNet 논문 리뷰
    identifier: acvnet-review
    parent: depth-estimation
    weight: 3
draft: true
---

## Abstract

- A novel cost volume construction method라고 소개한다.
- Reliable attention volume 만들기란..?
- 제안: Multi-level adaptive patch matching
    - 패치 매칭을 썼나보다
    - 다양한 disparity 값들에 대한 matching cost의 distinctiveness 향상이 목적 (even for textureless region)
- 여기서 제안하는 cost volume의 이름이 attention concatenation volume - 스테레오 매칭 네트워크쪽에 접목시키기 좋은 방법이라고 주장

## Intro

- CNN 기반 스테레오 매칭은 다음의 네 스텝으로 진행되나보다
    - Feature extraction → Cost Volume Construction → Cost Aggregation → Disparity Regression
    - 주로 이 방법에서는 cost volume construction과 cost aggregation에 초점을 맞추는 듯
    - CNN으로써 이 모델은 Regression을 수행하나보다
    - 이전 사례
        - DispNetC: 양안 feature map으로부터 Single-channel full correlation volume 계산
        - GC-Net: 4D concatenation volume 만듦 (모든 disparity level 사이의 left-right feature map들을 쌓아서 만든다라..?!)
        - 등등등.. 잠시 생략..
    - 이런 류의 연구에서의 목적은 Cost volume 그럴싸한거 뽑아내는 것인가봐?
- 본 논문에서의 key observations
    - 풍부한 정보를 가지고 있는 concatenation volume
    - correlation volume: 특징 유사성..

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8851a898-259b-450e-a31d-d4d2c52141b6/Untitled.png)

- Step: initial concatenation volume construction → attention weights generation → attention filtering
- 그림을 먼저 알아서 해석해볼까..
    - 양안에서 feature 나온걸 concatenate 시키는 것 같은데 우선 이미지 특징 뽑듯이 뽑아서 쌓니? 어떻게 하나?
    - 양안 영상 간 weight는 share한다고?
    - 그리고 attention weight 만든 결과 자체를 쓰는 경우와 cost aggregation 두 경우 모두 depth map을 뽑을 수는 있나봐?
    - MAPM: multi-level adaptive patch matching
    - cost aggregation은 말 그대로 기본 스테레오 매칭에서 쓰이던 flow였었어 내 기억으론
    - 대충 그림만 보면.. 양안으로부터 얻은 concatenation volume이 있고, 병렬적으로 뽑은 attention weights map과 filtering 연산을 거치고 attention concatenation volume을 얻는다. 그렇게 되면 최종적으로 얻은 이 volume을 통해 cost aggregation을 수행하는 것으로 보임
    - 근데 attention weight 단독으로도 depth를 뽑을 수 있어? 그림이... 음...

## 토막상식 - Stereo matching

1. Census transform
    - 픽셀 블록 범위에서 동작
    - bit vector 계산: bit vector는 해당 블록에서의 image block 구조를 나타냄
    - kernel size: 5x5, 7x7, 7x9 사용
    - Classic census transform
        - block 내 center pixel과 이웃한 pixel들을 비교한다.
        - bit vector 내에 각 comparison 결과들을 저장하는 듯 (center pixel이 더 작으면 0, 더 크면 1을 저장한다.)
    - Modified census transform
        - center pixel이 아닌 전체 block 내 pixel의 평균값을 먼저 계산하고, block 내 pixel들과 이 평균값을 비교한다.
    - Thresholding census transform
        - 이웃 픽셀 값이 center pixel 값 보다 더 커야 하는게 아니라 threshold 값보다 확실히 더 커야 하는 방법
    - Census transform with Mask
        - 별도의 마스크를 만들어서 bit vector 구성요소로 쓸 지 말 지를 결정해주는 역할을 함
2. Cost matching
    - 특정 위치의 픽셀과 상응하는 위치를 찾기 위한 matching cost 계산
    - configuration parameter와 descriptor type에 따라 다른 linear equation이 cost 계산을 위해 적용된다.
    - disparity width: 64, 96 두 option이 있음
    - disparity companding
        - sparse matching을 이용한 무언가..
        - N disparity 내에 있는 pixel과 매칭을 하고
        - M disparity 내에 있는 2번째 pixel과 매칭을 하고
        - T disparity 내에 있는 4번째 pixel과 매칭한다
        - ex) disparity=96이라면 N=48, M=32, T=16이 될 수 있다.
        - 근데 이 결과는 depth map에만 영향을 미치고 원본 disparity map에는 영향을 끼치지 않는다고 한다.
    - Thresholding을 통해 특정 confidence 이상의 값은 reject
    - Linear equation parameters
        - combinational cost (알파와 베타 값은 user defined)
        $Cost = \alpha * |p_1 - p_2| + \beta * (Census\ Transform\ Cost)$
        - Census transform cost: 2 pixel 간 cost로, hamming distance 기반의 값을 얻는다고 한다.
3. Cost aggregation
    - **semi global block matching** 기반 - cost map aggregation이 목적
    - stereo matching 알고리즘에 inertia를 부여,.,?
        - 픽셀에 텍스쳐가 적으면 disparity 값이 이전 픽셀의 disparity에 가까울 확률이 높다.
        - 그렇기 때문에 low texture에서의 결과 정확도를 올릴 필요가 있다.
    - Cost parameter에 penalty를 주는 것 같다.
        - Horizontal penalty P1, P2
        - Vertical penalty P1, P2

## Related works

### Cost volume construction

- 현존하는 cost volume 표현법은 3가지로 크게 나눌 수 있다.
    - Correlation volume
    - Concatenation volume
    - Combined volume = Correlation + Concatenation volume을 concatenate시킴
- DispNetCorr
    - DispNet 나왔던 그 논문!
    - 거기서 correlation layer에 대한 부분이 나온건가
    - Correlation layer 활용
    - 이를 활용해 left - right image feature 유사도를 직접적으로 판별한다.
    - 이를 기반으로 각 disparity level에 대해 single channel cost volume 만든다.
    - 그 다음 2차원 컨볼루션으로 contextual information을 aggregate한다.
    - Correlation volume은 적은 메모리와 적은 계산 복잡성을 요구한다.
    - 그러나 encoded information이 너무 제한적
- GC-Net
    - Concatenation volume 활용
    - 여기서는 left, right CNN feature를 concatenate시킴 - 4D cost volume이 모든 disparity에 대해 만들어진다.
    - 이러한 볼륨으로 풍부한 content information을 보존한다.
    - 당연하겠지만 정확도는 correlation volume에 비해 높다.
    - 다만 유사도에 대해서는 별도의 measure를 하지 않은 볼륨인지라 처음부터 모든 disparity의 cost를 aggregate해야한다.
- GwcNet
    - Group-wise correlation volume을 제안하고, 이걸 concatenation volume과 쌓아 combined volume을 구성한다.
    - 근데 각각의 특성을 전혀 고려하지 않고 이 두 볼륨을 직접적으로 바로 쌓는 방법이 두 볼륨간 보완적인 강점을 사용하기에는 비효율적이라고 한다.
- Cascade cost volume?
    - 메모리와 복잡도를 더 줄여줄 수 있는 방법이라고 주장
    - cost volume pytamid를 만든다?! - Coarse-to-Fine 접근법
    - 당연하겠지만, 에러 누적의 문제가 있다곤 한다.
        - 이전 stage에서의 error가 발생한다면 나중 stage에서 이를 보완하기 쉽지 않다
- ACVNet
    - 다른 값을 갖는 변위에 대해서만 weight를 조정한다.
    - 물론 attention weights가 완벽하진 않을 순 있는데
    - Rich context를 보유하는 concatenation volume이 에러를 완화해주는 역할을 하기도 한다고
    - Subsequent aggregation network를 기반으로..

### Cost aggregation

- 여기서는 초기 cost volume에서 정확한 similarity measure를 하기 위해 contextual information을 잘 aggregate하는 것이 목표
- 해서.. 이 단계에서의 CNN 기반 접근들은 similarity function을 잘 얻어내기 위한 학습을 수행 (입력이 사실상 cost volume)
- 하지만 시간 제약이 있는 applications에서는 메모리나 계산 복잡 관련 소비가 너무 크다.
- AANet
    - 복잡도를 줄이기 위해 intra-scale, cross-scale cost aggregation algorithm 제안
    - 기존의 3D convolution을 대체했다고 하는데
    - 매우 빠른 inference가 가능했다고 한다만.. 정확도 저하라는 희생도 있었다고.. (심지어 사소하지도 않대)
- GANet
    - 여기도 3D convolution을 two guided aggregation layers로 대체했다고 한다.
    - Spatially dependent 3D aggregation 이용해서 더 높은 정확도를 취득
    - aggregation time은 늘어났다고..
    - 심지어 얘네 최종 모델은 15개의 3D 컨볼루션을 사용해서 시간이 어마어마…

- Cost volume construction과 cost aggregation은 서로 엄청나게 긴밀한 연결이 되어있는 방법들이다.
- Attention concatenation volume: 논문에서 제안하는 효율적인 cost volume 표현방법이라고 한다.
- 여기서는 유사도 정보(correlation volume에 encoded)를 사용해서 concatenation volume을 정규화한다고 한다.
- 결과적으로 더 가벼운 aggregation network를 구성할 수 있고, 정확도와 효율성도 높였다고 한다.

## Method

### Attention concatenation volume

1. Initial concatenation volume construction
- Input: H x W x 3의 stereo image pair
- 두 쌍에 대한 feature map f_l, f_r을 얻는다. (CNN을 통해)
- feature map size: N_c x (H/4) x (W/4) (N_c = 32)
- 초기 concatenation volume: 두 feature map을 각각의 disparity level에 대해 쌓는다.
$C_{concat}=Concat(f_l(x, y), f_r(x-d, y))$
- volume size: 2N_c x (D/4) x (H/4) x (W/4)

1. Attention weights generation
- 여기서는 초기 concatenation volume이 유용한 정보를 강조할 수 있게끔 하는 것이 목표
- 이 과정에서는 결과적으로 attention weight를 만들어낸다.
    - 두 쌍의 stereo image로부터 뽑아낸 correlation을 통해 geometric information을 얻음으로써
- 기존 correlation volume: pixel 단위에서의 유사도를 검사해서 얻었다
    - 이 방법이 텍스쳐가 없는 영역에 매우 취약했음
- 해서..제안하는 방법이 multi-level adaptive patch matching(MAPM)
    
    ![스크린샷, 2022-07-01 17-18-01.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/48f85954-c415-497c-ac09-830dd39d9d02/스크린샷_2022-07-01_17-18-01.png)
    
    - 3x3 커널과 각각 다른 rates가 적응적으로 weight를 학습한다?
    - a, b, c가 각각 3 레벨의 feature map들임
    - 패치 사이즈가 크면 contextual information을 더 다룰 수 있고, 이로 인해 matching cost를 식별하기 더 좋다
    (다른 disparity들에 대해)
    - 이 feature map들은 feature extraction 모듈에서 얻을 수 있다고 하고
    - 채널 수: 64, 128, 128 (l1, l2, l3)
    - 각각의 pixel에서 새까만 patch를 쓴다라..(사전에 정의된 사이즈로..)
    - 그러고 적응적으로 weight를 학습한다. (목적: matching cost 계산)
    - Dilation rate를 조정하면서 patch의 scope가 feature map level과 연관이 있음을 알 수 있었다고..
    - Center pixel의 similarity calculation을 위한 pixel number는 유지..
    - Similarity: 두 상응하는 pixel에 대해 weighted sum 진행(패치 안에서.. → 위 그림에서 빨간색, 오렌지색 픽셀 의미)
- 추가로 GwcNet에서의 group-wise idea도 차용
    - feature를 group안에 분할하고 group by group으로 correlation maps을 계산 (논문을 봐볼까..)
    - 세개 레벨의 feature map들이 쌓이고 (N_f-channel 형태…? N_f=320..인데 무슨 근거인디)
        - 320 = 64 + 128 + 128!
    - N_f 채널의 맵을 N_g 그룹으로 분할한다 (여기서 N_g=40..이건 무슨 근거?)
        - 첫 8개 그룹은 l_1으로부터
        - 중간 16개, 마지막 16개 그룹은 차례대로 l_2, l_3로부터
    - 각각의 feature map은 서로 간섭하지 않는다
    - g번째 feature group을 $f_l^g, f_r^g$라고 할 때, multi-level patch matching volume은 아래 공식으로 계산된다.
    
    []()
    
    - C가 각각 다른 feature map level에 대한 matching cost
    - <> 괄호 안에 있는건 두 맵에 대한 inner product
    - (x, y)는 당연하겠지만 픽셀 위치
    - d: 각각 다른 disparity level이라고 한다.
    - 오메가 기호: coordinate set(9개 포인트 - 빨강과 오렌지색..)
    - w: 픽셀 (i,j)에서의 weight (여기 값이 학습된다)
    - k가 patch level
    - 최종 multi-level patch matching volume은 각 level에 대한 matching cost를 쌓은 결과라고 한다. (그럼 3x1?)
    - 이거 size = N_g x (D/4) x (H/4) x (W/4)
    - 그런 다음 3D 컨볼루션을 두번 → 3D hourglass network 적용(GwcNet에 있음)
    - 다음 컨볼루션 레이어로 channel을 1로 압축한대 → 그 다음 attention weight를 파생 (weight size: 1 x (D/4) x (H/4) x (W/4))
- 서로 다른 disparity에 대한 정확한 attention weight를 얻기 위해..
    - 목적: initial concatenation volume을 제대로 필터링해내기..
    - Supervised method 사용하나보다. (GT disparity요구)
    - 아래 식을 적용하는데, 이건 GC-Net에서도 사용한 soft argmin function이라고 한다.
    
    []()
    
    - 이 식의 목적은 attention weights로부터 disparity estimation 결과를 얻기 위함
    - 계산된 disparity와 GT 간 L1 loss를 계산해서 이를 기반으로 attention weight가 잘 학습되게끔 가이드해준다!
1. Attention filtering
- Attention weight를 얻고 나면 이걸 이용해 initial concatenation volume으로부터 많은 정보들을 필터링해준다. (표현력 향상)
- 최종적으로 채널 i에서의 Attention concatenation volume은 Attention weights와 Initial concatenation volume 간 element-wise product로 계산
- 하나의 attention weight map이 모든 채널에 대해 동일하게 적용됨

### 관련 지식 - Hourglass network

### ACVNet architecture

1. Feature extraction
- 3개 level의 ResNet과 같은 구조를 사용(이것도 GwcNet)
- 처음 세 layer: 3x3 conv with stride 2, 1, 1 순서 (입력 다운샘플)
- 16개의 residual layer가 뒤따름 (1/4 크기로 축소된 feature map 얻기 위함) → l_1
- 그 다음으로 6개의 residual layer가 뒤따름 (large receptive fields + semantic information) → l2, l3
- 1/4 사이즈로 줄어든 모든 feature map들이 concatenate됨 (320 채널 feature map 생성)
    - 압축 전 이 원본이 attention weight 만들 때 활용됨
- 그 다음 두 개의 convolutions으로 320 channel짜리가 32 channel로 압축 (왜??) → Initial concatenation volume 만들기 위함이라고.. f_l, f_r이 이렇게 최종적으로 만들어짐

1. Attention concatenation volume construction
- 320channel feature map을 이용해 만든다.
- filtering output: 4D cost volume (모든 disparity에 대해)

1. Cost aggregation
- pre-hourglass module 이용
    - 4개의 3D conv (w/ batchnorm, relu)
    - 2개의 stacked 3D hourglass networks
        - 3D convolution 4개
        - 3D deconvolution 2개

1. Disparity prediction
- 최종적으로 3개의 output이 이전 단계에서 얻어지는데
    - 이거에 상응하는 disparity map이 만들어지는 것 같다 (결국 map도 3개)
- 각각의 output에 대해 2번의 3D conv를 통해 1 채널 4D volume이 나옴
- 이걸 업샘플링하고, probability volume으로 전환시킨다. (softmax)

[]()

- k: disparity level
- p_k: corresponding probability

### ACVNet-Fast

[]()

- ACVNet의 실시간 버전
- feature extraction 과정은 동일
    - 그러나 layer 수가 더 적고
- disparity prediction도 동일
- attention weight generation
    - Multi-level patch matching volume은 1/8 크기의 feature maps를 이용
    - 2개의 3D conv - 3D hourglass network
    - output도 1/8로..
- disparity search space 좁힘
    - h를 샘플링한다라..
    - h: hypotheses 의미인 듯. 6으로 설정
    - 원본에 절반 사이즈에 해당하는 disparity를 동일 크기로 업샘플링한 attention weight를 이용해 얻는다.
    - Disparity hypotheses
        - (h x (H/2) x (W/2))크기
        - 아래 범위 내에서 uniformly sampled
        - range: (d_att - h/2, d_att + h/2)
        - 즉.. h를 설정하면 해당 범위 하에서 어떤 가설맵을 만들어내는 건가보다.
    - 위 hypotheses map을 이용해 sparse concatenation volume을 만들고, attention weight를 sample한다.(sparse attention weight를 얻는 과정)
- Sparse라고 하는걸 보면 빠르고 정확도는 떨어지지만, 어느정도 어림 짐작은 할 수 있는 결과를 얻고자 하는 듯하다.
- 최종적인 sparse attention concatenation volume의 크기: (2N_c x 6 x (H/2) x (W/2)) (식 4를 참고..)
- Cost aggregation
    - 두개의 3D conv만 이용하고, 하나의 3D hourglass network만 이용

### Loss function

[]()

- lambda_att: predicted d_att에 대한 상관계수
- lambda_i: i번째로 predicted된 disparity에 대한 상관계수
- 그리고 아래는 ACVNet-Fast에 대한 final loss
    - d^f가 Fast 버전에 대한 final output
    

## Experiments

- .

### Ablation study

- .