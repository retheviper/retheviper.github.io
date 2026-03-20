---
title: "Python으로 XML 파일 다루기 1"
date: 2019-06-09
categories:
  - python
image: "../../images/python.webp"
tags:
  - xml
  - python
translationKey: "posts/python-xml-modifier-1"
---

IT 관련 일을 하면서 가장 좋다고 느끼는 점은 여러 상황에 놓이며 얻는 경험이 많다는 것입니다. 독학만으로는 언어의 기초 문법은 알더라도 실제 코딩에서 어떻게 설계해야 하는지, 어떤 모듈(라이브러리 포함)을 써야 하는지 모르는 경우가 많기 때문입니다. 무엇을 만들어야 할지도 막막한 경우가 많죠. 웹 애플리케이션인지, 배치로 도는 코드인지 같은 큰 분류부터, 어떤 DB를 쓰고 어떤 작업을 하고 싶은지 같은 세부 사항까지 혼자 전부 상정하기는 꽤 어렵습니다. 하지만 업무에서는 어느 정도 요구사항이 정해져 있으니, 연결 방법과 필요한 작업만 알면 나머지는 해내면 됩니다. 목표 설정이 무엇보다 중요하다는 말이 딱 맞는 상황입니다.

이번에도 업무로 맡게 된 작업이었습니다. 요구사항은 이렇습니다. 어떤 도구를 사용해 개발하고 있는데, 이 도구는 GUI에서 맵 위에 아이콘을 배치하고, 그 아이콘 하나하나가 작업 단위가 됩니다. 그리고 아이콘끼리 연결하고 각 아이콘에 동작을 설정하면 전체 워크플로가 완성되는 구조입니다. 예를 들어 DB 아이콘에 접속 정보와 실행할 SQL을 넣고 파일 출력 아이콘에 연결하면, 실행 시 DB 결과가 파일로 저장됩니다. 이 도구로 만든 워크플로는 xml 파일로 저장됩니다.

문제는 개발 환경과 현업 환경의 설정이 다르다는 점입니다. 주로 이 도구로 하는 작업은 DB 관련이었는데, 개발 환경과 현업 환경의 DB 접속 정보가 서로 달랐습니다. 그리고 그 접속 정보는 xml 파일에 저장되어 있습니다. 그래서 개발이 끝난 xml 파일을 현업 환경에 배포할 때는 단순 복사만으로는 부족했고, `DB 접속 정보 재작성` 작업이 필요했습니다. 이걸 어떻게 구현했는지가 이번 글의 주제입니다.

## 사전 준비

배포 대상과 배포 원본 서버는 모두 Linux를 사용합니다. 그리고 xml 파일의 소스는 Git으로 관리합니다. 그러니 먼저 배포 원본에서 Git pull을 하고, DB 정보를 다시 쓴 뒤 rsync 같은 명령으로 복사하면 됩니다. 이 작업은 Jenkins로 자동화하기로 했습니다. 그렇다면 남은 문제는 DB 정보를 다시 쓰는 작업을 어떻게 구현하느냐입니다.

xml을 분석해 보니 각 아이콘의 태그 아래에 해당 아이콘의 상세 설정이 있었습니다. DB 처리를 하는 아이콘은 두 개였는데, 하나는 SELECT를 발행하는 것(이하 From), 다른 하나는 INSERT를 발행하는 것(이하 To)이었습니다. From과 To는 접속 대상 DB의 종류도 다르고, 스키마와 테이블명도 다르기 때문에 분리해서 처리해야 했습니다.

환경을 생각하면 Linux에서 쓸 수 있는 언어를 고르는 것이 좋습니다. 처음에는 셸 스크립트로 해 보려고 했습니다. 이게 제 첫 셸 스크립트였습니다. Java보다 쉽지 않을까 하는 근거 없는 자신감으로 셸을 골랐습니다. Linux에는 Python도 들어 있었지만 그쪽도 만져 본 적이 없어서, 일단 셸로 해 보고 안 되면 Python을 도전해 보자는 정도로 생각했습니다. 결과적으로는 처음부터 Python으로 썼으면 더 좋았겠다는 결론이 나왔지만요.

어쨌든 목표와 환경, 도구가 갖춰졌으니 바로 구현에 들어갔습니다.

## 셸 스크립트로 코딩

셸은 처음이라 시행착오가 많았습니다. 처음 배운 언어가 Java라서 같은 감각으로 쓰려고 하면 전혀 동작하지 않았습니다. 여러 번 실패하며 얻은 결론은, 함수를 쓴다는 생각을 버리고 명령을 어떻게 조합할지가 더 중요하다는 것이었습니다. 그걸 깨닫는 데 시간이 꽤 걸렸지만, 중요한 점은 알았으니 이제 실제로 무엇을 할지 구현하면 됐습니다.

