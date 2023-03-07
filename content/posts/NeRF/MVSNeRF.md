---
title: "MVSNeRF"
date: 2023-03-07T19:10:08+09:00
draft: true
---

## Abstract

- 우리는 복잡한 indoor scene까지도 해서 강인하게 잘 만들어내낟ㅇ
- 뉴럴 래디언스 필드를 3장의 input image만으로 만들어낸다…?! → 3개의 시점이 들어간다.
- 새로운 뷰포인트에 대한 가상 이미지를 강인하게 만들어낸다
- 거기에 파인 튜닝을 더해 아티팩트를 확 줄임
- nerf가 5시간 트레이닝 할 때의 퀄리티를 우리는 6분만에 했대 (장면당)
- 3개의 view만을 이용해 빠른 inference를 가능하게 한다가 핵심
- approach
    - mvs에서 많이 쓰위는 plane-swept cost volume 만들고 (geometry-aware scene 만들기 위함)
    - 이를 기반으로 볼륨 렌더링 수행한다라..?
    - 자세한건 뒤를 읽자
- 결과에 대해
    - training한 데이터셋과 연관없는 indoor 데이터가 들어와도 렌더링을 잘한대

## Intro

- .

## MVSNeRF

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/27d4c294-be3b-474f-8be5-7e8b64166fb1/Untitled.png)

- 그림 설명 먼저 보면
    - cost volume을 만든다
        - 아마 뎁스 만드는 메커니즘하고 동일해보이네
        - 방식은 plane sweep으로 2차원 image를 warping
        - 이게 근데 코스트 볼륨이라 할 수 있나..? 디테일을 봐야해
        - **input이 세 장인거에 주목 (코드에서 세 장의 입력이 들어가는지 잘 확인하자)**
    - neural encoding volume
        - 3D conv를 쓰네?
            - per-voxel neural feature 얻는다
        - output (x, d, f, c)는 무엇?
            - x: NDC(Normalized Device Coordinate) position (3차원 위치)
            - d: ray direction
            - f: volume feature
            - c: image color
            - delta: density
        - 이제 이게 MLP 통과해서 RGB, density를 얻고
    - 마지막으로 볼륨 렌더링
        - 결과를 gt와 비교하며 L2 loss 최소화
- NeRF: per-scene network memorization을 통해 radiance field 복원하는 방식이래
- MVSNeRF
    - 주어진 M개의 image + 각각의 알려진 카메라 파라미터가 있을 때 (논문에서는 M=3)
    - 뉴럴 인코딩 볼륨을 통해 표현할 수 있는 방법
    - 표현을 아래 식처럼 할 수 있다고 한다.
    
    ![스크린샷 2022-11-12 오후 8.44.46.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cafeb532-ebab-4db0-a4b4-ec05f4d0d573/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-11-12_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_8.44.46.png)
    
    - volume rendering: novel image와 novel viewpoint를 이용해 differentiable ray marching을 사용한다고 한다.
    

## Cost volume construction

- m개의 input image를 reference plane sweep에 워핑하는 것에 대한 볼륨인가?
1. Extracting image features
    - 2D CNN을 이용하여 각각의 입력 이미지에 대한 feature를 얻는다.
    - 목적은 2차원 neural feature를 효과적으로 뽑기 위함인데, 이 neural feature는 local image appearance를 의미
    - 네트워크 구성: downsampling convolution layers
    - input I가 들어가면 2차원 feature map F(H/4 x W/4 x C)를 만든다.
2. Warping feature maps
    - 카메라 파라미터는 K,R,t 모두 주어짐
    - 고전적인 호모그래피 워핑 방식 사용하나본데
        
        ![스크린샷 2022-11-12 오후 8.52.28.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e526be9c-e96f-4d44-a898-e1224f24fcd4/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-11-12_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_8.52.28.png)
        
    - view i에서 reference view로 warping하는 공식 (여기서 reference의 depth는 z)
        - 저 아래첨자 1이라고 하는게 무엇일까 (1번 뷰?)
        - n: normal?? depth가 있으니..?
    - 워핑은 다음 공식을 기반으로
        
        ![스크린샷 2022-11-12 오후 8.54.41.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/437a8dc7-1543-4926-b701-ec947a22ef93/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-11-12_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_8.54.41.png)
        
        - output이 warped feature map at depth z
        - u, v는 pixel location (reference view의)
        - 논문에서는 normalized device coordinate를 이용해 u, v, z를 파라미터화
        - **NDC가 먼데?**
