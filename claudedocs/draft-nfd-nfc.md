---
title: "DB에서 보이는데 검색이 안 된다? — macOS 한글 파일명과 Unicode NFD/NFC"
date: 2026-02-28 00:00:00 +0900
categories: [Development, Backend]
tags: [kotlin, jpa, unicode, mariadb, debugging, spring-boot]
---

## 현상

QA 환경에서 버그 리포트가 올라왔습니다.

> "첨부파일명으로 검색하면 결과가 나오지 않습니다"

게시글 제목, 내용, 작성자명 검색은 모두 정상이었습니다. **첨부파일명 검색만** 동작하지 않았습니다.

## 첫 번째 가설: Hibernate dialect 문제

QA 로그를 확인하니 Hibernate가 생성한 SQL이 이상했습니다.

```sql
WHERE file_name LIKE replace('%디지털%', '\\', '\\\\')
```

Hibernate 6의 MariaDBDialect이 LIKE 절을 `replace()`로 감싸고 있었습니다. "이게 원인이구나"라고 생각했습니다.

CustomMariaDBDialect를 만들어 `replace()` 래핑을 제거하려는 시점에, 혹시나 하는 마음에 DB에서 직접 SQL을 실행해봤습니다.

```sql
SELECT * FROM post_attachment
WHERE file_name LIKE '%디지털%';
```

**결과: 0건.**

DB에서 직접 LIKE 쿼리를 날려도 검색이 안 됐습니다. Hibernate 문제가 아니었습니다.

## 눈에 보이는 것을 의심하다

DB 클라이언트에서 해당 레코드를 조회하면 파일명이 정상적으로 보입니다.

```
file_name: 디지털 비즈니스 모델은 어떻게 완성되는가.jpg
```

분명히 "디지털"이라는 글자가 있는데 `LIKE '%디지털%'`에 안 걸립니다. HEX 값을 확인해봤습니다.

```sql
SELECT HEX(file_name) FROM post_attachment WHERE post_attachment_id = ?;
```

```
-- 실제 저장된 값 (NFD)
E18483E185B5E18488E185B5E18490E185A5E186AF ...
-- ㄷ+ㅣ  ㅈ+ㅣ  ㅌ+ㅓ+ㄹ ...

-- 키보드로 입력한 "디지털" (NFC)
EB9494ECA780ED8380 ...
-- 디      지      털
```

같은 "디지털"인데 바이트가 완전히 달랐습니다.

## Unicode NFD와 NFC

유니코드는 같은 한글을 두 가지 방식으로 표현할 수 있습니다.

| 형태 | 정식 명칭 | 구성 | "디"의 HEX |
|------|----------|------|-----------|
| **NFC** | Normalization Form Composed | 완성형 한 글자 | `EB9494` |
| **NFD** | Normalization Form Decomposed | 초성+중성+종성 분리 | `E18483E185B5` |

사람 눈에는 동일하게 보이지만, 컴퓨터 입장에서는 완전히 다른 바이트열입니다. 당연히 LIKE 비교도 실패합니다.

### 왜 NFD가 DB에 들어갔나

**macOS의 파일시스템(APFS/HFS+)은 한글 파일명을 NFD로 저장합니다.**

서비스의 데이터 입력 경로를 비교해보면 이유가 명확해집니다.

```
게시글 제목/내용:
  사용자 키보드 입력 → 브라우저 에디터 → API → DB
  (키보드 입력 = 항상 NFC ✅)

첨부파일명:
  macOS에서 파일 선택 → File API → 업로드 서버 → 파일명 반환 → API → DB
                         ↑
                   macOS가 NFD로 반환 ❌
```

키보드로 직접 타이핑한 텍스트(제목, 내용, 댓글)는 항상 NFC입니다. OS 파일시스템을 거치지 않기 때문입니다. 반면 파일명은 브라우저의 `File.name` API가 OS에서 읽어오는데, macOS가 NFD로 저장하고 있으니 NFD 문자열이 그대로 업로드 서버를 거쳐 DB까지 전달됩니다.

