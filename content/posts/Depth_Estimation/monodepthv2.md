---
title: "Digging into Self-Supervised Monocular Depth Prediction (Monodepth2)"
date: 2022-07-07T17:47:01+09:00
author:
  name: Sunho Kim
menu:
  sidebar:
    name: Monodepth2 논문 리뷰
    identifier: monodepth2-review
    parent: depth-estimation
    weight: 1
draft: true
---

## Problem

- Depth-from-color 문제는 비싼 LIDAR 센서를 대신할 좋은 대안으로 매력적인 분야이다.
- 하지만 이를 supervised learning을 통해 학습하기 위해서는 정확한 ground truth depth 데이터가 필요하다.
- 최근 이를 극복하기 위하여 self-supervised 방법이 제안되고 있는데
    - monocular video 환경에서의 연구들은 depth map의 추정 뿐 아니라 **ego-motion 또한 동시에 추정**해야 하는 어려움이 있고
    - stereo data를 활용한 방법에서는 **occlusion**이나 **texture-copy artifacts** 등이 풀어야할 문제이다.

## Essence

- 가려진 픽셀(Occlusion)을 처리하는 새로운 appearance matching loss
- 카메라와 상대적인 움직임이 없는 경우를 처리할 간단하면서도 새로운 auto-masking 방법
- Depth artifacts를 줄이고자 입력 해상도에서 수행되는 multi-scale appearance matching loss

## Detail

### 1. Self-supervised training

$$
L_p = \sum_{t'} pe(I_t, I_{t' \rightarrow t})
$$

