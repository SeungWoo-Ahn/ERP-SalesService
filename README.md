### 소개
본사 1개소, 직영 대리점 80개소, 협력 정비소 200개소, 월간 입출고 평균 5만건 규모의 부품 유통 회사의 **ERP 시스템**을 구축하는 프로젝트입니다.

이 시스템은 MSA 기반으로, 다섯 개의 독립적인 서비스로 구성되어 있습니다.
- UserSevice
- ItemService
- InventoryService
- ProcurementService
- SalesService

4인 팀으로 개발했고, 저는 ProcurementService와 SalesService 개발을 담당했습니다.

SalesService는 이 중 **발주** 기능을 담당합니다. <br>
**발주**는 대리점이 본사에 필요한 부품을 요청하는 과정이며, 핵심 기능은 다음과 같습니다.

1. 대리점의 발주 생성/임시저장/수정/제출/입고
2. 본사의 발주 승인/반려/취소
3. 본사와 대리점의 발주 KPI/목록/상세/이력 조회

<br>

### 기술 스택
| 구분 | 기술 |
| --- | --- |
| Backend | Java 21, Spring Boot, Spring Data JPA, Spring Security |
| Database | PostgreSQL, Flyway |
| Messaging | RabbitMQ |
| Docs | Swagger |
| Test | JUnit5, JaCoCo |
| Build | Gradle |

<br>

### 시스템 아키텍처
<img width="1306" height="811" alt="ERP007 시스템 아키텍처" src="https://github.com/user-attachments/assets/5df011c7-6826-4d85-96b1-696eb15c840c" />
<br>

### ERD
<img width="1022" height="671" alt="ERP007 발주 erd" src="https://github.com/user-attachments/assets/b759d7c4-9a87-4213-a6fb-805b57a7c511" />
<br>


### 입고/출고 처리의 이벤트 기반 비동기 전환

#### 배경
입고와 출고는 창고의 실제 부품 이동이 시스템에 반영되는 과정이라, 데이터 정합성이 가장 중요한 부분입니다.
그래서 초기에는 InventoryService에 재고 이동을 동기로 요청하고, 그 결과를 하나의 트랜잭션 안에서 처리했습니다.

문제는 명절 전처럼 요청이 몰리는 시기에 재고 이동 요청의 응답이 크게 지연된다는 점이었습니다. <br>
InventoryService는 재고 정합성을 위해 비관적 락을 사용하는데, 동시 요청이 많아지면 락 대기 시간이 길어지기 때문입니다.
동기 호출 구조에서는 이 대기 시간 동안 호출하는 쪽의 요청 스레드와 트랜잭션까지 함께 묶여, 한 서비스의 지연이 다른 서비스로 번질 위험이 있었습니다.

#### 해결
재고 이동 요청을 RabbitMQ를 통한 이벤트 기반 비동기 통신으로 전환했습니다. <br>
이 전환으로 두 가지 효과를 얻을 수 있었습니다.

**1. 가용성 향상**

메시지 브로커가 버퍼 역할을 하므로, 요청이 몰려도 InventoryService는 처리 가능한 속도로 메시지를 소비할 수 있습니다.
호출하는 쪽은 응답을 기다리지 않아 지연이 전파되지 않고, InventoryService에 장애가 발생하더라도 메시지는 큐에 남아 복구 후 처리됩니다.

**2. 서비스 간 결합도 감소**

기존에는 다른 서비스 호출이 하나의 트랜잭션에 묶여 있어 서비스 간 의존성이 높았습니다.
이벤트 기반으로 전환한 뒤에는 각 서비스가 필요한 이벤트를 구독해 자신의 책임만 처리합니다.
실제로 사용자 활동 로그 조회 요구사항도 이벤트를 구독하는 방식으로 손쉽게 추가할 수 있었고, 이후 새로운 후속 처리도 같은 방식으로 확장할 수 있습니다.

<br>

#### 메시지 브로커로 Kafka가 아닌 RabbitMQ를 선택한 이유

**1. 운영 부담**

팀에 Kafka 운영 경험이 없는 상황에서 리스크가 되고, 인프라 담당자에게 부담이 됐습니다.
RabbitMQ는 단일 브로커로도 운영이 가능하고 관리 UI를 통해 큐 상태와 적체를 바로 확인할 수 있어, 현재 팀 규모에서 감당 가능한 선택이었습니다.

