---
title: "Nerf"
date: 2022-07-07T17:44:00+09:00
draft: true
---

[Code Review는 여기에](https://www.notion.so/Code-Review-f1f707d50e404116b42a3ca03ccef8be)

# Representing Scenes as Neural Radiance Fields for View Synthesis

![스크린샷 2022-07-05 오후 11.46.36.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3b114e7c-3c8d-4f90-a394-1d637b5d50e6/스크린샷_2022-07-05_오후_11.46.36.png)

## 초간단 요약!(박은수님)

- Radiance Field 생성에 Neural Network를 적용했기에 이를 Neural Radiance Field, 줄여서 NeRF라고 부름 (보통 ‘너ㄹ프’라고 읽는듯 합니다^^)
- 3D 모델의 좌표 X,Y,Z와 이를 바라보는 방향벡터 theta와 phi를 입력받아 그 위치 및 시점의 RGB값과 그 값의 density를 출력 함
- 모델은 간단한 MLP(Multi Layer Perceptron)를 사용함
- 좋은 결과를 얻기위해서 입력을 positional encoding한 벡터들로 변환하고 ray의 coarse smapling과 fine sampling을 결합하여 효율을 높임

## Intro

- 어떤 3차원 객체가 있다고 가정할 때, 이 객체는 3차원 좌표를 가지고 있을 것이다.
- 이 3차원 물체는 어느 시점에서든 바라볼 수 있을 거야
- 그 바라보는 시점에 따라 다른 시점에서 안보이던 면이 보이기도, 빛이 반사되는 부분이 있을수도 있고…
- 즉, 특정 물체를 우리 눈에 보이게 하려면 객체 고유의 좌표 (x, y, z)와 바라보는 방향 값($\theta , \phi$ )이 필요하다. (실제로 2차원 viewing direction은 3D vector를 사용한다고 한다.)
- 이것도 하나의 implicit neural representation 방법
- Discrete한 N개의 시점에서 찍은 2차원 image로 학습해서 임의의 continuous한 시점에서 찍은 2차원 image를 만들어내라
- 입력이 (x, y, z, $\theta , \phi$ )일 때, 해당 시점에서의 객체에 대한 출력 값 (R, G, B, $\sigma$)을 얻는다. (여기서 RGB는 emitted radiance, sigma는 볼륨의 density)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/31b8caa3-3d7c-47af-ac20-eea929442ca5/Untitled.png)

- NeRF는 연속적인 scene 표현을 위해 parameter를 직접 최적화하는 방식으로 오류를 줄였다고 한다.
- 방법론에 대해
    - camera rays를 통해 샘플링을 거쳐 3차원 위치 집합을 생성한다. (ray를 지나는 공간상의 3차원 포인트 집합)
    - 입력 3차원 정보에 대한 RGB + density 값을 알게 되면 이거를 기반으로 volume rendering 수행 (2차원 projection)
- NeRF의 모든 과정은 미분 가능한 과정이기에 gradient descent를 통해 관측 이미지와 생성된 이미지의 오차를 최소화하여 모델을 최적화할 수 있다.
- 큰 의의가.. RGB 이미지에서 실제 객체의 사실적인 뷰를 렌더링할 수 있는 최초의 방법이라고는 한다.

## Related works

- 기존 방법은 복잡한 geometry가 들어오면 성능이 떨어졌다고 한다.
- Neural 3D shape representations
    - x, y, z 좌표를 바탕으로 signed distance function 기반으로 shape 표현
    - 배경화면이 깔끔한 물체이어야만 이 방법  적용이 가능하다는 한계가 있음
- View synthesis
- Image-based rendering

## Neural Radiance Field Scene Representation

- Neural radiance field: 각 위치 값을 입력으로 주면 RGB 값을 출력으로 주도록 그 값을 연산하는 함수 형태

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/dfb6bc30-0fad-48ec-a3f3-848f0a97026c/Untitled.png)