$$
I_{t' \rightarrow t} = I_{t'} \langle proj(D_t, T_{t \rightarrow t'}, K) \rangle
$$

$$
pe(I_a, I_b) = \frac{\alpha}{2} (1 - \text{SSIM} (I_a, I_b)) + (1 - \alpha) || I_a - I_b||_1, \alpha=0.85
$$

- 다른 시점의 another 이미지에서 target 이미지로의 view-synthesis 를 prediction 하는 network를 학습시켰다.(이전과 같음)
- 보통 이러한 방법들은 photometric reprojection error를 loss로 설정하고 이를 줄이도록 학습시킨다.
    - pe: photometric reprojection error
        - L1 distance와 SSIM을 활용
        - SSIM: 시각적 유사도보는거.. 이미 모노뎁스 1에서도 정의
        - alpha는 실험적으로 0.85로 결정한 것인지..?
    - proj: projection된 위치 (pose $T_{t \rightarrow t'}$를 이용)
        - $D_t$: depth(시점 t에서의)
        - $T_{t \rightarrow t'}$: pose(Image t에서 Image t’으로의)

$$
L_s = |\partial_x d_t^\ast|e^{-|\partial_x I_t|} + |\partial_y d_t^\ast|e^{-|\partial_y I_t|}
$$

- Edge-aware smoothness loss도 추가되었다.
    - mean-normalized inverse depth: $d_t^\ast = d_t / \bar{d}_t$
    - Monodepth1을 보면
        - .
- 학습에서는 Target 이미지의 앞, 뒤 2개 이미지를 source 이미지로 사용하고, 이를 통해 pose와 depth를 추정할 수 있게 학습되었다.
- Stereo에서는 stereo pair의 다른 한 쪽이 source가 될 것이고

### 2. Improved Self-Supervised Depth Estimation

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/155dd735-c12c-4be4-bb76-04251e7f4d5b/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/da3fd754-b39d-4266-89a7-1779bc0948d9/Untitled.png)

- Per-pixel Minimum reprojection loss
    - 기존에는 reprojection error를 평균냈었나 봄 (다수의 source image에 대해)
    - 근데 occlusion 영역에 대해서는 reprojection error 계산이 원활하지 못해
    - 분명 source image에 target image 대비 동일 위치에 occlusion이 생겼다면 match가 이루어지지 않을거야. (depth를 잘 추정했다면?)
    - 결국 높은 photometric error penalty가 생기게 된다
    - Out-of-view pixels
        - Image boundary에서의 ego-motion으로 가려지게 되는 경우 → 카메라가 이동하면서 해당 영역이 이미지 좌표계에서 벗어남
        - 일부 pixel에 대한 masking으로 reprojection loss를 줄일 수 있다고는 한다
        - 하지만 disocclusion(평균 reprojection 결과가 blurred depth 불연속 현상을 초래하는 영역을 말하는 듯)을 다루지는 않아
    - Occluded pixels 문제도 있음
    - 모든 Source image에 대해 photometric error를 평균내는 것이 아니라 최소 값을 사용하는 방법을 제안하나봄
    - $L_p=\min_{t'}pe(I_t, I_{t'->t})$
    - 결과적으로 Image border에서의 artifact 생성을 줄여주고, occlusion 영역에 대해 sharpness도 늘려준다.
- Auto-masking stationary pixels
    - 통상적으로 training 과정은 카메라가 움직이고, 정적인 화면이라는 것을 전제로 동작시킨다.
    - 그럼 이 전제가 깨진다면…? (카메라가 정지해있거나.. 다이나믹 오브젝트가 들어온다거나..)
    - 중요!! 우리 카메라는 정지!!!!
    - Hole을 만드는게 이런 부분인건가..? 네트워크에서? (depth 값이 무한대로 발산해버리는..) → 주로 다이나믹 오브젝트에서 발생하나 봄
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8e5c46e0-9efd-4f69-9c00-9abf7659ef82/Untitled.png)
    
    - 이걸 새로이 제안하는 auto-masking방법을 제안하며 접근
    - 이 방법에서는 “**외형**”이 바뀌지 않는 픽셀 값들에 대해서 필터링하는 접근법으로 보인다. (한 프레임에서 다른 프레임으로..)
    - 효과
        - 네트워크가 카메라와 같은 속도로 움직이는 객체를 무시하도록 하고
        - 카메라가 움직임을 멈추면 단안 비디오의 전체 프레임을 무시하도록 할 수 있음
    - per-pixel mask를 loss에 적용했다라..
    - pixel에 weight 주기 위함?
    - 근데 이전 방법들에 비해서는 마스크가 binary라고 한다. 그리고 자동으로 네트워크의 forward-pass 단계에서 계산되어진다고..
        - 특징이라면.. 이전 방법들은 아마 객체 모션을 기반으로 추정하나봐? 근데 이 방법에서는 그렇게 접근하지 않는다라..
    - 이웃한 프레임들 간 pixel이 같은지 아닌지를 검사하는 접근법인듯.. 같은지 여부를 binary화 한다?
        - same이 발생하는 케이스는..
        - 카메라 정지
        - object transformation == camera transformation
        - low texture region
    - per-pixel mask는 pixel loss만 담는다. binary 값은 다음 부등식을 기반으로 판별
        - warp된 source image와의 reprojection error가 warp하지 않은 source image보다 작은 경우, 이러한 상황으로 판단하여 mask를 0으로, 그렇지 않으면 1로 할당하였다.
        - $\min_{t'}pe(I_t, I_{t'->t}) < \min_{t'}pe(I_t, I_{t'})$
        - $I_{t'->t}$: Warped image (Target to source)
        - $I_{t'}$: Unwarped source image
    - 보면서 느낀건.. 결국 static case는 이전꺼 그대로 가려는게 큰건지?!
    - 아래 사진이 one epoch 거치고 난 auto-mask 결과인 것 같은데
        - 까만 픽셀이 loss로부터 지워진 결과(0)
        - 위: 자동차들이 까만 이유는 카메라와 유사한 속도로 이동중이기 때문
        - 아래: 대부분이 0이 된 이유는 현재 정지 중이기 때문
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c5e355e1-fa7d-479b-aa22-bd4c129d79db/Untitled.png)
    
- Multi-scale Estimation
    - Bilinear sampler에 의한 local minimum에 빠지는 것을 막기 위해 multi-scale depth prediction을 사용한다.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fcd0cb37-c4b4-4c6f-929e-7733bd5df38a/Untitled.png)
    
    - 기존에 나왔던 방법들은 멀티 스케일 depth prediction과 image reconstruction을 수행함
    - Total loss: decoder 내에서 각각의 scale에서의 개개인의 loss의 조합
    - 근데 이런 기존 방법들은 중간 layer에서 hole을 만들어내는 경향이 있다. (큰 low-texture region에 대해..)
    - 게다가 texture-copy artifacts도 발생
    - 제안하는 방법에서는 이런 multi-scale loss에 대해서.. 어느 해상도에서의 disparity와 color를 decouple함
    - 모호한 저해상도에서 loss를 계산하는 대신 중간 레이어에서 나온 저해상도 depth map을 입력 이미지 크기로 업샘플링한다.
    - 이걸 기준으로 reprojection을 수행해서 에러를 구하자!
