---
title: "Filter, Interceptor, AOP: 언제 어떤 것을 사용해야 할까?"
date: 2023-08-01 10:00:00 +0900
categories: [Development, Spring]
tags: [spring, filter, interceptor, aop, java]
---

## 개요

ISMS 심사 대비로 URL별 권한 관리 기능을 추가하면서, Interceptor를 사용하게 되었습니다. 비슷한 역할을 하는 Filter와 AOP도 있는데, 각각 언제 사용해야 하는지 정리합니다.

> **📌 2025-01 업데이트**: Spring 6 기준으로 HandlerInterceptorAdapter deprecated 내용을 반영했습니다.

## 요청 처리 흐름

```
HTTP 요청
    ↓
[ Filter ] ← Servlet Container 영역
    ↓
DispatcherServlet
    ↓
[ Interceptor ] ← Spring MVC 영역
    ↓
Controller
    ↓
[ AOP ] ← 메서드 레벨
    ↓
Service/Repository
```

- **Filter**: Servlet Container가 관리, Spring Context 외부
- **Interceptor**: Spring MVC가 관리, Spring Context 내부
- **AOP**: Spring이 관리, 메서드 레벨에서 동작

## Filter

Java Servlet 스펙에 정의된 기능으로, 모든 요청/응답에 대해 전처리/후처리를 수행합니다.

### 주요 메서드

```java
public interface Filter {
    // 필터 초기화 (애플리케이션 시작 시 1회)
    default void init(FilterConfig filterConfig) throws ServletException {}

    // 실제 필터링 로직
    void doFilter(ServletRequest request, ServletResponse response,
                  FilterChain chain) throws IOException, ServletException;

    // 필터 종료 (애플리케이션 종료 시)
    default void destroy() {}
}
```

### 구현 예시: 요청 로깅 필터

```java
@Component
@Order(1)  // 필터 순서 지정
public class RequestLoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        long startTime = System.currentTimeMillis();

        // 요청 정보 로깅
        log.info("Request: {} {}", httpRequest.getMethod(), httpRequest.getRequestURI());

        // 다음 필터 또는 서블릿으로 전달
        chain.doFilter(request, response);

        // 응답 후 로깅
        long duration = System.currentTimeMillis() - startTime;
        log.info("Response: {} ms", duration);
    }
}
```

### 사용 사례

- 인코딩 설정 (`CharacterEncodingFilter`)
- CORS 처리
- XSS 방어
- 요청/응답 로깅
- 인증 토큰 검증 (Spring Security)

## Interceptor

Spring MVC에서 제공하는 기능으로, Controller 전후에 처리를 수행합니다.

### 주요 메서드

```java
public interface HandlerInterceptor {

    // Controller 실행 전 (false 반환 시 요청 중단)
    default boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws Exception {
        return true;
    }

    // Controller 실행 후, View 렌더링 전
    default void postHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler,
                            ModelAndView modelAndView) throws Exception {}

    // 요청 완료 후 (예외 발생해도 실행)
    default void afterCompletion(HttpServletRequest request,
                                 HttpServletResponse response,
                                 Object handler,
                                 Exception ex) throws Exception {}
}
```

> **참고**: `HandlerInterceptorAdapter`는 Spring 5.3에서 deprecated되었고 Spring 6에서 제거되었습니다. `HandlerInterceptor`를 직접 구현하면 됩니다.

### 구현 예시: 권한 검사 인터셉터

```java
@Component
@RequiredArgsConstructor
public class AuthorizationInterceptor implements HandlerInterceptor {

    private final AuthorizationService authService;

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {

        // 핸들러 메서드가 아니면 통과
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }

        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // @RequireRole 어노테이션 확인
        RequireRole requireRole = handlerMethod.getMethodAnnotation(RequireRole.class);
        if (requireRole == null) {
            return true;
        }

        // 권한 검사
        String userId = request.getHeader("X-User-Id");
        if (!authService.hasRole(userId, requireRole.value())) {
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.getWriter().write("권한이 없습니다.");
            return false;
        }

        return true;
    }
}
```

### 인터셉터 등록

