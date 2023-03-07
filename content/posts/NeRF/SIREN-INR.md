---
title: "SIREN INR"
date: 2023-03-07T19:09:20+09:00
draft: true
---

Title: ****Implicit Neural Representations with Periodic Activation Functions****

- INR 쪽에서 거의 처음으로 나온 논문같은 느낌이랄까..

## 간단한 느낌

- Activation function이 바뀐 느낌인데
- MLP에서 ReLU 활성화 함수를 sin 함수 꼴로 바꾼 느낌
- 요약
    - 이 논문에서는 image, audio signal을 fit하고, 간단한 편미분 함수를 풀 때, 제안한 SIREN 구조를 사용하는 방법 설명
    - Room scale 3D scene을 단일 INR을 통해 파라미터화한다 (그 과정에서 사인파 활성화 함수를 사용)
    - 주기적인 비선형성을 활용해 고주파 데이터에 대한 디테일을 fit해주는 역할을 한다.
        - NeRF는 이런걸 positional encoding을 통해 풀었다면
        - SIREN은 periodic nonlinearity로 풀었다고 함
    - 하이퍼네트워크를 통해 image의 INR을 일반화했다..?
    - 주기함수인 sin파 함수를 활성화 함수로써 활용했다. → 어려운 고차 미분이 필요한 함수..? 의 파라미터화를 가능하게 했다.
    

## 10분 영상 리뷰

[Implicit Neural Representations with Periodic Activation Functions](https://www.youtube.com/watch?v=Q2fLWGBeaiI)

- 아래 이미지를 보니 왜 audio signal이 discrete한지 알겠다

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2ca8aa45-3da2-4d30-9ae3-e6c0141008a3/Untitled.png)

- ReLU MLP로 이미 이전에 continuous representation을 하는 INR 시도가 있었나보다
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ed0008db-85df-49e0-95c9-b596d0ea4538/Untitled.png)
    
    - 문제점: large scene에서 취약한 모습을 보인다고 함
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5478fa11-4749-4d94-b841-b154977e5576/Untitled.png)
    
    - 고주파 디테일에 취약한게 ReLU라고 한다.
- 그래서 논문에서 제안한게 sin파 기반 MLP
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fc43d1a9-5c40-476d-a18b-da6f6f75201f/Untitled.png)
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7cddd2a2-484f-4766-a6f2-621d39de9d36/Untitled.png)
    
    - 위에 비교 그림을 보면 고차 미분에서 0으로 값이 소실되는 것을 잡아주는걸까..?
    - 뭔가 원하는 값으로 수렴을 빨리 하고, ReLU는 2차 미분에 취약하고 등등등…
        
        ![inr.gif](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c698a9ee-8859-4631-af34-9cc6372ec522/inr.gif)
        
    - Image
        - input: spatial coordinates
        - output: RGB value
    - Audio
        - input: time point
        - output: amplitude
    - Video
        - input: space-time coordinates
        - output: RGB value
    - Poisson equation
        - image gradient..?
        - input: spatial coordinates
        - output: gray level
    - Eikonal equation
        - 아마 3d model 같은데..
        - input: spatial coordinates
        - output: signed distance