# SupplyChain workflows

The implemented path is to create a purchase order in the browser, find it in the pending list, and approve it through the backend API. Worker notification is a separate, best-effort path. See [architecture](architecture.md) for queue contracts and [research.md](research.md) for the full source review.

## Browse the dashboard

The [dashboard controller][dashboard] gets summary counts from the backend. It displays zero counts on exceptions, so zeros alone do not confirm an empty database. The backend provides read endpoints for [orders][order-api] (list, detail, pending), [inventory][inventory-api] (list, SKU, low stock), and [vendors][vendor-api] (list, code, rating).

## Create a purchase order

1. Open `/orders/new` and enter a unique order number, a vendor, a positive amount, and a requester. Description is optional. The [form][new-form] hardcodes vendor IDs `1` through `4`; it does not load them from the vendor API.
2. The [frontend order controller][frontend-orders] forwards the form to backend `POST /api/orders` through its [HTTP client][frontend-api]. It redirects to `/orders` on success or redisplays the form with an error if the backend call fails.
3. The [backend order service][create] validates a positive amount and an active vendor before saving. If status is absent, [persistence defaults it to `PENDING`][order-default].
4. After saving and before the database transaction commits, the service attempts to publish `order.created`. The checked-in converter is [expected to fail on persisted timestamps](architecture.md#async-contracts); the service logs publish errors without failing the order. If the message reaches the [worker listener][listeners], it logs simulated processing rather than sending a real notification.

## Approve a pending order

Open `/orders/pending` to find the order. The [pending page][pending-page] shows an Approve link that sends `GET /orders/{orderNumber}/approve`, but the [frontend controller][frontend-orders] has no matching handler. Use the [backend endpoint][order-api] instead: `PUT /api/orders/{orderNumber}/approve?approvedBy=...`.

The [service][approve] transitions only `PENDING` orders to `APPROVED` and records the approver and approval time. It does not publish an event or adjust stock. The reviewed [order][order-api] and [inventory][inventory-api] APIs do not implement order receipt.

## Review low stock

The [inventory API][inventory-api] exposes a low-stock view for items whose [quantity is at or below safety stock][threshold]. This is a read path: opening it does not trigger a notification. The [alert publisher][inventory-service] has no caller or scheduler in the reviewed code, and the worker handler simulates rather than sends an alert.

## Local smoke-check sequence

1. Provision MySQL and RabbitMQ and start the backend. The [backend defaults][backend-settings] are port `8080`, MySQL at `mysql:3306/supplychain`, and RabbitMQ at `rabbitmq:5672`. Provision an active test vendor separately with an ID the form offers. Keep credentials out of shared commands and docs.
2. Set the frontend's [`BACKEND_API_URL`][frontend-env] to the backend API base URL, including `/api`, and start the frontend. Start the worker with the same broker and queue names as the backend. For the frontend WAR in [external Tomcat][frontend-docker], verify the reachable port rather than assuming Spring's port setting controls the connector.
3. Submit a valid order and verify that it appears at `/orders` and `/orders/pending`. Check backend publish logs and worker receipt separately. If the order is saved but the worker receives nothing, check for the expected timestamp conversion error before treating it as a broker connectivity problem.
4. Approve using the backend API. Verify that `GET /api/orders/{orderNumber}` returns `APPROVED` and that the order no longer appears in `/orders/pending`.

This sequence is derived from the reviewed source; the services were not run together during the research.

[dashboard]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/java/com/acme/scm/frontend/controller/HomeController.java#L30-L62
[new-form]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/webapp/WEB-INF/jsp/orders/new.jsp#L58-L101
[frontend-orders]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/java/com/acme/scm/frontend/controller/OrderController.java#L21-L71
[frontend-api]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/java/com/acme/scm/frontend/service/BackendApiService.java#L61-L65
[pending-page]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/webapp/WEB-INF/jsp/orders/pending.jsp#L74-L83
[frontend-docker]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/Dockerfile#L2-L22
[frontend-env]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/README.md#L14-L20
[backend-settings]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/resources/application.yml#L1-L31
[order-api]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/controller/PurchaseOrderController.java#L25-L57
[create]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/service/PurchaseOrderService.java#L36-L61
[approve]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/service/PurchaseOrderService.java#L77-L90
[order-default]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/model/PurchaseOrder.java#L56-L63
[inventory-api]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/controller/InventoryController.java#L20-L35
[inventory-service]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/service/InventoryService.java#L41-L63
[threshold]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/model/Inventory.java#L48-L50
[vendor-api]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/controller/VendorController.java#L20-L36
[listeners]: https://github.com/luxu-ms/SupplyChain-Worker/blob/458f2bab94b08b3acd167feb04fc485a7ecf6133/src/main/java/com/acme/scm/worker/listener/OrderMessageListener.java#L16-L56