3. Cost volume(P)
    - warped feature map F로부터 만들어지는 것 (on the D sweeping planes)
    - Cost 계산: variance-based metric..? (MVS에서 이미 많이 쓰는 방법이래 레퍼 참고)
    - Cost feature vector 계산
        
        ![스크린샷 2022-11-12 오후 9.03.03.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a08e3f9d-8c19-4a87-9dc0-5a95592555f2/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-11-12_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_9.03.03.png)
        
        - 각 voxel별 계산
        - u, v, z가 voxel center 값인가봐..?
        - var: M개 view에 대한 variance가 계산된다.
    - 위 방법을 쓰는 이유..?
        - 다른 input view들로부터 appearance variance를 인코딩하는 역할이라고 한다.
        - variance는 geometry와 view-dependent shading effect로부터 만들어진다.
        - 단순 geometry만을 기반으로 한 방법에 비해서는 realistic result 얻을 수 있다.
    - 궁금한건 이 벡터의 아웃풋은 몇차원이고 어떻게 구성된건데..

## Radiance field reconstruction

- 3D CNN을 이용해 neural encoding volume을 cost volume으로부터 복원한다
- Neural encoding volume은 per-voxel feature로 구성됨
- 이 feature는 local scene geometry와 appearance를 인코딩하고 있음
- Encoding volume에서부터 volume rendering properties를 얻기 위해 MLP 디코더 사용

1. Neural encoding volume
    - 기존 MVS는 depth probability를 cost volume으로부터 직접 구함
    - 이 방식은 scene geometry만을 표현
    - 논문에서는 여기에 appearance-aware information을 얻고자 함도 포함
    - 그래서 만들어진 image-feature cost volume으로부터 새로운 C-채널의 neural encoding volume을 얻기 위한 3D CNN 사용
    - 여기서 3D CNN은 3D UNet 사용
        - downsampling, upsampling, skip-connections…
        - scene appearance information을 추론하고 전파해서 더 의미있는 encoding volume 만들기 위해..!
    - 여기서 이 볼륨은 unsupervised way로 만들어진대!
    - 그래서 최종적으로는 per-voxel meaningful scene geometry and appearance 담기
        - 나중에 이 특징들이 연속적으로 interpolated
        - 그리고 volume density & RGB로 converted
    - 이 볼륨은 상대적으로 저해상도 결과물이 된다.
        - 원인: downsampling of 2D feature extraction
        - 고주파 appearance를 추론하기 위해서는 이러한 조건이 challenging
    - 위와 같은 부분에 대해..
        - original input image pixel을 volume regression stage에 함께 병합했다 하는데
        - 이거는 3.4 섹션에서..
2. Regressing volume properties
    - 임의의 3D location과 viewing direction이 주어졌을 때
    - MLP를 통해 이에 상응하는 volume density와 RGB 값을 neural encoding volume으로부터 얻는다.
        - 음.. 3차원 위치와 viewing direction을 통해 encoding volume에서 해당 위치의 값을 가져와서 이를 기반으로 표현을 얻는다가 절차일까?
    - pixel color가 추가적인 입력으로 들어간다.
        - pixel 위치 u,v에 대해..
        - 3차원 위치를 view i로 투영했을 때의 그 위치라고 한다!
        - color는 전체 M개 view에 대한 값을 concatenate
        - 즉, 3M-channel vector가 만들어진다
    - 최종적인 입력
        - 3D position
        - viewing direction
        - 3M-channel color vector
        - volume feature from given 3D position (trilinear interpolation?)
    - 이것들을 MLP에 넣어서 RGB-density 얻기
    - 3D position x는 reference view의 NDC space안에 파라미터화 된다(???)
    - d: unit vector로 표현됨
    - NDC space 씀으로써 normalize가 용이했나봄 (데이터 소스가 다양하게 들어와도 그거에 맞게 스케일 맞출 수 있나보다. 일단은 기본 노말라이즈된 값이 있으니)
    - positional encoding은 얘네도 갖다 씀 (이 인코딩이 확실히 고주파 값 표현에 강인한가봐)

## Volume rendering

- MVSNeRF에서는 주로 neural encoding volume을 복원하는게 메인 컨트리뷰션인 듯
- Differentiable ray marching 기법을 썼는데 알고보니 nerf도 이걸 썼었다고 한다?!
    - 설명 넣어주면 좋겠다
    - 공식말고

