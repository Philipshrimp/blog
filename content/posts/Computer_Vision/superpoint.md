---
title: "Superpoint"
date: 2022-07-07T17:43:41+09:00
draft: true
---

# SuperPoint: Self-Supervised Interest Point Detection and Description

## Abstract

- MagicLeap에서 나온 연구 - SLAM, 그것도 AR쪽에 접근하는 것이 주된 목적일 듯

[Enterprise augmented reality (AR) platform designed for business | Magic Leap](https://www.magicleap.com/en-us/)

- Interest point detectors와 descriptors를 train하기 위한 self-supervised framework 제안
- 특이한게.. fully convolutional model이고, 전체 사이즈 이미지에서 동작하나봄 (패치 영역 기반이 아닌..)
- interest point와 descriptor를 한 번의 forward pass로 찾는다!
- Homographic adaptation?
    - multi-scale, multi-homography approach
    - interest point 반복적으로 찾는 것을 가속화하는 역할인가..
- Cross-domain adaptation (synthetis to real)적용
- 제안된 방법이 반복적으로 풍부한 양의 포인트 검출을 잘 하는 것을 알 수 있었다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1157fef7-f03a-49e3-9567-3be49bef1e9d/Untitled.png)

### SuperPoint for Geometric Correspondences

- 480 x 640 image에서 70fps (Titan X GPU)
- 이미지를 보고 유추,,
    - 두 쌍의 이미지가 입력으로 들어가고
    - 각각의 네트워크에서 Interest points와 Descriptors가 결과로 나온다.
    - 두 이미지 쌍의 특징점들을 매칭

## Intro

- Geometric computer vision (SLAM, SfM, Calibration, Matching)에서 특징점 찾는 과정은 매우 중요하다. (항상 첫 단계에서 고려된다는 것!)
- Interest point = Image 상에서의 2D 위치를 의미
- 논문에서는 Human supervision 대신 self-supervised 방법을 제안한다고 이야기함
- 거대한 pesudo-ground truth dataset을 만들었다고 한다. (interest point locations에 대해) - Real image로부터
- Human annotation 공수는 없는 방법이라고 언급한다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/724e7513-709b-42b4-8518-5bb113e07215/Untitled.png)

- Pseudo-ground truth interest points는 어떻게 만들었는가..?
    - Fully convolutional neural networks를 백만여개의 examples를 이용해 학습 (synthetic data 이용 = Synthetic Shapes라고 명명)
    - 위 그림의 (a)에 있는 부분이라고 하는데
    - 일종의 인위적으로 interest point를 쉽게 검출할 수 있는 데이터인가..
    - 간단한 geometric shapes로 이루어졌다고 한다. → 특징점 뽑기에 ambiguity따위 없는 명확한 모양
- MagicPoint: 위에서 언급한 synthetic data를 이용해 학습된 detector
- 물론 단순 MagicPoint만을 이용하면 동작은 잘 하지만, 잠재적인 points를 아직은 많이 놓친다. (실제 이미지에서)
- 이러한 문제를 해결하기 위해 multi-scale, multi-transform 기술인 Homographic Adaptation 적용했다고 한다.
    - interest point 검출에 있어 self-supervised training이 가능하게 해준다.
    - 입력 이미지를 여러 차례 워핑을 한다라..? → 디텍터가 다양한 뷰와 스케일에서 이미지를 보고 특징점을 뽑을 수 있게끔 하기 위함이라고 한다.
    - 결과적으로 pseudo-ground truth를 이 과정에서 만들어낸다고는 한다.
- 이렇게 적용된 결과물을 SuperPoint라고 한다.?
- 그 다음 같은 네트워크에서 나오는 descriptor와 matching을 진행

## SuperPoint Architecture

- 특징: single forward pass로 interest point와 이에 상응하는 고정된 사이즈의 descriptor를 뽑는다.
- 전반적으로 SegNet처럼 encoder-decoder 구조

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/34b00d37-ba75-4d96-a0e7-882494eebdfb/Untitled.png)