먼저 파일을 읽어야 했습니다. xml 파일도 결국 텍스트 기반이니 셸로 읽을 수 있습니다. `for` 문 하나로 특정 확장자를 가진 파일을 순회하면서 한 줄씩 읽을 수 있었습니다. 그리고 기존 아이콘(툴 용어로는 컴포넌트)의 DB 접속 정보가 들어 있는 행을 찾은 뒤 다시 쓰면 됩니다.

다만 앞서 말한 대로 From과 To 컴포넌트를 구분해야 했습니다. xml을 들여다보니 컴포넌트 구조, 즉 태그 종류가 거의 같아 보였습니다. 셸에서는 xml을 제대로 파싱할 모듈도 없는 것 같아서, 우선 행 번호를 비교해 From 컴포넌트가 더 위에 있으면 첫 번째로 잡힌 DB 설정이 From이라고 판단하기로 했습니다. 다음은 구현 코드입니다.

## 코드 예제(셸 스크립트)

```bash
#!/bin/bash
# 아래 폴더를 순회하면서 xml 확장자를 가진 파일을 변수 fileName에 넣는다
for fileName in `\find . -name '*.xml'`; do
    # 컴포넌트의 행 번호를 찾는다(grep으로 컴포넌트 태그를 확인하고 sed로 행 번호만 추출)
    getComponentLine=$(grep -n RDBGet ${fileName} | grep Component | sed -e 's/:.*//g')
    putComponentLine=$(grep -n RDBPut ${fileName} | grep Component | sed -e 's/:.*//g')

    # Connection의 행 번호를 찾는다
    ConnectionOneLine=$(grep -n Connection ${fileName} | sed -e 's/:.*//g' | awk 'NR==1')
    ConnectionTwoLine=$(grep -n Connection ${fileName} | sed -e 's/:.*//g' | awk 'NR==2')

    # Connection 이름을 찾는다(cut으로 DB 설정명만 뽑고, awk로 여러 결과 중 하나를 고른다)
    ConnectionOneName=$(grep Connection ${fileName} | cut -d ">" -f 2 | cut -d "<" -f 1 | awk 'NR==1')
    ConnectionTwoName=$(grep Connection ${fileName} | cut -d ">" -f 2 | cut -d "<" -f 1 | awk 'NR==2')

    if [ "${getComponentLine}" -lt "${putComponentLine}" ]; then
        # get < put이면 ConnectionOneLine이 get의 Connection
        sed -i -e "${ConnectionOneLine} s/${ConnectionOneName}/HONBADBFROM/g" ${fileName}
        sed -i -e "${ConnectionTwoLine} s/${ConnectionTwoName}/HONBADBTO/g" ${fileName}
    else
        # get > put이면 ConnectionOneLine이 put의 Connection
        sed -i -e "${ConnectionTwoLine} s/${ConnectionTwoName}/HONBADBFROM/g" ${fileName}
        sed -i -e "${ConnectionOneLine} s/${ConnectionOneName}/HONBADBTO/g" ${fileName}
    fi
done
```

## 문제점

xml 형식의 파일을 태그 파싱 없이 대충 나눠 처리하는 현재 방식은 안전하다고 보기 어렵습니다. 그리고 이 방식이라면 From과 To 컴포넌트가 각각 하나씩 있을 때는 괜찮겠지만, 어느 한쪽이 하나라도 늘어나면 처리 방식을 바꿔야 합니다. 경우에 따라서는 그 케이스마다 다른 코드를 작성해야 할 수도 있습니다. 그러면 파일을 하나씩 확인해서 그에 맞는 코드를 대응해야 하니, 손으로 바꾸는 것과 별로 다르지 않게 됩니다.

결국 범용성도 없고 안전하지도 않은 코드가 되어 버렸습니다. 이런 코드는 현업에서는 쓸 수 없습니다. 그래서 방법을 바꾸기로 했습니다.

## Python으로 다시 작성

다음 방법으로는 Python을 사용해 제대로 파싱하기로 했습니다. 직접 해 보니 이런 간단한 작업에는 Python이 정답이라는 생각이 들 정도로 쉬웠습니다. Linux 환경에서는 기본적으로 Python이 들어 있는 경우도 많고(`yum`이 Python을 사용하는 대표적인 예입니다), 따로 설치하지 않아도 된다는 점도 장점입니다. Bash가 Linux의 기본 기능이니 Python보다 빠를 거라 생각했지만 꼭 그렇지도 않은 것 같았습니다. 그러니 셸 스크립트에 굳이 집착할 이유도 없어졌습니다.

