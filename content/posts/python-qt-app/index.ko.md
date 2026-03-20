---
title: "PyQt로 자막 번역 도구 만들기"
date: 2022-05-01
categories:
  - python
image: "../../images/python.webp"
tags:
  - python
  - qt
translationKey: "posts/python-qt-app"
---

친구가 유튜브 채널을 시작했습니다. 해외 드라마 메이킹이나 인터뷰 영상에 자막을 붙여서 전달하는 채널인데, 매번 자막을 손으로 만드는 것은 꽤 번거로운 일이었습니다. 그래서 다운로드와 자막 처리 일부를 자동화하는 작은 도구를 만들어 주었습니다.

그런데 채널 구독자가 늘면서 다국어 자막이 필요해졌고, 자동 번역을 붙일 수 있는 도구도 있으면 좋겠다는 요청이 들어왔습니다. 그래서 이번에는 아예 자막 번역 GUI 도구를 만들어 보게 됐습니다.

## 디자인

먼저 요구 사항과 기술 스택을 정리했습니다. 복잡한 기능은 빼고, 실제로 필요한 흐름만 중심으로 설계했습니다.

### 자막

[youtube-dl](https://youtube-dl.org/)이나 [yt-dlp](https://github.com/yt-dlp/yt-dlp)로 내려받은 동영상에는 자막이 함께 포함될 수 있습니다. 자막 파일은 대부분 [WebVTT](https://developer.mozilla.org/ko/docs/Web/API/WebVTT_API) 형식이므로, 우선 `.vtt` 파일만 지원하기로 했습니다.

포맷은 단순합니다. 시간 정보와 자막 문자열이 반복되는 구조라서, 번역 대상인 문자열만 골라내면 충분했습니다.

```vtt
WEBVTT

00:01.000 --> 00:04.000
액체 질소를 절대 마시지 마세요.

00:05.000 --> 00:09.000
- 그것은 위에 구멍을 낼 수 있습니다.
- 죽을 수도 있습니다.
```

### 번역 API

번역에는 Google Translate 대신 [Papago](https://papago.naver.com/)를 사용했습니다. 무료 API가 있었고, Python 예제도 있어서 시작하기 쉬웠습니다.

### 기술

구현 언어는 Python으로 정했습니다. 이 정도 규모의 도구는 JVM 기반으로 만들기보다 Python이 더 빠르고 간단했습니다. 이미 파일 읽기, 편집, 저장 같은 CLI 도구를 몇 개 만들어 본 경험도 있어서 구현 방식이 익숙했습니다.

다만 실제로 사용할 사람은 개발자가 아니라 친구였기 때문에, CLI 대신 GUI를 선택했습니다. 배포도 `.py` 파일보다 실행 파일이 낫다고 판단해서 [PyInstaller](https://pyinstaller.org/en/stable/)로 바이너리 형태로 묶기로 했습니다.

GUI 프레임워크는 여러 가지가 있지만, 이번에는 Qt 계열을 써 보고 싶었습니다. 샘플도 많고 Python뿐 아니라 C++ 예제도 참고할 수 있어서 학습 자료가 풍부했습니다. 그래서 [PyQt](https://wiki.python.org) 기반으로 만들었습니다.

## 코드

### 파일 로드

먼저 `.vtt` 파일을 읽어 자막 문자열만 추출합니다. 원본 파일은 그대로 보관하고, 번역할 문자열만 따로 모아 둡니다.

```python
# vtt 파일에서 원본 자막을 읽는다
def get_content(self, file_path: str):
    exported_contents: dict[int, str] = {}
    global source
    with open(file_path, 'r') as file:
        original_contents: list[str] = file.readlines()
        for index, line in enumerate(original_contents):
            content = line.strip()
            if source == '' and 'Language:' in content:
                source = content.split(':')[1].strip()
            if '-->' not in content and content != '':
                exported_contents[index] = content
    return original_contents, exported_contents
```

원본 언어는 헤더에서 읽어 `source`에 넣고, 번역해야 할 문장은 인덱스와 함께 `dict`에 저장합니다. 최종적으로는 원본 데이터와 번역 대상 데이터를 둘 다 돌려줍니다.

### 번역 API 호출

추출한 문자열을 Papago에 하나씩 보내 번역합니다. API에 요청 제한이 있으므로 호출 간 잠깐씩 쉬어 주었습니다.

```python
# Papago에 번역을 요청하고 결과를 가져온다
def send_request(self, contents: dict[int, str]):
    translated_contents: dict[int, str] = {}

    headers = {
        'X-Naver-Client-Id': client_id,
        'X-Naver-Client-Secret': client_secret,
        'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
    }

    for index, content in contents.items():
        payload = {
            'text': content,
            'source': source,
            'target': target
        }

        response = requests.post(api_url, headers=headers, data=payload)

        if response.status_code == 200:
            body = response.json()
            translated_text: str = body['message']['result']['translatedText']
            translated_contents[index] = translated_text
            sleep(0.12)
        else:
            msgBox = QMessageBox().critical(
                self,
                'Error',
                response.json()['errorMessage'],
                buttons=QMessageBox.StandardButton.Abort
            )
            if msgBox == QMessageBox.StandardButton.Abort:
                sys.exit(255)

    return translated_contents
```

### 파일 저장

번역 결과를 원본 파일에 다시 덮어씌워 저장합니다. 인덱스를 기준으로 원문을 교체하고, 언어 코드도 함께 바꿉니다.

```python
# 번역한 내용을 파일로 저장한다
def write_result(self, file_path: str, original_contents: list[str], translated_contents: dict[int, str]):
    contents = original_contents.copy()

    for index, content in translated_contents.items():
        if 'Language:' in content:
            contents[index] = content.replace(source, target) + line_separator
        else:
            contents[index] = content + line_separator

    root = os.path.dirname(file_path)
    file_name = os.path.basename(file_path).replace(source, target)
    target_file_path = os.path.join(root, file_name)

    with open(target_file_path, 'w') as file:
        file.writelines(contents)
```

### 파일 드롭다운

파일을 드래그 앤 드롭으로 넣을 수 있게 `QListWidget`를 사용했습니다. 확장자가 `.vtt`인 파일만 목록에 추가하고, 화면에는 파일명만 보여 줍니다.

```python
# 파일 목록 위젯
class FileListView(QListWidget):
    def __init__(self, parent=None):
        super(FileListView, self).__init__(parent)
        self.setAcceptDrops(True)

    def dragEnterEvent(self, event):
        if event.mimeData().hasUrls:
            event.accept()
        else:
            event.ignore()

    def dragMoveEvent(self, event):
        if event.mimeData().hasUrls:
            event.accept()
        else:
            event.ignore()

    def dropEvent(self, event):
        if event.mimeData().hasUrls:
            event.accept()
            for url in event.mimeData().urls():
                file_path = url.toLocalFile()
                if file_path.endswith('.vtt') and str(file_path) not in original_files:
                    original_files.append(str(file_path))
                    self.addItem(file_path.split('/')[-1])
        else:
            event.ignore()
```

### 파일 삭제

목록에서 선택한 파일을 삭제할 수 있도록 버튼도 만들었습니다.

```python
# 삭제 버튼 생성
def create_remove_button(self):
    button = QPushButton()
    button.setText('Remove file')
    button.clicked.connect(self.remove_file)
    return button


# 목록에서 파일 삭제
def remove_file(self):
    global original_files
    for selected_item in self.list_view.selectedIndexes():
        index = selected_item.row()
        self.list_view.takeItem(index)
        original_files.pop(index)
```

### 언어 선택

번역 대상 언어는 드롭다운으로 선택하게 했습니다. 화면에는 사람이 읽기 쉬운 언어 이름을 보여 주고, 실제 요청에는 API가 요구하는 언어 코드를 넘깁니다.

```python
supported_language_code: dict[str, str] = {
    'Korean': 'ko',
    'English': 'en',
    'Japanese': 'ja',
    'Chinese(China)': 'zh-CN',
    'Chinese(Taiwan)': 'zh-TW',
    'Vietnamese': 'vi',
    'Indonesian': 'id',
    'Thailand': 'th',
    'German': 'de',
    'Russian': 'ru',
    'Spanish': 'es',
    'French': 'fr'
}

def create_target_language_selector(self):
    label = QLabel()
    label.setText('Select target language')
    selector = QComboBox()
    selector.addItems(supported_language_code.keys())
    selector.textActivated.connect(self.set_target_language)
    return label, selector

def set_target_language(self, selectd: str):
    global supported_language_code
    global target
    target = supported_language_code[selectd]
```

### 번역 버튼

마지막으로 번역 버튼을 추가했습니다. 버튼을 누르면 파일 목록을 순회하면서 번역을 수행하고, 작업 중에는 버튼을 잠깐 비활성화합니다.

```python
# 번역 버튼 생성
def create_translate_button(self):
    self.translate_button = QPushButton()
    self.translate_button.setText('Translate')
    self.translate_button.clicked.connect(self.translate_files)
    self.translate_button.setDisabled(False)


# vtt 파일을 읽어 번역한다
def translate_files(self):
    self.translate_button.setDisabled(True)
    for file_path in original_files:
        original_contents, exported_contents = self.get_content(file_path)
        if source == target:
            continue
        translated_contents = self.send_request(exported_contents)
        self.write_result(file_path, original_contents, translated_contents)
    self.translate_button.setDisabled(False)
```

실제 소스 코드는 [여기](https://github.com/retheviper/PythonTools/blob/master/PapagoVtt/papago_vtt.py)에서 볼 수 있습니다.

## 개선하고 싶다

기능은 원하는 대로 동작했지만, PyQt나 비동기 처리 쪽은 아직 부족한 점이 많았습니다. 그래서 다음과 같은 부분은 더 손보고 싶었습니다.

### 텍스트를 함께 번역

API 요청 제한 때문에 한 줄씩 번역하도록 만들었는데, 자막이 길어지면 처리 시간이 꽤 오래 걸립니다. 여러 줄을 묶어서 보내는 방법도 생각했지만, 마지막에 다시 원래 위치로 되돌려 놓는 로직이 복잡해져서 아직은 보류 중입니다.

### 진행률 표시줄 추가

지금 구조는 단일 스레드라서 처리 중 화면이 멈춘 것처럼 보입니다. 파일 단위나 전체 진행률을 보여 주는 Progress Bar를 추가하고 싶지만, 스레드와 진행률 계산 방식을 같이 설계해야 해서 아직 미뤄 두고 있습니다.

### 버튼 비활성화

번역 대상 파일이 없거나 처리 중일 때 버튼을 비활성화하고 싶었는데, 아직 깔끔하게 구현하지는 못했습니다. 이것도 다음에 손보고 싶은 부분입니다.

## 마지막으로

이번 프로젝트는 GUI를 처음부터 끝까지 직접 다뤄 볼 수 있어서 꽤 재미있었습니다. 서버 쪽 도구만 만들다가 UI/UX까지 신경 써야 하니 시행착오도 많았지만, 그만큼 배운 것도 많았습니다. 평소 잘 다루지 않던 영역을 건드려 보는 일이 개발자로서 성장하는 데 꽤 도움이 된다고 느꼈습니다.
