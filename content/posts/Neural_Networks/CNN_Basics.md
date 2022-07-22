---
title: "CNN의 기본적인 구조 및 원리"
date: 2022-06-28T20:25:21+09:00
author:
  name: Sunho Kim
menu:
  sidebar:
    name: CNN의 기본적인 구조 및 원리
    identifier: cnn-basics
    parent: neural-networks
    weight: 10
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
![MLP](https://scikit-learn.org/stable/_images/multilayerperceptron_network.png)  
<!-- <img src="https://scikit-learn.org/stable/_images/multilayerperceptron_network.png" height="300px" title="Multi layer perceptron" alt="MLP"></img><br/> -->
CNN이 본격적으로 활용되기 이전에 뉴럴 네트워크를 통해 image classification 등의 문제를 풀 때는 주로 perceptron 구조를 사용했다. 특정 차원의 벡터로 이루어진 입력이 들어오면 이 것을 여러 층의 hidden layer를 거쳐 출력 값을 내놓는 구조가 Multi layer perceptron이다. 여기서 이 다중 레이어의 퍼셉트론이라는 구조가 왜 나왔는지가 중요한데, 우리가 일반적으로 선형 함수로 되어 있는 특정 문제를 풀기 위해서는 단일 레이어내에 정의된 함수를 통과하면 원하는 값을 도출할 수 있을 것이다.

그러나, 실제로 우리가 풀어야 할 무수히 많은 문제들은 선형 함수만으로는 풀 수 없는 비선형성을 갖는 것들이 매우 많다. 그렇다면 선형 문제를 풀기 위해 정의된 하나의 layer를 통해서는 이 비선형 문제를 풀기가 어렵다는 소리가 된다. 이를 풀기 위한 접근법이 다중 레이어를 통한 접근 방법이라 이해하면 좋을 것이다.  

![XOR_Gate](https://cdn1.byjus.com/wp-content/uploads/2020/06/xor-equivalent-circuit.png)  
좀 더 쉽게 설명하자면, 논리 게이트의 XOR Problem을 보면 좋다.  

일반적인 논리 게이트를 AND, OR, NOT에 대한 게이트는 쉽게 볼 수 있는데, XOR 문제의 경우 단일 게이트로 이를 표현하기 어렵다는 문제가 있다. 따라서 XOR 문제를 논리 회로로 표현하기 위해서는 위 그림처럼 OR Gate와 NAND, AND gate들을 잘 조합해야 한다.

![XOR_Graph](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=http%3A%2F%2Fcfile22.uf.tistory.com%2Fimage%2F99612E4B5C0B73DD3417CA)  
직관적으로 위 표와 그래프를 보면 알 수 있을 것이다.  
AND, OR 문제에 비해 XOR은 결과값을 선형 그래프로 표현하는 것이 매우 어렵다. 결국 이 문제는 비선형성을 표현하는 문제와도 직접적으로 연결되어 있기도 하다. 다중 레이어 퍼셉트론 구조는 이와 같은 고민에서 출발했다고 볼 수 있다.  


기본적인 퍼셉트론 구조의 뉴럴 네트워크는 크게 두 가지 문제를 폴기 위한 목적으로 사용될 수 있다. 하나는 분류(Classification) 문제고 다른 하나는 회귀(Regression) 문제이다.  
분류 문제는 입력으로 넣은 데이터가 별도로 정의된 클래스들 중 어느 클래스에 해당하는지 식별하는 문제이며, 회귀 문제는 내가 찾은 어떠한 결과가 특정 정답에 해당하는 어떠한 평면과 얼마나 근접한 가를 찾는 문제로 볼 수 있다. 사실 이와 관련된 부분은 모델 정의 뿐만 아니라 각 문제별로 loss function을 어떻게 정의하느냐에 관한 문제와도 연관되어있다. (loss 이야기는 차후에)


### Convolutional Neural Networks  
이름 그대로, 


기존에 나온 Fully connected layer 기반의 네트워크 대비 CNN은 다음과 같은 차별점을 가지고 있다.

- 각 레이어의 입출력 데이터 형상을 유지할 수 있다.
- 이미지의 공간 정보를 유지하면서 인접 이미지와의 특징을 효과적으로 인식할 수 있다.
- 복수의 필터로 이미지의 특징을 추출하고, 이에 대한 학습을 진행할 수 있다.
- 추출한 이미지의 특징을 한데 모으고, 원하는 정보에 대해서 특화할 수 있는 pooling layer를 사용한다.
- 필터를 공유 파라미터로 사용하기 때문에 일반 인공 신경망과 비교하여 학습 파라미터가 매우 적다.
- CNN은 필터 사이즈를 조정하거나, 영상 사이즈를 점점 줄여가거나(pooling)하는 방법으로 특징 맵을 얻는다.


### Loss functions

