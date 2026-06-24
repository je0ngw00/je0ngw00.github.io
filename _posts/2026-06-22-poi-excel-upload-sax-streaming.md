---
title: "대용량 엑셀 처리 — 업로드편: XSSF OOM과 POI SAX 스트리밍 읽기"
date: 2026-06-22 00:00:00 +0900
categories: [Development, Spring]
tags: [poi, excel, sax, streaming, outofmemory, kotlin]
---

최근 어드민 기능을 개발하다 보니 엑셀 업로드를 다시 손볼 일이 있었습니다. 관리자 시스템에는 회원 일괄 등록, 정산 데이터 적재처럼 사용자가 채운 엑셀을 받아 DB로 옮기는 기능이 빠지지 않는데, 몇백 행짜리는 `WorkbookFactory.create()` 한 줄로 끝납니다. 문제는 행이 수만을 넘어가면서부터입니다. 같은 코드가 `OutOfMemoryError`로 죽기 시작합니다.

파일은 몇 MB밖에 안 되는데 왜 수백 MB 힙이 부족할까. 이번에는 이 질문을 그냥 넘기지 않고 PoC로 끝까지 재현하면서 힙을 직접 재봤습니다. 코드는 [kotlin-excel-streaming-s3](https://github.com/je0ngw00/kotlin-excel-streaming-s3)에 정리해 뒀습니다. 결론부터 말하면 범인은 파일 크기가 아니라 **POI가 엑셀을 메모리에 올리는 방식**이었습니다.

## XSSF는 시트 전체를 객체로 올린다

POI로 xlsx를 읽는 가장 흔한 코드는 이렇습니다.

```kotlin
// 행이 수만을 넘으면 여기서 터진다
WorkbookFactory.create(inputStream).use { wb ->
    val sheet = wb.getSheetAt(0)
    for (row in sheet) { /* 셀 읽어서 DB 적재 */ }
}
```

`WorkbookFactory`는 확장자를 보고 구현체를 고르는데, xlsx면 `XSSFWorkbook`을 만들고 이 방식은 시트 전체를 객체 모델로 메모리에 올립니다. 행 하나, 셀 하나가 전부 자바 객체로 만들어지고, 거기에 중복 문자열을 모아두는 공유 문자열 테이블(SharedStringsTable)과, 압축을 푼 XML을 파싱해 만든 객체들이 더해집니다. 그래서 디스크의 몇 MB짜리 파일이 힙에서는 수십 배로 불어납니다. 힙을 키워도 동시에 두세 명이 업로드하면 같은 증상이 다시 납니다. 근본 원인이 "한 번에 다 올린다"는 데 있기 때문입니다.

말로만 "터진다"고 하면 와닿지 않아서, 힙을 일부러 작게 잡고 행 수를 늘려가며 실제로 어디서 죽는지 재봤습니다.

## 힙을 256MB로 묶고 행을 늘리자 XSSF가 터졌다

`-Xmx256m`으로 띄운 뒤, 행 수별로 만든 엑셀을 (1) XSSF 전체 적재 방식과 (2) 뒤에서 설명할 SAX 방식으로 각각 읽으면서 피크 힙을 기록했습니다. DB 적재는 빼고 읽기 자체의 메모리만 봤습니다. 인메모리 H2에 행을 쌓으면 그게 또 힙을 먹어 측정이 흐려지기 때문입니다.

| 행 수 | XSSF 전체 적재 | SAX 스트리밍 |
|---|---|---|
| 5만 | 176MB | 151MB |
| 10만 | **OOM** | 155MB |
| 25만 | **OOM** | 139MB |
| 50만 | 미측정 | 163MB |

XSSF는 5만 행은 버티지만 10만 행부터 256MB 천장을 못 넘고 OOM이 났습니다(이미 10만에서 터지니 50만은 따로 재지 않았습니다). 반면 SAX는 5만이든 50만이든 140~160MB에서 평탄했습니다. 행을 10배 늘려도 힙이 거의 그대로라는 게 핵심입니다. 힙을 키우는 건 OOM이 나는 행 수를 잠깐 미루는 것뿐이고, 기울기 자체를 눕히려면 읽는 방식을 바꿔야 했습니다.

## SAX 이벤트 방식으로 한 행씩 읽기

POI에는 전체를 올리지 않고 `<row>`를 만날 때마다 콜백을 받는 이벤트 API가 있습니다. `XSSFReader`로 시트 XML 스트림을 열고, `XSSFSheetXMLHandler`에 핸들러를 물려 행 단위로 받는 구조입니다.

```kotlin
class StreamingXlsxReader {
    fun read(xlsxPath: Path, onRow: (Map<String, String?>) -> Unit) {
        OPCPackage.open(xlsxPath.toFile()).use { pkg ->
            val strings = ReadOnlySharedStringsTable(pkg)
            val reader = XSSFReader(pkg)
            val sheets = reader.sheetsData as XSSFReader.SheetIterator
            while (sheets.hasNext()) {
                sheets.next().use { sheet ->
                    val xml = XMLHelper.newXMLReader()
                    xml.contentHandler = XSSFSheetXMLHandler(
                        reader.stylesTable, strings, RowHandler(onRow), false,
                    )
                    xml.parse(InputSource(sheet))
                }
            }
        }
    }
}
```

콜백을 받는 핸들러는 "현재 행"만 들고 있다가 행이 끝나면 넘기고 비웁니다. 업무 데이터로 보면 메모리에 남는 건 현재 행뿐이고(스타일·공유 문자열 같은 전역 구조는 따로 남습니다), 위 표의 평탄한 선이 이렇게 나옵니다. 라이브러리를 갈아끼운 게 아니라 같은 POI 안에서 읽는 방식만 바꾼 것입니다.

## SAX를 쓰며 걸린 함정 셋

편해 보이지만, 전체 모델이 주던 편의를 잃기 때문에 직접 메워야 하는 구멍이 몇 개 있었습니다.

**1. 빈 셀이 컬럼을 민다.** SAX `cell()` 콜백은 시트 XML에 실제로 들어 있는 셀에 대해서만 불립니다. 중간에 빈 칸이 있으면 그 셀은 XML에서 빠져 있어, 건너뛰고 다음 값이 당겨져서 `A, (빈칸), C`가 `A, C`로 들어옵니다. 단순히 순서대로 담으면 컬럼이 한 칸씩 밀립니다. 그래서 셀의 좌표(`A1`, `C1`)를 보고 열 인덱스를 복원해야 합니다.

```kotlin
override fun cell(cellRef: String?, value: String?, comment: XSSFComment?) {
    // "C5" → 열 인덱스 2. 건너뛴 빈 칸 자리는 null로 채운다.
    val col = cellRef?.let { CellReference(it).col.toInt() } ?: current.size
    while (current.size < col) current.add(null)
    current.add(value)
}
```

처음엔 이걸 모르고 순서대로 담았다가, 중간 열이 비어 있는 파일에서 이메일 자리에 이름이 들어가는 식으로 데이터가 어긋났습니다. 좌표 기반으로 바꾸고 나서야 정렬이 맞았습니다.

**2. 공유 문자열 테이블은 여전히 메모리에 있다.** SAX가 묶어주는 건 "행·셀 객체"이지 문자열 전체가 아닙니다. `ReadOnlySharedStringsTable`은 이름 그대로 읽기 전용이라 가볍긴 하지만, 고유 문자열이 수백만 개면 이 테이블이 그대로 힙에 올라갑니다. SAX로 바꿨다고 문자열 메모리까지 공짜가 되는 건 아니라는 점은 알아둬야 합니다. 이게 병목이라면 공유 문자열을 디스크로 빼는 `poi-shared-strings` 같은 보조 라이브러리도 있습니다.

**3. xlsx는 ZIP이라 스트림만으로는 깔끔하지 않다.** xlsx는 내부적으로 XML 여러 개를 묶은 ZIP(OPC 패키지)입니다. `OPCPackage.open(InputStream)`으로 스트림에서 바로 열면 POI가 ZIP 전체를 힙에 버퍼링하지만, 파일로 열면 디스크에서 랜덤 액세스로 필요한 파트만 찾아 읽어 메모리를 덜 씁니다. POI 문서도 파일 경로 쪽이 가볍다고 못 박습니다. 그래서 업로드를 받으면 일단 로컬 임시파일로 떨어뜨린 뒤 그 파일을 SAX로 읽는 경로가 가장 말썽이 없었습니다.

## 측정을 믿으려고 신경 쓴 것

그래프 한 장으로 "평탄 vs 우상향"을 보여주려면 측정 자체가 흔들리지 않아야 했습니다. 세 가지를 정리해 뒀습니다.

- **순수 읽기만 측정**: DB 적재를 넣으면 인메모리 H2가 행을 쌓아 힙이 올라갑니다. 읽기 방식의 차이를 보려고 적재는 뺐습니다.
- **행을 늘리기보다 힙을 조이기**: 100만 행까지 안 가도 됩니다. `-Xmx`를 256m으로 조이면 10만 행만으로 XSSF가 천장에 부딪힙니다. 큰 힙이면 XSSF가 더 버티다 터질 뿐, 기울기 차이는 같습니다.
- **한계도 적어두기**: 피크는 `totalMemory - freeMemory`로 쟀는데, 아직 수거되지 않은 가비지가 섞일 수 있어 절대값은 다소 높게 잡힙니다. 100ms 간격이라 더 짧은 스파이크는 놓칠 수 있고요. 그래도 "평탄이냐 우상향이냐"를 가르는 데는 충분했습니다. 더 정밀히 보려면 GC 로그(`-Xlog:gc`)나 JFR를 같이 보는 편이 낫습니다.

## 돌아보며

업로드가 터진 건 파일이 커서가 아니라 전체를 한 번에 메모리에 올렸기 때문이었습니다. 적은 데이터에서 잘 돌던 `WorkbookFactory.create()`가 그대로 발목을 잡았고, 해결은 라이브러리를 바꾸는 게 아니라 같은 POI 안에서 읽는 방식을 이벤트 기반으로 바꾸는 것이었습니다. 기능이 더 단순해도 괜찮다면 fastexcel 같은 경량 라이브러리도 선택지였지만, 스타일·수식까지 다루는 표준이 필요해 POI의 SAX를 택했습니다.

두 가지가 남았습니다. "메모리에 다 올리지 않는다"는 원칙이 쓰기뿐 아니라 읽기에도 똑같이 통한다는 점, 그리고 추측만 하던 걸 직접 재보니 어디까지 버티는지가 숫자로 보였다는 점입니다. 힙을 256m으로 묶고 10만 행에서 갈리는 표를 한 번 보고 나니, "힙을 키우자"는 임시 처방으로는 돌아가지 않게 됐습니다.

다음 편에서는 반대 방향, DB를 조금씩 읽어 큰 엑셀을 만들어 내보내는 쪽을 다룹니다. 쓰기는 SXSSF로 메모리를 잡고, 서버 디스크와 HTTP 타임아웃까지 가면 S3로 흘려보내는 이야기입니다.