- 임의의 입력 뷰 집합으로부터 continuous volumetric scene function을 학습하여 새로운 뷰를 합성하는 방법
- 근데 여기서 위 함수를 아예 deep neural net으로 정의한대
    - 일반적으로는 train → test하는데
    - 여기서는 deep neural net을 initialize하고, 대상 이미지 하나에 대해서 학습하는 과정 자체를 rendering으로 간주(일종의 zero-shot learning)
    - 즉, train부터 그냥 그 자체가 inference! test할 때 input이 들어오면 그 input에 대해서만 학습을 새로 하는 개념
- density
    - 투명도의 역수 개념이랄까…
    - 이 값이 크면 값이 견고해서 물체 자체의 rgb값이 더 잘 보이고 물체 뒤는 거의 안보임
    - 반대로 값이 작으면 rgb 값이 견고하지 못해 물체 뒤가 반투명처럼 서서히 보이기 시작하는…
- 출력 값을 얻으면 이를 토대로 고전적인 volume rendering 기술을 적용
- Multi layer perceptron 구조
- 아래 그림이 핵심 알고리즘

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8b5975cb-9e96-4340-9dd3-282830fca2da/Untitled.png)

- pixel 값: 한 ray 위에 존재하는 point들의 RGB값을 weighted sum!
- 모든 point에 대한 RGB+Density 값이 분포도로 (c) 나오게 되고
- 출력 값을 실제 정답과 rendering loss 함수를 통해 학습 진행
- 뉴럴 네트워크가 갖게 될 weight 파라미터들 자체가 scene의 representation이라 말할 수 있다고 한다 (무슨 의미이지..)
- 네트워크 구조
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/dee18dcb-cd2c-45c2-abf7-6a040eb0d3eb/Untitled.png)
    
    - 3차원 객체가 가지고 있는 고유한 특징 벡터는 위치 정보만을 입력으로 사용하여 얻어낸다. → 여기서 density 값을 얻고
    - 특징 벡터와 입력 카메라 방향 정보를 연결하고, 이를 128 채널의 MLP에 통과시켜서 RGB 값을 얻는다.
    - 위 네트워크 구조에서 감마 함수는 입력 좌표를 positional embedding한 것
    - 이 모델에서의 가정
        - density: 위치에 의존적
        - RGB: 공간 정보, 방향 정보, 밀도 정보에 의존적
    - 중간에 60차원짜리 하나 추가로 들어간 이유: 정보 손실을 고려하여 입력 값을 추가로 넣어줌

## Volume Rendering with Radiance Fields

![스크린샷 2022-07-05 오후 9.54.01.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ac7f1c28-458d-4c9f-a5f6-57c4219530f9/스크린샷_2022-07-05_오후_9.54.01.png)

- 연속적인 ray r에 대한 color function은 아래와 같다.
    - 목적은 결국 weight 찾기 같은데 (weighted sum이니)
    - Density가 클 수록 weight가 커야 한다.
    - 그 지점을 가로막고 있는 점들의 density 합이 작을수록 weight가 커야 한다.
        - 예를 들어.. 렌더링하고 싶은 객체가 있는데
        - 객체 자체 density 큰 것은 좋은데
        - ray 전체에 density가 있을텐데
        - 객체 앞에 장애물이 있거나 한다고 해보자. 얘의 density가 높으면 아무리 객체 density가 높아도 객체는 가려질거야
        - 결국 객체와 카메라 사이 전체 파티클에 대한 density 합은 작아야만 한다!

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/aa0ae4d5-a598-4430-9dcd-56a826f03500/Untitled.png)

- r(t) = o+td (o: 카메라 원점 벡터, d: 방향 벡터, t: continuous한 시점?)
    
    ![스크린샷 2022-07-05 오후 10.28.30.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/61accce8-cc34-4c99-ab3d-3043b9c92e1a/스크린샷_2022-07-05_오후_10.28.30.png)
    