```java
@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {

    private final AuthorizationInterceptor authorizationInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authorizationInterceptor)
            .addPathPatterns("/api/**")           // 적용할 경로
            .excludePathPatterns("/api/public/**"); // 제외할 경로
    }
}
```

### 사용 사례

- 권한/인가 검사
- 로그인 체크
- API 요청 로깅 (사용자 정보 포함)
- 실행 시간 측정

## AOP (Aspect-Oriented Programming)

메서드 실행 전후에 공통 로직을 적용하는 방식입니다. HTTP 요청과 무관하게 모든 메서드에 적용 가능합니다.

### 주요 용어

| 용어 | 설명 |
|------|------|
| Aspect | 공통 기능을 모듈화한 것 |
| Join Point | Aspect를 적용할 수 있는 지점 (메서드 실행 등) |
| Advice | 실제 수행할 로직 (Before, After, Around 등) |
| Pointcut | 어떤 Join Point에 적용할지 정의 |

### 구현 예시: 실행 시간 측정 AOP

```java
@Aspect
@Component
@Slf4j
public class ExecutionTimeAspect {

    // Service 계층 모든 메서드에 적용
    @Around("execution(* com.example.service..*(..))")
    public Object measureExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {

        long startTime = System.currentTimeMillis();
        String methodName = joinPoint.getSignature().toShortString();

        try {
            return joinPoint.proceed();
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("{} executed in {} ms", methodName, duration);
        }
    }
}
```

### 구현 예시: 트랜잭션 로깅 AOP

```java
@Aspect
@Component
@Slf4j
public class TransactionLoggingAspect {

    // @Transactional이 붙은 메서드에 적용
    @Before("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void logTransactionStart(JoinPoint joinPoint) {
        log.info("Transaction started: {}", joinPoint.getSignature().getName());
    }

    @AfterReturning("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void logTransactionSuccess(JoinPoint joinPoint) {
        log.info("Transaction committed: {}", joinPoint.getSignature().getName());
    }

    @AfterThrowing(
        pointcut = "@annotation(org.springframework.transaction.annotation.Transactional)",
        throwing = "ex"
    )
    public void logTransactionRollback(JoinPoint joinPoint, Exception ex) {
        log.error("Transaction rolled back: {} - {}",
                  joinPoint.getSignature().getName(), ex.getMessage());
    }
}
```

### 사용 사례

- 트랜잭션 관리 (`@Transactional`)
- 로깅/모니터링
- 캐싱 (`@Cacheable`)
- 보안 검사 (`@PreAuthorize`)
- 예외 처리

## 비교 정리

| 항목 | Filter | Interceptor | AOP |
|------|--------|-------------|-----|
| 관리 주체 | Servlet Container | Spring MVC | Spring |
| Spring Bean 접근 | 제한적 | 가능 | 가능 |
| 적용 대상 | 모든 요청 | Controller | 모든 Bean 메서드 |
| Request/Response 접근 | O | O | X (직접 접근 어려움) |
| 세밀한 제어 | X | O (URL 패턴) | O (메서드/어노테이션) |
| 예외 처리 | 직접 처리 | afterCompletion | @AfterThrowing |

## 권한 관리를 Interceptor로 한 이유

1. **Spring Bean 접근**: 권한 검사에 필요한 Service를 주입받을 수 있음
2. **URL 패턴 매칭**: 특정 경로에만 선택적으로 적용 가능
3. **HTTP 접근**: Request 헤더에서 사용자 정보 추출 가능
4. **Handler 정보**: Controller 메서드의 어노테이션을 읽을 수 있음

Filter는 Spring Context 외부라 Bean 접근이 제한적이고, AOP는 HTTP Request에 직접 접근하기 어렵습니다.

## 선택 가이드

```
HTTP 요청과 관련된 공통 처리가 필요한가?
├─ Yes → Spring Bean이 필요한가?
│        ├─ Yes → Interceptor
│        └─ No → Filter
└─ No → 메서드 레벨 처리가 필요한가?
         └─ Yes → AOP
```

**Filter**: 인코딩, CORS, 저수준 보안
**Interceptor**: 인증/인가, 로그인 체크, API 로깅
**AOP**: 트랜잭션, 캐싱, 메서드 레벨 로깅
