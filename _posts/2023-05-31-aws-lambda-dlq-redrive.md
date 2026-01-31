---
title: "AWS Lambda로 DLQ 메시지 자동 Redrive 구현하기"
date: 2023-05-31 10:00:00 +0900
categories: [Development, AWS]
tags: [aws, lambda, sqs, dlq, python, eventbridge]
---

## 개요

AWS SQS + SNS를 통해 비동기 처리를 하다 보면 외부 연동 문제 등으로 메시지 처리에 실패하는 경우가 있습니다. 10번 이상 처리에 실패한 메시지는 DLQ(Dead Letter Queue)에 적재되는데, 문제 해결 후 매번 콘솔에서 수동으로 redrive하는 것이 번거로웠습니다.

이 작업을 자동화하기 위해 AWS Lambda를 활용했습니다.

> **2023년 6월 업데이트**: AWS에서 DLQ Redrive API를 공식 지원합니다. 새로 구현하신다면 아래 [AWS Redrive API 활용](#aws-redrive-api-활용) 섹션을 참고하세요.

## AWS Lambda란?

서버를 프로비저닝하거나 관리하지 않고 코드를 실행할 수 있는 서버리스 컴퓨팅 서비스입니다.

**주요 사용 사례:**
- 파일 처리: S3 업로드 시 실시간 데이터 처리
- 스트림 처리: Kinesis를 통한 실시간 데이터 처리
- API 백엔드: API Gateway와 결합한 서버리스 API
- 스케줄 작업: EventBridge를 통한 정기 실행

## Lambda 콜드 스타트 문제

Lambda를 사용할 때 반드시 알아야 할 것이 **콜드 스타트(Cold Start)** 입니다.

### 콜드 스타트란?

Lambda 함수가 처음 호출되거나 오랜 시간 호출되지 않으면, AWS가 새로운 실행 환경을 생성해야 합니다. 이 초기화 과정에서 발생하는 지연을 콜드 스타트라고 합니다.

```
# CloudWatch 로그에서 콜드 스타트 확인
REPORT RequestId: xxx Duration: 368.80 ms Init Duration: 3569.10 ms
                                         ^^^^^^^^^^^^^^^^^^^^^^
                                         Init Duration이 있으면 콜드 스타트
```

### 콜드 스타트 영향 요소

| 요소 | 영향 |
|------|------|
| 런타임 | Java, .NET > Python, Node.js |
| 패키지 크기 | 클수록 초기화 시간 증가 |
| 메모리 할당 | 높을수록 CPU 할당 증가, 초기화 빠름 |
| VPC 연결 | VPC 내 Lambda는 ENI 생성으로 추가 지연 |

### 해결 방법

**1. 코드 최적화**
```python
# 핸들러 밖에서 초기화 (콜드 스타트 시 1회만 실행)
import boto3
sqs_client = boto3.client('sqs')  # 재사용

def lambda_handler(event, context):
    # sqs_client 재사용
    pass
```

**2. 패키지 크기 최소화**
- 불필요한 의존성 제거
- Lambda Layer 활용

**3. Provisioned Concurrency (비용 발생)**

미리 실행 환경을 준비해두어 콜드 스타트를 제거합니다.

```bash
aws lambda put-provisioned-concurrency-config \
    --function-name my-function \
    --qualifier prod \
    --provisioned-concurrent-executions 5
```

| 장점 | 단점 |
|------|------|
| 콜드 스타트 제거 | 비용 증가 |
| 일관된 응답 시간 | 설정 수 초과 시 다시 콜드 스타트 |

**4. 아키텍처 선택 (패키지 크기 기준)**
- 4MB 이하: x86_64가 유리
- 4MB 초과: arm64가 콜드 스타트 시간 감소에 유리

### DLQ Redrive Lambda에서의 고려사항

이번 사례처럼 **하루에 한 번 실행**되는 Lambda는 매번 콜드 스타트가 발생합니다. 다만 DLQ redrive는 실시간 응답이 필요 없는 배치 작업이라 콜드 스타트가 큰 문제가 되지 않습니다.

실시간 API라면 Provisioned Concurrency를 고려해야 하지만, 이런 배치 작업에는 비용 대비 효과가 없습니다.

## 구현 방향

DLQ 메시지 처리 방식:

- **실행 시점**: 매일 새벽 6시
- **처리 대상**: 전날 발생한 DLQ 메시지
- **처리 방식**: DLQ에서 메시지를 읽어 원래 SQS로 재전송

## Lambda 구성

### 1. 함수 생성

Python 런타임으로 Lambda 함수를 생성합니다.

### 2. 코드 작성

boto3를 사용하여 SQS에 접근합니다. `receive_message`의 `MaxNumberOfMessages`는 최대 10개이므로, 반복적으로 메시지를 받아 처리하는 구조로 작성했습니다.

```python
# lambda_function.py
import json
import boto3
from datetime import datetime, timedelta

# 핸들러 밖에서 초기화 - 웜 스타트 시 재사용되어 성능 향상
sqs = boto3.client('sqs')

DLQ_URL = 'https://sqs.ap-northeast-2.amazonaws.com/123456789/my-dlq'
SQS_URL = 'https://sqs.ap-northeast-2.amazonaws.com/123456789/my-queue'

def lambda_handler(event, context):
    yesterday = (datetime.now() - timedelta(days=1)).strftime('%Y-%m-%d')
    processed_count = 0

    while True:
        # 최대 10개씩 메시지 수신
        response = sqs.receive_message(
            QueueUrl=DLQ_URL,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=5
        )
        messages = response.get('Messages', [])

        if not messages:
            break

        for message in messages:
            # 전일자 메시지만 처리 (메시지 본문에 날짜 포함된 경우)
            if yesterday in message['Body']:
                # 원래 큐로 메시지 전송
                sqs.send_message(
                    QueueUrl=SQS_URL,
                    MessageBody=message['Body']
                )
                # DLQ에서 메시지 삭제
                sqs.delete_message(
                    QueueUrl=DLQ_URL,
                    ReceiptHandle=message['ReceiptHandle']
                )
                processed_count += 1

    return {
        'statusCode': 200,
        'body': json.dumps({'processed': processed_count})
    }
```

### 3. 권한 설정

Lambda 역할에 SQS 정책을 추가합니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:GetQueueAttributes"
            ],
            "Resource": "arn:aws:sqs:ap-northeast-2:123456789:my-dlq"
        },
        {
            "Effect": "Allow",
            "Action": "sqs:SendMessage",
            "Resource": "arn:aws:sqs:ap-northeast-2:123456789:my-queue"
        }
    ]
}
```

### 4. 제한 시간 설정

메시지가 많으면 기본 3초로는 부족합니다. **30초 이상**으로 설정합니다.

### 5. EventBridge 트리거

매일 새벽 6시에 실행되도록 EventBridge 규칙을 생성합니다.

```
cron(0 21 * * ? *)  # UTC 21시 = KST 06시
```

## AWS Redrive API 활용

2023년 6월부터 AWS에서 DLQ Redrive API를 공식 지원합니다. 새로 구현한다면 이 API를 사용하는 것이 더 간단합니다.

### 제공되는 API

| API | 설명 |
|-----|------|
| `StartMessageMoveTask` | DLQ에서 메시지 이동 시작 |
| `CancelMessageMoveTask` | 이동 작업 취소 |
| `ListMessageMoveTasks` | 최근 10개 이동 작업 조회 |

### 목적지 지정

`DestinationArn` 파라미터로 목적지를 지정할 수 있습니다:
- **생략 시**: 원래 소스 큐로 자동 이동
- **지정 시**: 지정한 큐로 이동 (다른 큐로 보낼 때 유용)

### CLI 사용 예시

```bash
# Redrive 시작
aws sqs start-message-move-task \
    --source-arn arn:aws:sqs:ap-northeast-2:123456789:my-dlq