**2. 메시지 유실 방지**

RabbitMQ는 처리량을 위해 메시지를 메모리에 두지만, 이는 설정으로 제어할 수 있는 부분입니다.
durable 속성을 준 exchange/queue에 persistent 메시지를 발행하면 메시지가 디스크에 기록되므로, 브로커가 재시작되어도 큐에 남아 정상 처리됩니다.
여기에 <a href="https://www.rabbitmq.com/docs/confirms">Publisher Confirms</a>를 함께 사용하면 브로커가 메시지를 책임지고 받았는지를 발행 측에서 확인할 수 있어, 재고 이동처럼 유실이 곧 재고 불일치로 이어지는 작업에도 충분한 보장을 얻을 수 있습니다.

**3. 사용 패턴의 적합성**

Kafka의 핵심 강점은 로그를 보존해 하나의 메시지를 여러 컨슈머 그룹이 각자의 속도로 반복 소비할 수 있다는 점입니다.
하지만 재고 이동 요청은 이벤트라기보다 "이 발주의 재고를 이동시켜라"는 **명령**에 가까워, 지정된 한 서비스가 한 번만 처리하면 충분했습니다.
오히려 필요한 것은 routing key로 명령과 응답을 목적지별로 나누는 유연한 라우팅이었고, 이는 RabbitMQ의 topic exchange가 더 적합했습니다.

> 관련 코드: <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/config/RabbitConfig.java">RabbitConfig.java</a>

<br>

### Outbox 패턴의 도입
#### 배경
동기 호출로 재고 이동을 요청하던 부분을 RabbitMQ 메시지 발행으로 그대로 교체하자, 입출고 트랜잭션이 롤백되어도 메시지는 이미 발행되는 문제가 생겼습니다.
예를 들어 메시지를 발행한 뒤 입고 처리 도중 예외가 발생하면, 입고 데이터는 롤백되지만 InventoryService는 메시지를 받아 재고를 이동시킵니다.

원인은 `@Transactional`이 보장하는 원자성의 범위가 DB에 한정된다는 점이었습니다.
`@Transactional`은 프록시를 통해 메서드 실행 전후로 트랜잭션을 시작하고 커밋하며, RuntimeException이나 Error가 발생하면 롤백합니다.
하지만 롤백되는 대상은 트랜잭션에 참여한 DB 작업뿐이고, 외부 시스템인 RabbitMQ로 이미 발행된 메시지는 되돌릴 수 없습니다.

그렇다고 발행 시점을 커밋 이후로 옮기면 반대의 문제가 생깁니다.
DB 커밋은 성공했지만 브로커 장애 등으로 발행이 실패하면, 입고는 완료되었는데 재고 이동은 일어나지 않습니다.

#### 해결
결국 DB 저장과 메시지 발행이라는 서로 다른 두 시스템에 대한 쓰기를 하나의 트랜잭션으로 묶을 수 없는 **이중 쓰기(Dual Write) 문제**였고, 이를 해결하기 위해 <a href="https://microservices.io/patterns/data/transactional-outbox.html">Transactional Outbox</a> 패턴을 도입했습니다.

메시지를 브로커로 바로 보내는 대신, 발행할 메시지를 같은 DB의 `outbox` 테이블에 **비즈니스 데이터와 동일한 트랜잭션으로** INSERT합니다.
쓰기 대상이 DB 하나로 줄어들기 때문에 "발주 상태 변경"과 "보낼 메시지"는 함께 커밋되거나 함께 롤백됩니다.
실제 브로커 발행은 커밋이 확정된 이후 별도의 릴레이가 담당하므로, 롤백된 트랜잭션의 메시지는 애초에 발행 대상이 되지 않습니다.

> 관련 코드: <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/adapter/outbound/messaging/StockEventMessagingAdapter.java">StockEventMessagingAdapter.java</a>, <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/adapter/outbound/persistence/outbox/OutboxEntity.java">OutboxEntity.java</a>

<br>

#### 적재된 메시지를 어떤 방식으로 발행할 것인가

**1. 폴링 방식의 한계**

