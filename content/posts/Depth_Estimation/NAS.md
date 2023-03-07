---
title: "Normal Assisted Stereo Depth Estimation"
date: 2023-03-07T19:06:23+09:00
author:
  name: Sunho Kim
menu:
  sidebar:
    name: NAS 논문 리뷰
    identifier: nas-review
    parent: depth-estimation
    weight: 3
draft: true
---

## Intro

- 최근에 딥 러닝 기반 MVS 알고리즘들이 제한된 뷰에서 경쟁력 있는 결과를 보여줌
- 근데 교차되는 뷰에 대한 correspondences를 얻기 힘든 경우에는 만족스런 결과를 보이지 못함 (교차뷰라는 말이 헷갈리는데 결국 correspondences 찾기 어려운 장면을 케이스로 드나보다)
- Training time에서 뎁스와 노말 간 일관성을 강화해서 성능을 끌어올리는 것이 이 논문의 목표
- 멀티뷰 노말과 뎁스의 학습을 묶었다.
- 새로운 consistency loss 제안: 목적은 독립적인 consistency module 학습을 위함 (이를 통해 depth/normal 쌍으로부터 depth를 refine한다)
- 논문 주장으로는 joint learning을 통해 성능 향상을 보았다고 (정확도, smoothness 모두

## Overview

- 엔드 투 엔드
- 모듈은 크게 둘로 나뉜다.
    1. Cost volume으로부터 depth와 normal을 추정하는 모듈 (Cost volume은 멀티 뷰 이미지 feature map으로부터 얻는다.)
        1. 여기서 normal을 추정하는 것이 depth 추정 성능 향상에 암시적으로 영향을 끼쳤다
    2. 일관성 강화를 통해 추정된 depth를 정제하는 모듈 
        1. 일관성 강화를 위한 명시적인 training이 진행된다고..(..?)

## Learning based plane sweep stereo

- 일반적인 stereo matching에서의 targets: single object reconstrution vs. Scene reconstruction
    - Scene reconstruction은 보통 다수의 객체가 포함된 형태
    - 그렇기 때문에 이미지 처리에 있어 더 큰 receptive field가 요구되고 있다.
    - 이러한 이유에서 DPSNet을 기반으로 depth prediction module을 구성
- 흐름
    - 입력: reference image I1과 neighboring view image I2를 사용
    - 둘은 같은 장면을 담고, 같은 카메라 내부 파라미터를 가지고 있으며, 둘 간의 외부 파라미터를 취득한 상태여야 한다.
    - SPP module을 통해 feature map을 얻는다.
    - plane sweeping과 3D cnn을 통해 cost volume을 얻는다. (그림에서 cost 0번이 plane sweeping, cost 1번이 3D CNN 결과인듯)
    - 만약 뷰가 더 많다고 가정하면 cost volume이 다수 생성될 것이고, 최종 볼륨은 이들의 평균 값으로 얻어진다.
    - Noisy한 cost volume을 regularize하기 위해 context-aware cost aggregation 사용
    - Final depth regression: soft argmin
- DPSNet에서..
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3c2bbc1e-9cd1-4c9b-bccd-242725b04087/Untitled.png)
    
    - Feature extraction
        - 기본 구조가 7개의 컨볼루션 레이어 (처음만 7x7, 나머지는 3x3)
        - extract hierarchical contextual information (SPP module)
        - 이를 통해 멀티 스케일 특징을 얻는다.
        - 멀티스케일 얻으면 업샘플링을 통해 오리지널 feature map과 동일 사이즈로 만들고
        - 모든 feature map을 concat 시켜 2D conv로 통과시킨다.
        - 이를 통해 모든 입력에 대해 32채널 feature representations를 얻는다.
    - Cost volume generation
        - 여러 이미지를 사용해서 평균 cost volume을 만드는게 잡음에 강인하다곤 한다.
        - virtual plane 갯수를 설정한다. (reference viewpoint의 Z방향으로, L개라고 하자.)
        - 그리고 depth 값은 virtual plane 갯수에 맞게 uniform sample
            
            ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1dfd8ba7-a5c5-4925-83ff-239847a64e5f/Untitled.png)
            
        - d_min: minimum scene depth(사전 정의)
        - N개의 viewpoint가 있을 때, 이 viewpoint 내에서 reference feature와 pair한 feature들을 warping
        - Feature volume: 전체 L개 plane에 대해 W x H x 2C x L 크기의 볼륨
            - L: sweeping plane 갯수
            - 2C: reference feature map과 warped image feature를 concat
            - 근데 두개만 쌓는 것을 보니 이러한 볼륨 자체는 camera view가 많을수록 다중의 cost volume이 나오나보다.
        - 기본적으론 동일 카메라로부터 얻는 것을 전제로 설명했으나, 입력을 잘 바꿔주면 다른 카메라 간 알고리즘 수행도 가능하다.
        - 여기까지 쌓였으면 이를 입력으로 최종적인 WHL 사이즈의 cost volume을 얻는다.
            - 얻는 방법은 3D CNN
            - training: 1쌍만 사용했다고 하고
            - test: 임의의 쌍으로 마음대로 넣었대
    - plane sweeping algorithm (아직도 잘 모르겠다)
        - Window-based stereo approach는 경사면 같은 부분에 있어 매우 취약한 결과를 보임
        - 이러한 부분에 대해서도 강인하게 추정할 수 있는 방식으로 plane sweeping 방식을 사용
        - 기본 아이디어는 이미지 set을 3차원 공간 내 가상 plane으로 역투영하는 것
        - 그리고 역투영된 결과를 바탕으로 photometric-consistency를 체크 (대상: each pixel, warped images들 간)
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3afb8054-1b77-43bc-b176-8f9311acb686/Untitled.png)
        
    - context-aware cost aggregation
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/128793c5-e6aa-4fd4-8fb5-11b4ee174a3e/Untitled.png)
        
        - 불안정한 매칭 때문에 발생하는 영향을 regularization을 통해 완화하는게 목적
        - cost volume의 개별 slice와 reference image feature를 context network로 보낸다.
        (참고로 전체 slice에 대해 weight는 sharing)
        - context network는 2D convolution으로 구성
        - residual: context network로 나온 값과 initial volume을 더해 aggregated volume을 얻는다.
        - reference feature map을 사용하기 때문에 문맥 정보를 제공해주기 좋다고 한다.
        - 목적은 값의 차이가 극단적인 경계부분을 파악하고, 이를 명확하게 하는 것 → 영역 구분이 확실해지고, 해당 영역에 속하는데 아닌 것 같은 값이 나온 잘못된 부분들을 완화한다.
        - **여기서 나온 볼륨으로 두번째 추정 뎁스를 얻는대 (무슨 차이인가)**
