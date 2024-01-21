---
title: "Cloud BuildのCI/CDを高速化してみた"
date: 2024-01-21
categories: 
  - gcp
image: "../../images/gcp.jpg"
tags:
  - gcp
  - gradle
  - docker
  - ci/cd
---

CI/CDは一度インフラとアプリケーションを構築すれば、あまり変わることはありません。極端な話、期待通り動いてくれれば問題はないといえることです。しかし、場合によってはCI/CDのフローを見直すことも必要になります。アプリケーションの機能追加やインフラの変更などにはいつ変化があるかわからないし、緊急対応が必要な場合は迅速なデプロイが必要とされる場合もあるからです。そのため、CI/CDの高速化が非常に重要な課題となるケースもあります。

実は去年転職をしましたが、転職先の会社ではCI/CDの高速化が課題となっていました。前職では2週間に一回というリリースのサイクルがあったのですが、今回は一日でも数回のリリースがあるため何よりもそのため、CI/CDの高速化を行うために、いくつかの方法を試してみました。今回はその中で、最も効果があった方法を紹介します。

## インフラ構成

基本的には、以下のような構成になっています。

- GitHubにソースコードおよびCI/CDの構成ファイルを配置
- Cloud Buildを利用してDockerコンテナをビルド
- Cloud Runにデプロイ

この構成はFrontend, Backendともに同じで、データベースマイグレーションも同じ環境となっています。マイグレーションの時にはCloud RunでFlywayを実行してAlloyDBにマイグレーションを行っているようになっています。

## キャッシュで高速化

CI/CDの高速化にはいくつかの方法がありますが、今回はキャッシュを利用することで高速化を行いました。

高速化のために考えられるのはいくつかの方法がありますが、まずは簡単な方法としてキャッシュを利用することにしました。今回利用したキャッシュは二つで、一つはGradleのキャッシュ、もう一つはDockerイメージのキャッシュです。

### Gradleのキャッシュ

まずGradleには[Build Cache](https://docs.gradle.org/current/userguide/build_cache.html)と[Configuration Cache](https://docs.gradle.org/current/userguide/configuration_cache.html)というものがあり、これらを有効にした場合、次回のビルドからはそれぞれ前回のビルドの成果物とビルド時の設定をキャッシュとして使うことになるのでビルドの時間をかなり短縮することができます。

これらのキャッシュを有効にするには、`gradle.properties`ファイルに以下のように設定を行います。

```properties
// build cache
org.gradle.caching=true

// configuration cache
org.gradle.configuration-cache=true
```

### Dockerイメージのキャッシュ

Gradleのキャッシュを有効にしても、CI/CDの環境で実際ビルドが走る時に利用できるキャッシュが存在しなければなりません。そのために利用したのがDockerのキャッシュです。Cloud Buildのフローが実行されるたびに前回のビルドの成果物をどこかにプッシュしておき、ビルドが走る時にそれを利用するとGradleのキャッシュを利用することができるようになります。

これを実現するためにはまず最初にCloud buildの構成ファイルは以前は以下のようになっていました。(一部抜粋)

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

上記のファイルでは、最初にコミットのハッシュでDockerイメージをビルドし、それをプッシュしています。その後、Cloud Runにデプロイするために、そのイメージを利用しています。この時、イメージのタグにはコミットのハッシュを利用しています。

ここでキャッシュを挟むようにすると、以下のようになります。

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

上記のファイルでは、まず最初に前回のビルドの成果物をプルしています。これにより、前回のビルドの成果物を利用することができます。その後、ビルドでは`--cache-from`オプションを利用して、前回のビルドの成果物をキャッシュとして利用しています。また、イメージのタグには`latest`とコミットのハッシュを利用しています。そして最後にContainer Registryに`latest`とコミット8種の両方をプッシュします。こうすると、Container Registryでは一つのイメージに対して二つのタグが付与され、次回のビルドでも`latest`を利用できるようになります。

### Dockerfileの修正

Dockerイメージをビルドする際にDockerfileを使っていましたが、キャッシュをより効率的に利用するためにDockerfileを修正しました。修正前のDockerfileは以下のようになっていました。(一部抜粋)

```dockerfile
FROM eclipse-temurin:17

RUN mkdir /api
WORKDIR /api

RUN ./gradlew build -x test -x compileTestKotlin
```

上記のDockerfileでは、まずGradleのビルドを行っていて、成果物はGradleの設定によりjarファイルとしてビルドされます。そしてCloud RunではリポジトリないのEntrypointを実行するように設定しているため、jarファイルを実行するようになっています。

一般的にはjarファイルだけを残すとコンテナのサイズを小さくすることができますが、今回はキャッシュを利用するためにjarファイル以外の成果物も残す必要があるので、ここは修正していません。ただ、Gradleのビルドが走る時に、毎回依存関係の解決を行うのは非常に時間がかかるので、依存関係の解決を行う部分だけをキャッシュするようにしました。結果は以下のようになります。

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

デプロイではほとんどの場合、ソースコードの変更が行われます。なので、あまり変わることのないGradle関係のファイルだけを先にコピーして、ソースコードのない状態でビルドを行います。これにより、実際のアプリはビルドされずに依存関係の解決だけが行われます。それからソースコードをコピーしてビルドを行います。

なぜこのような構造にしたかというと、Dockerイメージのビルド時にはキャッシュが存在していても、コピーしたファイルに変更があった場合にキャッシュが無効になるためです。依存関係の更新があった場合は仕方なくキャッシュは効かなくなりますが、ソースコードの変更がない場合は依存関係の解決まではキャッシュが有効になるため、ビルドの時間を短縮することができます。

ここでは主にBackendの設定を紹介しましたが、Frontendの設定もDockerfileを同じように修正することでキャッシュを利用できるようにしています。backendと多少は設定が違いますが、基本的にはyarnもGradleと仕組みとしては同じく先に依存関係の解決を行うようにするだけです。

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

データベースマイグレーションの場合、Cloud Buildで直接DBに接続してマイグレーションを行うのが難しかったのと、Backendと同様の設定をすることでDBアクセスは簡単にできたので基本的にはBackendと同じくDockerイメージを作成し、Cloud Run上でEntrypointのスクリプトを利用しFlywayを実行するようにしていました。

ただ、ここでBackendと同じDockerfileを使って同じくアプリをビルドしたあと、実際のアプリをCloud Runで実行するようになっていましたが、ここは必要なかったので（Cloud Runでは指定したポートからコンテナが応答するようにすればいいだけなので）アプリのビルドは行わなく、依存関係の解決だけを行うようにしました。ポートからの応答に関してはnginxを起動して応答するようにしています。

## 最後に

今回はCI/CDの高速化について紹介しました。キャッシュを利用することで、ビルドの時間が大幅に短縮できて、既存だと14分ほどかかっていたのが今は6分ほどになっているので、一日中でも数回のリリースが行われる現状はかなり大きい成果だと言えます。ただ、データベースのマイグレーションではCloud Runを使わないようにするなど依然として改善の余地はありますね。

では、また！