가장 단순한 방법은 스케줄러로 `PENDING` 상태의 행을 주기적으로 조회해 발행하는 것이라, 처음에는 1초 주기 폴링으로 구현했습니다.
하지만 이 방식은 두 가지가 아쉬웠습니다. 발행할 메시지가 없어도 매초 DB를 조회해 자원을 낭비하고, 반대로 최악의 경우 폴링 주기만큼 재고 이동이 지연됩니다.

**2. 커밋 직후 즉시 발행**

그래서 커밋이 끝나는 시점을 폴링으로 "발견"하는 대신, 커밋 시점에 직접 "통지"받는 방식을 택했습니다.
outbox 행을 적재할 때 Spring의 `ApplicationEventPublisher`로 적재 사실을 알리고, 릴레이가 `@TransactionalEventListener(phase = AFTER_COMMIT)`으로 이를 수신해 해당 행만 즉시 발행합니다.
커밋이 확정된 뒤에만 리스너가 호출되므로 롤백된 트랜잭션의 메시지는 발행되지 않으면서, 지연은 폴링 주기가 아니라 커밋 직후 수준으로 줄어듭니다.

**3. 응답 지연 문제와 비동기 전환**

그런데 이 구조를 적용하자 브로커 발행이 지연될 때 API 응답까지 함께 느려지는 문제가 나타났습니다.
원인은 <a href="https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html">Spring 트랜잭션 이벤트</a>의 기본 동작이 **동기**라는 점이었습니다.
`AFTER_COMMIT` 리스너는 커밋을 수행한 스레드, 즉 사용자 요청 스레드에서 그대로 실행되기 때문에, 발행과 confirm 대기 시간이 고스란히 응답 시간에 더해진 것입니다.

이를 `@Async`로 분리하고 릴레이 전용 스레드풀을 할당해 해결했습니다.
요청 스레드는 커밋 후 바로 응답을 반환하고, 발행은 `outbox-relay` 스레드에서 진행됩니다.
풀이 포화되면 `CallerRunsPolicy`로 일시적으로 호출 스레드에서 처리해 무제한 스레드 생성을 막았습니다.

```java
@Async("outboxRelayExecutor")
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onAppended(OutboxAppendedEvent event) {
    outboxJpaDao.findById(event.eventId())
            .filter(o -> o.getStatus() == OutboxStatus.PENDING)
            .ifPresent(this::publish);
}
```

**4. 폴링은 보조 수단으로 유지**

비동기로 분리한 뒤에는 발행 스레드가 죽거나 애플리케이션이 내려가면 메시지가 `PENDING`으로 남을 수 있습니다.
그래서 폴링을 제거하지 않고, 즉시 발행이 주 경로, 폴링이 **누락분 보조 경로**가 되도록 역할을 나눴습니다.
주기는 1초에서 5초로 늘려 평상시 DB 조회 비용을 줄였고, 오래된 순으로 최대 100건을 조회해 순차 발행합니다.
발행 중 브로커 장애 신호가 감지되면 남은 건을 계속 시도하지 않고 배치를 중단해, 다음 폴링으로 미루는 자연스러운 백오프가 동작합니다.

> 관련 코드: <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/adapter/outbound/messaging/OutboxRelay.java">OutboxRelay.java</a>, <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/config/AsyncConfig.java">AsyncConfig.java</a>

<br>

#### 발행 실패는 어떻게 처리할 것인가

**1. 성공의 기준을 무엇으로 볼 것인가**

`RabbitTemplate.send()`는 예외 없이 반환되어도 브로커가 메시지를 받았다는 보장이 되지 않습니다.
그래서 `publisher-confirm-type: correlated` 설정으로 Publisher Confirms를 활성화하고, **브로커의 ack를 성공 기준**으로 삼았습니다.
ack를 받으면 outbox 행을 `PUBLISHED`로 전환하고 해당 발주의 saga를 `PROCESSING`으로 전진시킵니다.

이때 두 작업의 순서를 **saga 전진 → outbox 상태 변경** 순으로 두었습니다.
반대 순서였다면 outbox는 `PUBLISHED`인데 saga는 `SENDING`에 멈춘 상태로 남아 복구할 방법이 없지만,
이 순서라면 중간에 실패해도 행이 `PENDING`으로 남아 폴러가 재발행하고 멱등 처리로 수렴하기 때문입니다.

