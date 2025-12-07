---
title: "AI는 어떻게 평가해야 하는가? (6) - AI 평가의 핵심 지표 (完)"
author: 1st-world
date: 2025-11-02 17:15:00 +0900
last_modified_at: 2025-12-07 17:35:00 +0900
categories: [Artificial Intelligence, Machine Learning]
tags: [confusion-matrix, accuracy, precision, recall, f1-score, pr-curve, roc-curve]
pin: false
math: true
---

우리는 [1부](/posts/ai1-how-does-ai-understand-language/)(트랜스포머 이해를 위한 핵심 부품)부터 [5부](/posts/ai5-how-ai-generates-the-world/)(이미지를 생성하는 모델들)까지, AI가 언어와 이미지를 '이해'하고 '생성'하는 세계를 탐험하며 핵심 원리들을 파악했습니다. 이제 마지막 질문이 남았네요.

**"그래서, 우리가 다룬 AI 모델이 '얼마나 좋은지'는 어떻게 알 수 있을까요?"**

모델을 평가하는 것도 AI 개발의 핵심입니다. "AI가 똑똑해졌다" 같은 감상적인 표현이 아니라 "AI가 특정 작업에서 **99%**의 성능을 보인다"처럼 말할 수 있게 하는 '숫자'가 필요합니다.

오늘은 AI, 그중에서도 특히 '분류(Classification)' 모델을 평가하는 핵심 지표들을 알아보겠습니다.

---

## 1. 모든 평가의 시작: 혼동 행렬(Confusion Matrix)

먼저 스팸 메일을 분류하는 모델을 간단한 예시로 들어보겠습니다.

* **"모델이 스팸 메일을 99% 맞혔다!" (정확도 99%)**

이런 결과가 나왔다면, 과연 이 모델은 '정말로' 좋은 모델이라 확신할 수 있을까요?

만약 전체 메일 100개 중 '정상 메일'이 99개이고 '스팸'은 1개뿐이라면? 모델이 모든 메일을 "정상!"이라고만 예측해도 99%의 정확도를 달성합니다. 하지만 이 모델은 '스팸'을 전혀 걸러내지 못하는, 쓸모없는 모델이죠.

이 '정확도의 함정'을 피하기 위해, 우리는 모델이 **무엇을 맞혔고 어떻게 틀렸는지** 분해해 봐야 합니다. 이것이 바로 **'혼동 행렬(Confusion Matrix)'**입니다.

가장 고전적인 예시인 '암(Cancer) 진단' 모델을 기준으로 4가지 상황을 살펴보겠습니다.

아래 표는 혼동 행렬의 4분면을 시각적으로 보여줍니다.

| | **예측: 정상 (Negative)** | **예측: 암 (Positive)** |
| :---: | :---: | :---: |
| **실제: 정상 (Negative)** | <span class="text-blue">**TN (True Negative)**<br>정상을 정상으로 (정답)</span> | <span class="text-red">**FP (False Positive)**<br>정상을 암으로 (오답)</span> |
| **실제: 암 (Positive)** | <span class="text-red">**FN (False Negative)**<br>암을 정상으로 (오답)</span> | <span class="text-blue">**TP (True Positive)**<br>암을 암으로 (정답)</span> |
{: style="border: 2px solid #DCE0E2;"}

1. <span class="text-blue">**True Positive(TP)**</span>
    * **실제 = O(암)** / **예측 = O(암)**
    * 모델이 '암'을 '암'이라고 **정확히** 맞혔습니다.

2. <span class="text-blue">**True Negative(TN)**</span>
    * **실제 = X(암이 아님)** / **예측 = X(암이 아님)**
    * 모델이 '정상'을 '정상'이라고 **정확히** 맞혔습니다.

3. <span class="text-red">**False Positive(FP / Type Ⅰ Error)**</span>
    * **실제 = X(암이 아님)** / **예측 = O(암)**
    * 모델이 '정상'을 '암'이라고 **잘못** 예측했습니다. ("양치기 소년")

4. <span class="text-red">**False Negative(FN / Type Ⅱ Error)**</span>
    * **실제 = O(암)** / **예측 = X(암이 아님)**
    * 모델이 '암'을 '정상'이라고 **잘못** 예측했습니다. (이 상황에서 **가장 치명적인 오류**)

#### Python 코드로 구현하기

```python
from sklearn.metrics import confusion_matrix

# y_true = [실제 정답 값], y_pred = [모델 예측 값]
y_true = [0, 0, 0, 0, 0, 0, 0, 1, 1, 1]  # '정상' 7명, '암' 3명
y_pred = [0, 0, 0, 0, 0, 1, 1, 1, 1, 0]  # (0.5 임곗값 기준 예측)

confusion_matrix(y_true, y_pred)
# array([[5, 2],  <- 실제 정상(0) 7명 중 5명(TN) 맞히고 2명(FP) 틀림
#       [1, 2]])  <- 실제 암(1) 3명 중 2명(TP) 맞히고 1명(FN) 틀림

# 개별 값으로 바로 받기
tn, fp, fn, tp = confusion_matrix(y_true, y_pred).ravel()
print(tn, fp, fn, tp)  # 5, 2, 1, 2
```

