# grails-transactional-events

Sample app for the **[Cross-Domain Logic in Grails with Transactional Spring Events](https://grails.apache.org/guides/grails-transactional-events/8/guide/index.html)** guide on [grails.apache.org](https://grails.apache.org/guides/).

The guide demonstrates the cleanest Grails 8 answer to a recurring problem: how to fan out business logic across multiple unrelated domains after a domain object changes state, *but only if the change actually commits*. The pattern is Spring's `ApplicationEventPublisher` + `@TransactionalEventListener(phase = AFTER_COMMIT)`, and it works out-of-the-box with GORM because GORM rides on Spring's `PlatformTransactionManager`.

## Layout

| Directory | What it is |
|---|---|
| `initial/` | Vanilla Grails 8 starter, what you have after generating a new app on [start.grails.org](https://start.grails.org). |
| `complete/` | The fully wired sample - three domain classes (`Customer`, `Order`, `AuditLog`), the `OrderPlacedEvent` POGO, the `OrderService` publisher, three `@TransactionalEventListener` beans, and the Spock `@Integration` spec that proves the AFTER_COMMIT contract. |

## Running

```bash
cd complete
./gradlew integrationTest
```

The integration spec asserts both halves of the contract:

1. AFTER_COMMIT listeners fire when (and only when) the publisher's transaction commits.
2. AFTER_COMMIT listeners are silently skipped when the publisher's transaction rolls back - no phantom audit rows, no phantom emails for orders that never durably existed.

## Requirements

- JDK 21
- About 30 minutes to read the guide end-to-end.

## The pattern, in one place

```groovy
// 1. Publish a POGO from a @Transactional service:
@Transactional
class OrderService {
    @Autowired ApplicationEventPublisher applicationEventPublisher

    Order placeOrder(Long customerId, BigDecimal total) {
        Order order = new Order(...).save(failOnError: true)
        applicationEventPublisher.publishEvent(new OrderPlacedEvent(order.id, customerId, total))
        order
    }
}

// 2. Consume it from any number of listener beans:
class AuditListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void onOrderPlaced(OrderPlacedEvent event) {
        new AuditLog(...).save(failOnError: true)
    }
}
```

That's it. No GORM events plugin, no `EventBus`, no `@Subscriber`. Spring's transaction-synchronization manager fires the listener after the database commit lands. On rollback, the listener is silently skipped.

See the [guide](https://grails.apache.org/guides/grails-transactional-events/8/guide/index.html) for the full walk-through, including a side-by-side contrast with GORM lifecycle callbacks and the `@Async` composition for off-thread dispatch.