**2. 실패의 종류를 구분**

실패를 한 가지로 취급하면, 브로커가 잠시 끊긴 것뿐인데 재시도 횟수만 소진하고 `FAILED`로 떨어지는 문제가 생깁니다.
그래서 실패를 두 가지로 나눴습니다.

- **일시적 인프라 장애** (`AmqpConnectException`, `AmqpIOException`, confirm `TimeoutException`) <br>
  브로커가 복구되면 그대로 성공할 수 있는 상황이므로 재시도 횟수를 올리지 않고 `PENDING`을 유지합니다. 폴러가 복구 후 자동으로 재발행합니다.
- **그 외 실패** (브로커 nack, 직렬화 오류 등) <br>
  재시도 횟수를 증가시키고, 5회를 초과하면 `FAILED`로 전환해 무한 재시도를 끊고 점검 대상으로 남깁니다.

```java
private boolean isTransient(Throwable t) {
    Throwable cause = (t instanceof ExecutionException && t.getCause() != null) ? t.getCause() : t;
    return cause instanceof AmqpConnectException
            || cause instanceof AmqpIOException
            || cause instanceof TimeoutException;
}
```

**3. 테이블 비대화 방지**

outbox 테이블은 발주가 발생할 때마다 계속 쌓이기 때문에 조회 성능이 점점 나빠집니다.
새벽 4시에 스케줄러를 돌려 `PUBLISHED` 상태로 3일이 지난 행만 삭제하고, `PENDING`(미발행)과 `FAILED`(점검 대상)는 보존해 정합성 추적이 가능하도록 했습니다.

> 관련 코드: <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/application/service/ConfirmStockEventPublishedService.java">ConfirmStockEventPublishedService.java</a>, <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/adapter/outbound/persistence/outbox/OutboxCleanupScheduler.java">OutboxCleanupScheduler.java</a>

<br>

### Saga 패턴으로 재고 이동 실패에 대응

#### 배경
비동기로 전환한 뒤에는 승인 요청이 성공적으로 응답되어도, 이후 InventoryService가 "재고 부족"으로 출고를 거절할 수 있습니다.
모놀리식이었다면 트랜잭션을 롤백하면 끝나지만, MSA에서는 두 서비스의 트랜잭션 경계가 이미 나뉘어 있어 DB 롤백으로 되돌릴 수 없었습니다.

#### 해결
실패 응답을 받으면 **반대 방향의 비즈니스 동작을 직접 수행해** 되돌리는 Saga의 <a href="https://microservices.io/patterns/data/saga.html">보상 트랜잭션</a>으로 최종 일관성을 맞췄습니다.

- 출고 실패 → `APPROVED`를 `REQUESTED`로 되돌려 재승인 또는 반려가 가능한 상태로 복구
- 입고 실패 → `DELIVERED`를 `APPROVED`로 되돌려 재입고 처리가 가능한 상태로 복구

이때 비즈니스 마일스톤(`SalesOrderStatus`)과 메시징 진행 상태(`SagaStatus`)를 분리해 관리했습니다.
두 관심사를 한 컬럼에 섞으면 "승인됐지만 출고 처리 중"과 "승인 확정" 같은 상태를 표현할 수 없고, 상태 값이 계속 늘어나기 때문입니다.

보상을 구현하면서 특히 주의한 부분은 세 가지였습니다.

**1. 멱등성**

RabbitMQ는 at-least-once 전달을 보장하기 때문에 같은 응답이 두 번 이상 도착할 수 있습니다.
이때 보상을 그대로 다시 수행하면, 재승인해서 정상 처리 중인 발주를 다시 `REQUESTED`로 되돌리는 사고가 발생합니다.

별도의 중복 처리 테이블을 두는 방법도 있었지만, saga 상태 자체가 이미 "진행 중인지 끝났는지"를 표현하고 있었기 때문에
**도메인 모델의 saga 상태를 가드로 사용**해 추가 테이블 없이 해결했습니다.
`DONE`/`FAILED`처럼 이미 종료된 상태면 응답을 무시하고, 진행 중일 때만 전이시킵니다.

