---
title: "Python으로 DB 다루기"
date: 2019-06-30
categories:
  - python
image: "../../images/python.webp"
tags:
  - oracle
  - postgresql
  - blob
  - python
translationKey: "posts/python-blob-to-binary"
---

개인적으로 DB 작업은 늘 어렵게 느껴졌습니다. 새로운 프로그램을 만드는 일보다 DB와 맞물리는 작업이 더 자주 따라왔고, 이번에도 Python 스크립트의 핵심은 DB 연동이었습니다. 그래서 여기서는 Python에서 DB에 연결하고, SQL을 실행하고, 그 결과를 어떻게 다뤘는지 정리해 보겠습니다.

이번에 맡은 일은 DB에서 바이너리 데이터를 꺼내 AWS S3[^1]에 업로드하는 스크립트를 만드는 것이었습니다. 업로드가 끝나면 다른 테이블도 업데이트해야 했는데, 추출 대상 DB와 업데이트 대상 DB가 각각 달랐습니다. 한쪽은 Oracle이고 다른 한쪽은 PostgreSQL이었습니다. 여기에 로깅, 외부 설정 파일 읽기, 최종 갱신일 기준 증분 처리 같은 조건도 붙어 있었습니다.

정리하면 스크립트의 흐름은 다음과 같았습니다.

1. 명령줄 인수를 받는다.
2. PostgreSQL의 테이블에서 해당 작업의 마지막 처리 시점을 읽는다.
3. Oracle 테이블에서 그 이후의 바이너리 데이터를 조회해 JPG로 저장한다.
4. PostgreSQL의 다른 테이블에서 파일 저장 경로를 구한다.
5. S3에 이미지 파일을 업로드한다.
6. 업로드 결과를 PostgreSQL 테이블에 기록한다.
7. 최종 처리 시점을 갱신한다.
8. 성공 시 `0`, 실패 시 `9`로 종료한다.

## 파이썬으로 DB 연동

Python에서 DB에 연결하는 일은 생각보다 어렵지 않았습니다. Oracle과 PostgreSQL에서 사용하는 모듈은 다르지만, 기본 흐름은 비슷합니다.

Oracle은 `cx_Oracle`을 사용했습니다. 연결에 필요한 값은 호스트 이름, 포트, 서비스 이름, 사용자 이름, 비밀번호입니다.

```python
import cx_Oracle

tns = cx_Oracle.makedsn('호스트명', '포트', '서비스명')
connect = cx_Oracle.connect('사용자명', '비밀번호', tns)
```

PostgreSQL은 `psycopg2`를 사용했습니다.

```python
import psycopg2

connect = psycopg2.connect(
    'host=호스트명 port=포트 dbname=DB명 user=사용자명 password=비밀번호'
)
```

둘 다 `pip`로 설치할 수 있지만, `psycopg2`는 시스템 라이브러리 의존성이 있어서 Amazon Linux에서는 `postgresql-devel` 같은 패키지가 필요했습니다.

DB 작업의 공통 흐름은 다음과 같습니다.

```python
# 실제 DB에 연결해 커서 가져오기
cursor = connect.cursor()

# SQL 실행
cursor.execute('실행할 SQL')

# 결과 가져오기
result = cursor.fetchone()   # 한 건
result = cursor.fetchmany(1000)  # 여러 건
result = cursor.fetchall()   # 전부

# 정리
cursor.close()
connect.commit()
connect.close()
```

조회 결과는 보통 행 단위의 리스트로 받아서 처리합니다.

```python
result = cursor.fetchmany(1000)

for row in result:
    print(row[0])
    print(row)
```

## 이미지 파일을 S3에 업로드하는 코드

전체 스크립트는 결국 다음 세 단계로 나눌 수 있었습니다.

### 1. 마지막 처리 시점 읽기

먼저 PostgreSQL에서 작업 코드에 해당하는 마지막 처리 시점을 읽습니다.

```python
def GetProcdate(args):
    global function_code
    function_code = args[1]

    try:
        print('>> Starting Job. The function Code is: ' + function_code)
        connect = psycopg2.connect(
            'host=' + HOST_POST +
            ' port=' + PORT_POST +
            ' dbname=' + DB_NAME_POST +
            ' user=' + USER_POST +
            ' password=' + PWD_POST
        )
        cursor = connect.cursor()
    except:
        print('>> Unable to connect PostgreSQL. quitting.')
        sys.exit(9)

    selectSQL = "SELECT last_date FROM date_table WHERE function_code='%s'" % (function_code,)
    cursor.execute(selectSQL)
    result = cursor.fetchone()

    if result is not None:
        last_date = result[0]
    else:
        print('>> No data matches with ' + function_code + '. quitting.')
        sys.exit(1)

    cursor.close()
    connect.close()
    GetImageFromTable(last_date)
```