- sigma: density term(model output으로부터)
- c: color term(model output으로부터)
- t_n: 이미지의 시작점 (렌더링되는 image plane)
- t: weighted sum을 진행할 객체를 ray가 통과하는 시점
- 즉, 전체 color에 대한 weighted sum
    - weight 1: density
    - weight 2: T (image plane to object 적분)
        - exp에 -가 붙은 함수 → density 작을수록 weight가 커짐
        - t_n부터 t까지. 즉, 가장 가까운 image plane부터 객체까지
- Radiance field
    - 어떤 위치에서든 density와 directional emitted radiance(RGB)로 장면을 표현
    - 고전적인 볼륨 렌더링 방식을 적용하여 모든 ray passing에 대한 색상을 rendering한다.
    - 위 식에서 T(t)는 t 지점까지 ray의 투과율을 나타낸다.
        - Density는 물체가 있는 곳에서 높을 텐데
        - 만약 장애물로 인해 density가 높은 곳이 있으면 오히려 높은 값에 대한 projection을 더 많이 해야하기 때문에
        - 어느정도 무효화를 시킬 수 있는 term이라..? (오히려 어려운데 이 말이)
    - 이는 ray가 위치 t까지 이동할 확률로 해석할 수도 있다.
    - 여기서 렌더링을 위해서는 이 적분함수의 결과를 추정해야하기 때문에 구분구적법을 사용하는 듯하다.
        
        ![스크린샷 2022-07-05 오후 11.27.39.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/31a62bce-df6d-4f11-82d5-98ac170bc71d/스크린샷_2022-07-05_오후_11.27.39.png)
        
        - [t_n, t_f]라는 범위를 준다. 왜냐하면 ray에 대해서 무한한 t를 다 볼수는 없으니 유한한 범위 t_n, t_f라는 범위를 준다.(near, far 줄임말)
        - [t_n, t_f]를 균등하게 분할하고 각 부분에서 uniform sampling한 데이터를 사용 (범위 하나하나의 단위에서 랜덤한 영역의 값을 sampling)
            
            ![스크린샷 2022-07-05 오후 11.10.17.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e7e5cc0e-1c4c-49f1-8d64-daffecb8ee21/스크린샷_2022-07-05_오후_11.10.17.png)
            
        - 샘플은 discrete하지만, continuous한 장면 표현이 가능했다고 한다.
        - 추정을 위해서 quadrature rule 적용
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7e7c50c1-75b0-4f6b-854e-7717d8bbdbe7/Untitled.png)
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6e0e7100-ff6c-4bb8-b371-f5d83358c1cc/Untitled.png)
        
        - 위 델타는 인접한 두 샘플 간 거리
        - color function에서 density 부분이 1-exp 함수로 바뀌었네?
            - y=x와 y=1-exp(x) 함수는 경향성이 비슷해서 근사화 목적으로 변형한 듯

## Optimizing a Neural Radiance Field

### 1. Positional encoding

- 목적: 뉴럴 네트워크로 픽셀 좌표를 직접 사용할 경우, 색상 및 고주파 변동을 잘 표현하지 못하는 것을 발견해서 이를 해결하고자 함
- (뉴럴 네트워크가 저주파 함수를 학습하는 쪽으로 biased되었다고 한대!)
- 입력을 고차원 공간에 맵핑하면 고주파 변동이 있는 데이터를 잘 표현할 수 있다고..!

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ab525628-33ff-4c5e-a0db-1f8f44dc1c95/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3ca132d4-c9f4-4c9b-8629-32209f7472e5/Untitled.png)

- 4, 5라는 값이 있을 때, 이걸 MLP를 태우면.. MLP는 결국 weighted sum이기 때문에 비슷한 결과가 나올 것이다.
- 가깝건 멀건 다른 벡터로 만들어줄 필요가 있을 때…
- 위 식에서는 많은 삼각함수를 넣어서 고주파로 표현을 바꿔준 결과
- 아래 결과를 보면 positional encoding을 적용할 때, 반사가 잘 표현된 것을 알 수 있다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5e9b545b-b7e2-4a5b-9651-2c112bb2dee2/Untitled.png)

