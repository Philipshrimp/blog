---
title: "CREStereo"
date: 2023-03-07T19:05:38+09:00
draft: true
---

# CREStereo: ****Practical Stereo Matching via Cascaded Recurrent Network with Adaptive Correlation****

[](https://arxiv.org/pdf/2203.11483v1.pdf)

## 파란 글씨: 2022년 7월 19일자 보충

## Abstract

- 실용적인 측면에서는 복잡한 요인들도 있고 (얇은 구조물 등) 해서 아직도 변위 추정이 challenging하다
- 실용적인 스테레오 매칭 제안
    - Fine depth detail을 더 잘살릴 수 있게 Recurrent refinement으로 hierarchical network 제안 → coarse to fine 방법으로 disparity update (inference 단계에서는 Stacked cascated architecture 사용)
    - Adaptive group correlation layer 제안 - 잘못된 rectification의 영향을 완화하기 위해
    - Dataset도 추가 제안.. (synthetic dataset인데 difficult case를 다뤄볼 수 있는..)
    

## Intro

- 기존 CNN 기반 stereo matching의 문제점들
    - 얇은 구조물이나 줄 같은 fine image detail을 캐치한 stereo matching에 대해서는 아직 약하다.
    - 완벽한 rectification 얻기란 쉽지 않아
    - hard case에 대한 dataset 불충분
- CREStereo: 현재 ETH3D two-view stereo와 Middlebuty benchmarks에서 둘 다 1위 차지
- Main contributions
    - Cascaded recurrent network: 실용적인 스테레오 매칭 위함, 해당 구조를 더 쌓아서 고해상도에서 활용도 쌉가넝
    - Adaptive group correlation layer: 이상적이지 못한 rectification 상황을 다루기 위함
    - New synthetic dataset: Real-world scene에 대해 보다 일반화된 데이터셋 제안

## Network overview

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d9edbc28-31ef-4d65-89bf-bafca85f3cac/Untitled.png)

- Input: I1, I2 두 stereo pair image가 들어간다.
- Feature extraction network
    - 두 이미지에 대해 각각 동작하나, shared weight
    - 그림을 보면 세 개의 conv layer가 있고 각각의 단계에서 feature map이 나오나 봄
    - output은 3-level feature pyramid인듯
    - 이 출력값은 각각 다른 스케일에 대한 correlations를 계산하는 것이 목적
    - 각각의 correlation은 cascaded recurrent network를 통해 구해지나보다. (계단식)
- 이미지에 대한 Feature pyramid: context information과 offset computation도 제공한다. (이미지에서의 문맥 정보란…?)
- Cascades의 각각의 stage에서
    - feature와 predicted disparity는 Recurrent Update Module을 통해 iterative하게 refined
    - 스테이지에서의 Final output: 초기치로써 다음 stage에 전달되나봄
- RUM 내에서의 각각의 iteration
    - Adaptive Group Correlation Layer가 적용됨
    - 이 layer를 기반으로 correlation 계산
- Inference
    - Image pyramid가 입력으로 들어가고
    - multi-level context를 활용한다고 한다.

## Method

### 1. Adaptive Group Correlation Layer

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/feec2d5e-2ae0-4f1b-b684-96dfafce6970/Untitled.png)

- 당연한 이야기겠지만 real-world에서 perfect calibration을 한다는 것은 불가능하다
    - 스테레오 baseline 맞추는쪽도…!
    - 한 예로.. OAK-D는 left right align한 형태로 나왔음에도 rectification을 거친다.
    - 결과적으로 이러한 배열은 3차원 공간 상에 약간의 rotation을 만들어낸다.
- 추가로.. lense로부터 나오는 이미지 자체가 왜곡이 있기도 해
    - 아무리 rectification이 이루어졌다고 해도
    - 여기서 이걸 residual distortion이라고 하네?
- corresponding point가 같은 scanline에 위치하지 못하는 에러가 결과적으로 발생한다.
- 그래서 논문에서 AGCL 제안
    - 이러한 상황들까지 고려해서 matching ambiguity를 줄이고자 함
    - 이전 방법들이 local correlation만 계산한 것과는 대조되는 방법이라고 주장함
- **Local feature attention**
    - 모든 pixel 쌍에 대한 global correlation을 계산하지 않고 local window 내에 있는 쌍끼리 local feature correlation을 계산한다.
    - 이유: 많은 양의 메모리 소모와 계산량을 줄이고자 함
    - LoFTR: Detector-free local feature matching with transformers
        - Sparse feature matching용인가?
    - 논문에서는 LoFTR에 attention module을 덧붙였다?
        - correlation 계산 전
        - cascade 첫 stage에서
        - 목적: global context information을 정합하기 위해 (single or cross feature map에서)
        - 거기에 backbone output에다가 positional encoding을 덧붙임 (NeRF에 나온거?)
        - 목적: feature maps의 positional dependence를 강화시키려고?
    - Attention layer에 대해
        - Self attention과 cross attention이 alternatively하게 계산된다고..
        - linear attention layer: 계산 복잡도를 줄여줄 수 있는 효과
