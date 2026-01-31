---
title: "Querydsl 대용량 엑셀 다운로드 시 OutOfMemory 개선하기"
date: 2023-05-26 10:00:00 +0900
categories: [Development, JPA]
tags: [querydsl, excel, outofmemory, performance, java, pagination]
---

## 개요

회사의 비즈니스 특성상 월말 정산 작업이 있는데, 엑셀을 수기로 다운받아 처리하는 업무가 많습니다. 따라서 월말에 대량의 엑셀 다운로드가 이루어지고, 그에 따라 OutOfMemory 메시지를 받는 경우가 많았습니다.

일단 조회 범위를 줄이는 방법으로 선 조치 후, 해당 내용에 대하여 개선한 내용을 남깁니다.

## 문제점

말 그대로 OutOfMemory라는 메시지를 받았으니, 엑셀 다운로드할 때 어느 부분이 메모리를 잡아먹는지 알아봤습니다.

크게 두 가지 문제점이 있었습니다.

### 조회

```java
List<Dto> resultList = getQuerydsl().....from(table).fetch()
```

- 대량 데이터를 조회하고 다운로드할 때 다운로드 대상 전체를 메모리에 모두 적재 후 처리하고 있었습니다.
- Heap Size를 늘려도 동시 요청 시 동일 증상이 계속 발생할 것입니다.

### 다운로드 진행

```java
public <T> InputStreamResource exportExcel(List<T> exportList, Class<T> exportClass) {
  try (ByteArrayOutputStream out = new ByteArrayOutputStream();
      SXSSFWorkbook workbook = ExcelWriter.builder()
        .type(exportClass)
        .write(exportList)) {
    workbook.write(out);
    workbook.dispose();
    return new InputStreamResource(new ByteArrayInputStream(out.toByteArray()));
  } catch (IOException e) {
    throw new ExportExcelException();
  }
}
```

- 엑셀 파일을 생성 후 메모리에 모두 적재하고 다운을 진행하고 있었습니다.

## 해결

### 조회

조회는 QueryDsl을 사용하고 있고, 조회한 데이터를 메모리에 올리지 않을 수 있는 방법을 찾아보았습니다. 크게 3가지 방법을 찾았습니다.

#### 방법 1: `iterate()` 메서드 사용

```java
CloseableIterator<Dto> resultList = getQuerydsl().....from(table).iterate()
```

`iterate()` 메서드를 사용하여 `CloseableIterator`를 받는 경우, 데이터를 반복적으로 조회하면서 필요한 만큼 데이터를 로드합니다. 데이터를 한 번에 메모리에 올리는 것이 아니라, 필요한 만큼 가져와서 처리합니다.

- **장점**: 대용량 데이터 처리에 특화, 메모리 사용량 최소화
- **단점**: 반드시 `close()`를 호출해야 하며, 그렇지 않으면 Connection Leak 발생

#### 방법 2: `stream()` 메서드 사용

```java
Stream<Dto> resultList = getQuerydsl().....from(table).stream()
```

Java 8의 Stream API와 유사한 방식으로 데이터를 스트림으로 처리합니다.

- **장점**: 코드의 가독성과 유연성이 좋음
- **단점**: 일부 데이터만 처리하는 경우에 더 적합

#### 방법 3: 청크 단위 페이징 처리 (권장)

위 두 방법도 좋지만, 실무에서는 **청크(Chunk) 단위로 페이징 처리**하는 방식이 더 안정적입니다. `offset`과 `limit`을 사용하여 데이터를 작은 덩어리로 나눠서 가져옵니다.

```java
public void processLargeDataInChunks(HttpServletResponse response) {
    final int CHUNK_SIZE = 1000;
    long offset = 0;

    try (SXSSFWorkbook workbook = new SXSSFWorkbook(100)) { // 메모리에 100개 row만 유지
        Sheet sheet = workbook.createSheet("Data");
        createHeaderRow(sheet);

        int rowIndex = 1;
        List<Dto> chunk;

        do {
            // 청크 단위로 데이터 조회
            chunk = queryFactory
                .select(new QDto(...))
                .from(entity)
                .where(...)
                .offset(offset)
                .limit(CHUNK_SIZE)
                .fetch();

            // 조회된 청크 데이터를 엑셀에 작성
            for (Dto dto : chunk) {
                Row row = sheet.createRow(rowIndex++);
                row.createCell(0).setCellValue(dto.getField1());
                row.createCell(1).setCellValue(dto.getField2());
                // ...
            }

            offset += CHUNK_SIZE;

            // 청크 처리 후 명시적으로 GC 힌트 (선택사항)
            chunk.clear();

        } while (chunk.size() == CHUNK_SIZE); // 더 이상 데이터가 없을 때까지 반복

        // 스트리밍 방식으로 응답
        response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        response.setHeader("Content-Disposition", "attachment; filename=\"data.xlsx\"");
        workbook.write(response.getOutputStream());
        workbook.dispose();
    }
}
```