이제 우리는 이 4가지 숫자(TP, TN, FP, FN)를 조합하여 모델의 성능을 입체적으로 분석할 수 있습니다.

---

## 2. 기본 지표: 정확도(Accuracy)

* **정의:** 모델이 전체 데이터 중에서 얼마나 정확하게 예측했는가?
* **계산:** $(TP + TN) \div (TP + TN + FP + FN)$
* 가장 직관적이지만, 앞서 언급한 '스팸 메일' 예시처럼 데이터가 **불균형(Imbalanced)**할 경우(예를 들어 99%는 '정상'이고 1%만 '스팸'일 때) 이 지표는 심각하게 왜곡될 수 있다는 문제가 있습니다.

#### Python 코드로 구현하기

```python
from sklearn.metrics import accuracy_score

y_true = [0, 0, 0, 0, 0, 0, 0, 1, 1, 1]
y_pred = [0, 0, 0, 0, 0, 1, 1, 1, 1, 0]

acc = accuracy_score(y_true, y_pred)
# (TP + TN) / (전체 합) = 7 / 10 = 0.7

print(f"정확도: {acc:.2f}")  # 정확도: 0.70
```

---

## 3. '정확도의 함정'을 피하는 법: 정밀도(Precision), 재현율(Recall)

데이터가 불균형할 때, 우리는 '정확도' 대신 '정밀도'와 '재현율'이라는 지표에 집중해야 합니다.

### 🎯 정밀도(Precision)

* **정의:** 모델이 "Positive(암)"라고 예측한 것들 중에서 **실제로 Positive(암)인** 비율
* **계산:** $TP \div (TP + FP)$
* 모델의 **'예측'** 관점입니다. ("내가 '암'이라고 진단한 환자들, 얼마나 정확하지?")
* **FP(False Positive)가 치명적일 때** 중요합니다.
    * **예시:** 스팸 메일 필터
    * 정상 메일(Negative)을 스팸(Positive)으로 잘못 예측하면(FP), 사용자는 중요한 메일을 놓칠 수 있습니다. 따라서 모델은 **정말 확실한 스팸만 잡도록** 예측해야 합니다. (차라리 스팸 몇 개를 걸러내지 못하더라도)

#### Python 코드로 구현하기

```python
from sklearn.metrics import precision_score

y_true = [0, 0, 0, 0, 0, 0, 0, 1, 1, 1]
y_pred = [0, 0, 0, 0, 0, 1, 1, 1, 1, 0]

prec = precision_score(y_true, y_pred)
# TP / (TP + FP) = 2 / (2 + 2) = 0.5

print(f"정밀도: {prec:.2f}")  # 정밀도: 0.50
```

### 🎣 재현율(Recall) / 민감도(Sensitivity) / TPR

* **정의:** 실제 Positive(암)인 것들 중에서 **모델이 "Positive(암)"라고 예측한** 비율
* **계산:** $TP \div (TP + FN)$
* 실제 **'정답'** 관점입니다. ("실제 '암' 환자들 중, 내가 몇 명이나 찾아냈지?")
* **FN(False Negative)이 치명적일 때** 중요합니다.
    * **예시:** 암 진단 AI
    * 실제 암 환자(Positive)를 정상(Negative)으로 잘못 예측하면(FN), 환자는 치료 시기를 놓쳐 생명을 잃을 수 있습니다. 따라서 모델은 **단 한 명의 암 환자라도 놓치지 않도록** 최대한 많이 찾아내야 합니다. (차라리 정상인을 암 환자라고 잘못 진단하더라도)

#### Python 코드로 구현하기

```python
from sklearn.metrics import recall_score

y_true = [0, 0, 0, 0, 0, 0, 0, 1, 1, 1]
y_pred = [0, 0, 0, 0, 0, 1, 1, 1, 1, 0]

rec = recall_score(y_true, y_pred)
# TP / (TP + FN) = 2 / (2 + 1) = 0.666...

print(f"재현율: {rec:.2f}")  # 재현율: 0.67
```

---

## 4. 두 지표의 균형점: F1 Score

정밀도(Precision)와 재현율(Recall)은 한쪽이 올라가면 다른 쪽이 내려가는 **Trade-off** 관계입니다.

* 암 진단을 위해 "조금이라도 이상하면 다 '암'이라고 해!" → FP 증가
    * 재현율(Recall) ↑ / 정밀도(Precision) ↓
