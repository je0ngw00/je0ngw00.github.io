---
title: "LocalStack으로 AWS SQS/SNS 로컬 테스트 환경 구축하기 (1편)"
date: 2023-07-15 10:00:00 +0900
categories: [Development, Testing]
tags: [localstack, aws, sqs, sns, docker, testing]
---

## 개요

운영 환경에서 AWS SNS + SQS를 사용하는데, 로컬 개발 시 매번 AWS 개발 환경에 연결해서 테스트하는 것이 불편했습니다. LocalStack을 도입하여 로컬에서 AWS 환경을 에뮬레이션하고, 테스트를 더 자유롭게 할 수 있게 된 경험을 공유합니다.

> **시리즈**: [AWS SQS+SNS 기본 개념](/posts/aws-sqs-sns-intro/) · [SQS+SNS vs Kafka 비교](/posts/sqs-sns-vs-kafka/) · [2편: Spring Boot 통합 테스트](/posts/localstack-sqs-sns-part2/)

## LocalStack이란?

LocalStack은 AWS 서비스를 로컬 Docker 컨테이너에서 에뮬레이션하는 도구입니다.

**장점:**
- AWS 의존 없이 로컬 개발 가능
- 테스트 비용 절감 (AWS 요금 없음)
- 빠른 피드백 루프 (배포 없이 테스트)
- CI/CD 파이프라인에서 통합 테스트 가능

**지원 서비스 (무료 버전):**
- SQS, SNS, S3, Lambda, DynamoDB, Kinesis
- API Gateway, CloudWatch, IAM, STS
- 그 외 다수 (유료 버전에서 더 많은 서비스 지원)

> **참고**: LocalStack 버전 0.11.0부터 모든 서비스가 단일 포트(4566)로 통합되었습니다.

## LocalStack 설치 및 실행

### Docker Compose 설정

```yaml
# docker-compose.yml
version: '3.8'

services:
  localstack:
    image: localstack/localstack:3.0  # 최신 버전 사용
    container_name: localstack
    ports:
      - "4566:4566"              # 모든 서비스 단일 포트
      - "4510-4559:4510-4559"    # 외부 서비스용 포트 범위
    environment:
      - SERVICES=sqs,sns         # 사용할 서비스만 지정
      - DEBUG=1                  # 디버그 로그 활성화
      - AWS_DEFAULT_REGION=ap-northeast-2
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./localstack-data:/var/lib/localstack"  # 데이터 영속화
```

### 실행

```bash
docker-compose up -d

# 상태 확인
docker-compose logs -f localstack

# health check
curl http://localhost:4566/_localstack/health
```

## AWS CLI로 리소스 생성

LocalStack이 실행되면 AWS CLI로 리소스를 생성합니다.

### SQS 큐 생성

```bash
# 표준 큐 생성
aws --endpoint-url=http://localhost:4566 sqs create-queue \
    --queue-name order-queue

# FIFO 큐 생성
aws --endpoint-url=http://localhost:4566 sqs create-queue \
    --queue-name order-queue.fifo \
    --attributes FifoQueue=true,ContentBasedDeduplication=true

# DLQ 생성
aws --endpoint-url=http://localhost:4566 sqs create-queue \
    --queue-name order-queue-dlq
```

### SNS 토픽 생성

```bash
# 토픽 생성
aws --endpoint-url=http://localhost:4566 sns create-topic \
    --name order-events

# 출력된 TopicArn 확인
# arn:aws:sns:ap-northeast-2:000000000000:order-events
```

### SNS → SQS 구독 설정

```bash
# SQS 큐 ARN 확인
aws --endpoint-url=http://localhost:4566 sqs get-queue-attributes \
    --queue-url http://localhost:4566/000000000000/order-queue \
    --attribute-names QueueArn

# SNS 토픽에 SQS 구독
aws --endpoint-url=http://localhost:4566 sns subscribe \
    --topic-arn arn:aws:sns:ap-northeast-2:000000000000:order-events \
    --protocol sqs \
    --notification-endpoint arn:aws:sqs:ap-northeast-2:000000000000:order-queue
```

### 초기화 스크립트

매번 수동으로 생성하기 번거로우니 스크립트를 만들어둡니다.

```bash
#!/bin/bash
# init-localstack.sh

ENDPOINT="http://localhost:4566"
REGION="ap-northeast-2"
ACCOUNT="000000000000"

echo "Creating SQS queues..."
aws --endpoint-url=$ENDPOINT sqs create-queue --queue-name order-queue
aws --endpoint-url=$ENDPOINT sqs create-queue --queue-name order-queue-dlq

echo "Creating SNS topics..."
aws --endpoint-url=$ENDPOINT sns create-topic --name order-events

echo "Setting up subscriptions..."
aws --endpoint-url=$ENDPOINT sns subscribe \
    --topic-arn arn:aws:sns:$REGION:$ACCOUNT:order-events \
    --protocol sqs \
    --notification-endpoint arn:aws:sqs:$REGION:$ACCOUNT:order-queue

echo "Done!"
```

Docker Compose에서 자동 실행하려면:

```yaml
# docker-compose.yml
services:
  localstack:
    # ... 기존 설정 ...
    volumes:
      - "./init-localstack.sh:/etc/localstack/init/ready.d/init-localstack.sh"
```

## 메시지 테스트

### SNS에 메시지 발행

```bash
aws --endpoint-url=http://localhost:4566 sns publish \
    --topic-arn arn:aws:sns:ap-northeast-2:000000000000:order-events \
    --message '{"orderId": "12345", "status": "CREATED"}'
```

### SQS에서 메시지 수신

```bash
aws --endpoint-url=http://localhost:4566 sqs receive-message \
    --queue-url http://localhost:4566/000000000000/order-queue \
    --wait-time-seconds 5
```

## Spring Boot 연동 준비

### application.yml 프로파일 분리

```yaml
# application.yml
spring:
  profiles:
    active: local

---
# Local 환경: LocalStack 사용
spring:
  config:
    activate:
      on-profile: local

cloud:
  aws:
    endpoint: http://localhost:4566
    region:
      static: ap-northeast-2
    credentials:
      access-key: test
      secret-key: test
    sqs:
      queue-url: http://localhost:4566/000000000000/order-queue
    sns:
      topic-arn: arn:aws:sns:ap-northeast-2:000000000000:order-events

---
# Production 환경: 실제 AWS 사용
spring:
  config:
    activate:
      on-profile: prod

cloud:
  aws:
    region:
      static: ap-northeast-2
    sqs:
      queue-url: https://sqs.ap-northeast-2.amazonaws.com/123456789/order-queue
    sns:
      topic-arn: arn:aws:sns:ap-northeast-2:123456789:order-events
```

## LocalStack 사용 시 주의사항

### 1. 버전 호환성

LocalStack 버전에 따라 지원하는 AWS API가 다릅니다. `localstack/localstack:3.0` 이상을 권장합니다.

### 2. 데이터 영속화

기본적으로 컨테이너를 재시작하면 데이터가 사라집니다. 볼륨을 마운트하거나 초기화 스크립트를 사용합니다.

### 3. 실제 AWS와의 차이

LocalStack은 에뮬레이션이므로 100% 동일하지 않습니다. 최종 테스트는 실제 AWS 환경에서 진행해야 합니다.

### 4. IAM 정책

LocalStack 무료 버전에서는 IAM 정책이 완벽하게 적용되지 않습니다. 권한 테스트는 별도로 진행합니다.

## 다음 편 예고

2편에서는 Spring Cloud AWS 3.x와 Testcontainers를 사용하여 통합 테스트를 작성하는 방법을 다룹니다.
