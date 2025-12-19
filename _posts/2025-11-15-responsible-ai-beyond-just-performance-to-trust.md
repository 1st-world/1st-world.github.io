---
title: "책임 있는 인공지능(Responsible AI): 단순한 성능을 넘어 '신뢰'로"
author: 1st-world
date: 2025-11-16 16:00:00 +0900
last_modified_at: 2025-12-14 14:50:00 +0900
categories: [Artificial Intelligence]
tags: [rai, xai, fairlearn, shap]
pin: false
math: true
---

생성형 AI(Generative AI)의 시대에 접어들면서 우리는 놀라운 기술적 진보를 목격하고 있습니다. 하지만 동시에 AI가 만들어낸 그럴듯한 거짓말(Hallucination), 학습 데이터에 내재된 편향성(Bias), 불투명한 의사결정 과정(Black Box) 등에 대한 우려도 커지고 있습니다.

특정 AI 모델의 성능 지표가 F1 Score 0.95를 가리키더라도, 만약 그 모델이 특정 인구 집단에 대해 일관되게 낮은 성능을 보이거나 의사결정의 근거를 설명할 수 없다면 실패한 모델이나 마찬가지입니다.

이제 AI 개발의 목표는 단순히 SOTA(State of the Art) 수준의 성능을 달성하는 것을 넘어 **'어떻게 사용자가 믿고 쓸 수 있는 AI를 만들 것인가'**도 중요하게 여겨집니다.

이번 포스트에서는 **'책임 있는 인공지능(Responsible AI, 이하 RAI)'**이 무엇이며, 이를 실무에는 어떻게 적용할 수 있는지 간단한 코드와 함께 알아봅니다.

---

## 1. Responsible AI, 왜 중요한가?

과거에는 모델의 성능 지표가 좋다면 그것만으로 뛰어난 모델로 평가받았습니다. 예를 들어 정확도(Accuracy)가 99%라면 대체로 훌륭한 모델이겠죠. 하지만 그 1%의 오류가 특정 인종이나 성별에 집중되어 있다면 어떨까요? 혹은 대출 심사 AI가 왜 대출을 거절했는지 설명하지 못한다면 금융 규제를 통과할 수 있을까요?

RAI가 왜 필수적인지 이해하기 위해, 실제 업계에서 발생했던 대표적인 실패 사례를 살펴볼 필요가 있습니다.

### Case A: 아마존(Amazon)의 채용 AI 폐기 (데이터 편향성)

* **상황:** 아마존은 채용 과정을 효율화하려는 목적으로 2014년부터 AI 채용 시스템을 개발했으나 실패했습니다. 이 AI는 "여성"이라는 단어가 들어간 이력서(예: "Women's Chess Club Captain")에 감점을 주었습니다.

* **원인:** AI는 아마존 기술직 채용에 사용된 지난 10년간의 이력서를 학습 데이터로 사용했는데, 당시 아마존 기술직은 남성의 비중이 압도적이었기에 자연스럽게 남성 지원자의 이력서가 대부분일 수밖에 없었습니다. 그러다 보니 모델은 '과거의 패턴(남성 중심 채용)'을 '미래의 정답'으로 **과적합(Overfitting)**한 것입니다.

* 개발진이 여성 관련 단어를 없애는 패치를 시도했지만, 모델이 여전히 다른 간접적인 신호(특정 이력, 문장 패턴 등)로 성별을 추론하는 등 편향을 제거하지 못했습니다. 결국 2018년에 해당 프로젝트를 폐기했습니다.

### Case B: 환자 케어 알고리즘의 인종 차별 (프록시 변수의 함정)

* **상황:** 2019년 10월, Science지에 발표된 "[Dissecting racial bias in an algorithm used to manage the health of populations](https://www.ftc.gov/system/files/documents/public_events/1548288/privacycon-2020-ziad_obermeyer.pdf)" 논문을 통해, 미국의 병원에서 고위험 환자를 선별하여 추가 케어를 제공하기 위해 사용하던 알고리즘에서 흑인 환자를 백인 환자보다 덜 위험하다고 판단하는 경향(약 6분의 1 수준)이 발견되었습니다.