* 스팸 필터를 위해 "정말 확실한 스팸만 '스팸'이라고 해!" → FN 증가
    * 정밀도(Precision) ↑ / 재현율(Recall) ↓

개발자는 이 두 값 사이에서 적절한 균형점을 찾아야 하며, 두 지표를 하나의 숫자로 요약한 것이 **F1 Score**입니다.

* **정의:** 정밀도와 재현율의 **조화 평균(Harmonic Mean)**
* **계산:** $2 \times \frac{Precision \times Recall}{Precision + Recall}$
* 단순 평균이 아닌 '조화 평균'을 사용하는 이유는, 두 지표 중 어느 한쪽이라도 0점에 가까우면 F1 Score 역시 0점에 가깝게 만드는, 즉 **극단적인 불균형에 큰 페널티**를 주기 위함입니다. 모델이 양쪽 모두에서 준수한 성능을 낼 때 F1 Score가 높게 나옵니다.

#### Python 코드로 구현하기

```python
from sklearn.metrics import f1_score

y_true = [0, 0, 0, 0, 0, 0, 0, 1, 1, 1]
y_pred = [0, 0, 0, 0, 0, 1, 1, 1, 1, 0]

f1 = f1_score(y_true, y_pred)
# 2 * (0.5 * 0.667) / (0.5 + 0.667) = 0.571...

print(f"F1 Score: {f1:.2f}")  # F1 Score: 0.57
```

---

## 5. 성능의 전체 그림: PR Curve, ROC Curve

모델은 "이 환자는 95% 확률로 암입니다.", "이 환자는 30% 확률로 암입니다."처럼 '확률'을 출력합니다. 여기서 우리는 "몇 %부터 암이라고 판단할까?"라는 **'임곗값(Threshold)'**을 정해야 합니다.

* 임곗값을 90%(매우 보수적)로 설정 시: 정밀도 ↑ / 재현율 ↓
* 임곗값을 10%(매우 공격적)로 설정 시: 정밀도 ↓ / 재현율 ↑

모델의 성능은 임곗값에 따라 달라집니다. **PR Curve**와 **ROC Curve**는 이러한 '모든 임곗값의 변화'에 따른 모델의 성능을 한눈에 보여주는 그래프입니다.

### 📈 PR(Precision-Recall) Curve

* **X축:** 재현율(Recall)
* **Y축:** 정밀도(Precision)
* **해석:** 그래프가 **우측 상단(Precision = 1, Recall = 1)에 최대한 붙을수록** 좋은 모델입니다.
* **불균형 데이터셋**을 평가할 때 가장 신뢰할 수 있는 지표입니다. 정확도(Accuracy)가 왜곡될 수 있는 상황에서 TN을 무시하고 나머지 Positive 클래스에만 집중합니다.
* **AUC-PR**
    * PR Curve의 **아래 면적(Area Under Curve)**을 의미하며, **AP(Average Precision)**라고도 부릅니다.
    * 1에 가까울수록 좋은 모델입니다.

#### Python 코드로 구현하기

```python
from sklearn.metrics import precision_recall_curve, average_precision_score
import matplotlib.pyplot as plt

# y_pred 대신, 모델이 예측한 '확률'(Positive 클래스에 대한) 사용
y_true = [0, 0, 0, 0, 0, 0, 0, 1, 1, 1]
y_score = [0.1, 0.2, 0.15, 0.3, 0.25, 0.7, 0.6, 0.85, 0.9, 0.4]

# PR Curve를 그리기 위한 값들
precision, recall, thresholds = precision_recall_curve(y_true, y_score)

# PR Curve의 면적 (AUC-PR)
ap_score = average_precision_score(y_true, y_score)
print(f"Average Precision(AP): {ap_score:.2f}")  # Average Precision(AP): 0.87

# PR Curve 시각화
plt.figure(figsize=(8, 6))
plt.plot(recall, precision, marker='.', label=f'AP = {ap_score:.2f}')
plt.title("Precision-Recall Curve")
plt.xlabel("Recall")
plt.ylabel("Precision")
plt.legend()
plt.grid(True)
plt.show()
```

### 📈 ROC(Receiver Operating Characteristic) Curve

* **X축:** FPR(False Positive Rate / 위양성률)
    * $FPR = FP \div (FP + TN)$
    * "실제 '정상'인 사람들 중, 모델이 '암'이라고 잘못 예측한 비율"입니다. (1 - 특이도)

* **Y축:** TPR(True Positive Rate / 진양성률)
    * $TPR = TP \div (TP + FN)$
    * 이 값은 재현율(Recall)과 완전히 동일합니다.

* **해석:** 그래프가 **좌측 상단(FPR = 0, TPR = 1)에 최대한 붙을수록** 좋은 모델입니다.
    * (0, 1): 모든 정상인을 정상이라 판단함과 동시에 모든 암 환자를 찾아내는, 궁극의 목표 지점

