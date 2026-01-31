---
title: "AWS Lambda CI/CD 파이프라인 구축하기: CodeCommit + CodePipeline"
date: 2023-06-01 10:00:00 +0900
categories: [Development, AWS]
tags: [aws, lambda, cicd, codecommit, codepipeline, codebuild]
---

## 개요

Lambda 함수를 콘솔에서 직접 수정하는 방식은 버전 관리도 안 되고, 실수로 코드를 날릴 위험도 있습니다. AWS CodeCommit과 CodePipeline을 이용해 Lambda의 CI/CD 파이프라인을 구성한 경험을 공유합니다.

## 전체 아키텍처

```
CodeCommit (Push) → CodePipeline (감지) → CodeBuild (빌드) → Lambda (배포)
```

1. 개발자가 CodeCommit에 코드를 push
2. CodePipeline이 변경을 감지
3. CodeBuild가 코드를 빌드하고 Lambda에 배포

## CodeCommit과 CodePipeline

### CodeCommit

AWS에서 제공하는 관리형 Git 리포지토리 서비스입니다. GitHub와 유사하지만 AWS IAM과 완벽히 통합되어 권한 관리가 편리합니다.

### CodePipeline

소스 변경 감지부터 빌드, 배포까지 전체 릴리스 프로세스를 자동화하는 서비스입니다.

## 구성 단계

### 1. CodeCommit 리포지토리 생성

AWS 콘솔 > CodeCommit > 리포지토리 생성에서 새 리포지토리를 만들고, 로컬 코드와 연결합니다.

```bash
# 리포지토리 클론
git clone https://git-codecommit.ap-northeast-2.amazonaws.com/v1/repos/my-lambda

# 브랜치 생성
git checkout -b develop
```

### 2. buildspec.yml 작성

CodeBuild가 사용할 빌드 명세 파일입니다. 프로젝트 루트에 생성합니다.

```yaml
# buildspec.yml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.9
    commands:
      - echo "Installing dependencies..."
      - pip install -r requirements.txt -t ./package

  build:
    commands:
      - echo "Packaging Lambda function..."
      - cd package
      - zip -r ../deployment.zip .
      - cd ..
      - zip -g deployment.zip lambda_function.py

  post_build:
    commands:
      - echo "Deploying to Lambda..."
      - aws lambda update-function-code
          --function-name my-lambda-function
          --zip-file fileb://deployment.zip
      - echo "Deployment completed on $(date)"

artifacts:
  files:
    - deployment.zip
```

**주요 포인트:**
- `runtime-versions`: Lambda와 동일한 Python 버전 사용
- `artifacts`: 빌드 결과물 정의 (디버깅 시 유용)

### 3. CodeBuild 프로젝트 생성

**환경 설정:**
- 소스 공급자: CodeCommit
- 리포지토리: 생성한 리포지토리 선택
- 브랜치: develop (또는 배포할 브랜치)
- 환경 이미지: Amazon Linux 2
- 빌드 사양: buildspec.yml 사용

### 4. CodeBuild IAM 권한 설정

CodeBuild 서비스 역할에 Lambda 배포 권한을 추가해야 합니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "lambda:UpdateFunctionCode",
                "lambda:GetFunction",
                "lambda:UpdateFunctionConfiguration"
            ],
            "Resource": "arn:aws:lambda:ap-northeast-2:123456789:function:my-lambda-function"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

### 5. CodePipeline 생성

**파이프라인 설정:**
1. 소스 스테이지: CodeCommit, 해당 리포지토리/브랜치 선택
2. 빌드 스테이지: CodeBuild, 생성한 프로젝트 선택
3. 배포 스테이지: 건너뛰기 (buildspec.yml에서 AWS CLI로 처리)

파이프라인 생성 후 첫 실행이 자동으로 시작됩니다.

## 다중 환경 배포

develop, staging, production 환경별로 다르게 배포하려면 환경 변수를 활용합니다.

```yaml
# buildspec.yml
version: 0.2

env:
  variables:
    ENV: "dev"  # CodeBuild 프로젝트별로 오버라이드

phases:
  post_build:
    commands:
      - |
        if [ "$ENV" = "prod" ]; then
          FUNCTION_NAME="my-lambda-prod"
        elif [ "$ENV" = "staging" ]; then
          FUNCTION_NAME="my-lambda-staging"
        else
          FUNCTION_NAME="my-lambda-dev"
        fi
      - aws lambda update-function-code
          --function-name $FUNCTION_NAME
          --zip-file fileb://deployment.zip
```

각 환경별로 CodeBuild 프로젝트를 만들고 `ENV` 환경 변수를 다르게 설정합니다.

## 추가로 고려할 사항

### 알림 구성

배포 성공/실패 알림을 받으려면 SNS를 연동합니다.

```yaml
# buildspec.yml post_build에 추가
- |
  if [ $CODEBUILD_BUILD_SUCCEEDING -eq 1 ]; then
    aws sns publish --topic-arn arn:aws:sns:ap-northeast-2:123456789:deploy-notifications \
      --message "Lambda 배포 성공: $FUNCTION_NAME"
  else
    aws sns publish --topic-arn arn:aws:sns:ap-northeast-2:123456789:deploy-notifications \
      --message "Lambda 배포 실패: $FUNCTION_NAME"
  fi
```

### 롤백 구성

배포 실패 시 이전 버전으로 자동 롤백하려면 Lambda 버전과 별칭을 활용합니다.

```yaml
# 버전 발행 및 별칭 업데이트
- VERSION=$(aws lambda publish-version --function-name $FUNCTION_NAME --query 'Version' --output text)
- aws lambda update-alias --function-name $FUNCTION_NAME --name live --function-version $VERSION
```

### 배포 전 테스트

```yaml
phases:
  build:
    commands:
      - echo "Running tests..."
      - python -m pytest tests/ -v
      - echo "Tests passed, proceeding with packaging..."
```

## 정리

| 구성 요소 | 역할 |
|----------|------|
| CodeCommit | Git 리포지토리, 소스 코드 관리 |
| CodePipeline | 변경 감지, 파이프라인 오케스트레이션 |
| CodeBuild | 빌드 및 배포 실행 |
| buildspec.yml | 빌드/배포 명령 정의 |

Lambda 함수를 Git으로 관리하고 자동 배포하면 코드 리뷰, 버전 관리, 롤백이 모두 가능해집니다. 처음 설정은 번거롭지만, 한 번 구축해두면 운영이 훨씬 편해집니다.