* **원인:** 알고리즘은 '의료비 지출'을 질병의 심각도를 나타내는 대리 변수(Proxy Variable)로 사용했습니다. 하지만 당시 미국 사회에서 흑인 환자는 동일한 질병을 앓더라도 의료 접근성, 보험 등 각종 사회·경제적 요인으로 인해 의료비를 덜 지출하는 경향이 있었고, 알고리즘은 "의료비가 적게 든다"를 "덜 아프다"라고 잘못 해석한 것입니다.

* 이후 연구진이 질병의 심각도를 나타내는 요소를 의료비 대신 만성질환, 합병증 수 등 다른 프록시로 바꿔 다시 학습시켰더니 인종 차별이 거의 사라졌습니다. 즉, 알고리즘 자체가 악의적이거나 인종 변수를 직접 쓴 것이 아니라, **겉보기엔 중립적인 프록시 변수(의료비)가 사회적 불평등을 반영하면서 편향이 발생**한 사례였습니다.

> **본질 파악하기**
>
> 데이터에는 사회적 편향이 반영되어 있는 경우가 많은데, 앞서 소개한 Case A(아마존 사례)처럼 단순히 해당 변수를 사용하지 않거나 관련 단어들을 없애는 것만으로는 편향이 해결되지 않을 수 있습니다. 다른 변수들이 편향된 속성을 대변(Correlation)하고 있기 때문입니다.
{: .prompt-tip }

RAI는 단순한 '윤리 캠페인' 같은 게 아닙니다. 이는 **리스크 관리(Risk Management)**이자 **제품의 품질(Quality Assurance)**과 직결된 문제입니다. 신뢰할 수 없는 AI는 결국 사용자에게 외면받을 것이고, 기업에 법적·사회적 비용을 초래할 수 있습니다.

---

## 2. Responsible AI의 6대 원칙

Microsoft 등 주요 빅테크 기업들이 정의하는 RAI의 6가지 원칙은 다음과 같습니다. 개발자는 이 추상적인 개념들을 구체적인 기술 목표로 연결할 수 있어야 합니다.

* **공정성(Fairness):** 인종, 성별, 나이 등에 따른 차별 없이 모든 사람에게 공정해야 합니다.
* **신뢰성 및 안전성(Reliability & Safety):** 예상치 못한 상황에서도 일관성 있게 동작하고, 인간에게 해를 끼치지 않아야 합니다.
* **개인정보 보호 및 보안(Privacy & Security):** 학습 데이터의 프라이버시를 보호하고, 외부 공격으로부터 모델을 방어해야 합니다.
* **포용성(Inclusiveness):** 장애 여부 등과 관계없이 모든 사람이 혜택을 누릴 수 있어야 합니다.
* **투명성(Transparency):** 시스템이 어떻게 설계되었고, 어떤 데이터를 사용했는지 명확해야 합니다.
* **책임성(Accountability):** AI의 결과에 대해 누가 책임을 질 것인지 거버넌스 체계를 마련해야 합니다.

위 6가지 원칙 모두 중요하지만, 엔지니어링 단계에서 우리가 코드로 직접 통제하고 구현할 수 있는 대표적인 영역은 **'공정성(Fairness)'**과 **'투명성(Transparency)'**입니다. 특히, 투명성을 확보하기 위해서는 모델이 왜 그런 결과를 냈는지 보여주는 **'설명 가능성(Explainability)'** 기술이 필수적입니다.

이제부터 이 두 가지를 Python 코드로 어떻게 구현할 수 있는지 살펴보겠습니다.

---

## 3. 도구를 활용한 실무 구현

모델의 공정성을 검증하기 위해서는 **정성적인 판단이 아닌, 정량적인 지표(Quantitative Metrics)**가 필요합니다. 이를 측정하기 위한 대표적인 도구 중 하나인 Microsoft의 오픈소스 라이브러리 'Fairlearn'을 사용해 봅니다.

### 진단(Diagnosis): 내 모델은 공정한가?

#### 시나리오: 대출 승인 모델 (Binary Classification)

모델의 성능이 높다고 해서 공정한 것은 아니죠. 예를 들어, 대출 심사 모델의 전체 정확도가 90%라도, 남성의 승인율은 80%인데 여성의 승인율이 40%라면 이는 편향이 있다고 봅니다.