다만 Linux에 기본으로 들어 있는 Python은 2인 경우가 많습니다. 확인해 보니 제가 쓰는 Mac에도 Python 2가 들어 있었습니다. Python은 2와 3 사이에 문법 차이가 꽤 있어서 특정 기능을 쓸 때 주의가 필요합니다. 실제로 제가 작성한 코드에는 Python 3에서만 쓸 수 있는 부분이 있습니다. `alternative` 같은 명령으로 Python 링크를 3으로 바꾸는 방법도 있지만, 그러면 Python 2를 사용하는 프로그램에 문제가 생길 수 있습니다.[^1] 그래서 처음부터 Python 2에 맞게 쓸지, Python 3로 실행하도록 할지, Python 2를 쓰는 다른 프로그램의 실행 환경을 바꿀지 등을 고민해야 했습니다.

저는 작성한 코드를 Python 3에서 실행하도록 했습니다(Jenkins에 넣기 때문에 그 설정에서 맞췄습니다). 코드는 다음과 같습니다.

## 코드 예제(Python)

```python
# -*- coding: UTF-8 -*-
# 일본어 주석을 쓰기 위해 먼저 인코딩을 지정한다

# xml 파서와 폴더에서 파일을 가져오는 모듈을 import
import xml.etree.ElementTree as ET
import glob

# 네임스페이스(prefix)를 맵으로 선언
ns = {'fb':'http://foo.com/builder', 'fe':'http://foo.com/engine', 'mp':'http://foo.com/mapper'}

# 파일명을 재귀적으로 가져온다(recursive 옵션은 Python 3 전용)
fileList = glob.glob("**/HOGE*.xml", recursive=True)

# 가져온 파일을 순회하면서 Connection 이름을 수정한다
for fileName in fileList:
    # 파일 파싱 시작
    tree = ET.parse(fileName)

    # To 컴포넌트에서 자식 요소인 Connection 이름을 가져온다(prefix 안)
    putCon = tree.find("fe:WorkFlow/fe:Component[@type='RDB(Put)']/fe:Property[@name='Connection']", ns)
    putCon.text = putCon.text + 'x'
 
    # From 컴포넌트에서 자식 요소인 Connection 이름을 가져온다(prefix 안)
    getCon = tree.find("fe:WorkFlow/fe:Component[@type='RDB(Get)']/fe:Property[@name='Connection']", ns)
    getCon.text = getCon.text + 'y'

    # 저장 시 prefix가 바뀌는 것을 방지한다
    ET.register_namespace('fb', 'http://foo.com/builder')
    ET.register_namespace('fe', 'http://foo.com/engine')
    ET.register_namespace('mp', 'http://foo.com/mapper')

    # 수정 내용을 저장한다
    tree.write(fileName, 'UTF-8', True)
```

## 마지막으로

코드 길이도 길지 않고, 파싱으로 필요한 요소를 제대로 집어낼 수 있어서 셸 스크립트보다 훨씬 안전한 방식이 되었습니다. 태그로 컴포넌트를 구분하므로 컴포넌트 수가 바뀌어도 비교적 안정적으로 대응할 수 있다는 점도 장점입니다. 태그에 네임스페이스가 있어서 초기 설정과 저장 직전에 추가 처리가 필요하다는 점은 조금 번거롭지만[^2], 그래도 셸 스크립트보다 유지보수하기 쉬운 코드가 되었습니다.

무엇보다 메인 함수나 클래스를 굳이 만들지 않아도 스크립트처럼 작성한 코드가 의도대로 잘 동작한다는 점이 좋았습니다. Linux 환경에서 간단한 반복 작업을 자동화해야 한다면 Python은 충분히 좋은 선택지라고 생각합니다. 다음 글에서는 이 스크립트를 실제로 적용하면서 드러난 문제와 수정 내용을 이어서 정리하겠습니다.

[^1]: 예를 들어 `yum`에서 문제가 발생했습니다. `yum`의 실행 환경을 바꾸는 방법(`/usr/bin/yum` 설정 참고)도 있지만, 어떤 프로그램이 Python 2를 쓰는지 일일이 확인하기 어려워서 추천하고 싶지는 않습니다.
[^2]: 특히 마지막 `ET.register_namespace()`가 없으면 네임스페이스가 마음대로 바뀝니다.
