---
title: "Point NeRF"
date: 2023-03-07T19:01:43+09:00
draft: true
---

## Arxiv GPT

> *The paper "Point-NeRF: Point-based Neural Radiance Fields" proposes a method to combine the advantages of volumetric neural rendering and direct multi-view stereo methods by using a neural point cloud with associated neural features to model a radiance field, which can be rendered efficiently by aggregating neural point features near scene surfaces in a ray marching-based rendering pipeline. The method achieves state-of-the-art results on multiple datasets and can handle errors and outliers in 3D reconstruction methods via a novel pruning and growing mechanism.*
> 

이 논문은 Volumetric neural rendering과 Direct multi-view stereo 방식의 이점을 결합한 방법을 제안한다. 

여기서 논문에서는 Radiance field를 얻기 위해 Neural feature와 결합한 Neural point cloud라는 것을 활용한다.

렌더링 파이프라인에서는 Scene surface 근처에 있는 neural point feature들을 잘 aggregate하여 ray marching 기반의 방법을 활용해 렌더링을 수행한다.

추가적으로 Pruning과 growing 메커니즘을 통해 에러와 아웃라이어 핸들링도 잘 해서 다양한 데이터셋에서 SOTA급 결과를 보이고 있다.

**Key insights and lessons learned:**

- Point-NeRF combines the advantages of volumetric neural rendering and direct multi-view stereo methods by using a neural point cloud with associated neural features to model a radiance field, which can be rendered efficiently by aggregating neural point features near scene surfaces in a ray marching-based rendering pipeline.
- Point-NeRF can be initialized via direct inference of a pre-trained deep network to produce a neural point cloud, which can be finetuned to surpass the visual quality of NeRF with 30X faster training time.
- Point-NeRF can handle errors and outliers in 3D reconstruction methods via a novel pruning and growing mechanism.
- The method achieves state-of-the-art results on multiple datasets, including the DTU, NeRF Synthetics, ScanNet, and Tanks and Temples.

**Questions for the authors:**

1. How does the proposed pruning and growing mechanism work to handle errors and outliers in 3D reconstruction methods?
2. Can Point-NeRF be applied to real-time applications, such as augmented reality or virtual reality?
3. How does the efficiency of Point-NeRF compare to other state-of-the-art methods for neural rendering and 3D reconstruction?
4. How sensitive is the performance of Point-NeRF to the density and quality of the input point cloud?
5. Are there any limitations or failure cases of Point-NeRF that you have observed in your experiments?

**Future research directions:**

1. Investigating the potential of combining Point-NeRF with other neural rendering and 3D reconstruction methods.
2. Exploring the scalability of Point-NeRF for large-scale scenes and datasets.
3. Evaluating the performance of Point-NeRF in different types of scenes and environments, such as indoor versus outdoor scenes or scenes with different lighting conditions.
4. Examining the interpretability and explainability of the learned neural radiance fields in Point-NeRF.
5. Generalizing Point-NeRF to handle dynamic scenes or objects that change over time.

**Relevant references:**

1. Mildenhall, B., Tancik, M., Barron, J.T., Gordon, D., & Ng, R. (2020). Nerf: Representing scenes as neural radiance fields for view synthesis. In European Conference on Computer Vision (ECCV) (pp. 405-421).
2. Tatarchenko, M., Dosovitskiy, A., & Brox, T. (2015). Single-view to multi-view: Reconstructing unseen views with a convolutional network. In IEEE International Conference on Computer Vision (ICCV) (pp. 3498-3506).
3. Wang, Y., Liu, Z., Zhu, F., Liu, C., & Loy, C.C. (2021). High-fidelity neural volumetric rendering using continuous hierarchical features. In IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR) (pp. 9071-9080).
4. Dai, A., Nießner, M., Zollhöfer, M., Izadi, S., & Theobalt, C. (2017). BundleFusion: Real-time globally consistent 3D reconstruction using on-the-fly surface reintegration. In ACM Transactions on Graphics (ToG)

## Intro