- shared encoder: 입력 이미지의 dimensionality 축소
- Two decoders: 특정 weights를 학습하는 역할을 한다고 보면 됨

![스크린샷 2022-06-07 오후 9.50.05.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7793dc5a-82ac-4740-b5b8-b2e577220d08/스크린샷_2022-06-07_오후_9.50.05.png)

1. Shared encoder
- VGG 스타일의 encoder 구조를 사용해서 입력 이미지의 dimensionality를 축소했다.
- 레이어 구성: Conv - Pool - Activation(무얼 썼나)
    - 풀링은 max pooling → 최종적으론 8배 축소
    - 2x2 non-overlapping max pooling이 세개
- Cell: lower dimensional output을 여기서 cell이라 하고
- 인코더 결과로 8x8 pixel cell이 만들어짐 → 8x8 픽셀의 대표 값이 압축된 하나의 셀을 일컫는 듯
- 결과: Input image → Intermediate tensor (가로 세로 크기는 줄어들고, 채널 차원은 F로 늘어났다.)

1. Interest point decoder
- 최종 결과를 얻기 위한 업샘플링은 upconvolution을 사용 (=Transposed convolution)
- 하는게 보통인데..
- 근데 체커보드 형태의 artifacts를 만들고, 계산량 자체도 많아져서
- 명시적 디코더가 있는 head 부분을 설계했다고 하는데..
    - Sub-pixel convolution
    - Depth to space(TensorFlow), Pixel shuffle(PyTorch)
    - 파라미터가 없는 디코더라고 한다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d8820fe9-53f5-40eb-bbc2-eac91adf2c72/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ddb131de-deae-420b-9f9b-6e8b7bcb6b41/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1f81f4f5-8e1e-45bc-bd53-09fc05ea3954/Untitled.png)

- 잠시 sub-pixel convolution 리뷰(간단히만)
    - 기존 Super Resolution 기법들처럼 모델의 첫 부분 혹은 전처리 부분에서 업샘플링 하는 것이 아닌 sub-pixel convolution layer를 사용해 모델 끝단에서 업샘플링 수행
    - 모델 연산량과 파라미터 수를 줄이고 성능도 높였다고 주장
    - low resolution 보다 r배로 업샘플링을 한다고 가정하면
    - low resolution 보다 $r^2$만큼 feature map 채널 수를 늘리고
    - feature map을 순서대로 조합해서 high resolution image를 만드는 원리 (feature map 하나하나씩 떼어서 이미지를 만드는 형식)
    - transposed convolutions보다 $\log2r^2$만큼 시간이 단축되고, 다른 일반적인 업샘플링보다 평균적으로 $r^2$만큼 시간이 단축된다고 주장
    - 위 식에서 왼쪽 공식이 레이어 동작을 수식화한 것, 오른쪽 공식은 원본 이미지와 픽셀 레벨에서 비교하여 loss를 계산하는 것인 듯
- 이 decoder의 head는 $\chi$를 계산
    - 채널이 65채널인 이유..? (8x8.. 하나의 다운스케일된 영역이 이만큼을 커버하니까)
    - 추가로 “No interest point” dustbin에 해당하는 한 채널 (포인트 없는 영역이다를 의미하니..?)
- 그 다음 channel-wise softmax 계산
    - 여기서 dustbin channel은 제거됨 (있냐 없냐 분류를 먼저하고 그 다음에 제거되는걸까)
- 마지막으로 reshape 진행해서 업샘플링 수행

1. Descriptor decoder
- 출력 결과의 사이즈는 W x H x D
- 출력 값은 고정된 길이의 descriptor이며, 이는 L2-Norm을 기반으로 정규화가 이루어짐
- 여기서는 UCN과 유사한 모델을 사용했다고 하는데.. (Universal Correspondence Network)
    - 8픽셀당 하나의 output을 내놓나봄..?
    - semi-dense grid..