- **2D-1D alternate local search**
    - 기존 RAFT의 Stereo version
        - 모든 pair의 correlation은 두 feature map C x H x W의 matrix 곱으로 계산
        - output이 4D의 H x W x H x W 혹은 3D H x W x W의 cost volume
    - CREStereo는 다른 방법
        - Local search window 내에서의 correlation 계산
        - Output volume이 훨씬 작아진다 (H x W x D)
            - H, W는 feature map의 width, height일테고
            - D는 correlation pairs 갯수 (근데 W보다 훨씬 작을거래..)
    - 이전 방법들
        - Search range가 foreground 물체의 최대 변위와 연관있었다.
        - 근데 이 써치 범위가 fixed이고, CREStereo가 제안하는 방법에 비해 큰 값인가봐
        - 이런 search range 값은 noisy result를 유발하나보다.
    - 추가로 CREStereo는 range를 미리 설정할 필요가 없다.
        - 모델이 각각 다른 baseline으로 stereo pair로 model을 일반화 할때..뭔소리야
    - position (x,y)에서의 local correlation은 아래 식으로 계산된다.
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e77e8af9-a90b-43b6-9be5-42d774ef1b6e/Untitled.png)
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/38ad4762-1f3d-45fb-9262-f06559cba067/Untitled.png)
        
        - F1: Resampled feature map
        - F2: Attended feature map
        - Corr(x, y, d)는 (H x W x D) 범위 내에 있고
        - 위 연산의 결과는 d(d는 (0~D-1) 사이)번째 correlation pair에 대한 matching cost
        - C: feature channels
        - f(d): fixed offset of current pixels in horizontal directions
        - g(d): fixed offset of current pixels in vertical directions
        - f, g는 하이퍼파라미터인가…
    - 고전적인 방법: 두 rectified image에 대한 search direction은 stereo matching에서의 epipolar line에만 의존했다.
    - 근데 practical한 방법에서는 이게 ideal하지 못하다는 점
    - 그래서 2D-1D alternate local search 방법을 사용했다는 것!
    - 목적: matching accuracy 높이기
    - 1D search mode
        - g(d) = 0으로 설정하고, f(d)는 -r과 r 사이에서 설정한다 (r=4로 둠)
        - f(d)의 양수 변위는 모든 iterative sampling 이후에 부정확한 결과 조정을 위해 보존된다?
        - 위에 local correlation 계산하는 식에 의해 나온 결과들이 concatenated될테고
        - 이게 최종 correlation volume이 된다.
    - 2D search mode
        - k x k grid를 쓴다?
        - 추가로 dilation l 수치를 추가로 설정 (dilated convolution과 유사한 접근법이 들어간다라)
        - 여기서 $k=\sqrt{2r+1}$ (output feature들이 서로 같은 channel 수를 가져야 한다. 1D와.. 같은 채널 의미하는 듯)
    - 2D와 1D가 shared weight인듯?
    - Alternate local search는 propagation module처럼 움직인대
        - network는 bias가 쏠린 예측 결과를 더 정확한 결과로 대체할 수 있도록 학습한대
- **Deformable search window**
    - 스테레오 매칭은 종종 occlusion이나 textureless 영역에 대해서는 모호성을 갖는다.
    - fixed-shape local search window를 사용해 corrleation을 계산하면 이런 문제에 취약하다고 한다.
    - Deformable convolution을 확장했다라..?
    - Content adaptive search window?
    - New correlation을 아래 식으로 계산됨
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9406475d-cb91-474a-95ca-1274b51f4bfa/Untitled.png)
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4044a294-6d70-41e0-b81d-8a2528eba147/Untitled.png)
        
        - additional offset dx, dy가 학습된 결과라고 하고..
    - 그림을 보면 이 offset이 기존 search window를 어떻게 바꾸는지 보인다고는 하는데..
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/eed9bfd2-d94c-42a5-afd1-a82ff4c02609/Untitled.png)
        
        - 2D와 1D는 같은 수의 searched neighbors를 공유한다고 한다.
        - 이는 같은 모양에서 correlation map을 만들기 위함이다라..
        - 문제점.. 그림만 두고 제대로 설명이 없다. deformable convolution을 봐야하니?
