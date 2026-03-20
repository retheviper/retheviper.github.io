---
title: "Jenkins Pipeline 시작하기"
date: 2020-01-26
categories:
  - jenkins
image: "../../images/jenkins.webp"
tags:
  - groovy
  - ci/cd
  - jenkins
translationKey: "posts/jenkins-pipeline"
---

앞서 소개한 Jenkins Job 생성 방식은 비교적 전통적인 형태에 가깝습니다. Jenkins 2.0 이후에는 스크립트로 Job을 정의할 수 있는 Pipeline 방식이 본격적으로 자리 잡았습니다.

Pipeline에서는 `groovy`를 사용해 실행하고 싶은 명령이나 작업을 `stage` 단위로 작성합니다. 작성한 스크립트는 위에서 아래로 순차 실행되며, 각 단계별 결과를 화면에 보여 줍니다. 기존 Jenkins Job과 사용 방식이 꽤 다르기 때문에, 이번에는 Pipeline Job을 어떻게 작성하는지 샘플과 함께 정리해 보겠습니다.

## Pipeline의 장점

먼저 일반 Job과 비교했을 때 어떤 장점이 있는지 보고 싶습니다. 기존 Freestyle Job이 아니라 Pipeline으로 Jenkins Job을 만들면 다음과 같은 이점이 있습니다.

- 스크립트라서 관리가 쉽습니다.
  - 파일로도 관리할 수 있으니 Git으로 버전 관리가 가능합니다.
- 만들기 쉽습니다.
  - Snippet 기능이 있어서 스크립트를 빠르게 작성할 수 있습니다.
- 성공과 실패 이력을 보기 쉽습니다.
  - 단계별로 실행되므로 어느 `stage`에서 성공하거나 실패했는지 바로 확인할 수 있습니다.

Pipeline Job의 stage 실행 기록은 GUI에서 확인할 수 있고, 실행 로그도 보기 좋게 제공합니다. 화면은 대략 다음과 같습니다.

![Jenkins Pipeline Stage View](jenkins_pipeline_stage_view.webp)

## Pipeline 작성하기

### Pipeline Job 만들기

먼저 Pipeline Job을 만드는 과정을 간단히 보겠습니다. Job 생성 화면에서 이름을 입력하고 Pipeline을 선택합니다.

![Jenkins Create Pipeline](jenkins_create_pipeline.webp)

### Pipeline 스크립트

Pipeline으로 Job을 만들면, 실행할 항목을 지정하는 화면도 Freestyle Job과는 조금 다릅니다. 빌드 트리거 같은 설정은 비슷하지만, 아래로 내려가 보면 `Pipeline` 탭이 있습니다. 여기서 직접 스크립트를 작성할지, Git 등으로 관리하는 스크립트 파일을 지정할지 선택할 수 있습니다.

![Jenkins Pipeline Script](jenkins_pipeline_script1.webp)

처음부터 스크립트를 쓰는 것은 쉽지 않습니다. 우선 화면 오른쪽의 `try sample Pipeline...`을 눌러 보겠습니다. 먼저 `Hello world`를 선택해 보죠.

![Jenkins Pipeline Script 2](jenkins_pipeline_script2.webp)

Pipeline 스크립트는 `groovy`를 사용하지만, 처음부터 문법을 전부 외울 필요는 없습니다. 샘플 코드도 여러 개 있으니, 그걸 참고하면서 감을 잡으면 됩니다.

또 Jenkins는 Pipeline에 익숙하지 않은 사람을 위해 Snippet 생성 기능도 제공합니다. 실행할 작업을 드롭다운에서 고르고 필요한 파라미터를 입력하면 스크립트를 자동으로 만들어 주는 기능입니다. Pipeline 스크립트 입력창 아래의 `Pipeline Syntax`를 누르면 다음과 같은 화면이 나옵니다.

![Jenkins Pipeline Snippet](jenkins_pipeline_snippet.webp)

처음부터 직접 작성해도 되지만, 어떻게 써야 할지 막막할 때는 이런 도구를 쓰는 편이 훨씬 편합니다.

### Pipeline 실행 결과

완성된 Pipeline Job을 실행하면 stage별 성공과 실패 결과가 표시됩니다. 방금 만든 Hello World 샘플의 실행 화면은 다음과 같습니다.

![Jenkins Pipeline Result](jenkins_pipeline_result1.webp)

각 stage를 누르면 해당 단계에서 실행한 작업의 결과를 볼 수 있습니다. `Logs`를 눌러 보겠습니다.