- 완전히 dense하지 않고 semi-dense하게 진행했다고 하고, 그 덕에 training memory를 줄일 수 있었다고 한다. (=실시간성도 얻고..)
- 그 다음 bicubic interpolation을 수행하고, L2-Norm 적용(SSD)

1. Loss functions
- 하나는 Interest point detector에 대한 loss
- 다른 하나는 descriptor에 대한 loss
- synthetically warped image pairs 이용
    - pseudo-ground trutu interest point locations 가지고 있고
    - ground trutu correspondence from a randomly generated homography도 가지고 있고
        - 호모그래피 매트릭스가 랜덤하게 만들어진다..? 아마 워핑과 연관있는 랜덤 값일 듯
        - 위에서 말한 이미지 쌍에 상응하는 매트릭스일듯
- 이 쌍을 이용하면 두 개의 loss를 동시에 최적화할 수 있다고 설명 (위 그림 joint training파트 참고)
- 람다 값을 조정하며 loss 값의 밸런스를 맞춘다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/51b0fdd1-dbe7-4be9-b56f-e33cb5cae01a/Untitled.png)

1) Interest point detector loss

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/37985b04-3ce3-4c0d-b968-cfd0f4bad0cb/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/73cac68a-369e-4a34-88ad-1d981a4466d6/Untitled.png)

- Fully convolutional cross-entropy loss
- $\chi$ 내에 있는 각 cell에 대한 계산을 진행하는 다음의 식
- Y: set of corresponding GT point labels
- 하나의 loss는 전체 channel에 대해서, 전체 cell에 대해서 계산함을 의미

2) Descriptor loss

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/be1b5600-8747-46d8-beaa-f0e235514d22/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/07d3cafd-568c-455c-ab75-7331b38440f4/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/aed037a7-154c-4b1a-bdbd-4c553d6e34bb/Untitled.png)

- 두 이미지에서 얻어지는 모든 descriptor cells 쌍에 대해서 계산
- 여기서 p는 (h,w) cell 안에서의 center pixel 위치
- H hat은 p x H 하고, 거기서 나온 마지막 좌표 값을 나눈 결과 (=Homogeneous coornates)
    - Homogeneous coordinates: transformation 연산을 동일하게 matrix 하나와의 곱셈 연산으로 풀고 싶어서 좌표 차원을 임의로 확장한 coordinates
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/32f4e83a-dab9-4c50-933c-2335e3774bfd/Untitled.png)
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/82e6dd72-9aeb-4186-80ac-aa6651b0ec6e/Untitled.png)
    
- 해석하자면.. 호모그래피 매트릭스 기반으로 워핑한 첫 이미지 포인트 값과 두번째 이미지 포인트 간 거리 차이가 8 pixel을 넘으면 이것을 문제있는 값으로 간주한다는 듯
- S: 모든 correspondences set
- hinge loss?
- m: margin (positive / negative)

## Synthetic pre-training

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/36c26f1c-bbdf-42f8-98f7-c6a456ccdbc4/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/21394434-86cf-4094-8218-60567363c7d1/Untitled.png)

- synthetic data 기반이기 때문에 상대적으로 잡음에 강하다고 주장
- Synthetic shapes
    - pre-training을 위해 직접 만든 synthetic dataset
    - 단순 기하학적인 구조들을 이용해 만든 심플한 데이터 셋 (선, 삼각형, 체커보드 등등 코너점 검출에 명확한 데이터 셋 생성한 듯)
    - label의 모호성을 제거할 수 있었다고 주장
    - 추가로 homographic warp를 적용해서 augmentations도 수행

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/68e77372-6871-4197-b3cb-cf802cf937eb/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/886e45e6-7b09-493b-bec1-efb63ba919f9/Untitled.png)