**청크 사이즈 선정 기준:**
- 너무 작으면: DB 쿼리 횟수 증가로 인한 오버헤드
- 너무 크면: 메모리 압박과 긴 트랜잭션 발생
- **권장**: 500 ~ 2000 사이에서 시작하여 모니터링 후 조정

#### Spring Data JPA의 Slice 활용

Spring Data JPA를 함께 사용한다면 `Slice`를 활용한 방법도 있습니다:

```java
public void processWithSlice(HttpServletResponse response) {
    Pageable pageable = PageRequest.of(0, 1000);

    try (SXSSFWorkbook workbook = new SXSSFWorkbook(100)) {
        Sheet sheet = workbook.createSheet("Data");
        int rowIndex = 0;

        Slice<Dto> slice;
        do {
            slice = repository.findAllByCondition(condition, pageable);

            for (Dto dto : slice.getContent()) {
                Row row = sheet.createRow(rowIndex++);
                // 데이터 작성
            }

            pageable = slice.nextPageable();

        } while (slice.hasNext());

        // 응답 처리
        workbook.write(response.getOutputStream());
        workbook.dispose();
    }
}
```

`Slice`는 `Page`와 달리 전체 count 쿼리를 수행하지 않아 대용량 데이터에서 더 효율적입니다.

### 다운로드 진행

```java
// Excel 파일을 스트림으로 전송하는 StreamingOutput 구현
StreamingOutput streamingOutput = outputStream -> {
    try {
        workbook.write(outputStream);
        outputStream.flush();
    } finally {
        workbook.close();
        resultList.close(); // CloseableIterator 닫기
    }
};

// 파일 다운로드 설정
response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
response.setHeader("Content-Disposition", "attachment; filename=\"data.xlsx\"");

// StreamingOutput을 사용하여 데이터를 클라이언트로 전송
try {
    streamingOutput.write(response.getOutputStream());
} catch (IOException e) {
    e.printStackTrace();
}
```

`StreamingOutput`을 구현하여 Excel 파일을 스트림으로 전송했습니다. 이 방법을 사용하면 Excel 파일을 메모리에 올리지 않고 바로 다운로드할 수 있습니다.

## SXSSFWorkbook 사용 시 주의사항

대용량 엑셀 생성 시 `XSSFWorkbook` 대신 `SXSSFWorkbook`을 사용해야 합니다:

```java
// 메모리에 유지할 row 수 지정 (기본값: 100)
SXSSFWorkbook workbook = new SXSSFWorkbook(100);
```

- `SXSSFWorkbook`은 지정된 row 수만 메모리에 유지하고 나머지는 임시 파일로 flush합니다.
- 작업 완료 후 반드시 `workbook.dispose()`를 호출하여 임시 파일을 정리해야 합니다.

## 정리

| 방법 | 메모리 효율 | 구현 복잡도 | 권장 상황 |
|------|------------|-------------|----------|
| `iterate()` | 높음 | 중간 | 단순 순회 처리 |
| `stream()` | 중간 | 낮음 | 함수형 처리 필요 시 |
| 청크 페이징 | 높음 | 중간 | 대용량 배치 처리, 진행률 표시 필요 시 |
| Slice | 높음 | 낮음 | Spring Data JPA 사용 시 |

대용량 데이터를 처리할 때는 **청크 단위 페이징**이 가장 안정적입니다. 진행 상황을 로깅하거나 중간에 실패 시 재시작 지점을 파악하기도 용이합니다.

마지막으로 `CloseableIterator`나 `Stream`을 사용할 경우 반드시 리소스를 닫아주어야 합니다. `@Transactional` 애노테이션을 사용하여 자원을 관리하는 방법도 있지만, 명시적으로 `close()`를 호출하는 것이 더 안전합니다.