- 기존의 Per-scene optimization 속도가 느리다는 점을 지적 (NeRF)
- Deep MVS: network inference를 바로 써서 (아마 NeRF처럼 씬 마다 트레이닝 안하는 점을 말하는 것 같다.) 빠르게 scene geometry를 복원할 수 있다.
- Point-NeRF는 Deep MVS의 빠른 scene geometry 복원과 NeRF의 high-quality view synthesis 이점을 모두 살린 방법
    - 여기서 Neural 3D point cloud를 사용했다고 한다.
    - 이 뉴럴 포인트 클라우드는 뉴럴 피쳐와 관련된 무언가 같고
    - 결과적으로 radiance field를 모델링하는 역할을 하나보다.
- 렌더링은 기존처럼 ray-marching 파이프라인대로 진행한다.
- 뉴럴 포인트 클라우드 생성: 사전 학습된 뉴럴 네트워크 이용한다?
    - 이 결과물은 파인 튜닝되어서 …

## Point-NeRF Representation

### 1. Volume rendering and radiance fields

- 보통은 알다시피 ray marching으로!
- r을 radiance라고 하는 중
    - 목적에 따라 컬러 혹은 밀도
- 델타가 distance

### 2. Point-based radiance field

- 뉴럴 포인트 클라우드라는 이야기가 계속 나온다.
    - 총 N개의 point에 대해서
    - 각 포인트는 포지션 p에 있고
    - 각 포인트는 뉴럴 피쳐 벡터 f와 연관되었으며
    - 이 뉴럴 피쳐 벡터에는 local scene contents가 인코딩되어 있다.
    - 추가로 각 포인트에는 scale confidence value가 할당되어 있다.
        - 감마라고 부르는 그것
        - 값은 0-1사이
        - 의미: 실제 scene 표면 근처에 해당 포인트가 있을 확률
- 3차원 포인트의 위치 값 x가 주어지면
    - K개의 이웃한 neural point들을 찾는다.
    - 이는 정해진 radius R 안에서 찾는다.
- point-based radiance field는 뉴럴 모듈로써 요약되어진다.
    - regress volume density & view-dependent radiance (color?)
    - 뉴럴 래디언스 필드의 역할을 말하는 듯.
- 그래서 포인트 너프를 공식화한 것을 보면
    - 컬러와 밀도를 구하는 함수에
    - 입력 값이 3D x position, viewing direction d, K개의 뉴럴 포인트 클라우드 값들
- 뉴럴 네트워크의 대략적인 구조
    - PointNet 형태의 구조라고 한다.
    - 다수의 sub-MLP들이 포함된 구조
- 전반적으로, 각각의 뉴럴 포인트에 뉴럴 프로세싱(네트워크 통과시키기)을 진행하고, 멀티-포인트 정보를 통합해서 최종 결과를 얻는 형태 (너프가 원래 그랬다.)

### 3. Per-point processing

- MLP F를 통해 각각의 이웃한 뉴럴 포인트로부터 새로운 피쳐 벡터를 추정한다. (셰이딩 위치 x에 대해)
    - 이 MLP의 입력: 기존 feature vector, x와 p_i간 거리
    - 원본 특징 벡터는 p_i 주변의 로컬한 3차원 scene content를 인코딩하고있는 중이라면
    - 새로운 특징 벡터는 x에서의 특정한 뉴럴 scene description을 갖는다 (무슨 소리야 이게)
    - 입력으로 relative position (x-p)를 사용하면서 네트워크 invariant를 갖게 만들었나봄
        - point translation에 대해.. 그래서 일반화가 더 잘되는 형태라고 함

### 4. View-dependent radiance regression

- inverse distance weighting?
    - 목적: neural feature aggregation
    - K개의 feature들을 하나의 feature로
    - w를 통해서 distance의 역수를 weight로 삼는다.
    - 선택한 이유: 흩어진 data들에 대한 interpolation 목적이라고 함 (보통 그러한 목적에서 많이 쓰임)
    - 즉, 더 가까운 뉴럴 포인트에 shading computation 비중을 높임
    - confidence(gamma)를 쓴 목적은 outlier rejection을 위함
- 다음 MLP R
    - view-dependent radiance를 regress하고 있다. (위에서 얻은 최종 피쳐로부터)
    - 그래서 입력이 viewing direction + feature vector

### 5. Density regression

- 위쪽은 radiance regression이고, 이번엔 밀도!
- 위랑 유사항 multi-point aggregation이 들어가지만
- 다만 먼저 각 포인트 i번째에 대해서 MLP T를 돌려서 각각의 density를 먼저 추정한다 (입력: feature vector)
- 그런 다음 inverse distance 기반의 weighting을 수행
- 각각의 뉴럴 포인트들은 volume density와 point confidence에 직접적으로 영향을 끼친다..
    - 반대로 영향 받기도 함!
    - 예를 들어 컨피던스가 낮으면 밀도도 낮아
    - 확 낮아지면 해당 영역은 empty일수도?