- MagicPoint
    - 사실상 synthetic data 기반의 pre-trained model
    - 이 구조는 기본적인 superpoint detector 구조만 사용
    - FAST, Harris corner, Shi-Tomasi’s method와 비교했을 때, 도드라진 성능 차이를 보였다고 주장
    - 잡음에 강해보이고 명확한 코너만 잘 검출한 것으로 보임
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/700325e3-a538-4c54-8cc6-fd56ceb1ab5c/Untitled.png)
    
    - synthetic image 뿐만 아니라 real image에서도 동작 잘했다고 주장 (실험결과 7.2에 있음) 그러나 완전 잘됐다 정도는 아님
        - 코너가 명확한 환경에서는 잘됐지만, 그렇지 못한 곳에서는 고전적인 방법들만도 못했다고..

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/eb07da3d-72e3-4cfa-97e6-66b4f7a6da15/Untitled.png)

## Homographic adaptation

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7b5ffdf1-fd41-4794-a322-4f125be68d64/Untitled.png)

- 그림 먼저 대략 해석..
    - Unlabeled real image(COCO등의 dataset으로부터..)와 MagicPoint 준비
    - Homographic adaptation
        - N회의 랜덤 호모그래피 적용
        - 디텍터 적용
        - point response 얻으면 이거 다시 unwarp
        - 전체 heatmaps을 merge
    - Interest point superset 얻음
- Random homography
    - 논문에서 이 방법에 self-supervised라고 주장하는 이유라고나 할까..
    - self-supervised: Network로 하여금 만든 pretext task를 학습하게 하여 데이터 자체에 대한 이해를 높일 수 있게 하고, 이렇게 network를 pretraining 시킨 뒤 downstream task로 transfer learning을 하는 접근 방법
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a9a69722-869c-4f2a-b48c-9f11f876bf3a/Untitled.png)
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7c0fbf06-fc8b-4277-b6a4-bf6f8f84fb88/Untitled.png)
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0eb634f0-2e53-4cec-af4b-b1ef9281f8bf/Untitled.png)
    
    - f: initial detector function
    - I: input image
    - x: interest points
    - H: random homography
    - f와 H는 보통 covariant한 관계라고 한다. 근데 실제로, detector는 완벽하게 covariant하지는 않고..
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5c4c6993-c04a-4341-9785-d6eb2d151f76/Untitled.png)
    
    - 위 식은 improved super-point detector를 나타냄
        - N이라 함은 충분히 큰 랜덤 호모그래피 샘플의 수를 말한다고 한다.
        - N은 하이퍼 파라미터. 논문에서는 10, 100, 1000을 실험했는데..
        - MS-COCO 기준으로 100 이상으로 주었을 때 좋은 성능을 보였닥 한다.
        - 근데 100을 주었을 때랑 1000을 주었을 때 큰 차이를 보이지 않아 100이 제일 적절하다고 본 듯
    - 무조건 3x3 matrix를 랜덤하게 만드는게 아니라 더 심플한 방법도 사용했다고는 함
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3851be97-b98f-4edd-b514-68422911c260/Untitled.png)
        
        - 사전에 정의된 range 안에서의 translation, scaling, in-plane rotation, perspective distortion…

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/81d879b5-6d3b-4a18-bea3-598b6af50fd7/Untitled.png)

- Iterative homographic adaptation
    - 최종적으로 training 과정에서 이 HA를 반복적으로 적용한 모델을 SuperPoint라고 명명
    - 그림: 위가 base detector(MagicPoint)이고, 반복적으로 할 수록 성능이 올라감을 보임
    

## Experimental details

- Encoder: VGG와 유사한 구조 사용
    - 8개의 3x3 conv layers (64 channel 4개, 128 channel 4개)
    - 두개의 레이어 당 하나씩 max pooling이 적용된 구조
- Decoder head: single 3x3 convolutional layer (256 channel)
    - for detector: 뒤에 1x1 convolution 사용해서 65채널로 축소
    - for descriptor: 그대로 사용
- 모든 convolutional layer에는 ReLU + BatchNorm 사용

## Results

- HPatches dataset을 이용해 repeatability 검사 (=다른 각도에서 보아도 동일한 위치에서 keypoint가 검출되는 능력)