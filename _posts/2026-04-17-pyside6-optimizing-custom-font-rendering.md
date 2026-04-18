---
title: "[PySide6] 커스텀 글꼴 렌더링 문제 해결 (QTableView/QHeaderView)"
author: 1st-world
date: 2026-04-17 07:10:00 +0900
last_modified_at: 2026-04-18 11:55:00 +0900
categories: [Development, Python]
tags: [python, pyside, rendering, aliasing, hinting]
pin: false
math: true
---

최근 PySide6를 이용해 데스크톱 애플리케이션을 개발하면서 개인적으로 가장 골머리를 앓았던 부분은 다름 아닌 '글꼴 렌더링'이었습니다. OS에 구애받지 않는 디자인 일관성을 위해 시스템 글꼴이 아닌 외부 커스텀 글꼴을 내장하여 적용했는데, 로드 자체에는 문제가 없었지만 **텍스트 외곽선이 계단 현상(Aliasing)처럼 깨지거나 거칠게 출력**되었습니다.

처음에는 글꼴 객체에 안티 앨리어싱(Anti-aliasing)과 힌팅 제거(NoHinting) 옵션을 주는 것으로 간단히 끝날 줄 알았습니다. 하지만 예상과 달리 **앱 전역에 QSS를 적용하는 순간 일부 요소에서는 해당 설정이 무시**되었으며, 제 경우에는 QTableView의 데이터 셀과 헤더 텍스트 부분이 특히 그러했습니다. 이를 해결하기 위해 Delegate를 적용해 내부 텍스트를 고치는 데는 성공했으나 표의 헤더(Header)는 여전히 깨져 나오는 기현상을 마주해야 했습니다.

이 글은 그 문제를 해결하기 위해 겪었던 시행착오의 과정과, 그 속에서 알게 된 Qt 렌더링 파이프라인의 맹점을 정리한 기록입니다.

---

## 플랫폼 렌더링 차이와 Hinting의 중요성

운영체제마다 텍스트를 렌더링하는 백엔드 엔진이 다르기 때문에, 글꼴이 화면에 출력되는 품질도 큰 차이를 보입니다.

* **macOS:** Quartz 렌더러의 특성상 기본적으로 안티 앨리어싱을 강하게 적용하여 텍스트가 매끄럽게 출력됩니다.
* **Linux:** FreeType 및 Fontconfig 설정의 영향을 받으며, 배포판 환경에 따라 렌더링 결과가 다를 수 있습니다.
* **Windows (ClearType / DirectWrite):** ClearType 및 DirectWrite 환경에서는 **힌팅(Hinting)의 영향력이 매우 강합니다.** 코드에 흔히 적용하는 `PreferAntialias` 플래그는 시스템에 "안티 앨리어싱을 선호"한다는 힌트를 주는 수준에 그칩니다. Windows에서 강한 힌팅이 적용되면 글꼴의 벡터 곡선이 픽셀 그리드에 강제로 맞춰지면서 오히려 외곽선이 뭉개지거나 계단 현상이 두드러질 수 있습니다.

즉, Windows 같은 환경에서 커스텀 글꼴의 계단 현상을 방지하고 실제 렌더링 품질을 극적으로 개선하려면 **힌팅을 끄는 `PreferNoHinting` 플래그가 핵심적인 역할**을 합니다.

---

## 문제의 원인: QSS와 렌더링 엔진의 간섭

글꼴 객체에 안티 앨리어싱과 힌팅 제거를 적용하려면 보통 아래와 같이 설정합니다.

```python
font.setStyleStrategy(QFont.StyleStrategy.PreferAntialias)
font.setHintingPreference(QFont.HintingPreference.PreferNoHinting)
```

하지만 애플리케이션 전역에 QSS(Qt Style Sheets)를 `QWidget { font-family: 'FONT_NAME'; }`과 같이 적용하면 설정이 무시되는 현상이 발생합니다.

