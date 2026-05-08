---
title: "AI와 도메인을 설계하며 짚어간 세 단계"
date: 2026-05-08 00:00:00 +0900
categories: [Development, Architecture]
tags: [domain-design, ai-pair-programming, layered-decisions, java, sealed-class, functional-core]
---

138 파일을 손댔습니다. +9906 줄, -3765 줄. 그런데 화면에서 보이는 동작은 어제와 같습니다. 운영자가 정책을 등록하고 시스템이 그 정책을 평가해 차단하거나 허용하는 흐름은 그대로입니다.

처음에는 이 작업을 글로 쓸 가치가 있는지 망설였습니다. "요즘 AI가 뛰어나니 설계 방향만 있으면 다 할 수 있을 법한 내용이라 고민이 됩니다" 라는 자기 의심이 들었기 때문입니다. 동일한 동작을 다르게 다시 쓴 일은, AI 시대에는 굳이 사람이 시간을 들일 일이 아닐지도 모릅니다.

작업이 끝난 뒤 돌아보니, 이 일에서 사람이 한 일은 *AI에게 코드를 받아 쓴 일* 이 아니었습니다. 도메인 설계는 한 번의 결정으로 끝나지 않고, 층층이 쌓이는 결정의 단계로 진행됐습니다. 한 단계의 답이 다음 단계의 답을 좁히고, 그 단계를 순서대로 짚어가는 일을 AI와 함께 했습니다. 이 글은 그 단계가 어떤 모양이었는지에 대한 기록입니다.

본문에 등장하는 "정책", "외부 시스템 X", "외부 시스템 Y" 같은 표현은 모두 일반화한 명칭입니다. 핵심은 도메인의 이름이 아니라 결정의 모양에 있어, 코드 예시도 일반화한 형태로 옮겼습니다.

## 어제까지의 모양 — 단계가 흐트러진 결과

운영자는 한 화면에서 네 종류의 제한 규칙을 설정할 수 있었습니다. 부르기 쉽게 정책 A·B·C·D라고 부르겠습니다.

| 정책 | 평가 방식 |
|-----|---------|
| A | 어떤 수치 N 종류를 합산해 임계치 비교 |
| B | 특정 사용자와 특정 대상의 연결 여부 |
| C | 외부 시스템 X의 카운트가 임계치 이상인지 |
| D | 외부 시스템 X의 진척도와 상태 조합 |

처음 코드 모양은 자연스러웠습니다. 정책 4개니까 Checker 4개. 각 Checker가 자기 전용 Repository, 자기 전용 Configuration을 가졌습니다. 비슷하지만 다른 모양이었고, *비슷하지만 다른* 이 모든 문제의 시작이었습니다.

```java
// 어제까지의 PolicyA Checker (요약)
@Service
class PolicyAChecker {
    public Result check(Context ctx) {
        var rules = repo.findActiveBy(ctx.tenantId(), ctx.tab());
        for (var r : rules) {
            if (r.getSubType() == SUB_X)      { ... }
            else if (r.getSubType() == SUB_Y) { ... }
            else if (r.getSubType() == SUB_Z) { ... }
            else if (r.getSubType() == SUB_W) { ... }
        }
    }
}
```

이 코드 안에서는 데이터 모양, 책임 경계, 호출 시점이 한 클래스에 섞여 있습니다. *정책 D를 추가하려면 어디부터 손대야 하지* 가 매번 발생했습니다. 단계가 흐트러진다는 게 무슨 뜻인지를 그대로 보여주는 출발점이었습니다.

## 첫 번째 단계 — 데이터 모양

도메인을 설계할 때 첫 단계는 *이 N 개의 것이 같은 것이냐, 각자 다른 것이냐* 를 정하는 일입니다. 이걸 답하지 않으면 다음 단계가 의미를 잃습니다.

리팩토링을 처음 AI에게 맡겼을 때 받은 답은 자연스러운 답이었습니다. 공통 record 한 개와 `subType` enum으로 분류하는 모양, 그리고 enum으로 분기하는 dispatcher. 동작은 합니다. 운영도 됩니다.

그런데 *우리 도메인의 답인가* 부터 의심이 들었습니다. AI가 학습한 자료의 다수는 입문 수준의 예제와 튜토리얼이라서, 첫 답은 *유연한 한 가지 모양으로 N 가지를 처리하는 모범 답안* 에 가까운 경우가 많습니다. 그 답이 우리 도메인에 맞을지는 아무도 검증하지 않습니다.

다시 물었습니다.

> "정책 4개가 같은 인터페이스의 변종일까요, 각자 다른 종족일까요. 운영상 한 정책에서 다른 정책으로 갈아타는 일이 거의 없고, 새 정책이 추가되면 거의 항상 새로운 데이터 필드와 함께 나타납니다."