* **AUC(Area Under Curve)**
    * ROC Curve의 **아래 면적**을 의미하며, 모델 평가에 가장 널리 쓰이는 단일 지표 중 하나입니다.
    * **AUC = 1.0:** 완벽한 모델
    * **AUC = 0.5:** 쓸모없는 모델 (무작위 선택 모델과 같으며, 그래프는 대각선으로 출력)
    * **AUC < 0.5:** 반대로 예측하는 모델 (예측 결과를 뒤집으면 해결)

#### Python 코드로 구현하기

```python
from sklearn.metrics import roc_curve, roc_auc_score
import matplotlib.pyplot as plt

# PR Curve와 마찬가지로 y_pred 대신 y_score 사용
y_true = [0, 0, 0, 0, 0, 0, 0, 1, 1, 1]
y_score = [0.1, 0.2, 0.15, 0.3, 0.25, 0.7, 0.6, 0.85, 0.9, 0.4]

# ROC Curve를 그리기 위한 값들
fpr, tpr, thresholds = roc_curve(y_true, y_score)

# ROC Curve의 면적 (AUC)
auc_score = roc_auc_score(y_true, y_score)
print(f"ROC AUC Score: {auc_score:.2f}")  # ROC AUC Score: 0.90

# ROC Curve 시각화
plt.figure(figsize=(8, 6))
plt.plot(fpr, tpr, marker='.', label=f'ROC AUC = {auc_score:.2f}')
plt.plot([0, 1], [0, 1], 'k--', label='Random Guess (AUC = 0.5)')  # 50% 기준선
plt.title("ROC Curve")
plt.xlabel("False Positive Rate(FPR)")
plt.ylabel("True Positive Rate(TPR / Recall)")
plt.legend()
plt.grid(True)
plt.show()
```

---

## 6. 그 외 지표 (FNMR 등)

* **FNMR(False Non-Match Rate)**
    * "본인(Match)인데, 본인이 아니다(Non-Match)"라고 잘못 판단한 비율입니다.
    * 안면 인식, 지문 인식 등 Biometrics 관련 도메인에서 사용되는 용어인데, 본질은 **FN(False Negative)**입니다. 즉, FNMR은 **FNR(False Negative Rate)**과 같습니다.
    * $FNR = FN \div (FN + TP)$

#### Python 코드로 구현하기

```python
# `scikit-learn`에는 이걸 계산하는 전용 함수는 없지만 `recall_score`(TPR)를 통해 계산 가능
# `FNR = 1 - TPR`이므로, `FNMR = 1 - recall_score(y_true, y_pred)`
from sklearn.metrics import recall_score

y_true = [0, 0, 0, 0, 0, 0, 0, 1, 1, 1]
y_pred = [0, 0, 0, 0, 0, 1, 1, 1, 1, 0]

fnr = 1 - recall_score(y_true, y_pred)
# 1 - (TP / (TP + FN)) = 1 - 0.666... = 0.333...

print(f"FNR (FNMR): {fnr:.2f}")  # FNR (FNMR): 0.33
```

---

## 6부 끝: 시리즈의 종착점 🏁

우리는 6부에 걸쳐 AI의 기초부터 최신 모델, 그리고 평가 지표까지 숨 가쁘게 달려왔습니다.

이번 포스트에서 마지막으로 기억해야 할 것은 **"완벽한 만능 지표는 없다"**라는 것입니다.
  * 암을 진단해야 한다면, 정확도(Accuracy)가 99%여도 재현율(Recall)이 50%라면 그 모델은 재앙입니다.
  * 스팸 필터를 만든다면, 재현율(Recall)이 100%여도 정밀도(Precision)가 30%라면 사용자는 그 메일 시스템을 쓰지 않을 것입니다.

AI를 개발하고 평가하는 것은 결국 우리가 **'무엇을 해결하고 싶은지'**라는 문제 정의로 다시 돌아오게 됩니다. 이 6편의 시리즈가 AI라는 거대한 세계를 탐험하는 데 유용한 길잡이가 되었기를 바랍니다.

### ⏪ 본 시리즈 다시 보기

* [1부 - 임베딩과 잔차 연결](/posts/ai1-how-does-ai-understand-language/)
* [2부 - 현대 AI의 심장, 트랜스포머](/posts/ai2-transformers-the-heart-of-modern-ai/)
* [3부 - 거대 언어 모델(LLM)의 등장](/posts/ai3-the-emergence-of-llm/)
* [4부 - LLM을 더 똑똑하게 만드는 RAG & RLHF](/posts/ai4-rag-rlhf-make-llm-smarter/)
* [5부 - VAE, GAN, Diffusion](/posts/ai5-how-ai-generates-the-world/)
