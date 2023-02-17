---
title: "BARF : Bundle-Adjusting Neural Radiance Fields"
date: 2023-02-17T09:52:20+09:00
author:
  name: Sunho Kim
menu:
  sidebar:
    name: BARF 논문 리뷰
    identifier: barf-review
    parent: nerf
    weight: 1
categories:
  - NeRF
tags:
  - Deep Learning
  - Neural Network
  - Neural Rendering
  - Pose Estimation
  - NeRF
  - Bundle Adjustment
draft: false
---

해당 논문에 대한 리뷰 영상은 아래 동영상을 통해서도 확인하실 수 있습니다.
{{< youtube iqEfKA7seNk >}}
{{< vs 3>}}

### Intro
{{< vs 3>}}
{{< img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbJqC9b%2FbtrRTDSDQcg%2F90VMcJTdbt2oc8Tx0VypC1%2Fimg.png" width="500" align="center" >}}
{{< vs 3>}}

Original NeRF 논문은 처음 ECCV 2020에서 발표되었을 때, 뉴럴 렌더링 부분에 있어 굉장히 큰 화두가 되었다. 그와 동시에 많은 한계점들을 가지고 있어서 많은 연구자들로부터 그 문제점들이 지속적으로 지적되어 왔다. 이번에 리뷰할 BARF 논문에서는 그 중에서 Original NeRF가 입력 데이터로 정확한 Camera pose를 요구하고 있다는 점을 지적하고 있는데, 다시 말하면 입력 이미지 데이터와 한 쌍으로 있을 transformation 값이 부정확하거나 알 수 없을 경우에는 뉴럴 렌더링의 정확도가 떨어지게 된다는 것이다. BARF는 이러한 문제를 해결할 수 있는 방안을 제시하고 있다. 이 논문에서는 부정확한 camera pose 오차를 줄이고, 동시에 본래 NeRF가 수행해야 하는 3차원 representation 정확도도 확보할 수 있는 파이프라인을 제안하며, 언급한 두 문제를 동시에 풀고자 한다. 
{{< vs 3>}}

추가로 논문에서는 Positional encoding의 문제점도 지적하고 있다. 이전에 NeRF에서 입력 데이터를 고차원으로 encoding할 수 있게 해주는 positional encoding 방법은 representation 정확도 측면에서는 디테일을 잘 살려준다는 장점이 있으나, BARF에서처럼 representation과 pose registration을 동시에 수행하고자 할 때, pose registration 측면에서 정확도가 떨어지는 문제가 발생했다고 한다. BARF에서는 이러한 문제점 또한 완화시키기 위해 이러한 positional encoding을 coarse-to-fine 방식으로 접근하고 있다.
{{< vs 3>}}

기본적인 아이디어는 고전적인 Image alignment에서 출발해서 제안을 하고 있고, 이러한 방법으로 실험을 진행했을 때, BARF가 neural scene representation 문제와 large camera pose misalignment 문제를 동시에 잘 해결하고 있다고 주장한다. 결론적으로 이러한 점은 view synthesis와 localization 문제를 동시에 풀 수 있게 한다고 하고 있다.
{{< vs 3>}}

### Image alignment
통상적으로 Image alignment는 특정 source image를 target view로 warping하는 것을 말하며, 여기서 발생하는 reprojection error를 최소화하는 것이 목표다. 논문에서는 이 reprojection error를 다음과 같은 수식으로 표현하고 있다.

{{< alert type="secondary" >}}   
$$ \min_\textbf{p}\sum_\textbf{x}||I_1(W(\textbf{x};\textbf{p}))-I_2(\textbf{x})||^2_2$$
{{< /alert >}}
{{< vs 3>}}

2차원 이미지 상에 있는 픽셀의 위치를 \\(\textbf{x}\\)라고 할 때, source to target warping parameter \\(\textbf{p}\\)를 이용하여 warping을 한 결과를 target과 비교하는 공식으로, 이 오차가 적을 수록 warping이 성공적으로 되었다고 볼 수 있다. (참고로 여기서 언급하는 warping은 컴퓨터 비전에서의 affine transform, homography와 같은 문제를 생각하면 좋다.) 최적화 관점에서 이 파라미터는 점진적인 update를 하며 최적의 값을 찾아나가기 때문에 매 term마다 아래의 변화량을 얻게 되고, 이 변화량을 더하여 파라미터를 갱신하는 형태로 진행한다.

{{< alert type="secondary" >}}   
$$ \Delta\textbf{p} = -A(\textbf{x};\textbf{p})\sum_\textbf{x}\textbf{J}(\textbf{x};\textbf{p})^T(I_1(W(\textbf{x};\textbf{p}))-I_2(\textbf{x}))$$
{{< /alert >}}
{{< vs 3>}}

논문에서는 \\(A\\)를 Generic transformation matrix라고 언급하고 있다. 이 Generic transformation matrix는 최적화 기법에 따라서 들어가는 값이 다르다고 하는데, 보통 Gauss-Newton method를 생각하면 이 자리에 Hessian 값이 들어가서 \\(A(\textbf{x};\textbf{p})=(\sum_\textbf{x}\textbf{J}(\textbf{x};\textbf{p})^T\textbf{J}(\textbf{x};\textbf{p}))^{-1}\\)로 정의된다고 한다. 최근의 딥 러닝 기술에서는 이렇게 2차 미분 형태의 Hessian까지 접근하기 보다는 first-order optimizer 방법을 사용하는 것이 더 선호된다고 하는데(Gradient descent), 이 경우의 A 값은 스칼라 값으로 정의한다고 한다. 딥 러닝에서 쓰고 있는 learning rate가 이 자리에 들어간다고 논문에서는 이야기한다. Steepst descent image term을 의미하는 J의 경우, Image에 대한 1차 미분(=Jacobian)으로 이해할 수 있는데, 보통 이미지의 Jacobian이라면 \\(\textbf{J}(\textbf{x};\textbf{p})=\frac{\partial I(\textbf{x})}{\partial\textbf{x}}\\)이라고 할 수 있을 것이다. 그러나 warped image 관점에서 보면 각 픽셀 좌표 값들이 잘 warp되었는지 검증이 필요하고, 이를 위해서는 warping parameter가 최적의 값을 가져야만 한다. 이로 인해 여기서의 Jacobian은 아래와 같이 정의하고 있다.

{{< alert type="secondary" >}}   
$$ \textbf{J}(\textbf{x};\textbf{p}) = \frac{\partial I_1(W(\textbf{x};\textbf{p}))}{\partial W(\textbf{x};\textbf{p})} \frac{\partial W(\textbf{x};\textbf{p})}{\partial\textbf{p}}$$
{{< /alert >}}
{{< vs 3>}}

식을 보면 원래의 Jacobian 식처럼 \\(\textbf{x}\\)에 대해서 편미분하는 것이 아닌, 워핑된 좌표 결과에 대해 편미분하는 것을 볼 수 있다. 이어서 이 워핑된 결과 자체에 대해서도 파라미터에 대해 편미분을 하는 모습을 볼 수 있는데, 식 자체는 마치 chain rule 형태처럼 이루어진 것을 볼 수 있다. 이런 식으로 식을 구성한 이유는 warping parameter를 딥 러닝 관점에서 바라볼 때, 학습의 대상이 되기 때문에 역전파 과정에서 이를 학습하기 위한 형태로 표현하고자 이렇게 접근한 것이 아닐까 싶었다. 참고로 \\(\frac{\partial W(\textbf{x};\textbf{p})}{\partial\textbf{p}}\\)는 사전에 정의된 warping에 대한 pixel의 변위를 의미한다고 한다.
{{< vs 3>}}


### Image alignment as neural networks
지금까지 언급한 설명이 통상적인 Image alignment에 대한 내용이라면, 이번에는 이 개념을 뉴럴 네트워크의 측면에서 바라보는 설명을 하고 있다. 앞서서 Image alignment에서는 reprojection error를 최소화 할 최적의 warping parameter p를 찾아야 한다고 했는데, 이제 이걸 뉴럴 네트워크의 메커니즘처럼 학습을 통해 최적의 값으로 수렴해나가는 방식으로 접근을 한다. 나아가서 warped image를 만들어내는 과정 전체를 뉴럴 네트워크를 통해서 진행한다고 가정하면, warping parameter도 중요하지만, 실제로 만들어지는 이미지의 representation 또한 target과 유사도가 높아야 한다. 그렇기 때문에 여기서는 image alignment가 만들어내는 이미지 자체의 유사도와 alignment 내에서 진행되는 warping의 정확도를 동시에 확보해야 한다. 이에 대한 목적 함수는 다음과 같이 나타낼 수 있다.

{{< alert type="secondary" >}}   
$$ \min_{\textbf{p},\Theta}\sum_\textbf{x}(||f(\textbf{x};\Theta)-I_1(\textbf{x})||^2_2 + ||f(W(\textbf{x};\textbf{p});\Theta)-I_2(\textbf{x})||^2_2) $$
{{< /alert >}}
{{< vs 3>}}

여기서의 함수 f가 뉴럴 네트워크를 의미한다. 이 문제를 alignment를 수행해야하는 이미지가 두 개고, 그 두 이미지 각각에 대한 ground truth가 있는 환경에서의 image alignment로써 다음과 같이 정의할 수도 있다.

{{< alert type="secondary" >}}   
$$ \min_{p_1,p_2,\Theta}\sum^2_{i=1}\sum_\textbf{x}||f(W(\textbf{x};\textbf{p}_i);\Theta)-I_i(\textbf{x})||^2_2 $$
{{< /alert >}}
{{< vs 3>}}

이 식을 더 확장해서 보면, 이미지 M개의 alignment 문제에 대해서도 이 식으로 풀 수 있다는 뜻이 되고, 이에 대한 warping parameter도 함께 M개로 늘어날 것임을 알 수 있다.
{{< vs 3>}}


### Neural Radiance Fields
이제 앞서 설명한 Image alignment 컨셉을 3차원으로 확장하여 NeRF에 해당 이론을 적용하여보자. 2차원에서 3차원으로 확장된 만큼, 앞서 언급한 warping parameter는 3D rigid transformation을 위한 파라미터로 간주될 수 있을 것이다. 그리고 앞서 뉴럴 네트워크라고 언급한 함수 f는 NeRF가 될 것이다. NeRF는 MLP layer를 통해 3차원 입력 좌표와 viewing direction이 들어가면 이에 상응하는 color와 density 값을 출력하는데, 이를 수식으로 표현하면 \\(y=[\textbf{c};\sigma]=f(\textbf{x};\Theta)\\)와 같이 볼 수 있다(논문에서는 간소화를 위해 viewing direction이 생략되어 있어 이후 언급되는 식에서는 viewing direction이 생략되어 표기됩니다. 물론 viewing direction은 계속 고려되고 있습니다!). 여기서 NeRF 과정을 y=f(x)꼴로 표기를 했는데, 여기서 확장해서 논문에서는 volumetric rendering 문제를 또 합성함수의 형태로 정리하고 있다. NeRF에서 사용하던 Volumetric rendering 공식을 pixel homogeneous coordinates 관점에서 보면 다음과 같이 표기할 수 있다.

{{< alert type="secondary" >}}   
$$ \hat{I}(\textbf{u}) = \int_{z_{near}}^{z_{far}}T(\textbf{u},z)\sigma(z\bar{\textbf{u}})\textbf{c}(z\bar{\textbf{u}})dz$$
{{< /alert >}}
{{< vs 3>}}


이 식은 픽셀 좌표계에 있는 2차원 좌표 \\(\textbf{u}\\)와 이를 homogeneous coordinates로 변환한 \\(\bar{\textbf{u}}\\), 그리고 depth 값으로 간주할 수 있는 ray에서의 step z에 대한 표현으로 나타내고 있다. 앞서 y=f(x)를 통해 얻은 3차원 포인트의 representation이 y에 담겨있다고 할 때, 포인트 y가 N개가 있다라고 하면 volumetric rendering은 다음과 같은 함수 형태로 나타낼 수 있다.

{{< alert type="secondary" >}}   
$$ \hat{I}(\textbf{u}) = g(\textbf{y}_1, \textbf{y}_2, ..., \textbf{y}_N)$$
{{< /alert >}}
{{< vs 3>}}

여기서 카메라 좌표계상에 있는 픽셀 좌표를 depth step에 따라 3차원 월드 좌표계로 transform하는 함수가 \\(W(z\bar{\textbf{u}};\textbf{p})\\)라고 할 때, \\(y=f(W(z\bar{\textbf{u}};\textbf{p});\Theta)\\)라고 할 수 있으며, 이에 따라서 위의 렌더링 공식은 다음과 같은 합성 함수 형태로 정리될 수 있다.

{{< alert type="secondary" >}}   
$$ \hat{I}(\textbf{u};\textbf{p}) = g(f(W(z_1\bar{\textbf{u}};\textbf{p});\Theta), f(W(z_2\bar{\textbf{u}};\textbf{p});\Theta), ..., f(W(z_N\bar{\textbf{u}};\textbf{p});\Theta))$$
{{< /alert >}}
{{< vs 3>}}

이 식을 통해 transformed image가 2차원 pixel 좌표와 transformation parameter에 대하여 정리됨을 볼 수가 있다. 앞서 Image alignment 말미에 정리한 공식으로 돌아가자. 해당 식은 여러 개의 warping parameter와 representation parameter를 최적화하여 목적 함수를 최소화하는 쪽으로 학습이 되어야 한다고 했다. 동일 공식을 NeRF의 Volumetric rendering에 적용하면 다음과 같이 표기할 수도 있다. 이미지 입력 M개가 있다고 할 때, NeRF는 아래의 식을 최소화해야 한다.

{{< alert type="secondary" >}}   
$$ \min_{p_1,...,p_M,\Theta}\sum^M_{i=1}\sum_\textbf{u}||\hat{I}(\textbf{u};\textbf{p}_i,\Theta)-I_i(\textbf{u})||^2_2 $$
{{< /alert >}}
{{< vs 3>}}

이 식을 통해 NeRF의 volumetric rendering에 대해 각 이미지 별 M개의 pose와 representation을 동시에 최적화하도록 유도했다. 그런데 앞서 Image alignment를 설명할 때, warping parameter update를 위한 Jacobian을 정리한 부분이 있었다. parameter의 delta 값을 얻기 위해 1차 미분으로 접근하였기 때문인데, NeRF에서 또한 transformation parameter update를 위해서는 gradient descent 기반의 접근이 필요하고, 이를 위해서는 steepest descent image term이 요구되고 있다. N개의 위치 값에 대한 Jacobian 식은 다음과 같이 정의할 수 있다.

{{< alert type="secondary" >}}   
$$ \textbf{J}(\textbf{u};\textbf{p}) = \frac{\partial g(\textbf{y}_1, \textbf{y}_2, ..., \textbf{y}_N)}{\partial \textbf{y}_i} \frac{\partial \textbf{y}_i(\textbf{p})}{\partial \textbf{x}_i(\textbf{p})} \frac{\partial W(z_i\bar{\textbf{u}};\textbf{p})}{\partial\textbf{p}}$$
{{< /alert >}}
{{< vs 3>}}

이 식은 렌더링 함수인 \\(g(\textbf{y}_1, \textbf{y}_2, ..., \textbf{y}_N)\\)를 편미분하고 있는 것을 알 수가 있는데, 렌더링 함수를 representation 결과인 y에 대해서 미분하고, 거기에 neural network의 Jacobian을 곱한 뒤 transformation 함수에 대한 pixel 변위를 곱하고 있다. 이 Jacobian 역시 chain rule 형태로 정의되어 있는 것을 볼 수가 있다. 이를 통해 네트워크에서 효율적인 pose update 파이프라인을 구성할 수 있었다.
{{< vs 3>}}


### On Positional Encoding and Registration
다음으로는 지금까지 정의된 BARF 파이프라인에 Original NeRF에서 사용하고 있는 Positional encoding을 적용하고 있다. 보통 (k, k+1)번째 차원에 대한 positional encoding 결과는 다음과 같은 함수로 표현할 수 있다.

{{< alert type="secondary" >}}   
$$ \gamma_k(\textbf{x}) = [\cos(2^k\pi\textbf{x}), \sin(2^k\pi\textbf{x})] $$
{{< /alert >}}
{{< vs 3>}}

이 식을 x에 대해 미분을 하면

{{< alert type="secondary" >}}   
$$ \frac{\gamma_k(\textbf{x})}{\textbf{x}} = 2^k\pi[-\sin(2^k\pi\textbf{x}), \cos(2^k\pi\textbf{x})] $$
{{< /alert >}}
{{< vs 3>}}

로 정리될 수 있는데, 이걸 그래프 형태로 보면 동일 frequency에서 signal 증폭이 일어난 꼴로 표현된다. 근데 여기서 이 signal 증폭이 문제가 되고 있다고 한다. 신호가 동일 주파수에서 증폭될 경우 signal 자체도 일관성이 점차 떨어지게 되고, ray별 증폭이 일어난다고 봤을 때, 이러한 증폭으로 인해 서로 다른 ray에서 같은 값을 갖는 positional encoding 결과가 발생할 수 있는데, 이로 인해 signal 간의 상쇄 현상도 발생한다. 이 문제가 이전에 representation detail 살리는 측면에서는 문제가 되지 않았으나, 정확한 pose 추정의 측면에서는 효율적인 update를 방해하는 문제가 있었다고 한다. 그래서 저자는 pose registration과 representation을 동시에 고려하는 BARF에서 positional encoding은 마치 양날의 검과 같은 존재라고 언급하고 있다.
{{< vs 3>}}


### Bundle-Adjusting Neural Radiance Fields
논문에서는 이러한 문제를 매우 간단한 접근으로 해결하고 있다. 마치 low-pass filter 컨셉과 같은 smoothing mask를 적용해서 처음에는 저차원의 encoding 결과에만 집중하고, 후반부로 갈 수록 고차원의 결과까지 모두 집중하는 접근법을 적용하였다. 이를 위해 원래의 positional encoding 함수인 \\(\gamma_k(\textbf{x}) = [\cos(2^k\pi\textbf{x}), \sin(2^k\pi\textbf{x})]\\)에다가 하나의 weight를 추가하였다.

{{< alert type="secondary" >}}   
$$ w_k(\alpha) = \begin{cases} 
0, & \alpha < k \\\
\frac{1-\cos((\alpha-k)\pi)}{2}, & 0 \le \alpha-k < 1 \\\
1, & \alpha-k \ge 1
\end{cases}$$
{{< /alert >}}
{{< vs 3>}}

여기서 정의한 가중치를 점진적으로 증가시키면서 positional encoding이 초점을 맞추는 차원을 원본 이미지 차원에서 출발하여 Full positional encoding을 위해 설정한 L에 비례한 차원까지 확대시키는 방식인데, 이를 위해서 \\(\alpha\\) 값을 iteration step에 비례하게 점진적으로 증가시키면서 weight를 부여하고 있다. 이 방법으로 인해 BARF는 초기에는 smooth signal에 대해서 보고, 뒤로 갈 수록 높은 정확도의 표현력을 위한 쪽으로 학습되도록 하게 초점을 맞추어 준다. 즉, 초기에는 coarse하게 pose registration을 하고 나중에는 fine하게 scene representation을 하는 방법이다.
{{< vs 3>}}


### Experimental results
(Coming soon...)
{{< vs 3>}}


### Conclusion
BARF는 Image alignment 이론에서 출발한 개념을 잘 녹여내어 representation과 pose registration을 동시에 최적화하는 방법을 제안했다. Bundle-adjusting이라는 이름이 들어간 이유는 최초의 컨셉인 alignment에서의 비선형 최적화 식이 bundle adjustment에서의 최적화 공식과 유사한 형태를 지니기 때문이지 않을까 싶었다. 추가로 제안했던 Coarse-to-fine positional encoding 방법은 결과적으로 registration과 reconstruction을 동시에 수행하기에는 매우 효율적이었다. 한계점으로는 기존 NeRF와 동일한 slow optimization & rendering, rigidity assumption 등을 지적하고 있는데, 아무래도 그런 문제들보다 pose registration에 집중해서 개선 방안을 제안했기 때문에 해당 문제들에 대해서는 당연하겠지만 남아있을 수 밖에 없는 것으로 보인다. 논문에서도 결국 기존에 남아있던 문제를 추가로 해결하기 위해서는 해당 문제들을 개선한 방법들과의 결합을 이야기하고 있는데, 그런 측면에서 BARF는 타 알고리즘에 녹여내기 쉽다라는 이점이 있다고 주장하고 있다.