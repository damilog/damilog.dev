---
title: Github Action을 이용하여 React 프로젝트 AWS S3 배포 자동화 구현하기
date: '2023-07-16'
tags: ['ci/cd']
draft: false
summary: Github Action을 이용하여 React 프로젝트를 AWS S3에 배포 자동화
---

리액트 프로젝트를 AWS S3에 배포하는 방법과 더불어 GitHub Action으로 빌드/배포를 자동화하는 방법에 대해 살펴보겠습니다.
AWS 계정을 생성했다는 가정하에 진행하도록 하겠습니다.

# S3 배포

우선 CRA를 통해 배포 실습에 사용할 리액트 프로젝트를 생성해줍니다. 배포 대상인 프로젝트가 있다면, 이 단계는 넘어가셔도 좋습니다.

```
npx create-react-app my-app
```

다음으로 아래 명령어로 프로젝트 빌드를 수행합니다.

```
npm run build
```

이제 생성된 build 폴더를 AWS S3에 배포를 해보겠습니다.
![](https://velog.velcdn.com/images/dami/post/68c6e5e1-6d2f-4c40-92f5-96b4d1c95d02/image.png)

![](https://velog.velcdn.com/images/dami/post/89bcf4d0-371c-40e8-9c77-105993e750a4/image.png)

우측 상단의 버킷 만들기 버튼을 클릭하여 버킷을 생성합니다.

![](https://velog.velcdn.com/images/dami/post/a0396e77-ed68-49fa-9a67-20e9397e4937/image.png)

객체 소유권은 ACL 비활성화됨(권장)으로 지정합니다.
![](https://velog.velcdn.com/images/dami/post/be49fb33-aa66-462c-9a37-0c5e18ccbd51/image.png)

CloudFront 없이 S3로만 배포를 하는 구조이기에 우선 모든 퍼블릭 엑세스 차단을 해제한 다음 '버킷 만들기'를 클릭하여 버킷을 생성해줍니다.

![](https://velog.velcdn.com/images/dami/post/45f797c1-f7bf-49f0-8636-80c55c3af118/image.png)

버킷 생성이 완료되었다면, S3 버킷 페이지 내에 생성한 아래와 같이 생성한 버킷을 확인 가능합니다.
![](https://velog.velcdn.com/images/dami/post/9db3ef80-19de-45a4-93f5-cd6a2fdb6ed9/image.png)

이제 해당 버킷의 엑세스를 '객체를 퍼블릭으로 설정할 수 있음'에서 '퍼블릭'으로 변경하는 작업을 진행하겠습니다.

버킷 > 권한 > 버킷 정책에서 편집을 눌러 버킷 정책을 생성해줍니다.
![](https://velog.velcdn.com/images/dami/post/bd17199e-35d7-4aac-b1d6-0bc7cc7ad083/image.png)

버킷 정책을 직접 편집할 수도 있겠지만, 정책 생성기를 이용해 정책을 생성해보도록 하겠습니다. 아래 이미지에서 보이는 정책 생성기를 클릭합니다.

![](https://velog.velcdn.com/images/dami/post/b7ec96f8-d68c-4dfe-83d7-8f60d47081b0/image.png)

그럼 아래와 같은 AWS Policy Generator 페이지가 등장합니다. Type of Policy에는 S3 Bucket Policy를 선택해줍니다.

다음으로 Statements를 정의해보겠습니다. Principal에는 `*`을, Actions는 GetObject를 선택해줍니다. 이때 ARN을 기입하는 란이 있는데요.

![](https://velog.velcdn.com/images/dami/post/c3983042-dcb8-42e2-93ed-3b899e41a6ff/image.png)

ARN 값은 직전 페이지의 버킷 ARN란에서 복사하여 붙여넣으면 됩니다.
![](https://velog.velcdn.com/images/dami/post/69514957-5ff2-43aa-84e5-ba178734b104/image.png)

ARN 값을 붙여 넣었다면 Add Statement 버튼을 클릭하고, Generate Policy 버튼을 클릭하여 정책을 생성해줍니다.
![](https://velog.velcdn.com/images/dami/post/5cbd9a4b-8b64-47ac-96ce-8278331ea7be/image.png)

그럼 아래와 같은 JSON Document를 확인할 수 있습니다. 이 JSON을 복사합니다.
![](https://velog.velcdn.com/images/dami/post/076d3289-f52b-4980-bffc-82be8e680ba1/image.png)

복사한 JSON을 해당 버킷의 정책에 붙여넣어 줍니다.
![](https://velog.velcdn.com/images/dami/post/aa9ef760-aaec-4d4b-a011-24e1b5b1814b/image.png)

# Github Action

이제 Github Action을 이용하여 빌드/배포 자동화를 적용해보겠습니다. Github Action을 적용한다면 main 브랜치에 변경이 감지되면 Workflow 프로세스가 실행되고 이때 S3에 배포가 자동으로 진행되게 됩니다.

## Acess Key 저장하기

AWS IAM에서 사용자를 추가하게 되면 접근 권한키를 발급 받을 수 있습니다. 이 접근 권한키가 있어야 S3 배포 스크립트 작성이 가능한데요. 배포 스크립트에 접근 권한키가 노출된다면 보안 문제를 발생시킬 수 있기 때문에, 이 접근 권한키를 암호화해야 합니다. Setting> Secrets and variables > Actions
에서 New repository secret을 클릭하여 AWS key id와 접근키를 생성해줍니다.

![](https://velog.velcdn.com/images/dami/post/4c052b5b-1c23-4a28-9697-813f10099939/image.png)

## Workflow 설정하기

Repositoryh 내 Actions 탭으로 이동하여 좌측 상단의 New workflow 버튼을 클릭합니다.

이미 생성된 workflow가 아닌 직접 배포 스크립트를 수정할 것이기에 set up a workflow yourself 버튼을 클릭합니다 .
![](https://velog.velcdn.com/images/dami/post/12990fe6-a322-402e-af5f-29ea02e82da6/image.png)

배포 스크립트를 작성해줍니다.

```yml
name: Deploy to Production

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout source code
        uses: actions/checkout@master

      - name: Cache node modules
        uses: actions/cache@v1
        with:
          path: node_modules
          key: ${{ runner.OS }}-build-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.OS }}-build-
            ${{ runner.OS }}-

      - name: Install Dependencies
        run: npm install

      - name: Build
        run: npm run build

      - name: Deploy
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          aws s3 cp \
            --recursive \
            --region ap-northeast-2 \
            build s3://앞서 생성한 s3 주소
```

이제 모든 준비는 끝났습니다. 리액트 프로젝트를 수정 후 빌드한 뒤 target이 되는 브랜치에 push하여 s3 배포까지 정상적으로 되는지 확인해봅니다.