![Jenkins Pipeline Result 2](jenkins_pipeline_result2.webp)

로그 화면에서는 각 stage에서 실행한 명령과 작업 결과가 출력되고, 실행 시간과 함께 자세히 확인할 수 있습니다.

![Jenkins Pipeline Result 3](jenkins_pipeline_result3.webp)

### Pipeline 스크립트 구조

이제 Pipeline 스크립트가 어떤 구조인지 간단히 보겠습니다. 기본 형태는 다음과 같습니다.

```groovy
pipeline {
  // 이 안에 실행할 agent와 stage를 작성한다
}
```

### 실행 에이전트 설정

`pipeline` 블록을 작성한 뒤에는 실행 환경을 설정합니다. 단순히 Jenkins가 실행 중인 인스턴스에서 돌릴 때는 `agent any`라고 쓰면 됩니다. 요즘은 작업 실행 전용 Docker 컨테이너를 쓰는 경우도 많습니다. 그런 경우에는 실행 환경으로 Docker 컨테이너를 지정합니다.

```groovy
pipeline {
  agent {
    docker {
      image '실행하고 싶은 이미지'
      args '이미지를 실행할 때 넘길 커맨드라인 변수' // 생략 가능
    }
  }
}
```

### Stage 만들기

환경까지 정했으면, 이제 실행할 작업을 작성합니다. 여기서 중요한 개념이 `stage`입니다. Jenkins 공식 문서에서는 `stage`를 Pipeline 전체에서 수행하는 작업의 명확한 하위 집합으로 설명합니다. 즉 하나의 stage가 하나의 작업 단계라고 보면 됩니다. stage 안에서는 `step`을 정의하고, 그 안에 실행할 명령을 적습니다.

```groovy
pipeline {
  agent {
    // Docker 환경
  }
  stages {
    stage('스테이지명1') {
      steps {
        // 실행하고 싶은 명령
      }
    }
    stage('스테이지명2') {
      // ...
    }
    // ...
  }
}
```

Pipeline 자체에 대한 설명은 여기까지입니다. 다음으로는 실제 Pipeline Job을 작성하면 어떤 형태가 되는지 보겠습니다.

### Pipeline 예제

아래 작업을 Pipeline으로 만든다고 가정하고 간단한 예제를 작성해 봤습니다.

1. 실행 환경은 openjdk 컨테이너, root 사용자
2. Git에서 소스 코드를 체크아웃하고 디렉터리는 `springboot`
3. `gradlew`를 실행해 `war` 파일 생성
4. 완성된 `war` 파일을 Azure Blob에 업로드

이를 코드로 표현하면 다음과 같습니다.

```groovy
pipeline {
  agent {
    docker {
      image 'openjdk' // openjdk 공식 이미지 사용
      args '-u root' // root 사용자로 실행
    }
  }
  stages {
    stage('Checkout') { // Git 체크아웃 단계
      steps {
        checkout([$class: 'GitSCM', branches: [[name: '*/브랜치']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: '저장할 디렉터리']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'Git 크리덴셜 ID', url: 'https://Git 저장소']]])
      }
    }
    stage('Build') { // 빌드 단계
      steps {
        dir(path: 'springbootapi') { // 작업 디렉터리 지정
          sh './gradlew bootWar'
        }
      }
    }
    stage('Upload') { // 빌드한 war 파일을 Azure Blob에 업로드하는 단계
      steps {
        dir('springbootapi/web/build/libs'){
          azureUpload storageCredentialId: '스토리지 크리덴셜 ID', storageType: 'blob', containerName: '컨테이너 이름', filesPath: '**/*.war'
        }
      }
    }
    stage('Finalize') { // 작업이 끝나면 워크스페이스를 정리하는 단계
      steps {
        cleanWs()
      }
    }
  }
}
```

Pipeline 코드에 익숙하지 않으면 어렵게 느껴질 수 있지만, Pipeline Syntax를 활용하면 금방 작성할 수 있습니다. 한 번 써 보면 생각보다 편합니다.

## 마지막으로

처음 Jenkins를 접했을 때도 충분히 편하다고 느꼈지만, Pipeline이 들어오면서 작업 정의가 훨씬 더 분명해졌다는 점이 특히 인상적이었습니다. 최근 CI/CD 도구들을 보면 언어나 문법은 달라도 대체로 이런 방향으로 수렴하는 듯합니다. Jenkins를 계속 쓸 계획이라면 Pipeline은 초반에 한 번 정리해 둘 가치가 충분합니다.