저희 서비스는 파일을 직접 업로드하는 것이 아니라 외부 업로드 서버에서 링크와 파일명을 받는 구조였습니다. 그래서 처음에 "직접 업로드가 아닌데 왜?"라는 의문이 있었지만, 파일명 자체는 프론트엔드에서 `File.name`으로 읽어 전달하므로 동일한 문제가 발생했습니다.

## 해결 전 검증

코드를 수정하기 전에, NFC로 바꾸면 검색이 되는지 SQL로 먼저 검증했습니다.

```sql
-- 1. 기존 NFD 값 확인
SELECT HEX(file_name) FROM post_attachment WHERE post_attachment_id = ?;

-- 2. NFC로 변환하여 업데이트
UPDATE post_attachment
SET file_name = '디지털 비즈니스 모델은 어떻게 완성되는가.jpg'  -- NFC
WHERE post_attachment_id = ?;

-- 3. 검색 테스트
SELECT * FROM post_attachment WHERE file_name LIKE '%디지털%';
-- → 결과 정상 반환 ✅

-- 4. 롤백
UPDATE post_attachment
SET file_name = UNHEX('원래HEX값')
WHERE post_attachment_id = ?;
```

검증 완료. NFC 정규화가 해답이었습니다.

## 해결: JPA AttributeConverter

파일명 NFC 정규화를 어디에 적용할지 세 가지 선택지가 있었습니다.

| 방식 | 장점 | 단점 |
|------|------|------|
| API Mapper에서 변환 | 간단 | 파편화, 새 API 추가 시 누락 가능 |
| Entity setter에서 변환 | 확실 | setter 오염, 단일 책임 위반 |
| **JPA AttributeConverter** | 엔티티 레벨 보장, 관심사 분리 | 별도 클래스 필요 |

AttributeConverter를 선택했습니다. DB 저장 시점에 자동으로 NFC 정규화가 적용되므로, API가 추가되더라도 누락될 걱정이 없습니다.

```kotlin
@Converter
class NfcNormalizeConverter : AttributeConverter<String?, String?> {

    override fun convertToDatabaseColumn(attribute: String?): String? {
        return attribute?.let { Normalizer.normalize(it, Normalizer.Form.NFC) }
    }

    override fun convertToEntityAttribute(dbData: String?): String? {
        return dbData
    }
}
```

엔티티에 적용합니다.

```kotlin
@Entity
class PostAttachment(

    @Convert(converter = NfcNormalizeConverter::class)
    @Column(name = "file_name", length = 200)
    var fileName: String? = null,
)
```

댓글 첨부파일 엔티티(`PostCommentAttachment`)에도 동일하게 적용했습니다.

파일명을 저장하는 엔티티가 늘어난다면 개별 `@Convert` 방식은 새 엔티티 추가 시 누락 위험이 생깁니다. JPA에서 권장하는 패턴은 래퍼 타입을 만들고 `autoApply = true`를 함께 쓰는 방식입니다.

```kotlin
data class FileName(val value: String)

@Converter(autoApply = true)
class FileNameConverter : AttributeConverter<FileName?, String?> {

    override fun convertToDatabaseColumn(attribute: FileName?): String? {
        return attribute?.value?.let {
            if (Normalizer.isNormalized(it, Normalizer.Form.NFC)) it
            else Normalizer.normalize(it, Normalizer.Form.NFC)
        }
    }

    override fun convertToEntityAttribute(dbData: String?): FileName? =
        dbData?.let { FileName(it) }
}
```

엔티티에서 `FileName` 타입을 쓴 필드는 `@Convert` 어노테이션 없이 자동으로 NFC 정규화가 적용됩니다. `status: String`이나 `email: String` 같은 다른 필드에는 영향을 주지 않습니다.

```kotlin
@Entity
class PostAttachment(
    @Column(name = "file_name", length = 200)
    var fileName: FileName? = null,  // @Convert 없이 자동 적용
)
```