- Group-wise correlation
    - 이쪽은 GwcNet으로부터 영감을 받았나봐 (4D group-wise cost volume)
    - feature map을 G 그룹으로 나누고 각각의 local correlation을 계산했다.
    - 그 각각에 대해 G개의 correlation volume이 나오면 이걸 concatenate시켰다.
    - 이를 통해 GD x H x W의 볼륨을 얻는다라..

### 2. Cascaded recurrent network

- Receptive field가 크고 semantic information이 충분한 low-resoution feature map(그러니까 나중 layer에서 나오는 것)은 non-texture와 texture가 반복되는 영역에 대해서는 강점을 보인다고 한다.
- 근데 얇은 구조물이 있는 영역 등에 대해서는 이 자체를 잃을수도 있다.
- 목적: robustness + preserve the details
- 그래서 correlation 계산과 disparity 갱신을 위한 Cascaded recurrent refinement가 제안됨
- **Recurrent update module**
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0b0e0870-1861-457d-b13a-ad4c7cbdb323/Untitled.png)
    
    - GRU block와 AGCL로 구성되어 있다.
    - RAFT가 계속 비교대상으로 올라옴..
        - 얘는 single correlation layer에서 feature pyramid가 올라오나봄?
        - output은 물론 하나의 volume으로 merged
    - 제안하는 방법에서는
        - 모든 feature map 각각에 대해서 correlation을 계산한다
        - 근데 그 계산이 각각 다른 cascade level에서 행해짐
        - 그리고 각각 독립적으로 iteration 몇번을 통해 disparity를 보정
    - 다시 AGCL 구조 그림을 보면
        - sampler는 grouped feature의 location을 sampling하는데
        - f_n(n번째 iteration에 대한 prediction 결과)을 입력으로 하고
        - 거기서의 coordinate grid로부터 얻는건가봄..? (약간 말이 어려운디)
    - correlation volume construction
        - learned offsets과 함께..! (offset 범위: 2 x (2r+1) x h x w)
        - 범위가 왜 이렇게 나오는데…?
            - 2r + 1은 왠지 k x k 의미하는거 같고
            - 그게 두개라는건 stereo pair?
    - GRU block: 현재의 prediction을 update하는 역할을 하고, 그 결과를 다음 iteration의 AGCL 블록으로 보낸다.
    - 그림을 통해 과정을 알아보자면
        - 입력으로 offset, left feature, right feature가 들어온다
        - 세 입력값이 먼저 AGCL 블록으로 들어가고
            - 여기서 left, right feature는 cross attention을 통해 grouped features를 만들고
            - right feature 쪽에서 sampler를 거치는데
                - 여기서 sampler input은 이전 prediction 결과가..!
                - 근데 첫 AGCL에서는 아무것도 안들어가는 듯한 그림인데..?
            - 무튼..결과적으로 right feature에 대해서는 sampled feature가 나오고, left feature는 그 자신 그대로
            - 다음으로 adaptive local correlation을 offset이 함께 임력으로 들어가며 수행됨
            - 결과적으로 grouped correlation 결과들이 나오는데
            - 최종 correlation block은 이 결과들을 concatnate!
        - correlation block은 GRU block의 입력으로 들어가서
        - prediction 결과를 도출한다.
            - 도출: f0와 델타f1을 더한다..?
            - refinement value를 출력해서 이전 iteration 결과와 더하는 듯
        - 이게 이제 반복적으로 수행됨!
    - 잠시 GRU가 무엇인지 보자
        - 일단 RNN 자체는
            - 히든 노드가 방향을 가지고 서로 연결되어 순환 구조를 이룬다고 한다
            - 시간이 지난 사건을 네트워크에 반영이 가능하다는 장점
            - 순차적으로 등장하는 데이터 처리에 적합함
                
                ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b82c03d1-4314-4815-9675-78957de2903e/Untitled.png)
                
            - 순환 구조이기 때문에 앞단계의 히든 레이어로부터 들어오는 gradient도 같이 더해진대
        - 여기서 LSTM이 다음에 나오는데
            - RNN이 vanishing gradient로 인해 오래된 과거정보를 이용할 때는 학습이 잘 안됨
            - RNN의 hidden state에 cell-state를 추가한 구조라고 한다.
            
            ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/51833155-8296-4482-959e-ae5bbb1b9ca4/Untitled.png)
            
            - 이 구조로 인해 gradient 전파가 잘된다.
            - forget gate에서 어떤 정보를 버릴지 결정하고(sigmoid)
            - 새로운 정보 중 어떤 것을 저장할 지를 이 구조에서 결정
            - sigmoid layer가 어떤 값을 업데이트할 지 결정하고
            - tanh layer가 새로운 후바 값을 만들고 cell state에 더할 준비
            - 이 것들을 이용해 cell-state를 update하고 output을 도출한다.
        - 그 다음 파생된 GRU는
            - LSTM 구조를 더 간단하게 개선
            
            ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/29267507-29c7-4363-8b3d-523f3a4c561b/Untitled.png)
            
            - reset gate, update gate 두 개만 사용
            
            ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/85c9953b-a74a-487e-98af-950fe7c2efe1/Untitled.png)
            
            ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f9f10dab-20dc-4acb-bdd4-4e719417c547/Untitled.png)
            
            - 그림만 보면 forget gate, input gate 두 개만 이루어진듯. 즉, output gate가 별도로 없고 그대로 가나봄?
    - 여기선 왜 GRU가 들어갔을까..?
        - 