```java
SagaStatus saga = order.getSagaStatus();
if (saga == SagaStatus.FAILED || saga == SagaStatus.DONE) {
    return;
}
```

또한 InventoryService가 "이미 처리됨(`ALREADY_PROCESSED`)"으로 응답하는 경우는 형식상 거절이지만 실제로는 재고 이동이 완료된 상태이므로, 보상하지 않고 성공으로 취급합니다.

**2. 동시성**

보상은 재고 응답을 받은 리스너 스레드에서, 사용자 요청은 별도의 요청 스레드에서 같은 발주를 수정할 수 있습니다.
여기에 비관적 락을 쓰면 락 대기 때문에 비동기로 전환한 이점이 줄어드는데, 한 발주에 동시 수정이 몰릴 확률 자체는 낮습니다.
그래서 발주 엔티티에 `@Version`을 두어 **낙관적 락**으로 처리했습니다. 충돌한 쪽만 예외로 실패하고, 리스너 경로에서는 재시도로 복구됩니다.

**3. 보상 자체의 실패**

보상 처리 중 예외가 발생하면 응답 메시지가 큐로 되돌아가 무한 재전달되는 문제가 생깁니다.
이를 막기 위해 리스너 재시도를 3회(1초부터 2배씩 증가, 최대 10초)로 제한하고,
`default-requeue-rejected: false`로 재시도가 소진된 메시지는 재큐잉 대신 <a href="https://www.rabbitmq.com/docs/dlx">Dead Letter Exchange</a>를 통해 DLQ로 격리했습니다.
정상 메시지 처리를 막지 않으면서, 실패한 메시지는 원본 그대로 남아 원인 파악과 수동 재처리가 가능합니다.

**추가로, 확정되지 않은 상태 변경은 이력에 남기지 않도록 했습니다.** <br>
승인 시점에 이력을 바로 쓰면 이후 출고가 거절됐을 때 "승인 → 되돌림"이 이력에 남아 사용자가 혼란스러워집니다.
그래서 승인·배송의 행위자와 부가 데이터를 `PendingStatusChange`에 임시 보관했다가, saga가 성공으로 확정될 때만 이력으로 승격하고 실패 시에는 폐기합니다.

> 관련 코드: <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/adapter/inbound/event/StockReplyEventListener.java">StockReplyEventListener.java</a>, <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/application/service/CompensateStockSagaService.java">CompensateStockSagaService.java</a>, <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/application/service/CompleteStockSagaService.java">CompleteStockSagaService.java</a>, <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/domain/model/salesorder/SalesOrder.java">SalesOrder.java</a>

<br>

### 비동기 처리의 진행 상태를 사용자에게 노출

#### 배경
비동기 전환으로 서버는 빨라졌지만, 사용자 입장에서는 오히려 불친절해졌습니다.
승인 버튼을 눌러도 재고 이동이 실제로 성공했는지는 응답에 담기지 않고, 실패하면 아무 안내 없이 목록의 상태만 조용히 되돌아가기 때문입니다.

#### 해결
**1. 전달 방식의 선택**

실시간 전달 방식으로는 WebSocket과 SSE도 검토했습니다.
다만 재고 처리는 보통 수 초 내에 끝나고, 사용자가 결과를 기다리는 구간은 승인·배송 직후의 짧은 화면 하나뿐입니다.
상시 연결을 유지하기 위한 서버 자원과 구현 복잡도에 비해 얻는 이점이 크지 않다고 판단해, **클라이언트 폴링**으로 결정했습니다.
진행 상태 조회는 외부 서비스 호출 없이 자체 DB만 읽는 `readOnly` 트랜잭션이라 짧은 주기로 반복해도 부담이 적습니다.

**2. 조합 규칙을 백엔드가 소유**

진행 상태는 발주 상태와 saga 상태의 조합으로 결정됩니다.
이 조합 규칙을 클라이언트가 해석하게 하면 규칙이 바뀔 때마다 양쪽을 함께 수정해야 하고, 화면마다 해석이 어긋날 수 있습니다.
그래서 조합 규칙을 백엔드의 `OrderProgress`가 단독으로 소유하고, 클라이언트에는 **파생된 단일 값**만 내려줍니다.

