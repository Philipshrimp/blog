---
title: "Mobilenet"
date: 2022-07-07T17:40:33+09:00
draft: true
---

가장 먼저 PR12를 보고 정리해보자!

## PR-044: MobileNet

![스크린샷 2022-02-22 오후 9.37.06.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3139c3d9-200a-410b-a34e-8418e4bc6e12/스크린샷_2022-02-22_오후_9.37.06.png)

[PR-044: MobileNet](https://www.youtube.com/watch?v=7UoOFKcyIvM&list=PLlMkM4tgfjnJhhd4wn5aj8fVTYJwIpWkS&index=44)

- 2017년에 Google에서 발표한 논문
- 부제: Efficient convolutional neural networks for mobile version applications

![스크린샷 2022-02-19 오전 11.18.30.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ba30f65d-9578-46ae-9acd-b7d7e9fcf7a4/스크린샷_2022-02-19_오전_11.18.30.png)

- 아이디어 자체는 Xception(유재준님 발표 논문인데.. 이게 뭘까?)하고 크게 다를바 없을 정도로 심플한 키 아이디어
- 다양한 분야에 이 논문을 적용해보고 좋은 성능을 보여주었다라는 것이..
- 연산량, 파라미터 수를 줄이면서 효율성을 올리는 네트워크를 제안

- 컴퓨터 비전 분야에서의 상용화 측면에서 고려해야할 사항들
1. 데이터센터 : 서버 말하는 거
    1. safety에 관한 부분은 크게 중요하지 않다
    2. 저전력..은.. 뭐 그냥 있으면 좋겠지 꼭 필요한 것도 아냐
    3. 실시간성도 있으면 좋은거고
2. 가젯?(스마트폰, 자율주행 차, 드론 등..)
    1. 스마트폰을 제외한 대부분의 경우에는 safety가 중요해
    2. 그리고 저전력과 실시간성이 매우 중요할 것이다

- 위에 가젯류에 해당하는 작은 장비들에서 딥러닝을 사용할라면 무엇이 중요할까?
    - SOTA급은 아니더라도 충분히 높은 정확도는 필요해
    - 시간 복잡도가 낮아야 한다
    - 에너지 사용도 적어야 한다
    - 모델 사이즈가 작아야 한다
        - 작게 만들면 트레이닝을 더 빨리 할 수 있다(적은 리소스로)
        - 임베디드 프로세서 등에 들어가기도 적합해
        - Over-The-Air...?
            - 소프트웨어 업데이트를 통신망을 통해서 쉽게 할 수 있어야 하는 것
            - 업데이트된 모델이나 웨이트 등의 사이즈가 크면 업데이트가 쉽지 않을거야

- Small Deep Neural networks를 만들기 위해선..?
    - Fully-connected layer를 없애야 한다
        - CNN기준
        - 파라미터의 90퍼센트 정도가 FC layer에 있어 (Parameter sharing을 안하잖아)
    - 커널을 축소해야 한다
        - 3x3 → 1x1?
        - 연산량이나 파라미터 수를 줄이기 위함
        - squeezenet이 대표적
    - 채널 수가 갈수록 뚱뚱해지니까.. 이것도 줄여야 해
    - Evenly spaced downsampling
        - 무조건 작게 만드는데에 초점을 맞추는 것이 아니라
        - 보통 pooling이나 stride 값을 준 convolution을 통해 다운샘플링을 하는데...
        - 초반에 다운샘플링을 많이해서 후반부 네트워크로 흘러버리느냐 vs. 처음에는 최대한 다운샘플링을 안하다가 막판에 주로 한다던가
            - 전자: 네트워크가 금방 작아지지만, 정확도가 떨어짐
            - 후자: 정확도는 올라가나, 파라미터 수나 연산량이 많아짐
        - 중간을 잘 절충하는 방법은..
            - VGG: 두번 컨볼루션 → 한번 풀링 or 세번 컨볼루션 → 한번 풀링..
            - 이런 식으로 골고루 분포시키는 느낌으로 갈 수도 있고
        - 여기서 evenly spaced라는게 네트워크 전반에 걸쳐 골고루 다운샘플링을 분포시키는 것을 의미한다!
        - 이걸 강조했다라는 것은.. 모바일넷에선 이걸 어떻게 했을까?
    - Depthwise separable convolutions
        - 재준님 Xception 논문을 보면 자세히 볼 수 있다
    - Shuffle operations
        - 간단한 언급정도만 한다고한다
        - 왜?
    - Knowledge distillation & compression
        - 이것도 지난번에.. 누가? 설명했다고 하던데...
- 본 논문에서 주로 사용한 접근법: Channel reduction, **Depthwise separable convolutions**, Distilation & compression

- 어떤 파라미터를 바꾸면 이게 이렇게 결과가 바뀌는구나 라는 것을 많은 실험을 통해 접근했다고 한다

![스크린샷 2022-02-19 오후 12.19.24.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e8b4eebb-5ad2-4b4d-aeb7-28776bd191b7/스크린샷_2022-02-19_오후_12.19.24.png)

- 연산은 그냥 잠깐 다시보고...
- VGG recap
    - VGG는 3x3 컨볼루션만 여러번 사용
    - 더 깊은 네트워크: non-linearities가 커지고
    - 적은 파라미터 요구
- Inception-V3 recap
    
    ![스크린샷 2022-02-19 오후 12.23.13.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fd0358b8-15f1-4134-be17-4107bf2d2cba/스크린샷_2022-02-19_오후_12.23.13.png)
    
    - 본래 3x3 필터 쓰던 구조를 V3에서 바꿨나보다
        - 1xn, nx1로 쪼갰다!!
        - 직사각형 형태로 쪼갰는데 성능도 잘나오고 파라미터수를 줄일 수 있었다고 한다
        - 가로, 세로 성분을 쪼갠 느낌인건가??? 뭐지???
    - 컨볼루션 연산
        - width, height, channel을 모두 고려해서 연산을 하는게 특징
        - Inception V3에서는 width와 height를 쪼갰다
        - 근데... 연산에서 모든 채널을 꼭 다 고려해야만 해? 안그럴수는 없는거야?

- Depthwise convolution
    - Standard convolution
        
        ![스크린샷 2022-02-19 오후 12.27.19.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/796bb9c8-4d9a-4165-b57d-3be14aeeba8d/스크린샷_2022-02-19_오후_12.27.19.png)
        
    - 한 채널씩 따로 떼어서 연산을 하겠다는 접근인가보다
        
        ![스크린샷 2022-02-19 오후 12.28.28.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/71f8ae87-b42e-438b-84ec-524e3d66d561/스크린샷_2022-02-19_오후_12.28.28.png)
        
    - 여기에 pointwise convolution까지 결합하면 같은 결과가 나오나보다
    (1x1 convolution)
        
        ![스크린샷 2022-02-19 오후 12.29.10.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7ae6b333-7e3e-4fee-9e78-de14fb19cd40/스크린샷_2022-02-19_오후_12.29.10.png)
        
    - 일반 컨볼루션과 뎁스와이즈를 비교해보자.
        
        ![스크린샷 2022-02-19 오후 12.32.27.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8388db34-bd52-4f0e-9eaa-ade2695bb678/스크린샷_2022-02-19_오후_12.32.27.png)
        
        1. Standard convolution
            - D_K: 필터의 가로,세로
            - M: 입력 map의 채널 수(예를 들어, 원본 이미지 3채널짜리면 M=3)
            - N: 필터의 갯수. 즉, 필터 채널이 되겠지?
        2. Depthwise separable convolution
            - 표기법은 위와 같아 그렇다면
            - 필터의 가로,세로는 동일하게 D_K
            - 근데 각 필터는 채널이 1인거고, 이 1채널짜리 필터가 입력 map의 채널 수만큼 M개가 있는 것(즉, 원본 이미지 3채널짜리가 입력이면 D_K x D_K x 1짜리 필터가 3개 있는 것)
            - 그래서 이 필터들을 가지고 각 채널마다 컨볼루션을 수행한다
            - 그 이후, 1x1 컨볼루션 - 1x1xM짜리 필터 N개 사용
        - 결과적으로 사이즈는 동일하나 파라미터 수를 많이 줄일 수 있는 output이 나와 (왠지.. 실제 갯수가 어떻게 되는지 그 비교 예제를 보면 좋을 것 같아.. 아직은 와닿지 않아)
        - computational cost 비교
            1. Standard convolution
                - D_K x D_K x M x N x D_F x D_F
                - D_F: Feature map의 가로/세로
            2. Depthwise separable convolution
                - (D_K x D_K x 1 x M x D_F x D_F) + (1 x 1 x M x N x D_F X D_F)
                - Computations가 $1/N + 1/D_K^2$ 만큼 줄어들었다고 한다
                - 식을 보면.. 더하기 연산 쪽이.. 왼쪽엔 N이 빠졌고, 오른쪽엔 $D_K^2$가 빠졌네..
                - D_K나 N 둘 중에서는 수치 자체는 보통 N이 훨씬 큰 경우가 많다
                - 예를 들어 3x3 컨벌루션을 사용한다면 8-9배정도 연산량을 줄일 수 있다(필터 사이즈 측면)
                - 그럼.. 필터 갯수 측면에서는..?
            - 다시 식 비교...
            
            $D_K * D_K * M * N * D_F * D_F = 
            (D_K * D_K * N)(M * D_F * D_F)$
            
            $(D_K * D_K * 1 * M * D_F * D_F) + (1 * 1 * M * N * D_F * D_F) = (D_K * D_K + N)(M * D_F * D_F)$
            
            이 부분 다시 파헤쳐야 해...
            
        
        ![스크린샷 2022-02-19 오후 1.25.42.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3e3818b1-055b-4cde-8204-7466d73ae6ac/스크린샷_2022-02-19_오후_1.25.42.png)
        
    - 왼쪽이 standard, 오른쪽이 depthwise separable
    - Xception과 비교하면 순서가 달라(1x1 먼저하고 3x3하는 순서)
        - 추가로 non-linearity도 넣었다
- MoblieNet 모델 구조
    
    ![스크린샷 2022-02-19 오후 1.27.34.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/90c3f296-e830-4a67-bc24-74eb0d4dd9f9/스크린샷_2022-02-19_오후_1.27.34.png)
    
    - dw: depthwise
    - s: stride
    - 첫 번째 컨볼루션을 보면 스탠다드를 적용하고 depthwise를 적용했다.
        - 왜?
    - 전체가 컨볼루션으로 가다가 7x7이 남았을 때 평균 풀링을 썼다
    - FC layer가 최하단 하나에는 추가되었다 (classification 결과를 얻기 위함으로 보인다.)
- 어떤 레이어가 얼마만큼의 비중을 차지할까? 그리고 파라미터 갯수의 비율은?
    
    ![스크린샷 2022-02-19 오후 1.31.35.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/af4d170a-4c97-446a-a993-255aa8dcdead/스크린샷_2022-02-19_오후_1.31.35.png)
    
    - 대부분이 1x1에
    - 게다가 1x1에 많은 파라미터 비중이 있고, FC에는 비중이 적다 (일단 전반적으로 파라미터 수가 줄었기 때문에.. 얼마나 줄었는지 체감이 필요)

- Width multiplier & resolution multiplier
    - Width multiplier $\alpha$
        - 채널 줄이기가 목적
        - 입력 채널: M → $\alpha$M
        - 출력 채널도: N → $\alpha$N
        - $\alpha$: 1, 0.75, 0.5, 0.25...
    - Resolution multiplier $\rho$
        - 입력 이미지의 가로,세로 해상도를 줄여서 넣겠대
        - $0<\rho<=1$
    - 이런 방법들을 썼을 때 파라미터 수, 정확도, 연산량이 어떻게 바꼈을까를 보기 위함
    - Computational cost
    
    $(D_K * D_K * \alpha M * \rho D_F * \rho D_F) + (\alpha M * \alpha N * \rho D_F * \rho D_F)$
    
- 실험결과
    
    ![스크린샷 2022-02-19 오후 3.33.23.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/aaae898c-6740-431a-962d-742dd4c75ee1/스크린샷_2022-02-19_오후_3.33.23.png)
    
    ![스크린샷 2022-02-19 오후 3.34.10.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cc742560-2691-474e-97ad-b619450574d9/스크린샷_2022-02-19_오후_3.34.10.png)
    
    ![스크린샷 2022-02-19 오후 3.34.19.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c2af8a72-6247-4666-bfa4-55c10ade1ffb/스크린샷_2022-02-19_오후_3.34.19.png)
    
    - Shallow network: 네트워크 레이어 수를 좀 잘라놓은 구조라고 한다
    - Narrow network: 채널 수를 줄여서 슬림하게 만든 구조
        - 실험 결과를 통해 Deep한 구조는 유지하고 채널수를 줄이는 방법이 더 정확도가 좋았다고 한다(이유는 무엇일까?)

![스크린샷 2022-02-19 오후 3.38.49.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/46f9ee8a-fb82-4445-aa9b-84fbfa67a461/스크린샷_2022-02-19_오후_3.38.49.png)

- Top-1 accuracy 기준으로 다른 모델들과 비교도 함

![스크린샷 2022-02-19 오후 3.39.51.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/474d7c9c-5eac-4842-bf4f-d1e191f9b0fd/스크린샷_2022-02-19_오후_3.39.51.png)

- ImageNet말고 개의 종을 분류해야하는 Stanford Dogs라고 하는 더 어려운 데이터셋으로 정확도 비교한 결과
    - Million Mult-Adds가 무슨 의미일까?
    - train: 웹에서 다른 개들의 데이터를 수집해서 pretrain → Stanford dogs로 fine tuning (왜 이랬을까?)

![스크린샷 2022-02-19 오후 3.42.21.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/99c683e4-5730-4579-b839-2ae88c16d9b3/스크린샷_2022-02-19_오후_3.42.21.png)

- PlaNet이라고 하는 구조를 depthwise convolution을 통해 MobileNet 구조화하여 접근한 결과

![스크린샷 2022-02-19 오후 3.43.12.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cd786553-391a-4eb5-84a4-df276ceb22be/스크린샷_2022-02-19_오후_3.43.12.png)

- COCO dataset 기반으로 detection 문제에도 적용해봄
    - Framework resolution: SSD, Faster-RCNN
    - Model: VGG, Inception, MobileNet
    - 질문: 그럼 OD를 위한 RCNN, SSD 등은 기존 classification을 위한 CNN 구조로 무언가 혼합된 형태야?

![스크린샷 2022-02-19 오후 3.45.30.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/31b9a248-b3df-4be2-bf01-503e8d77e6ee/스크린샷_2022-02-19_오후_3.45.30.png)

- Knowledge distilation을 같이 혼합해서 사용한 결과 (보충강의 봐야해!)

[PR-009: Distilling the Knowledge in a Neural Network (Slide: English, Speaking: Korean)](https://www.youtube.com/watch?v=tOItokBZSfU&list=PLlMkM4tgfjnJhhd4wn5aj8fVTYJwIpWkS&index=10)

- 진원님이 말하신 remaining issue
    - 1x1 convolution 연산은 모든 채널을 고려하는데, 이것도 꼭 이렇게 해야 해? (레이어를 하나만 쌓을 것이 아니니깐)
    - 일부만큼 쪼개서 접근하는 방법?
    - 이 아이디어로 접근한게 ShuffleNet이지 않을까?

## PR-108: MobileNet V2

부제: Inverted residuals and linear bottlenecks

- 연관 논문이 MobileNet, ShuffleNet(이게 비교대상인가 봄)
- Width multiplier, Resolution multiplier가 똑같이 다뤄진다고 한다. 좀 다르게 했나..?

Key features

- Depthwise separable convolutions
    - 그대로 사용한건가..
- Linear bottlenecks
- Inverted residuals

Linear bottlenecks

- 수학적으로도 증명되어있음
- 자세히 봐야할 듯(진원님 생략)
- 리얼 이미지를 입력으로 받았을 때, 뉴럴 넷의 레이어들은 “Manifold of interest”를 형성한다고 모두들 생각한다 (그게 뭔디..?)
- manifold of interest는 저차원 서브스페이스로 맵핑이 가능하다.
- 라고 하는 두 전제조건하에 출발
- 중요한 두 개의 property에 대해 이야기를 하는데
    - Manifold of interest가 ReLU 활성화를 통과한 이후에 non-zero volume을 가진다면, 이것은 linear transform을 만들 수 있다.
        - ReLU는 살아남은 양수 값들에 대해 y=x를 유지시켜준다.
        - 이 살아남은 친구들을 보면 선형적인 구조를 가질 수 밖에 없다
        - 왜? y=x만 살아남으니까?
    - ReLU는 입력 매니폴드에 대한 전체 정보를 보존할 수 있으나, 입력 매니폴드는 오직 입력 공간 내 저차원 서브스페이스에만 기댈 수 있다(?)
        - 이것도 무슨 말인지 도통...
- 결론적으로 위 두 속성을 만족하고, manifold of interest가 low-dimensional이라면, linear bottleneck layer를 ReLU 대신 사용하면 더 효율적이라고 한다.
    - 좀 더 찾아보자

Residual blocks

![스크린샷 2022-02-22 오후 10.19.44.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5b588a8e-e381-4884-9bd9-125c86c801e5/스크린샷_2022-02-22_오후_10.19.44.png)

- 보통은 Wide → Narrow → Wide 구조라고 하는데..
- 그림대로면 1x1 convolution을 하고 채널수를 줄인 다음 필요한 부분에 대해 3x3 필터로 컨볼루션 연산 → 앞단에서의 입력 레이어와 채널을 맞춰주어야 하기 때문에 1x1 컨볼루션으로 다시 늘려준다.
- 그래 보통은 이렇지.. 근데 논문에선 뒤집었다!!!

 Inverted residuals

![스크린샷 2022-02-22 오후 10.26.10.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0d33abcc-eca9-4d83-8451-3e96eaab3ff0/스크린샷_2022-02-22_오후_10.26.10.png)

- Narrow→wide→narrow 구조로 간다!
- 빗금친 네모가 linear bottlenecks(ReLU 없이 들어갔다)
- bottleneck operation을 할 때, 채널 수가 많은 거에서 작은 것으로 갈 때, 그 정보를 다 잃지 않고 전부 앉고 간다고 가정한다.
- 정보를 다 담고 있으니 residual connection을 담아도 문제 없겠다 해서 이렇게 갔다
- 메모리 사용에 있어서도 효율적이라고 한다.

Bottleneck residual blocks

![스크린샷 2022-02-22 오후 10.30.12.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/557eaa37-fb28-4f10-841e-a61693595734/스크린샷_2022-02-22_오후_10.30.12.png)

![스크린샷 2022-02-22 오후 10.30.02.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3acf4546-de15-4f3c-b4da-86d17e933008/스크린샷_2022-02-22_오후_10.30.02.png)

- MoblieNet V2에서 기본적으로 사용하는 residual blocks
- Block 1
    - 1 x 1 Expansion으로 채널을 키우고
    - Batch Norm → ReLU6(6 이상 되는 값들은 모두 6으로 saturation 시키는 함수)
- Block 2
    - Depthwise conv → Batch Norm → ReLU6
    - 질문: MobileNet V2에서 ReLU6를 사용한 이유
- Block 3
    - 1 x 1 Projection으로 다시 채널을 원상복구
    - 그 다음 Batch Norm
    - 결과가 입력과 만나서 더해짐
- 채널을 얼마나 팽창시킬 것인지: expansion factor로
    - 논문에서는 6으로 설정
    - 논문에서 말하기로는 5-10으로 했을 때 성능이 제일 좋았다.
    

연산량 계산

- block size = h x w
- expansion factor = t
- kernel size = k
- input channels = d’
- output channels = d’’
- $(h * w * t * d' * d') + (h * w * t * d' * k * k) + (h * w * d'' * t * d') = h * w * d' * t * (d' + k^2 + d'')$
    - 첫 번째 괄호: 1x1 conv로 t배만큼 채널 확장, 사용되는 필터는 1x1xd’ 크기
    (1 x 1 x d’ 이 t개 있다랄까)
    - 두 번째 괄호: depthwise conv(인데 pointwise부분 빠진거) 공식은 위에꺼 다시 보기 (필터 사이즈: k x k x 1)
    - 세 번째 괄호: 1x1 conv (1 x 1 x d’ x t)
    - 공식 다시풀이...
- 실제로 V1보다 더 적은 채널을 가질 수 있는 장점이 있다.

Information flow interpretation(?)

![스크린샷 2022-02-22 오후 10.42.36.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6e46ea8d-a7ec-4afb-949d-af1939a5f90d/스크린샷_2022-02-22_오후_10.42.36.png)

- expansion layer에서 얼마나 채널 수를 늘려주느냐에 따라 expressiveness를 결정
- projection layer에서 얼마나 채널 수를 줄여주느냐에 따라 capacity를 결정
- expressiveness와 capacity에 대한 부분이 잘 분리되어 있는 게 이 구조의 장점이라고 주장한다.
- 보통의 모델들은 expressiveness와 capacity가 묶여있다
    - 출력 채널이 크면 둘다 커지고, 작아지면 둘다 작아진다.
- 이 논문은 따로따로 컨트롤할 수 있어서 네트워크 설명함에 있어서 많은 도움이 될 것이다라고 주장!

Architecture

![스크린샷 2022-02-22 오후 10.47.47.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ffd5ce93-b121-4164-ae52-56df952397c9/스크린샷_2022-02-22_오후_10.47.47.png)

- t = expansion, c = channel, n=구조 반복, s=stride
- Hyper parameters
    - Input resolution: 96-224까지 조절해봄
    - Width multiplier: 0.35-1.4까지 조절해봄
- stride가 1일 때와 2일 때가 구조가 다름
    - 1일 때만 residual connection이 있고, 2일 때는 없다.
    - 왜에 대한 부분이 없는데, 추측으로 보자면..
    - stride=2이면 가로,세로가 줄어들기 때문에 추가 연산이 필요해
    - shufflenet에서는 average pooling으로 줄여주는 방법으로 접근한 듯 한데
    - mobilenet은 굳이 풀링 등으로 추가 연산을 하기 싫어서 뺀걸로 보여..

메모리를 효율적으로 활용할 수 있는 구조

- 얼추 맞으나, 100퍼센트 다 맞는 말은 아닌거 같다고 진원님 주장
- residual block의 처음과 마지막만 메모리에서 사용하고, 그 사이에 있는 레이어들은 아닌가봐? 메모리 안거치고 내부 연산?? (엥???)
- 처음과 끝단에 있는 채널이 적으면 더 효율적인 메모리 사용을 할 수 있겠지란 이야기지..?
- 더 효율적으로 쓸 수 있는 방법
    - 머리아프다.. 다시보자..

실험 결과

![스크린샷 2022-02-22 오후 10.56.12.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c8ae403c-4b10-45e2-9ccf-53cc9398228a/스크린샷_2022-02-22_오후_10.56.12.png)

![스크린샷 2022-02-22 오후 10.57.58.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e7219a26-fe0e-4c2b-b45a-4754aef909b8/스크린샷_2022-02-22_오후_10.57.58.png)

- fps를 잰 것 같다.
- 아래는 object detection 결과

![스크린샷 2022-02-22 오후 10.58.51.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2618d60e-afe0-4fe4-98fe-139afed6fd81/스크린샷_2022-02-22_오후_10.58.51.png)

![스크린샷 2022-02-22 오후 10.59.42.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/df1f8848-1c1a-4108-8236-d53dd0ae570d/스크린샷_2022-02-22_오후_10.59.42.png)