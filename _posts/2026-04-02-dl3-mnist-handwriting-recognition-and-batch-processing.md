---
title: "딥러닝 맛보기 (3) - 실전! MNIST 손글씨 인식과 배치(Batch) 처리"
author: 1st-world
date: 2026-04-02 22:05:00 +0900
last_modified_at: 2026-04-18 21:20:00 +0900
categories: [Artificial Intelligence, Machine Learning]
tags: [mnist, flatten, normalization, batch, forward-propagation]
pin: false
math: true
---

지난 [2부 - 활성화 함수와 3층 신경망 구현](/posts/dl2-activation-function-and-3-layer-neural-network/)에서는 NumPy의 행렬 곱을 이용해 가상의 데이터를 흘려보내는 3층 신경망의 순전파(Forward Propagation)를 직접 구현해 보았습니다.

하지만 가상의 장난감 데이터만 가지고 노는 건 재미도 없고 별로 와닿지 않겠죠? 이제 우리가 만든 신경망에 진짜 데이터를 넣어서 스스로 판단할 수 있도록 만들어 볼 시간입니다.

이번 글에서는 머신러닝계의 "Hello World!"라 불리는 **MNIST 손글씨 이미지 데이터셋**을 활용하여 숫자를 분류하는 파이프라인을 간단히 만들어 보고, 대량의 데이터를 빠르게 처리하기 위한 핵심 기법인 **배치(Batch) 처리**의 원리와 구현 방법에 대해서도 알아봅니다.

---

## 실습 준비 단계: 환경 및 파일 구성

코드를 직접 타이핑하며 실습해 보려면 몇 가지 준비가 필요합니다. 이 예제 코드는 딥러닝 입문서의 고전이라 할 수 있는 '밑바닥부터 시작하는 딥러닝'의 도우미 소스코드와 파일 구조를 기준으로 작성했습니다.

### 필수 라이브러리 설치

컴퓨터에 Python뿐만 아니라, 수치 계산을 위한 필수 라이브러리인 `numpy`도 함께 설치되어 있어야 합니다. 만약 설치한 적이 없다면 터미널을 열고 아래 명령어를 입력해 주세요.

```terminal
pip install numpy
```

### 필요한 파일 다운로드

신경망에 데이터셋을 불러오고 사전 학습된 가중치를 가져오기 위해 다음 파일들이 필요합니다.

