---
title: "Pillow로 EXIF 데이터를 불러오는 올바른 방법"
author: 1st-world
date: 2025-10-15 01:30:00 +0900
categories: [Python]
tags: [pillow, ifd, exif, datetime]
pin: false
---

Python의 Pillow 라이브러리를 사용해 이미지의 EXIF 메타데이터를 다루던 중 촬영 날짜를 가져오는 간단한 작업에서 예상치 못한 문제에 부딪힌 적이 있습니다. 이 글에서는 `getexif()` 함수가 `DateTimeOriginal` 등 여러 태그들을 왜 바로 보여주지 않는지, 그리고 이 문제를 안정적이고 올바른 방법으로 해결하는 과정을 단계별로 정리합니다.

---

## `getexif()`와 `_getexif()`, 무엇을 써야 할까?

이미지 파일에서 EXIF 정보를 추출하기 위해 Pillow의 `getexif()` 함수를 사용했는데, 출력 결과 `DateTimeOriginal` 같은 상세한 태그는 없고 `DateTime` 등 일부 태그들만 보였습니다. 반면, 내부용 함수인 `_getexif()`를 사용하니 `DateTimeOriginal`을 포함한 수많은 태그들의 데이터를 모두 잘 반환해 주었습니다.

**`getexif()`의 반환 값 출력 (일부)**

```
{... 306: '2025:08:20 07:52:21', ...}   # DateTime
```

**`_getexif()`의 반환 값 출력 (일부)**

```
{... 306: '2025:08:20 07:52:21',        # DateTime
... 36867: '2025:08:20 07:52:21', ...}  # DateTimeOriginal
```

현 시점에서는 `_getexif()`를 쓰면 당장 문제는 해결됩니다. 아무래도 그게 제일 단순하고 편한 방법이기도 하고, 구글링을 해도 `_getexif()`를 쓴 코드가 대부분이더라고요.

하지만 이름 앞에 밑줄(`_`)이 붙은 내부용 함수들을 사용하는 건 권장하는 방법도 아닐 뿐더러 라이브러리 버전이 업데이트되면서 예고 없이 변경되거나 사라질 위험이 있습니다. 최근 Pillow에서는 `_getexif()` 사용 시 경고도 띄우고 있죠. 그렇다고 정보가 부족한 `getexif()`만 쓰자니 `DateTime`은 파일 수정 시 덮어쓰일 수도 있어 정확한 원본 촬영 일시라고 보장할 수 없습니다.

이 딜레마를 해결하기 위해 먼저 두 함수가 다르게 동작하는 이유부터 알아봐야 합니다.

## `getexif()`와 `_getexif()`의 근본적인 차이

`getexif()`와 `_getexif()`가 다른 결과를 보여주는 이유는 반환하는 데이터의 구조 때문입니다.

EXIF 데이터는 하나의 평평한 목록이 아니라 여러 개의 '**IFD** (**I**mage **F**ile **D**irectory)'가 중첩된 구조를 가지고 있습니다. 주요 IFD는 다음과 같습니다.

- **메인 IFD (0th IFD):** 이미지 크기, 제조사, `DateTime` 등 기본 정보가 담겨 있습니다.

- **Exif IFD:** `DateTimeOriginal`, 조리개, 셔터 스피드 등 사진 촬영 관련 상세 정보가 담겨 있습니다.

- **GPS IFD:** GPS 위치 정보가 담겨 있습니다.

이 구조를 바탕으로 두 함수의 동작 방식 차이를 이해해 보겠습니다.

* **`getexif()`**
    * 함수 호출 시 Pillow는 내부적으로 **모든 IFD(메인, EXIF, GPS 등)를 파싱하여 메모리에 로드**합니다.
    * 하지만 반환되는 `Exif` 객체는 메인 IFD를 중심으로 구성되어 있기 때문에 `for tag, value in exif.items():` 같은 형태로 객체를 직접 순회할 경우 **메인 IFD의 내용만 보입니다.**
    * 이 객체 안에 **이미 로드되어 있는** 하위 IFD(EXIF, GPS 등)의 데이터에 접근하기 위한 공식적인 통로가 바로 `get_ifd()` 메서드입니다. 즉, `get_ifd()`는 파일을 다시 읽는 것이 아니라 메모리에 있는 데이터의 다른 구역을 가리키는 역할을 합니다.

* **`_getexif()`**
    * 내부적으로 모든 IFD를 읽는 것은 동일하지만, 그 결과를 **중첩된 구조 그대로 두지 않고 모든 IFD의 내용을 하나의 딕셔너리로 평탄화해서 반환**합니다. 이것이 `_getexif()`의 결과에서 모든 태그 정보가 한 번에 보이는 이유입니다.
    * 이 과정에서 IFD 계층 정보가 손실되는 등의 한계점이 있습니다.

결론적으로, `getexif()`는 Pillow 개발진이 의도한 대로 **구조화된 데이터를 반환**하고, 사용자는 `get_ifd()`를 통해 그 구조에 맞게 탐색해야 합니다. 반면 `_getexif()`는 편의를 위해 모든 데이터를 하나로 합쳐 보여주는 내부용 함수인 셈입니다.

---

## 올바른 구현 방법: 코드 가독성 높이기

이제 `getexif()`로 얻은 객체에서 `get_ifd()`를 사용해 EXIF IFD에 접근해야 한다는 것을 알았습니다. `get_ifd()`를 사용하려면 Exif IFD를 가리키는 포인터의 태그 ID가 필요한데, EXIF 공식 정보에 따르면 이 태그의 이름은 `ExifOffset`이고 ID는 `34665`입니다.