#### 공정성 지표

우리가 살펴볼 주요 지표는 **인구통계적 동등성 차이(Demographic Parity Difference)**입니다. 민감 변수(성별, 인종 등)에 따른 긍정적인 예측의 비율(대출 승인율)의 차이가 얼마나 나는지를 보는 것입니다.

$$Metric = |P(\hat{Y}=1 | A=\text{group}_1) - P(\hat{Y}=1 | A=\text{group}_2)|$$

#### 코드 구현 (Fairlearn 활용)

```python
# pip install fairlearn
from fairlearn.metrics import MetricFrame, selection_rate
from sklearn.metrics import accuracy_score

# (데이터 로드 및 전처리 과정 등은 이미 완료되었다고 가정하고 생략)

# y_true: 실제 값, y_pred: 예측 값, X_test['sex']: 성별 정보

# 그룹별 지표 계산
grouped_metric = MetricFrame(
    metrics={
        "accuracy": accuracy_score,
        "selection_rate": selection_rate  # 긍정 예측(대출 승인) 비율
    },
    y_true=y_true,
    y_pred=y_pred,
    sensitive_features=X_test['sex']      # 성별, 인종 등 민감 변수 (여기서는 성별로 가정)
)

print("전체 정확도: ", grouped_metric.overall['accuracy'])
print("그룹별 승인율:\n", grouped_metric.by_group['selection_rate'])
```

위 예시 코드의 출력값 중 그룹별 승인율이 남성 0.85, 여성 0.45로 나타났다고 가정하면, 약 0.40의 격차(Difference)가 발생하므로 해당 모델은 편향이 존재한다고 판단할 수 있습니다.

### 완화(Mitigation): 편향 줄이기

문제를 인지했다면 해결해야겠죠. 편향 발견 시, 전처리 단계(Pre-processing)에서는 Reweighing 기법을 통해 학습 데이터 내 소수 그룹의 가중치를 높이는 등 데이터 불균형을 해소하고, 모델링 단계(In-processing)에서는 **공정성 제약 조건(Fairness Constraint)**을 주는 것이 좋습니다.

`ExponentiatedGradient` 알고리즘은 손실 함수 최적화 과정에서 공정성 제약 조건을 함께 고려하도록 학습시킵니다. 이로써 모델의 오차를 줄임과 동시에 공정성 위배 정도를 제약 조건으로 관리할 수 있습니다.

#### 코드 구현 (Fairlearn 활용)

```python
from fairlearn.reductions import ExponentiatedGradient, DemographicParity
from sklearn.tree import DecisionTreeClassifier

# 공정성 제약 조건 설정
constraint = DemographicParity()

# 기존 모델(Classifier)을 감싸서 공정성을 고려한 방향으로 재학습
mitigator = ExponentiatedGradient(
    estimator=DecisionTreeClassifier(),
    constraints=constraint
)

mitigator.fit(X_train, y_train, sensitive_features=X_train['sex'])
```

이후 `mitigator.predict()`의 결괏값은 기존 모델보다 공정성이 개선되었을 확률이 높습니다.

### 설명 가능성(Explainability): 블랙박스 열기

Responsible AI의 또 다른 핵심은 '설명 가능성'입니다. 현업 담당자나 고객이 "AI가 이 대출을 왜 거절했나요?"라고 물었을 때 "알고리즘이 그렇게 계산했으니까요."라고 답하면 곤란하겠죠.

XAI(eXplainable AI) 구현을 위해서는 본질적으로 해석 가능한 모델(Decision Tree, GAMs 등)을 우선 적용하는 것이 권장되고, 성능상 불가피하게 블랙박스 모델을 사용하는 경우 **SHAP, LIME 등의 사후 분석(Post-hoc) 기법을 활용**하여 보완해야 합니다.

**SHAP(SHapley Additive exPlanations)**는 게임 이론에 기반하여 각 Feature(소득, 연체 기록 등)가 예측 결과에 어느 방향으로 얼마나 기여했는지를 수치화하여 보여줍니다.

```python
import shap

# 모델 설명자 생성
explainer = shap.Explainer(model)
shap_values = explainer(X_test)

# 특정 고객의 데이터에 대한 Waterfall Plot
# shap_values[0, :, 1]
# 첫 번째 샘플(0)의, 모든 Feature(:) 중, Class 1(승인)에 대한 기여도(1)만 추출
shap.plots.waterfall(shap_values[0, :, 1])
```