이 질문은 AI가 먼저 던지지 않았습니다. 이 질문을 던지자 답이 달라졌습니다.

자리잡은 모양은 sealed 계층이었습니다. 정책마다 자기 데이터를 자기 필드로 들고 있는 record가 됩니다.

```java
public sealed interface Rule
        permits RuleA, RuleB, RuleC, RuleD {

    Finding evaluate(EvaluationInputs inputs);
}

public record RuleD(
        Long ruleId,
        Integer threshold,
        Set<StatusCode> statusFilters
) implements Rule { ... }
```

`record`가 implicitly final이라 sealed의 permits 규칙을 자동으로 충족합니다. JDK 21의 switch pattern matching으로 분기하면 (단, `default` 절 없이 exhaustive하게 작성한 경우) 새 정책이 추가될 때 컴파일러가 누락된 분기를 잡아냅니다. 이 record를 보면 *이 정책이 무엇을 보고 무엇을 차단하는가* 가 한 화면에 들어옵니다. 운영자가 하는 말이 그대로 코드로 옮겨져 있는 모양입니다.

이 단계의 결정이 다음 단계를 좁힙니다. 종족이 다르다는 답이 자리잡으면, 각자 자기 책임을 진다는 다음 단계의 출발점이 정해집니다.

## 두 번째 단계 — 책임 경계

종족이 다르면, 다음 질문은 *각 종족이 알아야 할 것과 모르게 할 것* 입니다. 이게 책임 경계의 단계입니다.

처음 AI에게 sealed 계층까지 알려주고 *각 Rule.evaluate() 안에서 필요한 외부 데이터를 가져오세요* 라고 부탁하면, 각 Rule이 `ExternalClient`와 `Repository`를 인자로 받아 그 안에서 직접 호출하는 모양이 옵니다. 동작은 합니다. 그런데 Rule이 인프라를 알아버립니다. 도메인 모듈에 spring/feign 의존성이 따라옵니다. 단위 테스트의 mocking 비용이 늘어납니다. 무엇보다, 정책 4개가 같은 외부 호출을 4번 하는 일이 일어납니다.

다시 물었습니다.

> "Rule 은 외부 시스템을 모르는 채로 두고 싶습니다. 그럼 누가 부르나요."

받은 답은 두 번째 자연스러운 답이었습니다. *Dispatcher가 미리 다 부르고 그 결과를 정책에 넘긴다*. 동작합니다. 그런데 정책 1개만 활성화된 상황에서도 외부 호출 5개가 발생합니다. 정책 A만 활성화된 케이스가 압도적으로 많은데도 정책 D의 외부 호출까지 합니다.

다시 물었습니다.

> "정책 자신이 필요할 때만 부르되, 인프라는 모른 채로 부르려면 어떤 모양이어야 하나요."

자리잡은 모양은 *호출 시점을 정책이 결정* 하는 모양이었습니다. 정책이 외부에서 받는 것은 데이터가 아니라 *데이터를 가져오는 함수* 입니다.

```java
public record EvaluationInputs(
        BigDecimal targetAmount,
        OffsetDateTime now,
        OffsetDateTime periodStart,
        OffsetDateTime periodEnd,
        Supplier<Long> aggregateCount,
        Supplier<BigDecimal> aggregateAmount,
        Supplier<Integer> negativeCount,
        Supplier<Boolean> hasPriorMatch,
        Supplier<BigDecimal> externalProgress,
        Supplier<StatusCode> externalStatus
) { }
```

정책 코드는 여전히 `Supplier`만 봅니다. 인프라 의존성이 안 들어옵니다. 단위 테스트에서는 `Supplier` 자리에 고정된 값을 주는 함수를 넣으면 끝납니다.

이 단계의 결정이 또 다음 단계를 좁힙니다. 호출 시점이 정책에 있다면, 같은 호출이 두 번 일어나지 않도록 해야 한다는 다음 단계가 나타납니다.

## 세 번째 단계 — 호출 시점

`Supplier`만 가지고는 부족했습니다. 정책 두 개가 같은 데이터를 보면, 같은 외부 호출이 두 번 일어납니다.

다시 물었습니다.

> "Supplier인 척하면서 한 번만 평가되는 모양은 어떻게 만드나요."

자리잡은 답에는 이미 이름이 있었습니다. Gary Bernhardt가 2012년에 제안한 [Functional Core, Imperative Shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell) 패턴입니다. 정책(Rule)은 functional core, Dispatcher와 외부 호출은 imperative shell, 그리고 그 경계에서 호출을 미루고 한 번만 평가하는 작은 장치가 lazy한 supplier입니다. 이름이 붙기까지 시간이 걸렸지만, 이름을 알아본 뒤에는 이 모양이 자리잡는 게 빨랐습니다.