```java
public static OrderProgress from(SalesOrderStatus status, SagaStatus saga) {
    return switch (status) {
        case DRAFT -> DRAFT;
        case REQUESTED -> saga == SagaStatus.FAILED ? OUTBOUND_FAILED : REQUESTED;
        case APPROVED -> switch (saga) {
            case SENDING, PROCESSING -> OUTBOUND_IN_PROGRESS;
            case FAILED -> INBOUND_FAILED;
            case NONE, DONE -> APPROVED;
        };
        case DELIVERED -> saga.inProgress() ? INBOUND_IN_PROGRESS : DELIVERED;
        case REJECTED -> REJECTED;
        case CANCELED -> CANCELED;
    };
}
```

**3. 폴링 종료 조건과 실패 사유까지 함께 전달**

`GET /sales-orders/{code}/progress` 응답에는 화면 라벨용 `progress`뿐 아니라 폴링을 계속할지 판단하는 `pending`, 최종 결과인 `outcome`, 실패 시 `failureReason`을 함께 내려줍니다.
클라이언트는 `pending`이 `false`가 될 때까지만 폴링하고, 실패한 경우 보상으로 되돌아간 이유를 그대로 안내할 수 있습니다.
언제까지 폴링해야 하는지를 클라이언트가 직접 판단하지 않아도 되므로, 이후 상태가 추가되어도 클라이언트 수정이 필요 없습니다.

> 관련 코드: <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/domain/model/salesorder/OrderProgress.java">OrderProgress.java</a>, <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/application/service/GetSalesOrderProgressService.java">GetSalesOrderProgressService.java</a>

<br>

### 발주번호 채번에서 락 제거

#### 배경
발주번호는 `SO-2026-09-0001`처럼 월별로 1번부터 순차 증가하는 형식입니다.
순번이 겹치면 PK 충돌로 발주 생성 자체가 실패하기 때문에, 초기에는 월별 시퀀스 행을 비관적 락으로 조회한 뒤 증가시켜 저장했습니다.

이 방식은 정확했지만 락을 잡은 트랜잭션이 커밋될 때까지 다른 발주 생성 요청이 모두 대기합니다.
발주 생성 트랜잭션에는 채번 이후에도 창고 검증, 품목 조회 등 외부 서비스 호출이 남아 있어 락 보유 시간이 길고,
요청이 몰리는 시기에는 이 구간이 그대로 병목이 됩니다. 조회와 증가가 별도 쿼리로 나뉘어 왕복이 두 번 발생하는 것도 부담이었습니다.

#### 해결
락으로 순서를 보장하는 대신, **DB의 행 수준 원자성에 맡기는 방식**으로 전환했습니다.
PostgreSQL의 <a href="https://www.postgresql.org/docs/current/sql-insert.html">`INSERT ... ON CONFLICT DO UPDATE`</a>는 충돌 시 UPDATE까지 하나의 원자적 연산으로 처리하고, `RETURNING`으로 증가된 값을 곧바로 돌려줍니다.

```java
Integer lastSeq = jdbcTemplate.queryForObject("""
        INSERT INTO so_number_sequences (seq_date, last_seq)
        VALUES (?, 1)
        ON CONFLICT (seq_date)
        DO UPDATE SET last_seq = so_number_sequences.last_seq + 1
        RETURNING last_seq
        """,
        Integer.class,
        monthKey);
```

이 전환으로 세 가지가 개선되었습니다.

1. **락 보유 구간 축소** — 애플리케이션이 명시적으로 락을 잡고 트랜잭션 끝까지 들고 있던 구조가 사라지고, 행 잠금이 단일 문장 실행 시간으로 줄었습니다.
2. **DB 왕복 감소** — 조회 + 저장 2회 왕복이 1회로 줄었습니다.
3. **예외 처리 제거** — 월 첫 채번 시 동시 INSERT 충돌을 막기 위해 `DataIntegrityViolationException`을 잡아 재조회하던 방어 코드가 필요 없어졌습니다. 충돌 처리를 DB가 담당하므로 분기 자체가 사라집니다.

> 관련 코드: <a href="https://github.com/SeungWoo-Ahn/ERP-SalesService/blob/develop/src/main/java/com/fallguys/salesservice/adapter/outbound/persistence/SalesOrderPersistenceAdapter.java">SalesOrderPersistenceAdapter.java</a>
