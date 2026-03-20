---
title: "Cloud Build로 CI/CD 더 빠르게 만들기"
date: 2024-01-21
categories: 
  - gcp
image: "../../images/gcp.webp"
tags:
  - gcp
  - gradle
  - docker
  - ci/cd
translationKey: "posts/application-build-cache"
---

CI/CD는 인프라와 애플리케이션을 한 번 갖춰 두면 자주 바뀌지 않습니다. 기대한 대로 안정적으로 돌아간다면 그대로 두어도 큰 문제는 없습니다. 다만 기능 추가나 인프라 변경, 긴급 대응처럼 배포 속도가 중요해지는 순간이 오면 흐름을 다시 점검해야 합니다. 그래서 CI/CD 자체를 빠르게 만드는 일도 생각보다 중요한 과제가 됩니다.

저는 작년에 이직을 했는데, 새 회사에서는 CI/CD 속도가 실제로 큰 과제였습니다. 이전 회사는 2주에 한 번 정도 릴리스하는 흐름이었지만, 지금은 하루에도 여러 번 배포가 일어납니다. 그래서 조금이라도 배포를 빠르게 만들기 위해 여러 방법을 시도해 봤고, 그중 가장 효과가 좋았던 방법을 정리해 봤습니다.

## 인프라 구성

기본 구조는 아래와 같습니다.

- GitHub에 소스 코드와 CI/CD 설정 파일을 둠
- Cloud Build로 Docker 컨테이너를 빌드
- Cloud Run에 배포

Frontend와 Backend는 같은 방식으로 구성했고, 데이터베이스 마이그레이션도 같은 환경에서 처리합니다. 마이그레이션은 Cloud Run에서 Flyway를 실행해 AlloyDB에 반영하는 형태로 운영하고 있습니다.

## 캐시로 속도 올리기

CI/CD를 빠르게 만드는 방법은 여러 가지가 있지만, 이번에는 캐시를 활용하는 방식으로 해결했습니다. 가장 손대기 쉬운 방법부터 적용해 보기로 했고, 실제로는 두 가지 캐시를 썼습니다. 하나는 Gradle 캐시이고, 다른 하나는 Docker 이미지 캐시입니다.

### Gradle 캐시

