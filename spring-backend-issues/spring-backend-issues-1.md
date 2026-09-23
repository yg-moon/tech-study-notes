# 웹 백엔드 개발자가 대충 넘어가는 문제들 — 1편: 웹·HTTP 경계 (Java/Spring)

> **시리즈 소개**: 에러가 나면 AI가 고쳐준다. 코드는 다시 돌아가지만, 개발자는 "왜 문제였고 왜 이게 해결인지"를 설명하지 못한 채 넘어간다. 이 시리즈는 그렇게 AI가 붙여준 반창고 아래에서 이해 없이 지나치기 쉬운 지점들을 모은 것이다.
>
> 각 항목은 **반창고 → 실제로는 → 제대로 → 핵심 한 줄** 순서다. 해법이 짧으면 '실제로는'에 녹였다. 1편은 브라우저·프론트엔드와 맞닿는 웹/HTTP 경계를, 2편은 트랜잭션·JPA·DB를, 3편은 외부 연동·운영을 다룬다.

**1편의 지도**

- **1~4번은 한 덩어리다.** CORS·CSRF·쿠키·인증은 "토큰을 헤더에 둘까 쿠키에 둘까" 하나로 줄줄이 결정된다.
- **5~6번은 프론트와의 계약이다.** 에러가 어떤 코드와 어떤 바디로 나가는지가 곧 API 명세다.
- **7번과 짧은 항목들은 경계를 지나며 변형되는 데이터다.** 시간·인코딩·헤더가 프록시와 브라우저를 거치며 바뀌는 자리다.

---

## 1. CORS — 서버 로그엔 200인데 브라우저는 왜 에러지?

**반창고**: 컨트롤러마다 `@CrossOrigin`을 붙이거나, `allowedOrigins("*")` 한 줄로 끝.

**실제로는**

- CORS는 서버의 보안 기능이 아니라 **브라우저가 강제하는 규칙**이다. 서버는 아무것도 막지 않는다. 브라우저가 응답 헤더(`Access-Control-Allow-Origin` 등)를 보고 다른 출처의 JS에 응답을 보여줘도 되는지 판단하고, 허락이 없으면 숨길 뿐이다.
- 그래서 CORS 에러가 났다고 요청이 서버에 안 닿은 게 아니다. 단순 요청이라면 서버는 이미 요청을 처리했고 부수효과도 일어났으며, 브라우저가 결과만 가린 것이다.
- Origin은 scheme + host + port다. `localhost:3000`과 `localhost:8080`은 다른 출처다.
- 가장 흔한 함정은 잘 되던 CORS가 **Spring Security를 붙이는 순간 깨지는** 것이다. preflight `OPTIONS`는 자격증명 없이 오는데, 보안 필터가 MVC의 CORS 처리보다 먼저 이를 거부하기 때문이다.

**Preflight** — 가장 많이 헷갈리는 지점

- 단순 요청은 바로 나간다. 메서드가 GET/HEAD/POST이고, 커스텀 헤더가 없고, Content-Type이 `text/plain`·`form-urlencoded`·`multipart` 중 하나인 요청이다.
- 하나라도 벗어나면 브라우저가 먼저 **`OPTIONS` preflight**를 보내고, 서버가 `Access-Control-Allow-*` 헤더로 허락해야 실제 요청을 보낸다.
- `application/json`, `Authorization` 헤더, `PUT`/`DELETE`/`PATCH`가 모두 여기 해당한다. "JSON POST인데 왜 OPTIONS가 먼저 오지?"의 답이 이것이다.

**Credentials 함정** — 쿠키나 인증 토큰을 함께 보낼 때

- 프론트가 `credentials: 'include'`나 `withCredentials`로 쿠키를 보내면 **`Access-Control-Allow-Origin: *`는 스펙상 금지**된다. 특정 출처를 명시하고 `Access-Control-Allow-Credentials: true`를 함께 줘야 한다.
- Spring에서 `allowedOrigins("*")`와 `allowCredentials(true)`를 함께 쓰면 예외가 난다.
- 와일드카드가 꼭 필요하면 `allowedOriginPatterns("https://*.example.com")`을 쓴다. 요청의 Origin을 그대로 돌려주므로 credentials와 함께 쓸 수 있다.