- Final Training loss
    - 하기 식에 대해 각각의 픽셀, 스케일, 배치에 대해 평균을 내어 계산한다.

$$
L=\mu L_p + \lambda L_s  = [\min_{t'}pe(I_t, I_{t'->t}) < \min_{t'}pe(I_t, I_{t'})]\min_{t'}pe(I_t, I_{t'->t}) + \lambda |\partial_x d_t^\ast|e^{-|\partial_x I_t|} + |\partial_y d_t^\ast|e^{-|\partial_y I_t|}
$$

$$
pe(I_a, I_b) = \frac{\alpha}{2} (1 - \text{SSIM} (I_a, I_b)) + (1 - \alpha) || I_a - I_b||_1, \alpha=0.85
$$

### 3. 추가 고려 사항들..

- .

### 4. 네트워크 디테일

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5c4a326e-70c0-494e-b415-5712436a21c2/Untitled.png)

![초기 모델 형태인듯…](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fbf595fe-f4e1-41f4-bf25-0998868a2df5/Untitled.png)

초기 모델 형태인듯…

- Encoder
    - 기본적인 ResNet-18을 사용 (Depth, Pose 모두)
    - Pose encoder는 아무래도 입력으로 두 프레임이 요구되기 때문인지 입력을 6채널을 받거나 두 이미지를 받게끔 수정되었다.
    - 그래서 Pose encoder의 첫 weight shape는 일반적인 ResNet-18의 3x64x3x3이 아닌 6x64x3x3이다.
    - 만약 Pose encoder를 위해 pretrained model을 가져온다면..? : First pretrained filter를 duplicate시켜서 6채널짜리로 바꿔주는 작업이 진행되나봄
    - 이 expanded filter의 모든 weight들은 2로 나누는데.. 왜냐하면 output을 original과 동일한 numerical range를 맞춰주기 위함이라고.. (One-image ResNet처럼..)
    - Pose encoder 비교
        - 실험에서 PoseCNN / Shared encoder(맨 처음엔 이렇게 접근했나보다) / Separate ResNet(최근의 MonoDepth2) 세 가지로 비교함
        - 처음 제안한 shared encoder의 경우, Depth network와 feature를 share하고
        - Training 과정에서 더 적은 파라미터를 최적화하면 됐었다고..
        - 근데 depth prediction 정확도도 떨어졌다고..?
        - 그래서 separate로 바꾼듯
- Decoder
    - Encoder까지 거치고 나면 256채널의 결과가 나와서 이게 각각의 Decoder로 들어가는거 같고
- Pose network가 이전 DispNet보다는 깊어짐
    - 근데 왜 이 당시에는 네트워크가 더 깊어졌다라는 것을 자꾸 강조하면서 신경쓰는 이유가 뭘까…
- 구현에 대해서..
    - 기본 틀은 U-Net 구조가 뼈대 - Encoder-decoder network를 구성해야 하기 때문
    - ResNet을 사용했기 때문에 Skip connection이 추가
        - Deep abstract feature와 Local information이 모두 필요하기 때문
        - ResNet 18은 ImageNet으로 Pre train됨
        - 이걸 이제 Encoder에 썼다!
    - Depth Decoder
        - 기본적으로는 Monodepth1과 유사하다.
        - 활성화 함수: ELU 사용 (마지막 output depth 나오는 부분 빼고.. 여긴 sigmoid)
        - decoder에서는 disparity가 나오기 때문에 계산을 통해 depth로 inverse시킨다.
        - zero padding 대신 reflection padding을 사용했다고 한다.
        - reprojection 과정에서는 0값 대신 가장 가까운 테두리 영역에 있는 pixel 값을 가져왔다고 한다. (image boundary 바깥 영역에 대해)
    - smoothness term을 0.001로 셋팅
- Inverse depth normalization trick?
    - 추정된 depth가 치명적으로 값이 shrink하는 것을 막기 위함
    - 

### 5. 추가 실험

1) Post-processing

- .