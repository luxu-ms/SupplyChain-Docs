# SupplyChain architecture

The frontend serves the browser UI and calls the backend synchronously over HTTP. The backend owns business rules and MySQL persistence. It attempts to publish events to RabbitMQ; the worker is outside the HTTP request path and simulates notifications rather than sending them.

```mermaid
flowchart LR
    Browser <-->|HTML and form submissions| Frontend["Frontend (Spring MVC / JSP)"]
    Frontend <-->|REST / JSON| Backend["Backend (Spring Boot)"]
    Backend <-->|JPA| MySQL[(MySQL)]
    Backend -.->|Best-effort publish| RabbitMQ[RabbitMQ]
    RabbitMQ -->|Queue deliveries| Worker["Worker (Spring AMQP)"]
    Backend -->|Vendor rating lookup| Ratings[External rating service]
```

## Components and ownership

| Component | Responsibility | Build and runtime |
| --- | --- | --- |
| Frontend | Renders server-side JSP pages. Its `RestTemplate` client handles calls to the backend rather than calling MySQL or RabbitMQ. | [Maven WAR][frontend-pom] deployed to [Tomcat][frontend-docker]. |
| Backend | Exposes REST APIs for orders, inventory, and vendors. Owns validation, status transitions, persistence, and event publication. | [Spring Boot REST/JPA][backend-pom], packaged as an [executable JAR][backend-docker]. |
| Worker | Consumes RabbitMQ messages independently of browser requests. Handlers log receipt, simulate processing, and log completion. | [Spring Boot AMQP][worker-pom], packaged as an [executable JAR][worker-docker]. |

All three projects target Java 8 and Spring Boot 2.7.18. MySQL stores [purchase orders][order-model], [inventory][inventory-model], and [vendors][vendor-model]. The backend uses Hibernate `ddl-auto: update` to update the schema, not to seed vendor records. Purchase orders store a vendor ID and total amount, but no line items, SKUs, or quantities. The [vendor rating endpoint][vendor-rating] calls an external service and returns `0.0` on error or an empty response; order creation checks the vendor's active status, not its rating.

## Async contracts

The backend [declares three durable RabbitMQ queues and a Jackson JSON converter][queues]. It publishes through the default exchange using the queue name as the routing key. The producer passes whole entities to the converter rather than using dedicated event DTOs. The worker's [listeners][listeners] accept strings and log simulated processing.

| Queue | Producer in the reviewed code | Worker behavior |
| --- | --- | --- |
| `order.created` | Order creation attempts to publish the saved `PurchaseOrder` before the database transaction commits. | Simulates order-created notification processing. |
| `inventory.alert` | A helper attempts to publish an `Inventory` entity for each item at or below safety stock, but no caller or scheduler was found. | Simulates low-stock notification processing. |
| `approval.pending` | Declared, but no publisher was found. Approving an order only updates MySQL. | Simulates an approval notification. |

The [order service][order-service] catches publish failures without failing order creation. The database write and queue publish are not atomic, and a successful order response does not prove worker receipt.

**Known serialization risk:** The checked-in converter uses Spring AMQP 2.4.17's no-argument [`Jackson2JsonMessageConverter`][amqp-converter], which creates a plain mapper without Java-time support. Under the pinned dependencies, the `LocalDateTime` timestamps on saved orders and persisted inventory are expected to fail JSON conversion before a message reaches RabbitMQ. This is a source-based diagnosis; runtime overrides and broker delivery were not verified. The reviewed configuration does not show an outbox, explicit publisher confirms, retries, or dead-letter handling.

## Scope

This is a source-code snapshot, not a description of a verified deployment. Source links pin the reviewed revisions: backend `a21f57e8`, frontend `c613adbc`, and worker `458f2bab`. See [typical workflows](workflow.md) for request paths and a local smoke check, and the [full research report](research.md) for evidence and limitations.

[frontend-pom]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/pom.xml#L8-L55
[frontend-docker]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/Dockerfile#L2-L22
[backend-pom]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/pom.xml#L8-L54
[backend-docker]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/Dockerfile#L6-L24
[worker-pom]: https://github.com/luxu-ms/SupplyChain-Worker/blob/458f2bab94b08b3acd167feb04fc485a7ecf6133/pom.xml#L8-L40
[worker-docker]: https://github.com/luxu-ms/SupplyChain-Worker/blob/458f2bab94b08b3acd167feb04fc485a7ecf6133/Dockerfile#L2-L18
[order-model]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/model/PurchaseOrder.java#L14-L76
[inventory-model]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/model/Inventory.java#L13-L50
[vendor-model]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/model/Vendor.java#L14-L25
[vendor-rating]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/service/VendorService.java#L46-L58
[queues]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/config/AppConfig.java#L43-L67
[amqp-converter]: https://github.com/spring-projects/spring-amqp/blob/v2.4.17/spring-amqp/src/main/java/org/springframework/amqp/support/converter/Jackson2JsonMessageConverter.java#L45-L60
[listeners]: https://github.com/luxu-ms/SupplyChain-Worker/blob/458f2bab94b08b3acd167feb04fc485a7ecf6133/src/main/java/com/acme/scm/worker/listener/OrderMessageListener.java#L16-L56
[order-service]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/service/PurchaseOrderService.java#L36-L61