## Point-NeRF Reconstruction

### 1. Generating initial point-based radiance fields

- Q개의 주어진 이미지가 있고, 포인트 클라우드도 있을 때
    - Point-NeRF representation은 랜덤하게 초기화된 per-point neural feature들을 최적화하는 것과 렌더링 loss 기반의 MLP를 통해 복원된다고 한다. (NeRF 이야기…?)
- 근데 이러한 원론적인 방법은 point cloud 의존성이 강하다고 함, 그래서 속도가 느리대..(아 그래?)
- Neural generation module 제안
    - 목적: 모든 뉴럴 포인트 property에 대해 추정을 수행하는 모듈
    - 그럼 한꺼번에 다 추정하는 형태야?
    - 뉴럴 포인트 클라우드가 입력으로 들어가는 모듈인가봄
    - 이 모듈의 인퍼런스 아웃풋: 좋은 initial point-based radiance field
    - 이후에 고퀄리티 렌더링을 위해 파인 튜닝도 이루어짐 (뭔진 모름)
- 효과: 짧은 주기만에 렌더링 퀄리티가 올라갔다! (이건 결과를 통해)

**Point location and confidence**

- Point location 취득 방법
    - Deep MVS 기법 - 특히 Cost volume 기반의 3D CNN
    - 도메인에 대해 일반화를 잘 하며, 고퀄리티의 dense geometry를 얻기에 좋음
    - 입력: 각 뷰포인트에 대한 이미지와 카메라 파라미터
    - MVSNet처럼 plane-swept cost volume을 취득하고
        - 방법: 2차원 이미지 피쳐 맵을 워핑 (이웃한 뷰포인트로부터..)
        - depth probability volume을 regress (deep 3D cnn 이용)
        - 그런 다음 뎁스 맵은 per-plane depth value를 선형적으로 결합한다. (확률 기반 가중치 적용)
        - 최종적으로 포인트 클라우드 얻을 때, 3차원 공간으로 프로젝션 시켜서 얻었다. (각 뷰에 대해)
- Point confidence 취득 방법
    - depth probability volume을 tri-linear sampling하여 얻었다고 설명
- 수식을 보면 여러개의 additional neighboring views를 넣는데, 논문에서는 주로 2개의 추가 뷰를 넣었다고 한다.

**Point features**

- 각 이미지로부터 2D CNN을 통해 뉴럴 피쳐를 뽑는다.
- 이 과정에서 VGG 네트워크 구조를 사용
    - 3개의 다운샘플링 레이어를 갖는 구조
    - 중간 단계에서 나오는 피쳐들도 조합해서 f를 구성
    - 그림 2a를 보면 3개 층 피라미드 피쳐로 보인다.
    - 이를 통해 나오는 피쳐 벡터는 8+16+32 = 56차원
    - 거기에 추가적으로 view-dependent effect를 위해 각 입력 viewpoint에 viewing direction 정보도 넣었다고
    - 그래서 3개 차원이 더 추가된 59차원 (이건 아까 3 view 썼다는 것과 연관이 있는 것인가)
    - 추가자료 봐야함!

**End-to-end reconstruction**

- 이제 다양한 시점에서 얻은 포인트 클라우드 들을 조합해서 최종 형태의 뉴럴 포인트 클라우드를 구성한다.
- Point generation network는 representation network와 함께 학습한다 (이게 엔드 투 엔드로?!)
    - 학습도 결국 렌더링 로스 잘 추정하는 형태로
    - 일단 본질적인 너프 목적만 충실하는건가…
    - 근데 이렇게 간 덕에 per-scene fitting time을 줄였다고 주장한다.

### 2. Optimizing point-based radiance fields