Qt의 스타일시트 엔진(`QStyleSheetStyle`)은 렌더링 과정에서 **스타일시트 속성을 기반으로 객체 정보를 다시 계산하거나 내부적으로 새로운 `QFont` 상태를 구성**합니다. 결과적으로 개발자가 설정해 둔 `StyleStrategy`와 `HintingPreference` 등의 저수준 플래그가 유지되지 않고 초기화되는 셈입니다.

---

## 셀(Cell) 렌더링 제어: QStyledItemDelegate

`QTableView` 내부의 데이터 셀은 뷰에 종속되지 않고 `Delegate`를 통해 독립적으로 렌더링됩니다. 셀이 화면에 그려지기 직전에 스타일 옵션을 가로채어 플래그를 다시 주입하면 QSS의 간섭을 받지 않습니다.

다음은 `QStyledItemDelegate`를 상속받아 `initStyleOption` 메서드를 오버라이딩하는 커스텀 클래스 예시입니다.

```python
from PySide6.QtWidgets import QStyledItemDelegate
from PySide6.QtGui import QFont

class EnhancedItemDelegate(QStyledItemDelegate):
    def initStyleOption(self, option, index):
        super().initStyleOption(option, index)
        # 렌더링 직전, 옵션에 포함된 font 객체에 렌더링 플래그 강제 주입
        option.font.setStyleStrategy(QFont.StyleStrategy.PreferAntialias)
        option.font.setHintingPreference(QFont.HintingPreference.PreferNoHinting)

    # (이하 paint() 오버라이딩을 통한 커스텀 아이콘 정렬 등 로직 추가 가능)
```

`initStyleOption` 메서드는 텍스트, 글꼴, 정렬 등 렌더링에 필요한 모든 `option`이 세팅된 직후, 실제 `paint()`가 호출되기 전에 실행됩니다. 즉, QSS가 글꼴까지 계산한 최종 결과물을 마지막 순간에 수정하는 방식입니다.

이 Delegate를 테이블 뷰에 적용(`setItemDelegate`)함으로써 목록 내부의 텍스트가 부드럽게 출력됩니다.

---

## 헤더(Header) 렌더링 제어: QHeaderView 커스텀 페인팅

흥미로운 건, 셀 내부 텍스트는 이제 정상적으로 출력되지만 표 상단의 헤더(Header) 텍스트는 여전히 설정이 무시된 채 거칠게 나온다는 점입니다. **헤더 렌더링을 담당하는 `QHeaderView`는 일반 아이템 Delegate 체계를 따르지 않고 `QStyle` 기반의 고유한 파이프라인을 사용**하기 때문입니다.

내부적으로 `CE_Header`, `CE_HeaderSection`, `CE_HeaderLabel` 같은 단계를 거쳐 텍스트가 그려지는데, 이 과정에서 글꼴 정보가 스타일시트 기반 값으로 다시 적용될 수 있습니다. 따라서 `QStyledItemDelegate`만으로는 헤더까지 제어할 수 없으며, 이를 온전히 제어하려면 `QHeaderView`의 `paintSection`을 직접 오버라이딩하여 텍스트 그리기 단계를 가로채야 합니다. QSS에는 배경 그리기만 맡기고 텍스트 렌더링의 주도권은 우리가 쥐는 것입니다.

> **구현 시 유의할 점**
>
> 1. **텍스트 안티 앨리어싱:** 일반 도형을 다루는 `Antialiasing`과 텍스트 윤곽선을 다루는 `TextAntialiasing`은 서로 다른 `RenderHint`입니다. 텍스트를 그릴 때는 `TextAntialiasing`을 사용해야 합니다.
>
> 2. **정렬 화살표(Sort Indicator) 오버랩 방지:** 텍스트를 직접 그릴 때, 헤더 영역(`rect`)을 전부 사용하면 오름/내림차순 정렬 화살표와 텍스트가 겹치는 버그가 발생할 수 있습니다. 이를 막기 위해 조건부로 마진(Margin)을 확보해야 합니다.
{: .prompt-warning }