**제대로**

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override public void addCorsMappings(CorsRegistry r) {
        r.addMapping("/api/**")
         .allowedOrigins("https://app.example.com") // credentials를 쓰면 "*" 불가
         .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
         .allowedHeaders("*")
         .allowCredentials(true);
    }
}
```

- Security를 쓴다면 시큐리티 체인에 `http.cors(Customizer.withDefaults())`를 넣는다. 시큐리티 필터가 위 설정이나 `CorsConfigurationSource` 빈을 먼저 적용해 preflight를 인증 없이 통과시킨다.

**핵심 한 줄**: CORS는 서버가 아니라 브라우저가 응답을 가리는 것이니, 에러가 나면 **preflight, 자격증명, Security 필터** 순서로 의심하라.

---

## 2. CSRF — 403이 나길래 껐는데, 괜찮은 건가?

**반창고**: Spring Security가 POST를 403으로 막길래 CSRF를 disable.

**실제로는**

- CSRF는 **브라우저가 쿠키를 자동으로 붙이는 성질**을 악용한다. 인증이 쿠키 기반이면 악성 사이트가 사용자의 브라우저를 시켜 내 도메인에 요청을 보낼 수 있고, 브라우저가 쿠키를 알아서 붙여 상태 변경이 일어난다.
- CSRF 토큰은 공격자가 읽을 수 없는 값을 요청에 요구해 이를 막는다. 동일 출처 정책 때문에 공격자는 그 값을 읽지 못한다.

**제대로** — 브라우저가 자격증명을 자동으로 붙이는가로 갈린다

- `Authorization` 헤더에 토큰을 담는다면 브라우저가 자동으로 붙이지 않으므로 CSRF 위험이 거의 없다. **stateless 토큰 API는 CSRF를 꺼도 된다.**
- 세션 쿠키로 인증한다면 CSRF를 끄는 순간 **진짜 취약**해진다. 켜둬야 한다.
- Chrome 계열은 SameSite 속성이 없는 쿠키를 `Lax`로 취급해 크로스사이트 POST에 싣지 않는다(3번). 다만 모든 브라우저가 그렇지는 않으니, 이 방어에 기대려면 `SameSite=Lax`를 직접 명시해야 한다.
- 크로스도메인 인증 때문에 `SameSite=None`으로 풀었다면 이 방어는 사라지고, 다시 토큰 방어가 필요하다.

**핵심 한 줄**: CSRF는 브라우저가 자동으로 붙이는 쿠키로 인증할 때 생기는 문제이니, **세션 쿠키를 쓴다면 끄지 마라.**

---

## 3. 쿠키 속성 — 로컬에선 되는데 배포하면 쿠키가 안 실린다

**반창고**: 원인을 모른 채 쿠키 설정을 이것저것 바꿔봄.

**실제로는**

- 세 속성이 관여한다. `HttpOnly`는 JS가 쿠키를 읽지 못하게 해 XSS 탈취를 막고, `Secure`는 HTTPS에서만 전송하게 하며, `SameSite`는 다른 사이트에서 온 요청에 쿠키를 실을지 정한다.
- `SameSite` 값은 `Lax`·`Strict`·`None`이고, 지정하지 않으면 Chrome 계열은 `Lax`로 취급한다.
- 프론트(`a.com`)와 API(`b.com`)가 다른 사이트라면 쿠키 인증에 `SameSite=None`이 필요하다. 그리고 **`None`은 `Secure`와 함께여야만** 브라우저가 쿠키를 보낸다.
- 로컬에선 프론트와 API가 포트만 다른 `localhost`라 같은 사이트로 취급되고, 그래서 `Lax`로도 잘 된다. 운영에서 도메인이 갈라지는 순간 위 조건에 걸린다.

**핵심 한 줄**: 크로스사이트 쿠키 인증이 운영에서만 깨지면 **`SameSite=None; Secure`**부터 확인하라.

---

## 4. 인증: 세션 vs JWT — 요즘은 다 JWT 아닌가?

**반창고**: "요즘은 다 JWT"라서 JWT. 토큰은 AI가 짜준 대로 localStorage에.

**실제로는**

| | 세션(쿠키) | 토큰(JWT) |
|---|---|---|
| 상태 | 서버가 보관(stateful) | 자기완결(stateless) |
| 확장 | 공유 세션 저장소(Redis 등) 필요 | 수평 확장 쉬움 |
| 폐기(로그아웃·차단) | 쉬움(서버에서 삭제) | **어려움**(만료 전 무효화엔 별도 블랙리스트) |
| CSRF | 필요 | 헤더에 담으면 거의 불필요 |
| 저장 위치 위험 | — | localStorage → XSS 노출 / 쿠키 → CSRF 재등장 |

- JWT의 진짜 비용은 **즉시 폐기가 어렵다**는 것과 저장 위치 딜레마다. 강제 로그아웃이나 권한 즉시 회수가 중요한 서비스라면 세션이 더 단순할 때가 많다.
- 백엔드 입장에서 핵심은 토큰이 **헤더에 사느냐 쿠키에 사느냐**다. 이것이 CSRF 필요 여부와 CORS 자격증명 처리를 결정하고, 1~3번이 전부 이 선택에 엮인다.

**핵심 한 줄**: 토큰을 헤더에 둘지 쿠키에 둘지가 CSRF와 CORS 설정을 전부 결정하니, **폐기와 저장 위치의 비용을 알고** 골라라.

---

## 5. HTTP 상태코드 — 에러인데 200을 돌려주면 안 되나?

**반창고**: 예외가 나길래 컨트롤러를 try-catch로 감싸 `200 OK + {"success": false}`를 반환. 또는 모든 예외가 스택트레이스째 500으로 노출.

**실제로는**

- 상태코드는 사람이 아니라 **기계가 읽는 계약**이다. 프론트의 응답 인터셉터, HTTP 캐시, 로드밸런서 헬스체크, 모니터링 알람, 재시도 로직이 전부 상태코드로 분기한다.
- 전부 200이면 이 인프라가 통째로 눈을 감는다. 에러율 대시보드에 아무것도 안 잡힌 채 장애가 진행되는 식이다.
- 첫 분기는 **4xx와 5xx**다. 4xx는 클라이언트가 고쳐야 하니 재시도해도 소용없고, 5xx는 서버 잘못이라 재시도가 의미 있을 수 있다.
- 자주 헷갈리는 쌍도 있다.
  - 401은 이름과 달리 인증 실패로, 누군지 모른다는 뜻이다. 403은 인가 실패로, 누군지는 알지만 권한이 없다는 뜻이다.
  - 400은 요청 형식 오류이고, 409는 중복 가입처럼 현재 상태와의 충돌이다.
  - 생성은 201, 본문 없는 성공은 204다.

**제대로**

컨트롤러마다 try-catch를 두면 중복인 데다 반드시 누락이 생긴다. `@RestControllerAdvice`와 `@ExceptionHandler`로 **예외 타입 → 상태코드 + 일관된 에러 바디** 매핑을 한 곳에 모은다. Spring 6부터는 RFC 9457의 표준 에러 바디인 `ProblemDetail`도 기본 지원한다.

```java
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(EntityNotFoundException.class)
    ProblemDetail notFound(EntityNotFoundException e) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
    }
}
```

- 500 응답에 예외 메시지나 스택트레이스를 담으면 내부 구조가 노출된다. 상세는 서버 로그에 남기고 클라이언트엔 추적용 ID만 준다.

**핵심 한 줄**: 상태코드는 기계가 읽는 계약이니, 예외→상태코드 매핑은 **`@RestControllerAdvice` 한 곳**에서 관리하라.

---

## 6. 요청 바인딩과 검증 — 415와 400은 뭐가 다른가?

**반창고**: 415/400이 나면 어노테이션을 이리저리 바꿔 되는 조합이 나오면 넘어감. `@Valid`를 붙였는데 검증이 안 돼도 원인은 모름.

**실제로는**

- `@RequestBody`는 요청의 `Content-Type`을 보고 맞는 `HttpMessageConverter`(JSON이면 Jackson)를 골라 본문을 역직렬화한다. 폼 데이터는 전혀 다른 경로인 `@ModelAttribute` 바인딩으로 간다.
  - **415 Unsupported Media Type**은 그 Content-Type을 읽을 컨버터가 없다는 뜻이다. JSON을 기대하는데 form으로 보냈거나 Content-Type 헤더가 빠진 경우다.
  - **400 Bad Request**는 컨버터는 찾았는데 파싱이나 바인딩에 실패했다는 뜻이다. JSON 문법 오류나 타입 불일치가 여기 해당한다.
- `@Valid`는 붙인 파라미터에만 동작한다. **중첩 객체는 그 필드에도 `@Valid`를 붙여야** 검증이 전파되고, 빼먹으면 에러 없이 조용히 스킵된다.
- 검증 실패는 `MethodArgumentNotValidException`으로 400이 되고, 5번의 `@RestControllerAdvice`에서 이 예외를 필드 에러 목록으로 바꿔 내려주면 된다. 이 바디 형태도 프론트와의 계약이니 한 번 정하고 유지한다.
- `@Validated`는 Spring 쪽 어노테이션으로, 검증 그룹과 서비스 계층 메서드 검증용이다. 컨트롤러 body 검증엔 표준 `@Valid`면 충분하다.

**핵심 한 줄**: 415는 읽을 컨버터가 없다는 뜻이고 400은 읽다가 실패했다는 뜻이며, **`@Valid`는 중첩 객체에 자동으로 전파되지 않는다.**

---

## 7. 시간대 — 운영에서만 시간이 9시간 어긋난다

**반창고**: `LocalDateTime` 쓰고 "되네" 하고 넘김.

**실제로는**

- `LocalDateTime`은 **타임존 정보가 없는 벽시계 숫자**다. 서버·DB·클라이언트의 타임존이 섞이면 오프셋을 잃고 어긋난다.
- 로컬은 전부 KST라 문제가 안 보이다가, 운영 서버나 DB가 UTC인 순간 9시간이 틀어진다. 오프셋 없는 `2026-06-21T12:00:00`을 받은 프론트도 그걸 자기 현지 시간으로 해석한다.
- 특정 시점은 `Instant`나 `OffsetDateTime`으로 다루고, DB 컬럼도 PostgreSQL이라면 `timestamptz`처럼 타임존을 보존하는 타입을 쓴다. JDBC 드라이버의 타임존 해석도 `hibernate.jdbc.time_zone=UTC` 등으로 맞춰야 한다.
- Spring Boot의 Jackson은 java.time을 기본으로 ISO-8601 문자열로 직렬화한다. `WRITE_DATES_AS_TIMESTAMPS`가 기본으로 꺼져 있어서, `Instant`는 `2026-06-21T12:00:00Z`처럼 나간다.

**제대로**: 저장·전송은 UTC, 변환은 가장자리에서 한다. API는 ISO-8601 UTC를 주고, 프론트가 현지 시간으로 바꿔 보여준다. 백엔드가 로컬 타임존으로 가공해 내려보내지 않는다.

**핵심 한 줄**: 시점은 **`Instant`로 UTC 저장·전송**하고, 현지 시간 변환은 화면에 표시할 때만 하라.

---

## 짧게 짚고 가는 것들

- **리버스 프록시 뒤(nginx/ALB)**: 프록시가 TLS를 끝내므로 서버가 보는 클라이언트는 프록시다. `getRemoteAddr()`가 전부 프록시 IP로 찍히거나 리다이렉트가 `http://`로 나가면 `server.forward-headers-strategy=framework`로 `X-Forwarded-For/Proto`를 반영하게 한다. 이 헤더는 위조할 수 있으니 신뢰하는 프록시가 붙인 값만 믿어야 한다.
- **파일 업로드 한도**: Spring Boot multipart 기본 한도는 파일당 1MB(`max-file-size`), 요청당 10MB(`max-request-size`)이고, 넘으면 `MaxUploadSizeExceededException`이 난다. nginx `client_max_body_size`(기본 1MB) 같은 프록시 한도도 함께 맞춰야 하며, 프록시에서 걸리면 서버 로그 없이 프론트만 413을 받는다.
- **문자 인코딩**: 깨진 글자는 어딘가 UTF-8이 아니라는 뜻이다. 응답 `Content-Type`의 charset, 소스 파일 인코딩, DB까지 한 줄로 꿰어 맞춰야 한다. DB 쪽 `utf8mb4`는 2편에서 다룬다.

---

> **공통 교훈**: 반창고가 에러를 없애줘도 "왜 없어졌는지"를 한 줄로 설명하지 못하면 빚이 쌓인 것이다. 각 항목의 "핵심 한 줄"이 그 설명이다.