- differentiable ray marching을 통해? 래디언스 필드 최적화 성능을 향상시킨다라…?
- 여기서는 뉴럴 포인트 클라우드를 최적화시키면서… (포인트 피쳐와 컨피던스에 대해서…)
- 그리고 특정 씬에 대한 MLP representation도?
- 그림 3번에서의 피치색 화살표와 하늘색 화살표 말하는 듯
- 보통 COLMAP 등으로 만든 포인트 클라우드는 hole이나 outlier가 존재하기도 하는데, 이로 인해 렌더링 퀄리티가 떨어질 수 있다.
- per-scene optimization 과정에서…
    - 이런 문제를 풀기 위해 point들의 위치를 직접적으로 최적화하는 방법으로 접근?
    - …을 했다가는 트레이닝 안정성을 떨어트리고, 큰 홀은 결국 메꾸지 못한다고 한다.
    - 그래서 point pruning & growing을 사용

**Point pruning**

- point confidence 이야기 다시 하는 중.. 이 pruning 과정에서 활용하고 있다!
- 목적은 outlier rejection
- 매 10,000회 이터레이션마다 컨피던스 값이 0.1 이하인 포인트들은 쳐낸다.
- 추가로 point confidence에 sparsity loss가 추가되어있음
    - 목적: confidence value를 좀 더 0과 1에 가깝게 (명확성을 높이는 목적으로 보인다)

**Point growing**

- hole filling 목적 (이것도 10000회 iter마다 한다고 하네)
- 점진적으로 point를 growing → 포인트 클라우드 주변에서
- 새로운 포인트 성장시킬 shading location 찾자
- 식 11번
- 알파값이 가장 큰 j를 찾아야 함
- 식에 안보이는 입실론을 계산한다고 하네? x_j_g의 거리 계산 용도 (뉴럴 포인트로부터 가장 가까운…)
- ray marching 과정에서
    - 새로운 뉴럴 포인트를 만들텐데
    - 알파 값이 임계치를 넘고 입실론 값도 임계치를 넘으면!
    - 즉, 표면 근처에는 있으나 이미 존재하는 뉴럴 포인트와 거리가 먼 포인트일 경우가 유효하다는 뜻
- 이러한 과정들을 반복하면서 홀 필링을 해내고 있다.

## Implementation details

### 1. Network details

- Positional encoding: relative position ,features, viewing direction에 적용
- Cost volume-base CNN (G_p, G_r)
    - MVSNet 구조 차용
    - 1/4 다운샘플링 레이어들이 있고, 최종적으로 32채널? 이게 코스트 볼륨
    - 코스트 볼륨 정규화 과정이 U-Net 구조라고 한다.
    - 완전 마지막으로는 1채널 확률맵
- Image feature extraction 2D CNN (G_f)
- Point-based radiance fields MLP
    - c1 = 56, c2 = 128
    - F, R, T는 2, 3, 2개의 레이어로 각각 이루어짐

### Training & optimization details

- MVS-Net 기반의 depth 생성 네트워크를 pretrain
    - GT depth를 이용해서 진행한다.
    - 깃헙을 보면 체크포인트를 다운받는다.
- feature extraction & point representation
    - 이것도 pretrained checkpoint를 다운받는다.
- 3개의 input view 기준, 포인트 클라우드 생성에 0.2초밖에 안걸린다고?
- per-scene optimization 관점에서는 sparsity loss가 추가된다고 보면 됨
    - 단, 이 loss의 가중치는 0.002 (실험을 통해 찾음)

### Initializing neural points from COLMAP

- COLMAP으로부터 나온 리컨스트럭션 결과도 활용이 가능하다.
- 포인트가 N개인 리컨 결과가 있다고 해보자
    - 감마를 초기에 0.3으로 통일했네?
    - 최적화 과정에서 0과 1로 가까워지는 형태로 보임
    - 각 포인트에 대한 피쳐 벡터는?
    - 포인트가 다른 포인트로 인해 가려지는 view가 있으면 우선 배제한다. (occlusion)
    - ?
    - 그 다음 포인트들을 reproject시켜서 피쳐맵 상으로 투영한다. (여기서의 피쳐 맵은 이미 뽑혀있다)
    - 이렇게 피쳐맵과 포인트 위치를 상응시킨다.

## Limitations

- 렌더링 스피드나 최적화쪽에는 초점을 맞추지 못했다고 함
- 그럼에도 오리지널 너프에 비해서는 3배 빠름
    - empty space skipping이 들어간 덕이라고 주장
- 다음 연구와 결합되면 좋은 시너지가 있을 것으로 기대한다
    - KiloNeRF, Plenoctree
    - 렌더링 속도 개선 논문