아래 이미지는 "왜 대출이 거절되었는가?"에 대한 SHAP Waterfall Plot 예시입니다.

![light mode only](/assets/img/posts/2025-11-15-responsible-ai-beyond-just-performance-to-trust_1_LIGHT.png){: .light .rounded-10 w='798' h='340' }
![dark mode only](/assets/img/posts/2025-11-15-responsible-ai-beyond-just-performance-to-trust_1_DARK.png){: .dark .rounded-10 w='798' h='340' }

SHAP에서 색깔은 예측값(확률)을 높이느냐 낮추느냐를 의미합니다. <span class="text-red">빨간색</span>은 양수(+)로서 확률을 높이는 요인(승인 가능성을 높임 → 긍정적), <span class="text-sky">파란색</span>은 음수(-)로서 확률을 낮추는 요인(승인 가능성을 낮춤 → 부정적)입니다.

위 이미지에서는 모든 요인이 왼쪽(<span class="text-sky">파란색</span>)을 향하고 있습니다. **매우 강한 "거절" 신호**인 것입니다.

* **시작점 $E[f(X)]$ (= 0.301):** 이 모델의 평균적인 대출 승인율은 약 30.1%입니다.
* **도착점 $f(x)$ (= 0.011):** 이 고객의 최종 대출 승인 확률은 **1.1%**로 계산되었습니다.
* **각 요인 해석**
  1. **Credit_Score (= 520):** 신용점수가 낮아서 승인 확률이 **13%p만큼 떨어졌습니다.**
  2. **Total_Debt (= 35000):** 부채가 많아서 승인 확률이 **11%p만큼 더 떨어졌습니다.**
  3. **Years_Employed (= 2):** 근속 연수가 짧아서 확률이 **5%p만큼 더 떨어졌습니다.**
  4. **Annual_Income (= 55000):** 연소득이 5.5만 불 정도 있지만, 이 모델 기준에서는 승인 확률을 높여주는 요소로 작용하지 못하고 **영향력이 거의 없거나(-0)** 미미하게 부정적이었습니다.

> **기술적 참고: SHAP 값의 단위**
>
> 분류(Classification) 모델에서 SHAP 값은 일반적으로 로그 오즈(Log-odds) 단위로 계산됩니다. 하지만 본문에서는 직관적인 이해를 돕기 위해 이를 확률(Probability) 공간으로 변환 및 근사한 값으로 가정하고 설명했습니다. 실제 구현을 위해 코드를 작성할 때는 `model_output='probability'` 옵션을 설정하면 확률 단위로 변환된 값을 얻을 수 있습니다.
{: .prompt-info }

결론적으로, 상담원은 고객에게 단순히 "거절되었습니다."가 아니라 **"소득은 나쁘지 않지만, 현재 신용점수와 부채 비율을 개선해야 승인 가능성이 높아집니다."**라고 설명할 수 있을 것입니다.

이러한 시각화 자료가 있으면 모델의 신뢰도가 높아져 현업 부서 및 규제 기관 등을 설득하기에 용이하며, 엔지니어에게는 디버깅을 돕는 결정적 근거로 작용하기도 합니다.

---

## 맺음말: 지속적인 개선으로 균형 맞추기

초기 AI 개발이 '기능 구현'에 집중했다면, 이제는 **'지속 가능성'**으로 초점이 맞춰지고 있습니다. 책임 있는 AI를 구축하는 것은 한 번의 코드 배포만으로 끝나는 일이 아니죠. 사회적 기준은 변하고, 데이터는 끊임없이 흐릅니다.

엔지니어로서 우리는 모델을 훈련시킬 때 이 모델이 실제 세상에 나간다면 누구에게 어떤 영향을 미칠지 한 번 더 고민할 필요가 있습니다.

기술적 우수함(Performance)과 윤리적 책임감(Responsibility)의 **균형점**을 찾는 것. 이것이 AI가 일상이 된 이 시대에 엔지니어가 지향해야 할 진정한 의미의 '엔지니어링'이 아닐까요?