- 논문에서의 하이퍼파라미터 L
    - 위치 정보(x, y, z)에 대해서는 L을 10으로 주어 3차원 정보를 총 60차원으로 확장 (읭..? sin cos 두개있는데 그럼 곱하기 2 아닌가?)
    - 방향 정보(theta, phi)에 대해서는 L을 4으로 주어 2차원 정보를 총 24차원으로 확장 (4x2x2)
        - 왜 발표 등에서는 자꾸 24라고 했는지.. 계산 결과는 16차원인데?
        - 방향 정보가 2차원이긴 한데, 실제로는 cartesian coordinates를 써서 3차원으로 표현했다고 한다!!

### 2. Hierarchical volume sampling

- Coarse한 샘플링으로 결과가 뽑히면 이 결과를 바탕으로 좀 더 fine한 샘플에서도 출력 결과를 뽑아서 최종 출력 값을 뽑는 방법
- 왜냐하면 camera ray를 따라 샘플링된 샘플 모두를 평가하는 것은 효율적이지 못함
- 즉, 어떤 sample을 선택해서 training에 사용할 지 선정하는 부분일 듯!
- 두 개의 네트워크가 동시에 제안되었다고 하는데
    
    ![스크린샷 2022-07-05 오후 11.34.13.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d032fc06-f7f8-4f79-9ae4-7b25aa72dc06/스크린샷_2022-07-05_오후_11.34.13.png)
    
    - Coarse network: hierarchical sampling을 사용하고 네트워크를 평가
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/831333bb-fc60-42e0-92b8-6b8c1320b892/Untitled.png)
        
        - c: sampled colors along the ray
    - Fine network
        - Coarse의 출력을 감안해 더 많은 정보를 가진 지역에서 다시 샘플링을 진행
        
        ![스크린샷 2022-07-05 오후 11.16.52.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/95e3b0ac-f328-4f0d-9a74-8402774e087e/스크린샷_2022-07-05_오후_11.16.52.png)
        
        - 위 그림에서 분포 반응이 큰 곳에 객체가 있을 확률이 높으니
        - 여기서 N_f개만큼 추가로 뽑는다
        - 이걸 판단하는 근거 - inverse transform sampling
            
            ![스크린샷 2022-07-05 오후 11.38.08.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ee367415-abf6-4653-a901-fef55911db1f/스크린샷_2022-07-05_오후_11.38.08.png)
            
            ![스크린샷 2022-07-05 오후 11.38.44.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/cb044488-c609-4f40-9054-7619c5c679fc/스크린샷_2022-07-05_오후_11.38.44.png)
            
            ![스크린샷 2022-07-05 오후 11.41.14.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/653432e4-943c-4828-82ec-d2b01af8b268/스크린샷_2022-07-05_오후_11.41.14.png)
            
            - 
        - 두 개의 샘플 세트의 합집합을 사용해 네트워크를 평가하여 최종 색상을 계산한다.
        - 즉, fine network는 N_c + N_f개만큼!

### 3. Implementation details

- 각 scene에 대해 별도의 네트워크를 사용한다.
- 각 iteration에서 hierarchical volume sampling을 수행하고
- 이후 volume rendering 사용
- loss: coarse & fine 모두에 대해 렌더링된 픽셀 색상과 실제 픽셀 사이의 L2 distance를 계산한다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/50697179-26e2-4fdf-bff2-75f828a621a6/Untitled.png)

- Batch size: 4,096 rays
- N_c: 64, N_f: 128 (뭔데 이게)
- 소요 시간이 미침… V100 기준 1~2일 (100~300k iteration)

[Results](https://www.notion.so/Results-d258f5748d75409a800f131f097d765c)

-