- **Cascaded refinement**
    - cascade의 첫 레벨
        - input resolution의 1/16 크기에서 출발
        - 그리고 disparity가 초기에 all zero로 초기화되어 있는 상태
    - 나머지 레벨
        - 이전 레벨 대비 upsampled version of prediction을 초기 값으로써 갖는다.
    - 모든 RUM은 same weight를 공유한다.
    - 마지막 refinement level에서는 convex upsampling (이것도 RAFT에서 썼대)을 통해 input resolution과 동일한 크기의 최종 예측 결과를 얻는다.
        - The network outputs optical flow at 1/8 resolution. We upsample the optical flow to full resolution by taking the full resolution flow at each pixel to be the convex combination of a 3x3 grid of its coarse resolution neighbors. We use two convolutional layers to predict a H/8×W/8×(8×8×9) mask and perform softmax over the weights of the 9 neighbors. The final high resolution flow field is found by using the mask to take a weighted combination over the neighborhood, then permuting and reshaping to a H ×W ×2 dimensional flow field. This layer can be directly implemented in PyTorch using the unfold function.
        - 네트워크 아웃풋이 1/8크기 옵티컬 플로우인데
        - 풀 레졸루션으로 업샘플이 필요해
        - We use two convolutional layers to predict a H/8×W/8×(8×8×9) mask and perform softmax over the weights of the 9 neighbors.
        - 아래부터 추가 보충..
    - Convex upsampling
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/422e2424-44cf-46a4-8f62-68cb55b30ecb/Untitled.png)
        
        - Full-resolution의 optical flow 이미지가 Convex combination이다라..
        - 3x3 weighted

### 3. Inference

- 트레이닝 과정에서는 3개 레벨의 feature pyramid를 통해 hierarchical refinement를 거쳤는데
- 더 큰 resolution에서는 downsampling이 더 필요하다?!
    - 목적은 receptive field 키우기 위함
    - 근데 small object with large disparity 케이스를 생각하면 direct downsampling 쓰는게 성능이 악화될거야
- Stacked cascaded architecture를 inference에서 쓰기 위해 제안
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b02bedbd-c21a-4b46-85b7-989889a32ac4/Untitled.png)
    
    - 이 구조에서는 shortcut 방법이 추가됐나봐?
    - image pyramid를 만들기 위해 image pair downsampling을 미리 했나봄
    - 그 다음 이 pyramid를 train된 feature extraction network로 보낸다
    - 목적은 multi-level context 활용하기 위함
    - 그림에는 생략되어 있는데 같은 stage에서는 skip connection이 있다고 한다.
    - 모든 stage는 same weight를 쓴다고..
    - 그림보고 유추해보면..
        - RUM에 대한 weight는 train 과정에서의 것들을 단계별로 가져오는 느낌
        - feature extraction 자체도 기존에 train한 구조 그대로 쓰고
        - fine-tuning 별도로 불필요

### 4. Loss function

- 각각의 stage s는 1/16, 1/8, 1/4를 나타냄 (scale에 대한 부분인 듯)
- 각 iteration에서 나오는 feature map f1~fn은 input과 동일한 크기로 upsample되어 각 feature map에 대해 loss가 계산되는 식인가 봄
- Exponentially weighted l1 distance?
    - gamma = 0.9
    - gamma 값에 n-i승 만큼 제곱이 가해진달까.. 이렇게 weight가
- total loss
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2ef3373f-2d42-425f-a976-82f5e0bf00f4/Untitled.png)
    
    - 즉, 모든 s와 모든 n에 대해 계산되는데
    - n은 말했듯이 iteration
    - weight 부분을 보면.. gamma가 소수점이니 곱할수록 작아지겠지
    - n-i니까 i가 n에 가까울수록 0에 가까워질테고, 즉, n에 가까울수록 원 값인 0.9에 가까워지지
    - 그 말은 즉, 초기 iteration에는 가중치가 적게 부여되고, 나중 iteration은 뽑고자하는 최종 output과 가까우니 가중치가 많이 부여됨
    - 근데 왜 모든 iteration마다 계산했을까? 이건 좀 생각해볼법하다..