- soft argmin에 대해 (GCNet)
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ac8ca336-4da1-4135-8c7a-e7919fed584e/Untitled.png)
    
    - regression에서 일반 argmin을 사용한다면 다음의 두 가지 문제점을 앉고 있다.
        - 결과가 discrete하고, sub-pixel disparity estimation을 하기 어렵다.
        - 미분 불능이라 back-propagation이 문제
    - smooth한 depth 추정과 differentiable하게 만들기 위해 soft argmin 제안
    - 과정
        - 이전 단계에서 얻은 cost(cost volume으로부터 얻은 각 disparity에 대한 cost)를 probability volume으로 convert한다.
        - 이 볼륨은 각 value에 대해 negative한 값을..? (그니까 음수 취한다)
        - 이후 disparity dimension에 따라 이 볼륨을 normalize (여기서는 softmax 사용) (위 식에서 시그마)
        - 이 확률을 기반으로 가중치를 주고, 가중치를 곱한 disparity를 더한다. (즉, softmax 결과는 일종의 가중치가 된다.)
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7b26d24f-e214-4118-a94c-fa0820c785f6/Untitled.png)
        
        - 일종의 soft attention mechanism과 비슷한 맥락이라고 이야기하기도..

## Cost volume based surface normal estimation

- 논문에서 NNet이라고 명명하는 그 네트워크를 기반으로 추정한다.
- Cost volume에는 모든 spatial information을 담고 있을 것이다 (거기에 image feature까지)
- Probability volume에는 depth 분포를 모델링함 (candidate planes에 대해)
- candidate plane 갯수가 무제한이라는 제한적인 케이스(??)에서 → 정확히 추정된 확률 볼륨은 표면을 설명하는 암시적 함수 표현이 될 수 있다.
    - 즉, surface에 포인트가 존재하면 1, 그 외의 영역은 0으로 표현한다고 한다(…?)
    - 여기서 Cost volume 1번을 사용하는 아이디어를 얻었다고 한다.. (3D 컨볼루션 거친 결과물)
    - 이 볼륨은 이미지 레벨의 특징을 포함하여 노말 추정에 사용함
- 방식
    - 주어진 코스트 볼륨에 월드 좌표계 값을 concat 시킨다 (그림에 나와있음)
    - 그림에서의 world coordinates volume: 모든 복셀의 좌표값을 말하는데, feature map에서 해당 상응하는 voxel 영역에 대한 월드 좌표 값을 쌓는다
    - 그 다음 3개 레이어의 2-strided convolution을 사용한다고 한다. (그림에선 3D CNN이라 나왔다. width, height, features 3차원인듯)
    - depth dimension의 차원 축소를 수행해서, 1/8로 줄인다. (이 아웃풋을 C_n이라고 부르는 중)
    - 이거를 depth 방향으로 slice한 ((f+3) x H x W) 크기의 3차원 슬라이스 맵을 S_i라고 하고 (전체 뎁스 차원 중 i번째인듯)
    - 이 각각의 슬라이스를 normal-estimation network로 흘린다.
    - 네트워크 구성: 7개의 3x3 2D conv 레이어들로 구성
    - receptive field 늘리기 위해 (1,2,4,6,8,1,1)의 수치를 준(?) dilated convolution으로 각각 구성됨
    - 출력은 3 x H x W (노말 맵이니까 3채널인듯)
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/429d8a0c-9475-412c-8368-682bcc4c7048/Untitled.png)
    
    - 각 슬라이스로 나온 전체 결과를 다 합하고 노말라이즈 한다.