```java
public final class LazyValue<T> implements Supplier<T> {
    private final Supplier<T> supplier;
    private volatile boolean evaluated;
    private T value;

    private LazyValue(Supplier<T> supplier) {
        this.supplier = supplier;
    }

    public static <T> LazyValue<T> of(Supplier<T> supplier) {
        return new LazyValue<>(supplier);
    }

    public static <T> LazyValue<T> ofValue(T value) {
        return new LazyValue<>(() -> value);
    }

    @Override
    public T get() {
        if (!evaluated) {
            synchronized (this) {
                if (!evaluated) {
                    value = supplier.get();
                    evaluated = true;
                }
            }
        }
        return value;
    }
}
```

`volatile`과 동기화 블록으로 다중 스레드 평가에 대해서도 한 번 평가가 보장됩니다. 다만 알아둘 점은, `supplier`가 예외를 던지면 `evaluated`가 그대로 `false`로 남아 다음 호출 시 재시도된다는 것입니다. 실패까지 캐시하지 않는 동작이 의도와 맞는지 한번 점검해야 합니다. 단위 테스트에서 정책의 입력은 `LazyValue.ofValue(true)` 한 줄로 정리됩니다.

## 단계가 흐트러질 때 일어나는 일

세 단계는 *순서* 가 있어서 의미가 있었습니다. 한 단계의 결정 없이 다음 단계로 가면, 코드는 어제까지의 모양으로 돌아갑니다.

데이터가 *같은 것이냐 다른 것이냐* 를 답하지 않으면, 한 클래스 안에서 분기로 처리하게 됩니다. 책임 경계가 정해지지 않으면, 정책이 인프라를 알아버리거나 반대로 모든 호출이 Dispatcher로 몰립니다. 호출 시점이 자기 자리를 찾지 못하면, 같은 데이터를 두 번 가져옵니다.

어제까지의 4개 Checker가 *비슷하지만 다른* 모양이었던 이유는 정확히 이것이었습니다. 첫 번째 단계의 답이 *같은 것* 으로 정해진 채로 시작했기 때문에, 책임 경계와 호출 시점도 자리를 못 찾았던 것입니다. 잘못된 게 아니라, 단계가 흐트러진 결과였습니다.

## AI와 단계를 짚을 때 잘 맞은 부분

단계만 짚어주면 채우는 일은 빠릅니다. 이 작업에서 AI와 잘 맞은 부분은 다음과 같았습니다.

- 정책 4종을 각자 record로 옮기는 작업. 첫 번째 패턴만 잡아주면 나머지 3개는 페이스가 빠르게 따라옵니다.
- 단위 테스트 매트릭스. 입력과 기대 결과의 정직한 매트릭스라서 AI가 가장 잘하는 영역이었습니다.
- 리뷰 지적사항 적용. 어떤 지적이 옳은지만 사람이 판단하면 코드 적용은 위임이 가능했습니다.
- Dispatcher 단일화. 분산되어 있던 Checker 호출 순서를 한 곳으로 모으는 일은 기계적인 머지에 가까웠습니다.

이 부분들은 단계가 정해진 *뒤* 에 일어나는 일들입니다. 단계가 흐트러진 채로 이 일들을 부탁하면, 잘 채워진 어제까지의 모양이 나옵니다.

## 그래서 AI와 같이 설계하는 모양은

이 작업에서 AI와 한 번의 답 받기로 끝난 일은 거의 없었습니다. 매 단계마다 결정의 종류가 달랐고, 윗 단계의 답이 정해져야 다음 답이 의미를 가졌습니다.

Thoughtworks가 'middle loop' 라고 부르기 시작한 작업이 결국 이 단계 짚기에 가깝습니다. 이너 루프(코드를 쓰는 일)와 아우터 루프(딜리버리) 사이에서, 결정을 단계로 나누고 그 단계를 순서대로 짚어가는 일. 이게 AI 시대에 도메인 설계가 들어가는 자리였습니다.

AI가 코드를 빠르게 쓸 수 있다는 사실은, 사람이 도메인 모델링에서 손을 떼도 된다는 의미가 아니었습니다. 코드 쓰는 시간이 줄어든 만큼 *어떤 단계의 무엇을 결정해야 하는지* 를 순서대로 짚어가는 시간이 늘어났습니다. 다음 작업에 들어갈 때도, 단계를 흩지 않고 짚어갈 수 있을지 다시 한 번 점검하게 됩니다.
