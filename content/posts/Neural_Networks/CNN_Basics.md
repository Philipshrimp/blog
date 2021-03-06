---
title: "CNN의 기본적인 구조 및 원리"
date: 2022-07-28T20:25:21+09:00
author:
  name: Sunho Kim
menu:
  sidebar:
    name: CNN의 기본적인 구조 및 원리
    identifier: cnn-basics
    parent: neural-networks
    weight: 1
categories:
  - Neural Networks
tags:
  - Deep Learning
  - Neural Network
draft: true
---

처음 회사에서 인공지능 업무를 배정받기 이전에, 기본적인 CNN 구조에 대해 확실히 알고 가야 제대로 활용할 수 있겠다는 이야기를 듣고 CNN 구조부터 확실히 짚어가기 위해 기본적인 구조에 대한 학습을 시작했다.  
이 포스트는 사내에서 세미나를 진행했던 CNN 기초를 다시 복습할 겸 저장하는 블로그 글이다.  


### Perceptron  
{{< img src="https://scikit-learn.org/stable/_images/multilayerperceptron_network.png" width="400" align="center" >}}   
CNN이 본격적으로 활용되기 이전에 뉴럴 네트워크를 통해 image classification 등의 문제를 풀 때는 주로 perceptron 구조를 사용했다. 특정 차원의 벡터로 이루어진 입력이 들어오면 이 것을 여러 층의 hidden layer를 거쳐 출력 값을 내놓는 구조가 Multi layer perceptron이다. 여기서 이 다중 레이어의 퍼셉트론이라는 구조가 왜 나왔는지가 중요한데, 우리가 일반적으로 선형 함수로 되어 있는 특정 문제를 풀기 위해서는 단일 레이어내에 정의된 함수를 통과하면 원하는 값을 도출할 수 있을 것이다.   
   
그러나, 실제로 우리가 풀어야 할 무수히 많은 문제들은 선형 함수만으로는 풀 수 없는 비선형성을 갖는 것들이 매우 많다. 그렇다면 선형 문제를 풀기 위해 정의된 하나의 layer를 통해서는 이 비선형 문제를 풀기가 어렵다는 소리가 된다. 이를 풀기 위한 접근법이 다중 레이어를 통한 접근 방법이라 이해하면 좋을 것이다.   
   
{{< img src="https://cdn1.byjus.com/wp-content/uploads/2020/06/xor-equivalent-circuit.png" width="400" align="center" >}}   
좀 더 쉽게 설명하자면, 논리 게이트의 XOR Problem을 보면 좋다. 일반적인 논리 게이트를 AND, OR, NOT에 대한 게이트는 쉽게 볼 수 있는데, XOR 문제의 경우 단일 게이트로 이를 표현하기 어렵다는 문제가 있다. 따라서 XOR 문제를 논리 회로로 표현하기 위해서는 위 그림처럼 OR Gate와 NAND, AND gate들을 잘 조합해야 한다.   
   