- 왜 이렇게 했을까?
    - 각 슬라이스는 모든 뷰에 대해 픽셀별 패치매치 유사도와 상응하는 정보를 담는다.
    - 이전 단계에서의 strided convolution으로 인해 특징 값은 정보를 축적해서 담고 있다. (이웃 평면간의 그룹에서의 정보를)
- 특정 픽셀에 대해서.. GT depth와 근접한 값을 갖는 slice는 좋은 노말 추정 결과를 도출한다.
- 반대로 GT와 멀면 0값을 내놓는다?
- 그냥 다시 말하면 노말 추정 성능은 뎁스에 비례하다.
- 추정된 노말값에 대해
    - magnitude: 해당 픽셀에 대한 해당 슬라이스에서의 correspondence probability처럼 보임
    - direction: 슬라이스의 픽셀 주변 로컬 패치에서 강한 correspondence와 정렬된 벡터로 보임
- 학습에 대해
    - GT depth를 기준으로 Z1, Z2 추정
    - GT normal map을 기준으로 n을 추정
- loss function
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1d24b8b9-aa81-4a29-b18f-36dcc43019c3/Untitled.png)
    
    - Huber Norm으로 계산한다.
    - depth loss: 2번 뎁스와 GT 간 로스를 기본으로 가되, 어느정도의 가중치를 주어 1번 뎁스에 대한 로스도 같이 계산한 듯 보인다 **(왜 그랬을까?)**
    - normal loss: 노말 로스 자체가 가중치가 들어가있는 상태

## Depth normal consistency

- 카메라 모델을 활용하여 뎁스의 spatial gradient를 추정 (픽셀 좌표계에서)
- 여기서는 픽셀 좌표계 중 u, v 각각에 대한 그래디언트를 얻는다.
- 핀홀 모델을 식으로 나타내자면

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ce796dbb-e7f2-41e6-9d46-881ba4af4631/Untitled.png)

![스크린샷 2022-11-22 오후 8.30.58.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7028ccc2-ea8f-4717-8255-942d3baf102a/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-11-22_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_8.30.58.png)

- 이를 기반으로 X, Y와 이의 u에 대한 gradients를 추정하면

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7fc10089-6d45-49b1-b23e-cf588b924c58/Untitled.png)

- Estimate 1)
    - Depth map의 spatial gradient는 처음에 Sobel filter를 이용해서 depth map으로부터 계산되어짐
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9367b158-3c4a-472a-b4ac-ad82a1e4d271/Untitled.png)
    
- Estimate 2)
    - 기본적인 전제 scene이 implicit function F(X, Y, Z)=0으로 표현될 수 있는 smoothed surface라고 가정
    - 노말 맵은 이러한 surface의 gradient 추정치라 할 수 있다.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9342ae5b-e384-419e-9057-ad780e1d52a6/Untitled.png)
    
    - 이를 토대로 뎁스의 2차 추정을 한다면
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/df0e978a-71f0-4fe8-97e8-1792e18ef4be/Untitled.png)
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9de91039-0a63-4bc8-85a7-70384d4bdac5/Untitled.png)
    
- Consistency loss
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/82409908-e76c-45c4-a46b-db4c16da1d45/Untitled.png)
    
    - 1차 추정 결과와 2차 추정 결과를 비교하여 그 차이가 적은 것을 일관성 로스로 취급
    - Huber norm을 계산한다.
- ~~2차 추정 결과는 해당 픽셀에서의 절대적인 뎁스 값에 의존한다. (이웃 뎁스에는 의존하지 않음)~~
- ~~노말 맵으로부터 로컬한 표면 정보를 얻을 수 있는데, 이로 인해 뎁스를 더 정확하고 쉽게 추정할 수 있다고 주장한다.~~
- ~~제안된 consistency loss는 이웃한 픽셀 간 상대적인 depth constraints를 강인하게 할 뿐만 아니라 절대적인 depth 값 자체도 강인하게 한다고 한다.~~
- ~~이전에 제안된 방법들 대비 차이점~~
    - ~~이전에는 월드 좌표계에서 제약을 걸었고~~
    - ~~제안된 논문은 픽셀 좌표계에서 제약을 걸었다.~~
- 이 loss를 독립적인 모듈로 구현했다
    - UNet 기반?
    - 입력: raw depth and normal estimates (직접 구한 뎁스 노말을 말하는건가?)
    - 목적은 이를 기반으로 정제된 결과를 얻는 것
    - 코드를 보니 독립적으로 별개로 하기보단 하나의 트레이닝 파이프라인에 녹여있음
    - 구현 자체만 클래스로 독립 구현한거 같다.
    - 초기에 first module을 loss를 기반으로 학습
    - 그 다음 second module을 붙혀 cons loss와 기존 loss 두 가지를 기반으로 학습을 한다.
- 코드를 보면..
    - train_cons 아규먼트를 붙여서 이걸 추가하는지 마는지 결정하는 것으로 보인다.