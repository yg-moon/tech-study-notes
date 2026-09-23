# PLAN

## 진행 상황

- [x] `spring-backend-issues` 1~3편: 완결. 브라우저·DB·외부 세계라는 세 경계로 짜인 시리즈라 3편에서 닫았다.
- [ ] `database-internals` ← 다음
- [ ] `distributed-consistency`
- [ ] `backend-security`
- [ ] `production-operations`
- [ ] `network-fundamentals`
- [ ] `jvm-performance`
- [ ] `backend-testing`
- [ ] 시스템 설계 사례

---

## 우선순위 기준

백엔드의 실패는 되돌리기 어려울수록 비싸다.

1. **데이터가 틀어지거나 새는 것**: 틀어진 데이터와 유출된 데이터는 되돌릴 수 없다.
2. **서비스가 멈추는 것**: 복구는 되지만 그동안의 요청과 신뢰를 잃는다.
3. **서비스가 느린 것**: 불편하지만 고칠 시간이 있다.
4. **개발이 느린 것**: 비용을 팀 안에서 치른다.

시리즈는 이 순서를 따른다. 앞 시리즈가 뒤 시리즈의 바탕이기도 하다. DB를 모르면 분산 일관성을 이해할 수 없고, 네트워크를 모르면 성능 문제의 절반을 놓친다.

---

## 다음 시리즈

### 1. `database-internals` — DB 안에서 벌어지는 일

- **왜**: 거의 모든 데이터가 DB를 거치고, 장애와 성능 문제도 결국 DB에서 끝난다. spring-backend-issues 2편이 DB를 쓰는 법이었다면, 이 시리즈는 DB 안쪽의 동작이다.
- **후보**: B+tree와 클러스터드 인덱스(랜덤 UUID PK가 느린 이유), MVCC와 undo, 긴 트랜잭션이 남기는 것(PostgreSQL bloat와 vacuum), 옵티마이저 통계와 실행계획이 바뀌는 순간, 복제 지연과 read-after-write, 파티셔닝과 샤딩, 무중단 스키마 변경(expand-contract)

### 2. `distributed-consistency` — DB 트랜잭션 하나로 끝나지 않을 때

- **왜**: 시스템이 둘 이상이면 트랜잭션 경계 밖에서 데이터가 어긋난다. 주문·재고·결제처럼 여러 시스템에 걸친 작업의 정합성을 다룬다.
- **후보**: 이중 쓰기와 outbox, 메시지 전달 보장과 소비자 멱등성, 순서 보장과 파티션 키, 사가와 보상 트랜잭션, 분산 락과 fencing token, 시계를 믿으면 안 되는 이유, 정산 대사(reconciliation)

### 3. `backend-security` — 코드가 만드는 구멍

- **왜**: 유출은 정합성 오류만큼 되돌릴 수 없다. spring-backend-issues 1편이 CORS·CSRF·인증의 기본이었다면, 이 시리즈는 애플리케이션 코드가 만드는 구멍이다.
- **후보**: 문자열 결합 쿼리와 SQL 인젝션, mass assignment, 객체 단위 인가 누락(IDOR), 토큰 검증과 갱신, 비밀값 관리, 로그와 저장소의 개인정보, SSRF, 의존성 취약점

### 4. `production-operations` — 운영과 관측

- **왜**: 멈춘 서비스를 빨리 알아채고 빨리 되돌리는 능력이다. 배포와 장애 대응은 백엔드 개발자가 직접 책임지는 영역이다.
- **후보**: 메트릭·로그·트레이스의 역할 분담, 증상 기반 알람과 SLO, 배포 전략과 롤백, API 하위 호환, 피처 플래그, 용량 산정과 부하 테스트, 장애 대응과 포스트모템

### 5. `network-fundamentals` — 요청이 서버에 닿기까지

- **왜**: 원인 모를 502·504와 간헐적 연결 오류는 대개 네트워크 계층에서 온다. 프레임워크가 가려주는 만큼 모르고 지나치기 쉽다.
- **후보**: TCP 연결 수명(keep-alive, TIME_WAIT, 포트 고갈), HTTP/1.1·2·3의 차이, TLS 핸드셰이크, DNS와 JVM의 DNS 캐시, L4·L7 로드밸런서와 idle timeout 불일치, 타임아웃 예산의 전파

### 6. `jvm-performance` — 느린 서비스

- **왜**: 성능 문제는 고칠 시간이 있지만, 원인을 재는 법을 모르면 추측으로 고치게 된다.
- **후보**: GC와 멈춤, GC 로그 읽기, `volatile`·CAS와 `ConcurrentHashMap`의 복합 연산, `CompletableFuture`의 공용 풀, 스레드풀 위 `ThreadLocal` 누수, 가상 스레드의 pinning, 캐싱 전략과 캐시 스탬피드, 프로파일링과 플레임 그래프, 부하 테스트의 함정(coordinated omission)

### 7. `backend-testing` — 테스트가 거짓말하는 순간들

- **왜**: 개발 속도를 떠받치는 영역이다. 비용은 팀 안에서 치르지만 매일 치른다.
- **후보**: 테스트의 `@Transactional` 롤백이 가리는 버그, 컨텍스트 캐시와 느린 테스트, H2 대신 Testcontainers, 시간 의존 테스트와 `Clock` 주입, 비동기 테스트, 서비스 간 계약 테스트, flaky 테스트의 원인

### 그다음: 시스템 설계 사례

위 시리즈들이 설계의 부품이다. 샤딩과 복제는 1번, 큐와 전달 보장은 2번, 캐싱은 6번에 있다. 부품이 모이면 URL 단축기, 피드, 결제처럼 전체 시스템을 조립하는 사례 시리즈로 넘어간다.