{{< img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=http%3A%2F%2Fcfile22.uf.tistory.com%2Fimage%2F99612E4B5C0B73DD3417CA" width="700" align="center" >}} 
직관적으로 위 표와 그래프를 보면 알 수 있을 것이다.   
   
AND, OR 문제에 비해 XOR은 결과값을 선형 그래프로 표현하는 것이 매우 어렵다. 결국 이 문제는 비선형성을 표현하는 문제와도 직접적으로 연결되어 있기도 하다. 다중 레이어 퍼셉트론 구조는 이와 같은 고민에서 출발했다고 볼 수 있다.   
   
기본적인 퍼셉트론 구조의 뉴럴 네트워크는 크게 두 가지 문제를 폴기 위한 목적으로 사용될 수 있다. 하나는 분류(Classification) 문제고 다른 하나는 회귀(Regression) 문제이다. 분류 문제는 입력으로 넣은 데이터가 별도로 정의된 클래스들 중 어느 클래스에 해당하는지 식별하는 문제이며, 회귀 문제는 내가 찾은 어떠한 결과가 특정 정답에 해당하는 어떠한 평면과 얼마나 근접한 가를 찾는 문제로 볼 수 있다. 사실 이와 관련된 부분은 모델 정의 뿐만 아니라 각 문제별로 loss function을 어떻게 정의하느냐에 관한 문제와도 연관되어있다.   
   
### 기존 MLP 방법의 문제점  
MLP 구조의 뉴럴 네트워크의 경우, 일반적인 이미지를 입력으로 받아 연산을 수행하기에는 그 기능이 확장됨에 어려움이 있다.   
가장 큰 문제는 학습을 위한 weight의 갯수인데, 이는 입력 이미지의 해상도가 크면 클 수록 기하급수적으로 늘어나게 될 것이다.   
   
예를 들어, CIFAR-10에서 제공되는 이미지처럼 32x32 사이즈의 3채널 이미지가 입력으로 들어오는 경우를 생각해보자. 단일 색상에 대한 픽셀 하나하나가 입력으로 들어간다고 가정할 때, 이 입력이 fully connected layer로 들어가게 되면 하나의 레이어를 거칠 때마다 32 * 32 * 3 = 3072개의 weight를 요구하게 된다. 작은 사이즈의 이미지에서는 이게 문제로 보이지 않을 수 있는데, 만약 입력 해상도가 200x200가 있다고 생각해보자. 그러면 레이어 하나당 200 * 200 * 3 = 120,000개의 weight가 요구된다. 물론 CNN이 본격적으로 image classification 문제를 풀기 위한 솔루션으로 활용되기 시작한 시점에는 224x224 해상도의 영상을 많이 사용했지만, 영상이 HD급 이상으로 올라가게 된다면 한 레이어당 필요하게 되는 weight 갯수개 백만개는 훌쩍 넘어갈 것이고, 이게 또 멀티 레이어로 구성된다면 요구되는 weight 갯수는 감당이 힘들 정도로 늘어날 것이다. weight 수의 증가는 연산량 증가로도 이어질 수 있고, 과적합(overfitting) 문제로까지 이어질 수 있다. 이에 대해서는 다음 포스트에서 자세히 짚어보도록 하겠다.   
   
뿐만 아니라 MLP 구조는 보통 flatten된 1차원 벡터가 입력으로써 들어오게 된다. 이미지가 입력으로 들어왔을 때, 2차원 픽셀로 구성된 이미지를 1차원 벡터 형태로 펴주면 흔히 local 영역에 대한 특성을 잘 잡지 못한다는 문제도 존재하게 된다. 이 문제에 대해서는 CNN의 특징을 보면 왜 문제로 제기되었는지 알 수 있다.   


### Convolutional Neural Networks  
기본적인 CNN 구조는 각각의 히든 레이어가 여러개의 neuron이 1차원 벡터 형태로 쌓여 있는 구조가 아닌 3차원 volume으로 구성되어 있는 구조다. CNN에서는 입력 영상에 대해 각각의 레이어마다 주어진 weight 커널과의 합성곱(convolution)을 수행해 3차원 feature map volume을 출력해주는 구조로, 이미지에 대한 가로, 세로 크기는 줄여가며 각 영역에 대한 특징을 의미론적으로 압축해준 다음 최종적으로 1차원 벡터 형태의 출력을 도출하는 구조다.  


{{< img src="https://www.mdpi.com/applsci/applsci-09-04500/article_deploy/html/images/applsci-09-04500-g001.png" width="700" align="center" >}}   
위 그림에 나와있듯이, CNN은 크게 Feature extraction과 Classification 구조로 나뉘어 있다. Feature extraction에서는 상술한 합성곱 과정을 통해 입력 영상의 해상도는 압축되고, 각 픽셀 영역마다 semantic한 정보들을 여러 dimension에 걸쳐 담아주는 feature map을 만들어낸다. 하나의 레이어 내에는 기본적으로 convolution layer, Activation function, Pooling layer로 구성되어 있다.  


{{< img src="https://cdn-images-1.medium.com/max/1200/1*1okwhewf5KCtIPaFib4XaA.gif" width="500" align="center" >}}   
convolution layer에서는 입력으로 들어온 map과 해당 layer에 주어져 있는 kernel 간 합성곱 연산을 map 전체 영역에 대해 sliding window 형식으로 수행한다. 이 연산은 입력 map의 전체 channel에 대해 동일한 kernel을 사용하며, kernel의 차원은 우리가 출력으로 얻고자 하는 차원만큼 다르게 존재한다.  


CNN이 이와 같은 방식으로 연산을 하기 때문에 두 가지 큰 강점을 가지고 있다. 


앞 단에서 feature map을 만들어내면 해당 map이 classification을 위한 fully connected layer에 입력으로 들어가 최종적으로 하나의 vector를 만들어낸다. 이 vector는 최종적으로 풀고자 하는 classification에 대한 class 갯수만큼, 그 class 각각의 확률을 의미하는 값을 포함하고 있다.  


기존에 나온 Fully connected layer 기반의 네트워크 대비 CNN은 다음과 같은 차별점을 가지고 있다.

{{< alert type="secondary" >}}   
   
- 각 레이어의 입출력 데이터 형상을 유지할 수 있다.
   
- 이미지의 공간 정보를 유지하면서 인접 이미지와의 특징을 효과적으로 인식할 수 있다.
- 복수의 필터로 이미지의 특징을 추출하고, 이에 대한 학습을 진행할 수 있다.
- 추출한 이미지의 특징을 한데 모으고, 원하는 정보에 대해서 특화할 수 있는 pooling layer를 사용한다.
- 필터를 공유 파라미터로 사용하기 때문에 일반 인공 신경망과 비교하여 학습 파라미터가 매우 적다.
- CNN은 필터 사이즈를 조정하거나, 영상 사이즈를 점점 줄여가거나(pooling)하는 방법으로 특징 맵을 얻는다.
{{< /alert >}}
