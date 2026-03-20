---
title: "Python으로 XML 파일 다루기 2"
date: 2019-06-20
categories:
  - python
image: "../../images/python.webp"
tags:
  - xml
  - python
translationKey: "posts/python-xml-modifier-2"
---

[이전 글](../python-xml-modifier-1)에서 xml 파일을 조작하는 스크립트를 소개했습니다. 그런데 업무에서 그 스크립트를 한동안 쓰지 않게 되어 그대로 두고 있었는데, 방침이 바뀌면서 저장해 둔 스크립트를 실제로 적용해 보니 문제가 하나 드러났습니다. 처음에는 스크립트가 문제라고는 생각하지 않아서 원인을 찾는 데 시간이 꽤 걸렸지만, 지금은 해결되어 다행입니다. 역시 코딩에서는 설계대로 구현되었는지보다, 이상한 동작이 없는지 확인하는 일이 더 중요하다는 걸 다시 느꼈습니다.

이번 글에서는 어떤 문제가 있었고, 어떻게 코드를 고쳤는지 정리해 보겠습니다. 엄밀히 말하면 버그라기보다 상세 설계 단계에서 실수가 있었다고 보는 편이 맞을지도 모릅니다. 어쨌든 처음 계획대로 움직이지 않는 코드를 쓴 건 반성해야 했고, 그 덕분에 다시 배웠다고 볼 수 있습니다.

## 이전 스크립트의 문제

이전 글에서 소개한 스크립트는 얼핏 보면 잘 동작하는 것처럼 보였습니다. 실제 테스트에서도 지정한 엘리먼트의 텍스트는 제대로 바꿔 주었습니다. 하지만 처리해야 할 xml 파일의 엘리먼트 수가 예상보다 많았던 것이 문제였습니다. 이전 스크립트는 SELECT를 발행하는 DB 엘리먼트 하나, INSERT를 발행하는 DB 엘리먼트 하나라는 단순한 구성을 전제로 했지만, 이번에는 여러 엘리먼트를 처리해야 했습니다.

이전 스크립트로 처리하려고 하면, 하나의 엘리먼트를 수정한 뒤 나머지는 무시하고 다음 파일로 넘어가 버렸습니다. 그걸 모른 채 처리가 끝났다고 믿고 결과 파일을 쓰면 에러가 났습니다. 이게 스크립트를 고치게 된 계기였습니다.

원인을 알았으니 목표를 수정합니다. 이번 목표는 "조건과 일치하는 모든 엘리먼트 수정"입니다.

## 코드 수정

목표에 맞춰 코드를 고치면서, 자잘한 문제도 함께 개선하기로 했습니다. 이전 스크립트에서는 폴더 안 파일 목록을 얻기 위해 `glob` 모듈을 썼습니다. 적은 코드로 하위 폴더까지 재귀적으로 모아 주니 편리했지만, `glob`의 `recursive` 옵션은 Python 3.5 이상에서만 사용할 수 있다는 문제가 있습니다. 평소 Python 3만 썼다면 괜찮겠지만, 다른 스크립트는 모두 Python 2 기준으로 작성돼 있었습니다. 그래서 이번에는 Python 2에서도 쓸 수 있는 방식으로 바꾸기로 했습니다.

핵심 개선점은 `find`를 `findall`로 바꾸는 것이었습니다. 우선 DB connection 엘리먼트를 모두 가져와서, `if` 문으로 DB connection 이름에 따라 분기하면 됩니다. 또 바꾸고 싶은 DB connection 이름은 `From_PostgreSQL_01`처럼 언더스코어 뒤에 연속 번호가 붙어 있는 형태인데, 이것을 `From_PostgreSQL`처럼 번호만 제거하려는 것이었습니다. `replace`를 쓰는 방법도 있지만, 그렇게 하면 모든 경우를 조건으로 나눠야 하고 중복 가능성도 있습니다. 그래서 `rsplit`[^1]을 써서 언더스코어 기준으로 문자열을 나눈 뒤, 앞부분만 남기기로 했습니다.

이 요구사항에 맞춰 바뀐 코드는 다음과 같습니다.