# 응답
{
    "TaskHandle": "eyJ0YXNrSWQiOiI4ZGJmNjBiMy00MmUwLTQz..."
}

# 상태 확인
aws sqs list-message-move-tasks \
    --source-arn arn:aws:sqs:ap-northeast-2:123456789:my-dlq

# 응답
{
    "Results": [
        {
            "Status": "COMPLETED",
            "SourceArn": "arn:aws:sqs:...:my-dlq",
            "ApproximateNumberOfMessagesMoved": 150,
            "ApproximateNumberOfMessagesToMove": 150,
            "StartedTimestamp": 1684135792239
        }
    ]
}
```

### 속도 제한

`MaxNumberOfMessagesPerSecond` 파라미터로 이동 속도를 제어할 수 있습니다 (최대 500/초).

```bash
aws sqs start-message-move-task \
    --source-arn arn:aws:sqs:...:my-dlq \
    --max-number-of-messages-per-second 100
```

### Lambda에서 Redrive API 사용

```python
import boto3

sqs = boto3.client('sqs')

def lambda_handler(event, context):
    response = sqs.start_message_move_task(
        SourceArn='arn:aws:sqs:ap-northeast-2:123456789:my-dlq',
        # DestinationArn 생략 시 원래 소스 큐로 이동
        # DestinationArn='arn:aws:sqs:...:other-queue',  # 다른 큐로 보낼 때
        MaxNumberOfMessagesPerSecond=100
    )

    return {
        'statusCode': 200,
        'taskHandle': response['TaskHandle']
    }
```

### 필요 권한

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sqs:StartMessageMoveTask",
                "sqs:CancelMessageMoveTask",
                "sqs:ListMessageMoveTasks",
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:GetQueueAttributes"
            ],
            "Resource": "arn:aws:sqs:ap-northeast-2:123456789:my-dlq"
        },
        {
            "Effect": "Allow",
            "Action": "sqs:SendMessage",
            "Resource": "arn:aws:sqs:ap-northeast-2:123456789:my-queue"
        }
    ]
}
```

> **참고**: 목적지 큐에 대한 `sqs:SendMessage` 권한이 필수입니다. SSE(서버 측 암호화) 큐의 경우 KMS 권한도 필요합니다.

## 직접 구현 vs Redrive API

| 항목 | 직접 구현 | Redrive API |
|------|----------|-------------|
| 커스텀 로직 | 날짜 필터링 등 가능 | 전체 메시지만 (필터링 불가) |
| 목적지 | 자유롭게 지정 | 원래 큐 또는 DestinationArn으로 지정 |
| 복잡도 | 코드 관리 필요 | 단순 API 호출 |
| 속도 제어 | 직접 구현 필요 | MaxNumberOfMessagesPerSecond (최대 500/초) |
| 진행 상태 | 직접 로깅 | ListMessageMoveTasks로 조회 |

**선택 기준:**
- **직접 구현**: 날짜/조건별 필터링, 메시지 변환 등 커스텀 로직 필요 시
- **Redrive API**: DLQ 전체 메시지를 그대로 원복할 때 (대부분의 경우)

## 정리

이 작업으로 DLQ 메시지 처리가 자동화되었습니다.

- Lambda 사용 시 **콜드 스타트**를 고려해야 하지만, 배치 작업에서는 큰 문제가 아닙니다
- 2023년 6월부터 **AWS Redrive API**가 제공되어 더 간단하게 구현할 수 있습니다
- 커스텀 로직이 필요한 경우에만 직접 구현을 고려하세요