### 2. Oracle에서 이미지 데이터 읽기

Oracle에서는 마지막 처리 시점 이후의 데이터를 읽어 JPG 파일로 저장합니다.

```python
def GetImageFromTable(last_date):
    global imageFolder
    try:
        tns = cx_Oracle.makedsn(HOST_ORAC, PORT_ORAC, SERVICE_NAME)
        connect = cx_Oracle.connect(USER_ORAC, PWD_ORAC, tns)
        cursor = connect.cursor()
    except:
        print('>> Unable to connect Oracle DB. quitting.')
        sys.exit(9)

    cursor.execute(
        "SELECT image_data, image_name FROM image_table WHERE updated_at >= :last_date",
        {'last_date': last_date}
    )

    if os.path.exists(imageFolder):
        shutil.rmtree(imageFolder)
    os.mkdir(imageFolder)

    while True:
        rows = cursor.fetchmany(1000)
        if len(rows) == 0:
            break

        for image in rows:
            file_name = imageFolder + '/' + str(image[1]) + '.jpg'
            with open(file_name, 'wb+') as imageFile:
                imageFile.write(image[0].read())

    fileCount = len([
        name for name in os.listdir(imageFolder)
        if os.path.isfile(os.path.join(imageFolder, name))
    ])
    print('>> As result: ' + str(fileCount) + ' files were written.')
    cursor.close()
    connect.close()
    FileUploadToS3()
```

### 3. S3 업로드

저장한 파일을 기준으로 PostgreSQL에서 경로 정보를 읽고, S3에 업로드합니다.

```python
def FileUploadToS3():
    global imageFolder
    global function_code

    files = os.listdir(imageFolder)
    item_cd_list = [os.path.splitext(f)[0] for f in files if os.path.isfile(os.path.join(imageFolder, f))]

    try:
        connect = psycopg2.connect(
            'host=' + HOST_POST +
            ' port=' + PORT_POST +
            ' dbname=' + DB_NAME_POST +
            ' user=' + USER_POST +
            ' password=' + PWD_POST
        )
        cursor = connect.cursor()
    except:
        print('>> Unable to connect PostgreSQL. quitting.')
        sys.exit(9)

    fileCount = 0

    for item_cd in item_cd_list:
        selectSQL1 = "SELECT file_path1, file_path2, file_path3 FROM item_table WHERE item_cd='%s'" % (str(item_cd),)
        cursor.execute(selectSQL1)
        resultMaster = cursor.fetchone()

        if resultMaster is None:
            print('>> Result was 0. continue job.')
            continue

        image_id = str(resultMaster[2])
        path = str(resultMaster[0]) + '/' + str(resultMaster[1]) + '/' + str(resultMaster[2]) + '/'

        selectSQL2 = "SELECT * FROM image_table WHERE image_id='%s';" % (image_id,)
        cursor.execute(selectSQL2)
        result = cursor.fetchone()
        uploadToS3(item_cd, path)
        fileCount = fileCount + 1
        print('>> Processing file upload.')

    updateProcSQL = "UPDATE date_table SET last_date = CURRENT_TIMESTAMP WHERE function_code='%s'" % (function_code,)
    cursor.execute(updateProcSQL)
    print('>> As result: ' + str(fileCount) + ' files were uploaded.')
```

실제 업로드 함수와 예외 처리, 환경 설정 읽기 부분은 생략했지만 흐름은 위와 같습니다. 핵심은 DB 두 개를 오가면서 증분 처리하고, 파일을 저장한 다음 S3에 올리는 구조였습니다.

## 마지막으로

이번 작업으로 Python에서 DB를 다루는 방식과, 여러 DB를 오가며 데이터를 옮기는 기본 흐름을 조금 더 익힐 수 있었습니다. Oracle과 PostgreSQL이 섞인 구조는 처음에는 번거로웠지만, 흐름을 단계별로 나누고 나니 생각보다 정리가 잘 됐습니다.

DB와 연동하는 스크립트는 손이 많이 가지만, 한 번 구조를 잡아 두면 이후 작업에도 재사용하기 좋습니다. 특히 증분 처리, 파일 저장, 업로드, 상태 갱신처럼 반복되는 패턴은 다음 작업에서도 그대로 응용할 수 있을 것 같습니다.

[^1]: AWS S3는 객체 스토리지 서비스입니다. 바이너리 파일을 보관하고 배포하는 용도로 쓰기 좋습니다.