다음은 `QStyle` 기반의 헤더 렌더링 중 텍스트 그리기 단계만 가로채어 안티 앨리어싱과 힌팅 제거를 성공적으로 수행하고, 동적인 마진 계산까지 포함하는 커스텀 클래스 예시입니다.

```python
from PySide6.QtWidgets import QHeaderView, QStyleOptionHeader, QStyle
from PySide6.QtGui import QPainter, QFont
from PySide6.QtCore import Qt

class EnhancedHeaderView(QHeaderView):
    def paintSection(self, painter, rect, logicalIndex):
        if not rect.isValid():
            return

        painter.save()
        option = QStyleOptionHeader()
        self.initStyleOptionForIndex(option, logicalIndex)
        option.rect = rect
        style = self.style()

        # QSS 적용된 배경 및 프레임 렌더링 (QStyle에 위임)
        style.drawControl(QStyle.ControlElement.CE_HeaderSection, option, painter, self)

        # 텍스트 영역 여백 (정렬 화살표 공간 확보)
        left_margin = 6
        right_margin = 6

        if option.sortIndicator != QStyleOptionHeader.SortIndicator.None_:
            right_margin += 16  # 정렬 화살표 크기만큼 우측 마진 추가
            style.drawPrimitive(QStyle.PrimitiveElement.PE_IndicatorHeaderArrow, option, painter, self)

        # 텍스트 직접 렌더링 (QSS 엔진 개입 차단)
        if option.text:
            # 텍스트 전용 안티 앨리어싱 활성화
            painter.setRenderHint(QPainter.RenderHint.TextAntialiasing, True)

            # 스타일 계산 완료된 option.font 기반으로 플래그 주입
            font = QFont(option.font) 
            font.setStyleStrategy(QFont.StyleStrategy.PreferAntialias)
            font.setHintingPreference(QFont.HintingPreference.PreferNoHinting)
            painter.setFont(font)

            alignment = option.textAlignment | Qt.AlignmentFlag.AlignVCenter
            metrics = painter.fontMetrics()

            # 여백 적용된 텍스트 영역 계산 및 말줄임표(Elide) 처리
            text_rect = rect.adjusted(left_margin, 0, -right_margin, 0)
            elided = metrics.elidedText(option.text, Qt.TextElideMode.ElideRight, text_rect.width())

            painter.drawText(text_rect, alignment, elided)

        painter.restore()
```

---

## 최종 적용

작성한 커스텀 Delegate와 HeaderView는 테이블 뷰 초기화 로직을 통해 적용할 수 있습니다.

```python
self.table_view = QTableView()

# 셀 텍스트 렌더링 제어 적용
self.table_view.setItemDelegate(EnhancedItemDelegate(self.table_view))

# 헤더 텍스트 렌더링 제어 적용
self.table_view.setHorizontalHeader(EnhancedHeaderView(Qt.Orientation.Horizontal, self.table_view))
```

이제야 비로소 앱의 커스텀 글꼴 렌더링이 일관되고 매끄럽게 출력되는 것을 확인할 수 있습니다.

---

## 결론

Qt의 강력한 `QStyle`과 QSS는 UI 개발 생산성을 크게 높여주지만, OS 종속적인 글꼴 렌더링 최적화 단계에서는 위와 같은 사례에서 볼 수 있듯 종종 한계 혹은 맹점을 드러내기도 합니다. 그렇지만 렌더링 파이프라인 구조를 이해하고 Qt가 제공하는 Delegate 패턴과 `paintSection` 등의 커스텀 페인팅 기법을 적재적소에 활용할 수 있다면 기존 스타일시트의 유연성을 훼손하지 않고도 원하는 품질의 결과를 만들어 낼 수 있을 것입니다.

저처럼 글꼴 렌더링 문제를 마주한 PySide/PyQt 개발자분들, 그리고 어쩌면 미래의 저에게도 이 시행착오와 해결 방법의 기록이 좋은 실마리가 되기를 바랍니다.