* 원본 소스코드 저장소: [WegraLee/deep-learning-from-scratch (GitHub)](https://github.com/WegraLee/deep-learning-from-scratch)
    * MNIST 데이터 로더: [`dataset/mnist.py`](https://github.com/WegraLee/deep-learning-from-scratch/blob/master/dataset/mnist.py)
    * 공통 함수 (시그모이드, 소프트맥스): [`common/functions.py`](https://github.com/WegraLee/deep-learning-from-scratch/blob/master/common/functions.py)
    * 사전 학습 가중치 파일: [`ch03/sample_weight.pkl`](https://github.com/WegraLee/deep-learning-from-scratch/blob/master/ch03/sample_weight.pkl)

### 디렉터리 세팅

실행하려는 Python 스크립트 파일(`mnist_test.py`)이 위치한 폴더 구조를 다음과 같이 만들어 줍니다. 파일들의 경로가 어긋나면 ModuleNotFoundError 및 FileNotFoundError가 발생할 수 있으니 정확히 확인해 주세요!

```
📂 이번에_작업할_폴더/
├── 📂 dataset/
│   └── 📄 mnist.py         # MNIST 데이터를 받아오고 평탄화하는 코드
├── 📂 common/
│   └── 📄 functions.py     # 시그모이드, 소프트맥스 함수가 담긴 코드
├── 📄 sample_weight.pkl    # 사전 학습 가중치 파일
└── 📄 mnist_test.py        # 우리가 작성할 Python 스크립트 파일
```

---

## MNIST 데이터셋을 활용한 실습

MNIST(Modified National Institute of Standards and Technology) 데이터셋은 손으로 쓴 숫자(0부터 9까지) 이미지로 구성되었습니다. 딥러닝과 이미지 처리 분야를 처음 공부할 때 꼭 거쳐 가는 관문 같은 존재죠.

![Desktop View](/assets/img/posts/2026-04-02-mnist-handwriting-recognition-and-batch-processing_1.png)

MNIST 데이터셋 구성 정보는 다음과 같습니다.

* 이미지 크기: $28 \times 28$ 픽셀 (총 784개의 픽셀로 이루어진 회색조 단채널 이미지)
* 데이터 구성: 학습용 데이터 60,000장, 테스트용 데이터 10,000장
* 정답(Label): 이미지 속 실제 숫자(0부터 9까지)를 가리키는 정수

> **검증(Validation) 데이터는 어디에 있나요?**
>
> MNIST 공식 데이터셋은 크게 학습(Train)과 테스트(Test)용 데이터로만 나뉘어 있습니다. 만약 모델의 하이퍼파라미터를 튜닝하기 위한 검증 데이터가 필요하다면, 일반적으로 학습용 데이터 60,000장 중 일부를 연구자가 직접 분리해서 사용하게 됩니다.
{: .prompt-tip }

### MNIST 데이터 전처리

이미지 데이터는 일반적으로 2차원 배열(높이 × 너비)로 표현하지만, 단순한 다층 신경망(DNN)에 입력하려면 이 2차원 데이터를 일렬로 길게 늘어뜨린 **1차원 데이터**로 변환해야 합니다. 이러한 작업을 **평탄화(Flatten)**라고 합니다.

또한, 픽셀 값은 원래 0부터 255 사이의 정수로 표현(값이 클수록 더 밝음을 의미)하는데, 이 값을 0.0과 1.0 사이의 실수 범위를 갖도록 변환하는 **정규화(Normalization)** 작업도 함께 진행합니다.

입력 데이터의 스케일을 일정하게 맞춰 정규화를 해주면 오차 표면(Loss Landscape)이 비교적 균일해져 경사 하강법이 더 안정적으로 동작하고 모델의 학습 수렴 속도도 개선됩니다. 물론 우리는 아직 모델을 학습시키는 것은 아니지만, 이번 예제에서 사용할 사전 학습 모델 역시 정규화된 입력을 기준으로 학습되었으므로 추론 시에도 동일한 방식으로 데이터를 정규화해야 올바른 결과를 얻을 수 있습니다.

아래는 MNIST 데이터를 읽어오는 전형적인 코드 구조입니다.

```python
from dataset.mnist import load_mnist

# 처음 실행할 때는 MNIST 원본 파일을 다운로드하는 시간이 소요됩니다.
(x_train, t_train), (x_test, t_test) = load_mnist(
    normalize=True,      # 픽셀 값을 0.0~1.0으로 정규화
    flatten=True,        # 1차원 배열(784 크기)로 평탄화
    one_hot_label=False  # 정답을 원-핫 인코딩이 아닌 원래 숫자로 유지
)
```

### 이미 학습된 가중치 적용

우리는 아직 가중치($W$)와 편향($b$)을 데이터로부터 스스로 배우게 하는 과정을 다루지 않았습니다. 따라서 이번 실습에서는 미리 충분히 학습된 가중치 매개변수가 저장되어 있는 `sample_weight.pkl` 파일을 사용하여 추론(Inference) 단계만 먼저 경험해 볼 예정입니다.

여기서 `.pkl` 확장자는 Python에서 객체를 그대로 파일로 저장하고 불러올 수 있게 해주는 `pickle` 라이브러리의 파일 형식입니다. 이 파일 안에는 3층 신경망에 필요한 가중치 행렬 `W1`, `W2`, `W3`와 편향 벡터 `b1`, `b2`, `b3`가 딕셔너리 형태로 패킹되어 있습니다.

### 단일 이미지 추론 파이프라인 구현

그럼 먼저 가중치를 불러와 MNIST 이미지 10,000장을 **한 장씩 차례대로** 신경망에 통과시키면서 정확도를 계산하는 파이프라인을 작성해 봅시다.

```python
import numpy as np
import pickle
from dataset.mnist import load_mnist
from common.functions import sigmoid, softmax

# 테스트 데이터 로드 함수
def get_data():
    # 우리는 모델을 '평가'만 할 것이므로 테스트 데이터(x_test, t_test)만 불러오기
    (_, _), (x_test, t_test) = load_mnist(normalize=True, flatten=True, one_hot_label=False)
    return x_test, t_test

# 사전 학습된 가중치 로드 함수
def init_network():
    with open("sample_weight.pkl", "rb") as f:
        network = pickle.load(f)
    return network

# 3층 신경망 추론(Forward) 함수
def predict(network, x):
    W1, W2, W3 = network['W1'], network['W2'], network['W3']
    b1, b2, b3 = network['b1'], network['b2'], network['b3']

    # Layer 1
    a1 = np.dot(x, W1) + b1
    z1 = sigmoid(a1)

    # Layer 2
    a2 = np.dot(z1, W2) + b2
    z2 = sigmoid(a2)

    # Layer 3 (Output)
    a3 = np.dot(z2, W3) + b3
    y = softmax(a3)

    return y
```

이제 위 함수들을 엮어서 10,000장의 이미지에 대한 최종 정확도(Accuracy)를 측정해 볼까요?

```python
x, t = get_data()
network = init_network()

accuracy_cnt = 0

for i in range(len(x)):
    y = predict(network, x[i])
    p = np.argmax(y)  # 가장 높은 출력값(확률)을 가진 클래스의 인덱스(0~9) 반환

    if p == t[i]:
        accuracy_cnt += 1

print(f"Accuracy: {float(accuracy_cnt) / len(x) * 100:.2f}%")
# 출력 결과: "Accuracy: 93.52%"
```

신경망이 가상 데이터를 넘어 실제 픽셀 이미지를 받아 **93.52%의 정확도**를 달성했습니다! 장난감 수준의 퍼셉트론에서 출발해 실제 손글씨 숫자를 대부분 맞히는 훌륭한 신경망으로 진화한 순간입니다.

---

## 속도의 한계에 도전하다: 배치(Batch) 처리

위의 코드는 논리적으로 완벽합니다. 하지만 치명적인 약점이 있는데, 바로 "느리다"는 것입니다.

예를 들어 처리해야 할 이미지가 10,000장이 아니라 1억 장이라면 어떨까요? 가장 큰 성능 병목은 Python 레벨에서 데이터 한 장마다 for 루프를 돌며 작은 연산 함수들을 반복 호출하는 데서 발생합니다. 하드웨어의 자원은 대부분 비어있는데 Python 인터프리터만 바쁜 비효율적인 상황이 되는 것이죠.

이를 해결하기 위해 데이터를 묶음 단위로 뭉쳐서 한 번에 신경망에 흘려보내는 기법을 **배치(Batch) 처리**라고 부릅니다.

> **엄밀한 개념 정리**
>
> 딥러닝 문헌에서는 '배치(Batch)'라는 용어가 문맥에 따라 조금 다르게 사용됩니다.
>
> 전통적으로는 한 번에 처리되는 데이터 묶음을 의미하며, 학습 알고리즘 관점에서는 **학습 데이터 전체를 일괄 처리**하는 Batch Gradient Descent와 일부만 사용하는 Mini-batch Gradient Descent를 구분합니다.
>
> 그러나 현대의 딥러닝 프레임워크(PyTorch, TensorFlow 등)에서 설정하는 `batch_size` 같은 값은 전체 데이터가 아니라 그중 일부를 떼어낸 **미니 배치(Mini-batch)**의 크기를 의미하며, 엔지니어들은 실무에서 **'미니 배치'를 축약하여 '배치'**라고 부릅니다. 현대 딥러닝에서는 사실상 미니 배치 학습만 사용하기 때문입니다.
>
> 본 포스트에서도 이러한 관례적 표현에 따라 편의상 '배치'로 명명하겠습니다.
{: .prompt-info }

### 배치의 수학적 원리: 행렬의 차원

우리가 설계한 3층 신경망 가중치들의 구체적인 행렬 차원(Shape) 관계를 펼쳐놓고 수학적으로 비교해 봅시다.

#### 배치 처리를 하지 않았을 때 (단일 이미지 추론)

데이터를 한 장씩 순전파 연산을 할 때의 차원 변화는 다음과 같습니다.

$$\begin{array}{rccccccc}
\text{입력 } X & \cdot & \text{가중치 } W_1 & + & \text{편향 } B_1 & = & \text{출력 } A_1 \\
(784,) & & (784, 50) & & (50,) & & (50,)
\end{array}$$

* **입력 $X (784,)$:** 손글씨 이미지 1장의 크기는 $28 \times 28$ 픽셀입니다. 이를 평탄화(Flatten)하여 일렬로 늘어놓았기 때문에 784개의 픽셀 값이 들어있는 1차원 벡터가 됩니다.
* **가중치 $W_1 (784, 50)$:** 784개의 입력 신호를 받아 다음 은닉층(Hidden Layer)의 50개 뉴런으로 전달하는 연결 강도(가중치)입니다. 784행 50열의 2차원 행렬입니다.
* **편향 $B_1 (50,)$:** 은닉층에 있는 50개의 뉴런 각각이 가지는 고유의 예민도(편향)입니다. 뉴런이 50개이므로 편향도 50개가 필요하여 $(50,)$ 크기의 1차원 벡터가 됩니다.
* **출력 $A_1 (50,)$:** 입력 $X$와 가중치 $W_1$을 행렬 곱하고 편향 $B_1$을 더한 최종 결과물입니다. 다음 층의 50개 뉴런으로 전달될 신호 값들이므로 똑같이 $(50,)$ 크기의 1차원 벡터가 나옵니다.

이렇게 한 장의 연산이 끝나면 최종 출력 $Y$의 형상은 $(10,)$이 되고, 이 연산을 10,000번 반복합니다.

#### 배치 처리를 할 때 (100장을 한 묶음으로 추론)

이제 100장의 데이터를 한꺼번에 묶어서 행렬 곱을 시도하면 어떻게 될까요? 배치 처리가 되면 입력이 1차원에서 2차원 행렬로 차원이 확장됩니다. 즉, 묶인 입력 $X$의 차원은 $(100, 784)$가 됩니다.

$$\begin{array}{rccccccc}
\text{입력 } X & \cdot & \text{가중치 } W_1 & + & \text{편향 } B_1 & = & \text{출력 } A_1 \\
(100, 784) & & (784, 50) & & (50,) & & (100, 50)
\end{array}$$

* **입력 $X$가 $(784,)$에서 $(100, 784)$로 변경:** 이제 입력은 784개의 픽셀을 가진 이미지(행)가 세로로 100장 쌓여있는 형태의 2차원 행렬이 됩니다.
* **출력 $A_1$도 $(50,)$에서 $(100, 50)$으로 변경:** 가중치 $W_1 (784, 50)$의 차원은 그대로 유지한 채 행렬 곱 연산을 수행하면, 출력 결과는 이미지 100장마다 각각 50개의 뉴런 출력을 계산한 2차원 행렬, $(100, 50)$이 됩니다.

우리는 가중치 $W_1$의 모양을 전혀 바꾸지 않았고, 단지 입력 데이터 앞쪽에 배치 차원(Batch Dimension)만 추가했을 뿐인데 연산 결과가 완벽하게 호환되어 $(100, 50)$이라는 묶음 형태의 출력을 뿜어냅니다!

동일한 논리로 출력층을 빠져나온 최종 예측 결과 $Y$의 차원은 $(100, 10)$이 될 것입니다. 이는 100장의 이미지에 대한 10가지 숫자의 확률 정보가 담긴 2차원 행렬입니다.

> **편향 $B_1$은 여전히 $(50,)$인데 어떻게 더하기가 되나요?**
>
> 행렬의 덧셈은 원래 두 행렬의 크기가 동일해야 가능하죠. 그런데 NumPy에는 **브로드캐스팅(Broadcasting)**이라는 똑똑한 기능이 있습니다. $(100, 50)$ 크기의 행렬 뒤에 $(50,)$ 크기의 벡터를 더하면, NumPy는 자동으로 이 $(50,)$짜리 편향 벡터를 100번 복사한 것처럼 취급하여 $(100, 50)$ 크기의 행렬과 연산하게 됩니다. 덕분에 우리는 일일이 복사하는 코드를 짜지 않아도 손쉽게 덧셈을 수행할 수 있습니다.
{: .prompt-tip }

### Python 코드로 배치 추론 구현

슬라이싱(`x[start:end]`)을 활용하여 전체 데이터를 배치 크기만큼 쪼개어 연산하는 코드입니다.

```python
x, t = get_data()
network = init_network()

batch_size = 100  # 배치 크기 설정
accuracy_cnt = 0

# batch_size만큼 건너뛰며 반복
for i in range(0, len(x), batch_size):
    x_batch = x[i:i+batch_size]          # 100장씩 슬라이싱 -> shape: (100, 784)
    y_batch = predict(network, x_batch)  # 100장을 한 번에 추론 -> shape: (100, 10)

    # 100장 각각에 대해 가장 출력값(확률)이 높은 인덱스 추출
    # `axis=1`은 각 행(row) 내에서 최댓값 인덱스를 찾으라는 의미입니다.
    p = np.argmax(y_batch, axis=1)  # shape: (100,)

    # 실제 정답(t[i:i+batch_size])과 예측값(p)이 일치하는 개수를 합산
    accuracy_cnt += np.sum(p == t[i:i+batch_size])

print(f"Accuracy (with Batch): {float(accuracy_cnt) / len(x) * 100:.2f}%")
# 출력 결과: "Accuracy (with Batch): 93.52%"
```

* **`range(0, len(x), batch_size)`:** `i`가 `0, 100, 200, ...` 단위로 증가합니다.
* **`x[i:i+batch_size]`:** 데이터를 `batch_size` 묶음만큼 떼어옵니다.
* **`np.argmax(y_batch, axis=1)`:** 100행 10열의 예측 행렬에서 각 행별 최댓값 위치를 찾습니다. (`axis=1`은 1번째 차원, 즉 열 방향 안에서 최댓값을 뜻합니다.) 결과적으로 이미지 100장에 대한 100개의 예측 인덱스 배열이 나옵니다.
* **`np.sum(p == t[i:i+batch_size])`:** NumPy의 브로드캐스팅과 조건 비교를 사용해 `[True, False, ...]` 배열을 만들고, `True` 개수만 세어 더합니다.

배치 처리를 적용한 코드와 그렇지 않은 코드의 실행 시간을 측정하면 환경에 따라 **눈에 띄는 성능 향상**이 나타나는 것을 볼 수 있습니다! 예측 결과와 정확도는 당연히 동일하게 유지하면서 말이죠.

> **배치 처리가 압도적으로 빠른 이유**
>
> 배치 처리는 수많은 Python 반복문 호출을 제거하고, 하나의 거대한 대형 행렬 연산으로 변환합니다. NumPy 같은 수치 계산 라이브러리는 내부적으로 C/C++ 기반의 고도로 최적화된 저수준 선형대수 연산 라이브러리(BLAS 등)를 사용하므로, 이 거대한 행렬을 하드웨어 단에서 강력한 병렬 연산(Vectorization/SIMD)으로 순식간에 처리해 냅니다. GPU 환경에서는 이러한 병렬 연산 속도 향상 효과가 더 극적으로 나타납니다.
{: .prompt-tip }

---

## 3부 끝

이번에는 장난감 신경망을 벗어나, 현실의 데이터인 **MNIST 손글씨 데이터셋**을 사용해 보았습니다.

2차원의 이미지 데이터를 가공하여 일렬로 정렬하는 **평탄화(Flatten)**, 입력 데이터의 스케일을 맞추기 위한 **정규화(Normalization)**라는 개념을 접하고, 단일 추론에서 발생하는 병목 제거 및 연산 가속을 실현하기 위한 **배치(Batch) 처리**의 수학적 흐름과 그 차원을 확인했습니다. 배치 단위로 데이터를 쪼개어 예측하는 Python 코드를 직접 작성해 보기도 했습니다.

이번에는 미리 만들어진 가중치 파일(`sample_weight.pkl`)의 도움을 받아 편하게 테스트를 완수했습니다. 하지만 진짜 딥러닝 모델을 만들려면 가중치와 편향을 모델이 데이터로부터 직접 학습할 수 있도록 해야겠죠?

다음 [4부 - 신경망 학습의 나침반, 손실 함수와 기울기]에서는 신경망의 예측이 맞았는지 틀렸는지, 만약 틀렸다면 어떤 방향으로 파라미터를 고치는 게 좋을지 가이드라인을 제공하는 **손실 함수(Loss Function)**, 변화를 감지하는 수치 미분, 학습의 핵심 원리인 **경사 하강법(Gradient Descent)** 등에 대해 본격적으로 탐구해 보겠습니다.