Gradle에는 [Build Cache](https://docs.gradle.org/current/userguide/build_cache.html)와 [Configuration Cache](https://docs.gradle.org/current/userguide/configuration_cache.html)가 있습니다. 이 둘을 활성화하면 다음 빌드부터는 이전 빌드의 결과물과 설정을 재사용할 수 있어서, 빌드 시간을 꽤 줄일 수 있습니다.

활성화 방법은 `gradle.properties`에 아래처럼 적으면 됩니다.

```properties
// build cache
org.gradle.caching=true

// configuration cache
org.gradle.configuration-cache=true
```

### Docker 이미지 캐시

Gradle 캐시를 켜도, 실제 CI/CD 환경에서 재사용할 캐시가 있어야 합니다. 그래서 Docker 이미지를 캐시 저장소처럼 활용했습니다. Cloud Build가 실행될 때마다 이전에 만들어 둔 이미지를 다시 가져오고, 그 이미지를 다음 빌드의 기준으로 삼는 방식입니다.

기존 Cloud Build 설정은 대략 이런 형태였습니다.

```yaml
steps:
  - name: gcr.io/cloud-builders/docker
    args:
      - build
      - '-t'
      - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
      - api
      - '-f'
      - api/Dockerfile
    id: Build
  - name: gcr.io/cloud-builders/docker
    args:
      - push
      - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
    id: Push
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk:slim'
    args:
      - run
      - services
      - update
      - $_SERVICE_NAME
      - '--platform=managed'
      - '--image=$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:COMMIT_SHA'
      - >-
        --labels=managed-by=gcp-cloud-build-deploy-cloud-run,commit-sha=$COMMIT_SHA,gcb-build-id=$BUILD_ID,gcb-trigger-id=$_TRIGGER_ID,$_LABELS
      - '--region=$_DEPLOY_REGION'
      - '--quiet'
    id: Deploy
    entrypoint: gcloud
images:
  - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:COMMIT_SHA'
```

이 구조에서는 커밋 해시를 태그로 붙인 이미지를 빌드한 뒤 곧바로 푸시하고, 그 이미지를 그대로 Cloud Run 배포에 사용합니다.

여기에 캐시를 추가하면 설정은 아래처럼 바뀝니다.

```yaml
steps:
  - name: gcr.io/cloud-builders/docker
    args:
      - '-c'
      - >-
        docker pull $_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest
        || exit 0
    id: Pull
    entrypoint: bash
  - name: gcr.io/cloud-builders/docker
    args:
      - build
      - '--cache-from'
      - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest'
      - '-t'
      - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
      - '-t'
      - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest'
      - api
      - '-f'
      - api/Dockerfile
    id: Build
  - name: gcr.io/cloud-builders/docker
    args:
      - push
      - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
    id: Push (Cache)
  - name: gcr.io/cloud-builders/docker
    args:
      - push
      - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest'
    id: Push (Latest)
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk:slim'
    args:
      - run
      - services
      - update
      - $_SERVICE_NAME
      - '--platform=managed'
      - '--image=$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest'
      - >-
        --labels=managed-by=gcp-cloud-build-deploy-cloud-run,commit-sha=$COMMIT_SHA,gcb-build-id=$BUILD_ID,gcb-trigger-id=$_TRIGGER_ID,$_LABELS
      - '--region=$_DEPLOY_REGION'
      - '--quiet'
    id: Deploy
    entrypoint: gcloud
images:
  - '$_GCR_HOSTNAME/$PROJECT_ID/$REPO_NAME/$_SERVICE_NAME:latest'
```

먼저 `latest` 태그가 붙은 이전 이미지를 가져오고, 그 이미지를 `--cache-from`으로 사용합니다. 빌드가 끝나면 새 이미지에는 `latest`와 커밋 해시 두 태그를 함께 붙여 푸시합니다. 이렇게 해 두면 다음 빌드에서 다시 `latest`를 재사용할 수 있습니다.

### Dockerfile 수정

Docker 이미지를 빌드할 때도 캐시가 잘 먹도록 Dockerfile을 손봤습니다. 수정 전 Dockerfile은 아래와 같았습니다.

```dockerfile
FROM eclipse-temurin:17

RUN mkdir /api
WORKDIR /api

RUN ./gradlew build -x test -x compileTestKotlin
```

이 방식은 단순하지만, 빌드가 돌 때마다 의존성 해결까지 매번 다시 하게 됩니다. Cloud Run에서 실행할 jar 자체는 유지해야 하므로, 결과물을 아예 줄이는 방식도 쓸 수 없었습니다. 그래서 의존성 관련 파일만 먼저 복사해 두고, 그 단계만 캐시되도록 바꿨습니다.

수정 후에는 아래처럼 나눴습니다.

```dockerfile
FROM eclipse-temurin:17
WORKDIR /api

# dependency cache
COPY build.gradle.kts settings.gradle.kts gradlew gradle.properties /api/
COPY gradle /api/gradle
COPY docker /api/docker
COPY detekt /api/detekt
RUN ./gradlew build -x test -x compileTestKotlin -x detekt || return 0

# build app
COPY src /api/src
COPY resources /api/resources

RUN ./gradlew build -x test -x compileTestKotlin
```

실제 배포에서는 대부분 소스 코드만 바뀌고 Gradle 설정 파일은 잘 바뀌지 않습니다. 그래서 먼저 Gradle 관련 파일만 복사해서 의존성 해소 단계까지 캐시를 태우고, 그다음 소스 코드를 복사해 본격적으로 빌드합니다. 이렇게 하면 소스만 바뀐 경우에는 의존성 해결 시간을 상당히 줄일 수 있습니다.

이 방식은 Backend에 먼저 적용했지만, Frontend도 같은 원리로 처리했습니다. Backend와 세부 내용은 조금 다르지만, 핵심은 Yarn도 Gradle처럼 의존성 설치 단계를 먼저 분리하는 것입니다.

```dockerfile
FROM node:18.17.1-slim AS base

WORKDIR /app

# dependency cache
COPY package.json yarn.lock ./
RUN yarn --frozen-lockfile

COPY ./tsconfig.json ./
COPY ./packages/app/package.json ./packages/app/package.json
RUN cd ./packages/app && yarn || return 0

# build app
FROM base AS builder
WORKDIR /app
COPY ./packages/app/ ./packages/app/
COPY --from=base /app/packages/app ./packages/app
RUN yarn build:app:production
```

데이터베이스 마이그레이션은 조금 다릅니다. Cloud Build에서 직접 DB에 붙는 구조는 다루기 까다로웠고, Backend와 비슷한 방식으로 처리하는 편이 접근성도 좋았습니다. 그래서 기본적으로는 Backend와 같은 방식으로 Docker 이미지를 만들고, Cloud Run에서 Entrypoint 스크립트로 Flyway를 실행하게 했습니다.

다만 이 경우에는 Backend처럼 애플리케이션 자체를 실행할 필요는 없었습니다. Cloud Run은 지정한 포트에만 응답하면 되므로, 실제 애플리케이션 빌드는 생략하고 의존성 해소까지만 수행하도록 바꿨습니다. 포트 응답은 nginx가 맡도록 했습니다.

## 마지막으로

이번에는 CI/CD를 더 빠르게 만든 과정을 정리했습니다. 캐시를 적용한 뒤에는 빌드 시간이 크게 줄어서, 예전에는 약 14분 걸리던 것이 지금은 6분 정도로 내려갔습니다. 하루에도 여러 번 릴리스가 나가는 환경에서는 꽤 큰 차이입니다. 다만 데이터베이스 마이그레이션처럼 아직 Cloud Run에 기대고 있는 부분은 더 손볼 여지가 있습니다.

그래도 이번 작업을 통해, 배포 파이프라인은 한 번 만들어 두고 끝나는 것이 아니라 운영 방식에 맞춰 계속 다듬어야 하는 대상이라는 점을 다시 느꼈습니다.
