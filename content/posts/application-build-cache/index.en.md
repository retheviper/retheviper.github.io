---
title: "Speeding Up Cloud Build CI/CD"
date: 2024-01-21
translationKey: "posts/application-build-cache"
categories: 
  - gcp
image: "../../images/gcp.webp"
tags:
  - gcp
  - gradle
  - docker
  - ci/cd
---

Once you build your infrastructure and applications, CI/CD doesn't change much. In extreme terms, if it works as expected, there should be no problem. However, in some cases, it may be necessary to review the CI/CD flow. This is because you never know when changes will occur, such as adding application functionality or changing infrastructure, and if an emergency response is required, rapid deployment may be required. Therefore, there are cases where speeding up CI/CD becomes a very important issue.

I changed jobs last year, and speeding up CI/CD was an issue at my new company. At my previous job, we released once every two weeks, but here we had to release several times a day, so I tried several ways to make CI/CD faster. In this post, I will introduce the method that worked best.

## Infrastructure configuration

Basically, the configuration is as follows.

- Place source code and CI/CD configuration files on GitHub
- Build a Docker container using Cloud Build
- Deploy to Cloud Run

This configuration is the same for both Frontend and Backend, and the database migration environment is also the same. During migration, we run Flyway on Cloud Run and migrate to AlloyDB.

## Speeding Up with Caching

There are several ways to speed up CI/CD, but this time I improved it by using caching.

There are several ways to speed things up, but I decided to use caching as the simplest method. There are two caches used this time, one is a Gradle cache and the other is a Docker image cache.

## Gradle cache

First of all, Gradle has [Build Cache](https://docs.gradle.org/current/userguide/build_cache.html) and [Configuration Cache](https://docs.gradle.org/current/userguide/configuration_cache.html). If you enable these, the artifacts of the previous build and the settings at the time of build will be used as a cache from the next build, which can significantly shorten the build time.

To enable these caches, configure the `gradle.properties` file as follows:

```properties
// build cache
org.gradle.caching=true

// configuration cache
org.gradle.configuration-cache=true
```

## Docker image caching

Even if Gradle's cache is enabled, a cache must exist that can be used when the build is actually run in the CI/CD environment. For this purpose, we used Docker cache. Every time the Cloud Build flow is executed, you can push the artifacts of the previous build somewhere and use them when the build runs to take advantage of Gradle's cache.

To achieve this, first of all, the Cloud build configuration file used to look like this: (Partial excerpt)

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

In the above file, we are first building a Docker image with the hash of the commit and pushing it. We then use that image to deploy to Cloud Run. At this time, the commit hash is used for the image tag.

If you insert a cache here, it will look like this:

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

The above file first pulls the artifacts from the previous build. This allows you to reuse artifacts from earlier builds. The build then uses the `--cache-from` option to use those artifacts as a cache. We also use both `latest` and the commit hash as image tags. Finally, push both `latest` and the commit-hash tag to Container Registry. This way, Container Registry stores two tags for the same image, and you can use `latest` as the cache source in the next build.

## Dockerfile modification

I was using a Dockerfile when building a Docker image, but I modified the Dockerfile to use caching more efficiently. The Dockerfile before modification was as follows. (Partial excerpt)

```dockerfile
FROM eclipse-temurin:17

RUN mkdir /api
WORKDIR /api

RUN ./gradlew build -x test -x compileTestKotlin
```

The above Dockerfile first runs the Gradle build, and depending on the Gradle settings the artifact is produced as a jar file. Cloud Run was configured to run the repository's entrypoint, which in turn executed that jar file.

Generally, you can reduce the size of the container by leaving only the jar file, but in this case we need to leave artifacts other than the jar file in order to use the cache, so we did not make any modifications here. However, it takes a lot of time to resolve dependencies every time a Gradle build runs, so I cached only the part that resolves dependencies. The result should look like this:

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

Deployment almost always involves source code changes. Therefore, first copy only the Gradle-related files that do not change much, and then build without the source code. This will only resolve the dependencies without building the actual app. Then copy the source code and build.

The reason for this structure is that even if the cache exists when the Docker image is built, the cache will be invalidated if the copied file changes. If there is an update to the dependencies, the cache will no longer be effective, but if the source code has not changed, the cache will be enabled until the dependencies are resolved, which can shorten the build time.

Although we mainly introduced the Backend settings here, the Frontend settings can also be made to use caching by modifying the Dockerfile in the same way. The settings are slightly different from the backend, but basically yarn works the same way as Gradle, just resolve dependencies first.

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

In the case of database migration, it was difficult to directly connect to the DB with Cloud Build and perform the migration, and DB access was easy by configuring the same settings as Backend, so basically I created a Docker image like Backend and used Entrypoint scripts on Cloud Run to run Flyway.

However, after building the same app using the same Dockerfile as Backend, I started running the actual app on Cloud Run, but since this was not necessary (with Cloud Run, I just need to make the container respond from the specified port), I decided not to build the app and only resolve the dependencies. Regarding the response from the port, I start nginx and respond.

## Finally

This time, I introduced how to speed up CI/CD. By using the cache, the build time has been significantly shortened, and the existing build time was about 14 minutes, but now it is about 6 minutes, so the fact that we are releasing several times a day can be considered a significant achievement. However, there is still room for improvement, such as not using Cloud Run for database migration.

See you soon!
