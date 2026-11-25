# SupplyChain workflows

The main path is creating a purchase order in the browser, storing it in MySQL, and attempting to notify the worker asynchronously. See [architecture](architecture.md) for component ownership and queue contracts.

## Browse the dashboard

The [dashboard controller][dashboard] gets summary counts from the backend. It displays zero counts on exceptions, so zeros alone do not confirm an empty database. The backend provides read endpoints for [orders][order-api] (list, detail, pending), [inventory][inventory-api] (list, SKU, low stock), and [vendors][vendor-api] (list, code, rating).

## Create a purchase order

1. Open the [new order form][new-form], select a vendor, and submit the order. The form uses hardcoded vendor options, not the vendor API.
2. The [frontend order controller][frontend-orders] forwards the create request to backend `POST /api/orders` through its [HTTP client][frontend-api]. The frontend redirects on success.
3. The [backend order service][create] validates a positive amount and an active vendor before saving. If status is absent, [persistence defaults it to `PENDING`][order-default].
4. After saving, the service attempts to publish `order.created` to RabbitMQ. If the event reaches the [worker listener][listeners], it logs and simulates processing. A successful frontend redirect confirms neither worker receipt nor a real notification.

## Approve a pending order

The [backend approval endpoint][order-api] accepts `PUT /api/orders/{orderNumber}/approve?approvedBy=...`. The [service][approve] transitions only `PENDING` orders to `APPROVED`. The [pending orders page][pending-page] shows an approval link, but the [frontend controller][frontend-orders] does not handle that GET route, so the reviewed UI does not complete approval. Use the backend API to exercise the implemented transition. No producer for `approval.pending` was found.

## Review low stock

The [inventory API][inventory-api] exposes a low-stock view for items whose [quantity is at or below safety stock][threshold]. This is a read path: opening it does not trigger a notification. The [alert publisher][inventory-service] has no caller or scheduler in the reviewed code, and the worker handler simulates rather than sends an alert.

## Local smoke-check sequence

1. Provide MySQL and RabbitMQ, configure the backend to use them, and start the backend. Ensure there is an active test vendor matching an option in the order form. Keep credentials out of commands and docs that will be shared.
2. Configure the frontend backend URL and start the frontend and worker. Point the worker at the same broker and queue names as the backend. When running the frontend WAR in the [external Tomcat container][frontend-docker], verify the reachable port rather than assuming the Spring `server.port` setting controls Tomcat's connector.
3. Submit a valid order, confirm that it appears in the order list and pending view, and check worker logs separately for receipt on `order.created`. If the order exists but no message arrives, check backend publish logs, broker connectivity, and message conversion.
4. Approve the order through the backend API, then verify that the order detail shows `APPROVED` and the order no longer appears in the pending list.

This sequence is derived from the reviewed source; the services were not run together during the research.

[dashboard]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/java/com/acme/scm/frontend/controller/HomeController.java#L30-L62
[new-form]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/webapp/WEB-INF/jsp/orders/new.jsp#L58-L81
[frontend-orders]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/java/com/acme/scm/frontend/controller/OrderController.java#L21-L71
[frontend-api]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/java/com/acme/scm/frontend/service/BackendApiService.java#L61-L65
[pending-page]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/webapp/WEB-INF/jsp/orders/pending.jsp#L74-L83
[frontend-docker]: https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/Dockerfile#L2-L22
[order-api]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/controller/PurchaseOrderController.java#L25-L57
[create]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/service/PurchaseOrderService.java#L36-L61
[approve]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/service/PurchaseOrderService.java#L77-L90
[order-default]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/model/PurchaseOrder.java#L56-L63
[inventory-api]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/controller/InventoryController.java#L20-L35
[inventory-service]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/service/InventoryService.java#L41-L63
[threshold]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/model/Inventory.java#L48-L50
[vendor-api]: https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/controller/VendorController.java#L20-L36
[listeners]: https://github.com/luxu-ms/SupplyChain-Worker/blob/458f2bab94b08b3acd167feb04fc485a7ecf6133/src/main/java/com/acme/scm/worker/listener/OrderMessageListener.java#L16-L56