```python
# -*- coding: UTF-8 -*-

import xml.etree.ElementTree as ET
import os

# 네임스페이스(prefix)를 맵으로 선언
ns = {'fb': 'http://builder',
      'fe': 'http://engine',
      'mp': 'http://mapper'}

# xml 파일명을 재귀적으로 가져온다(Python 2용)
fileList = []
base_dir = os.path.normpath('./baseFolder') # 검색할 디렉터리의 시작점

for (path, dir, files) in os.walk(base_dir):
    # xml2 파일은 삭제하고, xml 파일만 다시 쓰고 싶으므로 분기 처리한다
    for fname in files:
        if ('.xml2' in fname):
            fullfname = path + "/" + fname
            os.remove(fullfname)
        elif ('.xml' in fname and not '.xml2' in fname):
            fullfname = path + "/" + fname
            fileList.append(fullfname)

# 가져온 파일을 순회하면서 Connection 이름을 수정한다
for fileName in fileList:
    # 파일 파싱 시작
    tree = ET.parse(fileName)

    # INSERT 컴포넌트의 Connection 이름에 '_01' 같은 문자열이 붙어 있으면 제거한다
    PutConnections = tree.findall("fe:Flow/fe:Component[@type='RDB(Put)']/fe:Property[@name='Connection']", ns)
    for PutConnection in PutConnections:
        if ('_0' in PutConnection.text):
            PutConnection.text = PutConnection.text.rsplit('_', 1)[0] # rsplit로 나눈 결과를 원래 텍스트에 넣는다

    # SELECT 컴포넌트의 Connection 이름에 '_01' 같은 문자열이 붙어 있으면 제거한다
    GetConnections = tree.findall("fe:Flow/fe:Component[@type='RDB(Get)']/fe:Property[@name='Connection']", ns)
    for GetConnection in GetConnections:
        if ('_0' in GetConnection.text):
            GetConnection.text = GetConnection.text.rsplit('_', 1)[0] # rsplit로 나눈 결과를 원래 텍스트에 넣는다

    # prefix가 바뀌는 것을 방지한다
    ET.register_namespace('fb', 'http://builder')
    ET.register_namespace('fe', 'http://engine')
    ET.register_namespace('mp', 'http://mapper')

    # 수정 내용을 저장한다
    tree.write(fileName, 'UTF-8', True)
```

## 마지막으로

파일을 가져오는 부분은 `glob`를 쓸 때보다 조금 복잡해졌습니다. `os.walk()`로 시작 디렉터리를 지정해 경로와 파일 이름을 가져오고, 파일명과 경로가 분리되므로 이를 다시 합치는 작업이 필요합니다. 그 뒤 처리할 파일 목록에 넣고 빼는 과정을 추가하면 이전 `glob`와 비슷하게 동작합니다.

그리고 `rsplit('_', 1)`을 쓰면 `From_PostgreSQL_01` 같은 문자열이 `From_PostgreSQL`과 `01`로 한 번만 나뉩니다. 나뉜 값은 배열이 되므로 `[0]`을 지정하면 `From_PostgreSQL_01`이 `From_PostgreSQL`로 바뀝니다. 또 `find`를 `findall`로 바꾸는 것만으로 조건에 맞는 모든 요소를 파일에서 찾아 리스트로 만들어 줍니다. 그때는 루프만 돌리면 됩니다. 이걸 몰랐을 때는 이전 코드에 루프를 더 얹어 보느라 실패했는데, 생각보다 간단한 해답이 있었습니다.

이렇게 고친 코드는 의도대로 잘 동작했습니다. 나중에 변경이 생겨도 일부만 손보면 되니 유지보수 측면에서도 훨씬 낫습니다. 더 깔끔한 작성법이 있을 수는 있겠지만, 적어도 처음 버전보다 범용성과 안정성은 분명히 좋아졌습니다.

결국 이번 수정에서 가장 크게 배운 점은 테스트의 중요성이었습니다. "한 번 돌아간다"는 것과 "실제 입력에서도 안전하게 동작한다"는 것은 전혀 다른 이야기라는 점을 다시 확인했습니다.

[^1]: `rsplit`과 `split`의 차이는 방향입니다. `rsplit`은 문자열의 오른쪽부터 나누고, `split`은 왼쪽부터 나눕니다. 이번에는 문자열 끝의 연속 번호를 취하고 싶어서 `rsplit`을 선택했습니다.