그래서 다음과 같이 코드를 작성할 수 있습니다.

```python
exif_ifd = exif_data.get_ifd(34665)
date_str = exif_ifd.get(36867)  # 36867은 DateTimeOriginal 태그 ID
```

하지만 이런 코드는 권장하지 않습니다. `34665`, `36867` 처럼 의미를 알기 어려운 숫자를 직접 쓰는 것을 '매직 넘버(Magic Number)'라고 부르며 일반적으로는 피해야 할 습관입니다.

- **가독성 저하:** 해당 코드를 처음 보는 사람은 숫자들의 의미를 바로 알아보기 힘듭니다.

- **유지보수 어려움:** 태그 ID가 여러 곳에서 사용될 경우, 수정 시 모든 숫자를 일일이 찾아 바꿔야 합니다.

- **오류 발생 가능성:** 복잡한 숫자가 오타를 유발할 수 있고, 오타가 발생해도 인지하기 어렵습니다.

다행히 Pillow는 이 문제를 해결하기 위해 `PIL.ExifTags` 모듈을 제공합니다. 이 모듈의 `TAGS` 딕셔너리에는 `'ExifOffset': 34665` 와 같이 태그 이름과 태그 ID가 짝지어 저장되어 있으므로 더 이상 의미 없는 숫자를 외울 필요가 없습니다.

```python
from PIL import ExifTags

# ExifTags.TAGS는 {태그_ID: '태그_이름'} 형태로 구성되어 있습니다.
# 따라서 가독성을 위해 이름을 키로 사용하는 딕셔너리를 만들어 사용하면 편리합니다.
TAG_NAME_TO_ID = {name: tid for tid, name in ExifTags.TAGS.items()}

# 이제 태그 이름으로 ID를 쉽게 찾을 수 있습니다.
exif_offset_id = TAG_NAME_TO_ID.get('ExifOffset')
date_original_id = TAG_NAME_TO_ID.get('DateTimeOriginal')

print(f"'ExifOffset'의 태그 ID: {exif_offset_id}")
print(f"'DateTimeOriginal'의 태그 ID: {date_original_id}")

# 출력 결과
# 'ExifOffset'의 태그 ID: 34665
# 'DateTimeOriginal'의 태그 ID: 36867
```

---

## 최종 구현: 안정성과 가독성을 모두 잡은 코드

이제 위에서 다룬 내용을 종합하여, Pillow 공식 API와 `ExifTags` 모듈을 사용한 이상적인 코드를 쓸 수 있습니다. 아래 내용은 사진 파일의 원본 촬영일을 추출하는 함수를 구현한 코드입니다.

```python
from PIL import Image, ExifTags

def get_shooting_date(path: str) -> str:
    """
    Pillow의 공식 API와 ExifTags 모듈을 사용하여 사진 파일의 원본 촬영일을 추출합니다.
    가장 안정적이고 가독성이 높은 Best Practice 코드입니다.
    """
    try:
        with Image.open(path) as img:
            # 1. getexif()로 메인 IFD의 EXIF 데이터를 가져옵니다.
            exif_data = img.getexif()
            if not exif_data:
                return ""

            # 2. 가독성을 위해 태그 ID와 이름을 매핑하는 딕셔너리를 준비합니다.
            tag_name_to_id = {name: tid for tid, name in ExifTags.TAGS.items()}

            # 3. Exif IFD 포인터의 '태그 ID'를 이름으로 찾습니다.
            exif_ifd_pointer_id = tag_name_to_id.get('ExifOffset')
            if not exif_ifd_pointer_id:
                # ExifOffset 태그가 없으면 Exif IFD도 없다고 간주합니다.
                exif_ifd = {}
            else:
                # 4. get_ifd()에 포인터의 '태그 ID'를 전달하여 Exif IFD 데이터를 가져옵니다.
                exif_ifd = exif_data.get_ifd(exif_ifd_pointer_id)

            # 5. 우선순위에 따라 날짜 정보를 탐색합니다.
            #    (태그 이름, 탐색할 IFD - 0:Main, 1:Exif)
            tag_priority = [
                ('DateTimeOriginal', 1),    # 원본 촬영일 (우선도 1순위)
                ('DateTimeDigitized', 1),   # 디지털화된 날짜 (우선도 2순위)
                ('DateTime', 0)             # 수정일 (우선도 3순위)
            ]

            for tag_name, ifd_type in tag_priority:
                tag_id = tag_name_to_id.get(tag_name)
                if not tag_id:
                    continue

                # 탐색할 IFD를 선택합니다.
                ifd_to_search = exif_data if ifd_type == 0 else exif_ifd
                date_str = ifd_to_search.get(tag_id)

                if date_str:
                    # "YYYY:MM:DD HH:MM:SS" 형식에서 날짜 부분만 추출
                    date_part = date_str.split(" ")[0]
                    return date_part.replace(":", "-")

    except Exception:
        # 파일을 열 수 없거나 EXIF 처리 중 오류가 발생할 경우
        pass
    
    return ""

```

이 코드는 내부용 함수 `_getexif()`에 의존하지 않으면서, 매직 넘버를 제거하여 가독성과 유지보수성을 높이고 Pillow 라이브러리의 의도된 동작 방식을 정확히 따르는 안정적인 방법입니다.

---

> _이 글의 초안은 ChatGPT, Gemini 등 AI와 주고받은 질의응답 내용을 토대로 작성되었습니다._
{: .prompt-info }