대신 `FileName` 타입으로 선언하면 접근 시 `.value`가 필요하다는 점은 고려해야 합니다. 엔티티가 적다면 개별 `@Convert`로도 충분하고, 여러 팀이 공유하는 공통 모듈이나 신규 프로젝트라면 래퍼 타입 방식이 의도를 더 명확히 표현합니다.

### 검색 키워드에도 방어적 적용

검색 키워드는 키보드 입력이라 보통 NFC이지만, 방어적으로 검색 유틸리티에도 NFC 정규화를 추가했습니다. `Normalizer.normalize()`는 이미 NFC인 문자열에 적용해도 그대로 반환하므로(멱등성) 일반 검색 키워드에는 영향이 없습니다.

```kotlin
object SearchUtils {

    fun sanitizeSearchKeyword(keyword: String): String {
        return Normalizer.normalize(keyword, Normalizer.Form.NFC)
            .replace("\\", "\\\\")
            .replace("%", "\\%")
            .replace("_", "\\_")
            .trim()
    }
}
```

## 테스트

```kotlin
@Test
fun `NFD 한글을 NFC로 정규화한다`() {
    val nfdFileName = Normalizer.normalize("디지털 비즈니스.jpg", Normalizer.Form.NFD)

    val actual = convertToDatabaseColumn(nfdFileName)

    assertThat(actual).isEqualTo("디지털 비즈니스.jpg")
    assertThat(Normalizer.isNormalized(actual, Normalizer.Form.NFC)).isTrue()
}

@Test
fun `NFD와 NFC의 바이트가 다름을 검증한다`() {
    val nfcText = "디지털"
    val nfdText = Normalizer.normalize("디지털", Normalizer.Form.NFD)

    assertThat(nfdText).isNotEqualTo(nfcText)

    val actual = convertToDatabaseColumn(nfdText)
    assertThat(actual).isEqualTo(nfcText)
}
```

## 기존 데이터 마이그레이션

코드를 배포해도 이미 NFD로 저장된 기존 데이터는 여전히 검색이 안 됩니다. DB 마이그레이션이 필요합니다.

MariaDB는 네이티브 NFC 변환 함수가 없으므로 애플리케이션 레벨에서 처리해야 합니다. AttributeConverter가 이미 적용되어 있으므로, 엔티티를 조회해서 `save()`만 호출하면 Converter가 자동으로 NFC 변환을 처리합니다.

```kotlin
val attachments = postAttachmentRepository.findAll()
attachments.forEach { attachment ->
    attachment.fileName?.let { name ->
        if (!Normalizer.isNormalized(name, Normalizer.Form.NFC)) {
            attachment.fileName = Normalizer.normalize(name, Normalizer.Form.NFC)
        }
    }
}
postAttachmentRepository.saveAll(attachments)
```

## 디버깅하면서 확인한 것들

이 버그를 추적하면서 몇 가지를 다시 확인했습니다.

첫 번째는 DB 클라이언트에서 정상으로 보이는 데이터를 믿으면 안 된다는 점입니다. 렌더링된 텍스트가 아니라 HEX 값을 봐야 실제 저장된 바이트를 알 수 있습니다. 두 번째는 가설을 빠르게 반증하는 습관입니다. Hibernate dialect 문제라고 확신한 상태에서 SQL 한 줄을 직접 실행해보지 않았다면, 엉뚱한 CustomDialect를 만드는 데 시간을 쏟았을 겁니다. 세 번째는 데이터 입력 경로 추적입니다. 같은 한글인데 제목 검색은 되고 파일명 검색은 안 된다는 차이가 결국 원인을 찾는 실마리였습니다.

macOS 사용자만 이 문제를 겪습니다. Windows는 NFC를 사용하기 때문입니다. 서비스 사용자 중 macOS 비율이 높다면 조용히 쌓이는 버그가 될 수 있으니, 파일명을 다루는 곳에서는 한 번쯤 확인해볼 만합니다.
