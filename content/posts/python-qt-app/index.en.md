---
title: "I created a subtitle translation tool with PyQt"
date: 2022-05-01
translationKey: "posts/python-qt-app"
categories: 
  - python
image: "../../images/python.webp"
tags:
  - python
  - qt
---
A friend of mine recently started a Youtube channel. This is a channel that mainly distributes subtitles to drama making and interview videos that are uploaded to overseas YouTube channels, but since creating subtitles every time is quite a tedious task, I decided to create and provide it as a small command alias. The original video has subtitles in the original language, and it seems that the editing tool can read them, so I created a command to download not only the video but also the subtitles. With this, the translated subtitle file can be read and used with an editing tool, which reduces the amount of work considerably.

However, as the number of subscribers to the channel increased, some of them seemed to come from various countries, and a friend of mine asked me if I wanted a way to add multilingual subtitles using automatic translation. It seemed like it would be a good idea to create a simple tool, so I decided to create one and release it to the public.This time, I would like to talk about how I created that tool.

## Design

First, we listened to a friend's request and created the basic design of the technology stack and app. However, I didn't want to do anything too complicated, so it was just a mix of things I wanted to try based on what my friend wanted to do.

## Subtitles

When downloading YouTube videos using [youtube-dl](https://youtube-dl.org/) or [yt-dlp](https://github.com/yt-dlp/yt-dlp), if the original video has subtitles, you can remove the subtitles by adding an option. When you download subtitles from Youtube, the file format is often [WebVTT](https://developer.mozilla.org/en-US/docs/Web/API/WebVTT_API), so I thought it would be necessary to support files in the `.vtt` format. It seemed like it would take some time to support other formats, so I'll stick with this one first.

Fortunately, when I downloaded the actual subtitle file, the format was simple (almost the same as [SubRip](https://en.wikipedia.org/wiki/SubRip)), and I felt that I could probably replace the translation by taking the string on each line next to the subtitle timing together with that line's index. The actual file is created in the following format.

```vtt
WEBVTT

00:01.000 --> 00:04.000
Do not drink liquid nitrogen under any circumstances.

00:05.000 --> 00:09.000
- It will perforate your stomach.
- You could die.
```

## Language translation API

For translation, there is also Google Translate, but at the request of a friend, I decided to use [Papago](https://papago.naver.com/). Fortunately, the API was available for free, and Python sample code and details of the API were provided, so it didn't seem difficult to use. I decided to use Postman after trying out the API in advance.

## Technology

First, I decided to create it using Python. Even though I mainly use Kotlin, I would like to write this small-scale app in Python. JVM may be slow to start up, and since it only handles text and CPU processing is not critical, there is no need to consider performance. Since I had already created various CLI tools using Python to read files, edit files, etc., I thought I could quickly implement the most important function of ``reading, editing, and saving files.''

However, although I can use it on the CLI if I use it myself, I decided to implement a GUI because my friend uses it. If Python is passed as a `.py` file, it may be inconvenient for non-engineers to use it, such as requiring the installation of dependencies between Python and the script, so the artifact will be a binary that can be executed with [PyInstaler](https://pyinstaller.org/en/stable/) etc.

Regarding GUI, I have used [PySimpleGUI](https://pysimplegui.readthedocs.io/en/latest/), but this time I wanted to try using a different framework. Speaking of Python GUI frameworks, there are various ones such as [tkinter](https://docs.python.org/3/library/tkinter.html), [Kivy](https://kivy.org), [wxPython](https://www.wxpython.org/), and [Libavg](https://www.libavg.de/site/). However, I chose [PyQt](https://wiki.python.org/moin/PyQt) because [Qt](https://www.qt.io/) is used not only in Python but also in many other ways, so I thought there would be many examples that could be referenced not only in Python code but also in C++ code.

If you're creating something at a production level, you might want to consider different criteria, but if you're writing hobby-level code like this, it might be best to choose something that's as easy to write as possible and has a lot of samples. Efficiency is also important.

## Code

## File loading

Let's start by loading the file. The `.vtt` file has subtitle information (language code, etc.) written in the header, and from then on, the subtitle display time and subtitle string are repeated.It might be okay to simply read all the lines, but since there was a limit on the number of characters that could be used in a day when calling the translation API, I wanted to reduce the amount of data included in the request as much as possible. Therefore, I read the data separately into two parts: the original file data to be replaced later, and the data to be added to the API request parameters for translation. Here's the code.

```python
# get original contents from vtt file
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

The API parameters require the original language (source) and the language you want to translate (target), so store the language information in the subtitle header as a global variable in `source`, and store the index and string of the line you want to translate as a dictionary. Finally, it returns the original data and the data to be translated.

## Translation API call

This is the part of the extracted data that passes the data to be translated and calls the API. The API has a restriction of ``allowing 10 requests per second,'' so I sent the translated string line by line for translation, and waited 120ms for each request. Basically, I just do POST using [Requests](https://docs.python-requests.org/en/latest/), but if an error is returned, I display an alert and immediately terminate the app.

```python
# send translate request to papago and get translated message
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

            # wait for API's limitation(only 10 request per second allowed)
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

## Save file

This is where the logic ends. To save the translated subtitle file in the location of the original file, pass the path of the original file, the original data to replace with the translated one, and the resulting data translated by the API.

Basically, the translated data contains an index as a key, so we use that key to replace the original data. Here, the header contains the code of the original language, so replace it with the code of the translated language. The file name also contains the language code, so let's change that as well.

Once all data has been replaced and you have decided on a file name, save and exit.

```python
# write translated contents to file
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

## File dropdown

Since I am using a GUI, I wanted to be able to drag and drop a file to [QListWidget](https://doc.qt.io/qtforpython-5/PySide2/QtWidgets/QListWidget.html) and the file name would automatically be displayed on the screen, and it would also be available for translation. I'm not very familiar with Qt, so I used what I searched on the internet.Basically, when you drag and drop a file, if the extension is `.vtt`, it will be added to the list. You can get the full path of the file here, but only the file name is visible on the screen, and the full path is stored in a separate global variable.

```python
# drop down file list
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
                    # add file path to list
                    original_files.append(str(file_path))
                    # add file name to file list view
                    self.addItem(file_path.split('/')[-1])
        else:
            event.ignore()
```

## Delete file

I thought there might be times when I wanted to delete a file after adding it to the list, so I wanted to be able to select a file from the list and press a button to make it disappear from the list. This is achieved with the following code. When you click the button, the selected item in the list disappears, and it also disappears from the translation target (full path).

```python
# create remove button
def create_remove_button(self):
    button = QPushButton()
    button.setText('Remove file')
    button.clicked.connect(self.remove_file)
    return button


# remove file from file list
def remove_file(self):
    global original_files
    for selected_item in self.list_view.selectedIndexes():
        index = selected_item.row()
        self.list_view.takeItem(index)
        original_files.pop(index)
```

## Language selection

The language of the original subtitle file can be extracted from the header, but I also made it possible to choose the target translation language. Using the translation API, I store the translatable language codes in a dictionary. I chose a dictionary because the [QComboBox](https://doc.qt.io/qtforpython-5/PySide2/QtWidgets/QComboBox.html) item is visible on screen, so I wanted to display the actual language names. Since the selected item becomes the dictionary key, the corresponding value is the language to translate into.

```python
# languages
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

# create drop down menu
def create_target_language_selector(self):
    label = QLabel()
    label.setText('Select target language')
    selector = QComboBox()
    selector.addItems(supported_language_code.keys())
    selector.textActivated.connect(self.set_target_language)
    return label, selector

# set translate target language when dropdown menu selected
def set_target_language(self, selectd: str):
    global supported_language_code
    global target
    target = supported_language_code[selectd]
```

## Translation button

Finally, I added a translation button so that the actual translation will be done when the button is pressed. When you press the button, it reads data while looping through the path of the file to be translated, calls the translation API if the original language and the language you want to translate are different, and saves the results. As a precaution, we have also added a process to deactivate this button while the process is being performed.

```python
# create translate button
def create_translate_button(self):
    self.translate_button = QPushButton()
    self.translate_button.setText('Translate')
    self.translate_button.clicked.connect(self.translate_files)
    self.translate_button.setDisabled(False)


# read vtt file and do translate
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

These are the important parts of the tool I actually created, and you can check the entire source code [here](https://github.com/retheviper/PythonTools/blob/master/PapagoVtt/papago_vtt.py).

## I want to improve

In this way, I was able to achieve the functionality that I particularly wanted, but since I'm still not used to PyQt, there are still some areas in the logic that I would like to improve, so I've listed a few.

## Translate texts in bulk

According to the translation API specifications, requests are limited to 10 per second, so `sleep` waits 120 ms between requests. Considering the I/O involved, a shorter interval might have been better, but the real issue is that translation requests are sent line by line from the file data. Even short videos can have subtitles spanning several hundred lines, so the full translation can take quite a while. At first I thought about sending all translatable text at once by joining the dictionary values into a single string, but the API specification did not state that clearly, and there also seemed to be a request size limit. In that case, I could split the data into chunks, but then I would need to track which rows to replace when saving the file and how to handle that properly.

This process takes a considerable amount of time to translate a file, so I would like to get rid of it someday, but it seems like it will take some time until I can come up with a way to properly save the translated results.

## Add progress bar

With the current design, since it is single-threaded, there is a problem that the screen freezes while processing is being performed on a single file. Since the processing is slow to begin with, it freezes for quite a long time, which is not very good from a UX perspective. Also, since I don't know how far the processing has finished, I would like to add a progress bar to visualize how much processing is being done for a single file and for the entire list.

However, when adding a progress bar, you have to separate the threads and think about the logic of what to base the progress bar on, so that also seems to take some time.

## Button deactivation

I would like to deactivate the button before the file to be translated is added and during processing, but this is also not implemented properly yet. Perhaps because the screen freezes during processing, the process of deactivating the button before and after processing did not work as expected, so I would like to fix this as well.

## Finally

This time, we were able to take on a variety of new challenges, mainly with the GUI, so the work was quite interesting. Also, perhaps because it was originally server-side, there were issues that came up that I couldn't fully consider from a UI/UX perspective, and there was a lot of trial and error involved in implementing the screen, so this was a good learning experience for me. This was possible because Python and PyQt were excellent, but I'm also curious to see what would happen if I rewritten it with Jetpack Compose or something like that. After all, I feel that trying things that I don't normally try is a good way to grow as an engineer.

See you soon!