[김성완 교수, "복잡하고 기묘한 도형의 구현은 단순하다"](https://www.inven.co.kr/webzine/news/?news=165131&site=hit)

- Ray tracing에 대해
    - 그러나, 레이 캐스팅은 정확하게 말한다면 전역 조명을 위한 것이 아니라 가시면 검출을 위한 것이다, 그럼에도 불구하고 이것을 이야기한 것은 다음에 이야기할 레이 트레이싱(ray tracing, 광선 추적)에 중요한 밑바탕이 되었기 때문이다. 레이 캐스팅은, 광선을 한번 던진 다음에 물체와 교차점 검사만 수행하기 때문에, 실질적으로 이 광선이 다른 물체에 영향을 미치는 것은 아니다. 즉, 각 픽셀로부터 가장 가까운 물체에 이르는 단일 광선만 고려한다. 레이 트레이싱은, 레이 캐스팅이 이렇게 한번 광선을 던진 이후에도 빛을 추적한다. 즉, 표면에 빛이 닿았을 때 이 표면이 빛을 받는 것인지 그림자를 만드는 것인지만 결정하는 것이 아니라, 표면에 닿는 순간 반사, 굴절, 그림자 광선과 같은 추가적인 빛이 다시 생성된다고 가정한다. 이렇게 생성된 각각의 빛들은 또 다시 다른 표면에 대해 같은 작용을 하게 되고, 이것은 레이 캐스팅보다 훨씬 현실에 가깝게 재현할 수 있다.
    - 그러나, 레이 트레이싱은 막대한 계산량이 필요하다는 단점이 있다. 특히, 광원에서 나오는 모든 빛에 대해 이렇게 광선 추적을 정방향 추적은 현실적으로 불가능하기 때문에, 관찰자의 시선으로부터 광선을 추적하는 역방향 추적을 많이 사용한다. 그럼에도 불구하고 레이 트레이싱은 실시간으로 처리하기 매우 어렵다. 이런 계산량을 줄이기 위한 여러가지 방법이 있는데, 그림자 광선을 먼저 추적하는 방법이 있다, 즉, 그림자 광선을 먼저 적용하여 보이지 않는 부분에 대해서 빛을 추적하지 않는 것이다. 어떤 표면이 빛을 향하고 있을 때, 광원과 표면의 교점을 구하고, 이 사이에 어떤 불투명한 물체가 존재한다면, 이 빛은 이 표면에 아무런 영향을 미치지 못하기 때문이다. 이외에도, 모든 빛들이 눈에 들어오지 않고 감쇄가 일어나므로, 이 감쇄 효과에 의해 완전히 소멸된 빛은 더 이상 추적할 필요가 없다.
    - 레이 트레이싱은, 다른 렌더링 알고리즘을 사용했을 때 구현하기 곤란한 그림자, 반사 효과 따위가 알고리즘 자체에 자연스럽게 포함되어 있다는 장점을 갖는다. 또, 상대적으로 다른 알고리즘에 비해 구현이 비교적 간단하며, 광선 하나 하나가 다른 빛과 상관없이 개별적인 계산 과정을 거치므로 병렬화에 유리하다는 장점도 있다.
- Ray marching에 대해
    1. **ray casting**
        - Ray가 처음으로 hit하는 곳에서 stop
        - Reflection, ray splitting 그런 기법 전혀 없음
        - per-pixel casting 단위도 아니라고 한다. (주로 screen의 per-row, per-column)
    2. **(back) ray trace**
        - 물리적인 특성이 고려됨 (light의)
        - 한 번이 아니라 ray가 여러번 hit하는 경우에 스톱한다고 한다.
        - 좀 더 디테일한 기법이라 할 수 있다.
    3. **path tracing**
        - 재귀적인 ray split을 막기 위한 최적화 기법
        - monte carlo integration 사용
        
        is optimization technique to avoid recursive ray split in ray trace using Monte Carlo (stochastic) approach. So it really does not split the ray but chose randomly between the 2 options (similarly how photons behave in real world) and more rendered frames are then blended together.
        
    4. **ray marching**
        - signed distace function 기반 최적화 (hit 기반 말고 단위 기반으로 sample)
- Ray marching은 fully differentiable

## End-to-end training

- 여기도 최종적으로 L2 loss를 통해 supervised learning을 하며, 모든 과정이 end-to-end다
- 차이점
    - nerf는 per-scene training
    - mvsnerf는 different scene을 이용한 training

## Optimizing the neural encoding volume

- input 갯수를 적게 가져간다는게 MVSNeRF의 강점일 수 있는데
- 반대로 이로 인해 완벽한 결과 얻는 부분에 있어서 이슈가 있다고 했다
- Vanilla NeRF
    - 일부러 hard generalization을 안했다
    - per-scene optimization방식이기도 해서고
    - 입력도 dense하게 잔뜩 가져가기도 하고
    - 그래서 실사 렌더링은 가능한데
    - 너무 계산량이 많아
- 최적화 기법을 통해 빠른 fast per-scene optimization을 할 수 있는 방법 제안
    - neural encoding volume fine-tuning
    - 근데 마치 dense image 쓴다는거처럼 나와있는데
    - 덴스한 인풋은 트레이닝할 때 쓰고, 추론 과정에서는 3개 뷰만으로도 가능하게 하는 전략인가
1. Appending colors
    - Neural encoding volume + input pixel color를 합쳐서 MLP로 보냄
    - 물론 이 방법이 동작은 잘하는데 3장의 input에 대한 의존도가 너무 높아진다.
    - independent neural reconstruction..?
        - per-view color를 encoding volume의 voxel center에 추가 (additional channel로써)
    - 이 방법만 쓰면 일단 blurring 문제가 있었다고 하네..?
2. Per-scene Optimization
    - Fast per-scene optimization 이용을 위함
        - 이건 dense image inputs을 사용할 수 있을 때
    - 이 과정에서의 최적화는 neural encoding volume과 MLP만
    - 즉, 전체 네트워크 최적화를 하지는 않는다
    - per-voxel local neural feature를 optimization을 통해 독립적으로 조정(?)
    - 코드 봐야 이해가 갈 프로세스인가
    - 아무튼 이 과정이 shared convolutional operation을 최적화하는 것보다 더 쉽대
    - 무엇보다 이 최적화는 다음 과정의 expensive processing을 피하게 해준다.
        - 2D CNN
        - plane-sweep warping
        - 3D CNN
    - 이점
        - clean neural reconstruction
        - independentn of any input image data (이건 컬러 채널 추가한게 한몫함)
    - More details
        - 정확하게는 각 scene에 대해 Neural encoding volume과 MLP decoder를 최적화하는 것이 목표
        - RTX 2080Ti 기준 15분 걸림 (10k training iterations)
        - 반면 nerfsms 10시간 걸림 (200k trainint iterations)
        - frustum 밖에 있는 부분에 대해서는 artifact가 만들어진다라..
        - boundary voxel에 그래서 padding 추가함
        - nerf는 two-stage ray sampling을 사용 (이건 처음 들었는디)
        - 근데 mvsnerf는 uniformly sample points (along each marching ray)
        - 128개를 기본으로 갔지만, challenging scene에 대한 fine-tuning 관점에서는 256 points를 쓰기도 함
    - Progress
        - 여기서는 progress 비교 (proposed fine-tuning vs. NeRF’s from-sctatch method)
        - 일단은 strong initial reconstruction이 한몫했다고 함 (아마 encoding volume)
        - 아니 비교만 넣었지 그래서 뭘 최적화했다고!!
        - 코드보는게 빠르겠고만

## Implementations details

1. Datasets
    - DTU dataset 사용 (일반화가 가능한 네트워크 학습)
    - Follow PixelNeRF (데이터 분할 측면)
        - 88개의 training scene
        - 16개의 test scene (아마 val 말하나보다)
        - resolution 512 x 640
    - Test는 Synthetic NeRF data & Forward-facing data 사용
        - training 데이터와 비교하면 다른 scene과 view distribution을 가졌다.
        - 20개의 인접한 view를 먼저 고르고
        - 여기서 최종 3개의 center view를 골랐다. (20개들 중에서 center에 해당하는 3개 뷰?)
        - 13개는 추가 input(per-scene fine-tuning 목적)
        - **4장 남은걸로 test 했다고 한 거 같은데 (이 view에 대한 rendering 검사한건가? 이건 돌아가는 코드를 봐야 알 거 같아)**
2. Network details
    - f=32 (volume feature channle)
        - feature extraction, cost volume, neural encoding volume 모두 연관
    - D=128 (depth hypotheses)
        - plane sweeping volume
        - sweeping 갯수인가보다
        - uniformly sampled from near to far
    - MLP 구조
        - NeRF랑 비슷한데
        - 6개의 layer만 사용 (?!)
        - NeRF가 two radiance field를 복원했다고?? (coarse, fine)
            - 너프 MLP 다시 봐야겠는뎅..
        - 근데 MVSNeRF는 하나의 radiance field만 복원
    - Ray marching
        - 128개의 shading point를 사용 (sampled..)
    - Training
        - RTX 2080 Ti 사용
        - Training image에서 1024 pixel을 sampling
        - 하나의 novel-viewpoint에서 픽셀 샘플링하며 이걸 하나의 batch로써
        - Adam optimizer
3. Architectures
    - 2D CNN: 2D feature map 뽑는 용도
        - input: Image
        - output: feature map
    - 3D CNN: Neural encoding volume 뽑는 용도
        - input: Image, feature map
        - output: neural encoding volume
    - MLP: volume properties 뽑는 용도
        - input:x, f, c, d
        - output: RGB-density
    - CBR2D: ConvBnReLU2D
    - CBR3D: ConvBnReLU3D
    - CTB3D: ConvTransposeBn3D
    - LR: Linear ReLU
    - PE: Positional encoding