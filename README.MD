## Investigation details

It was hard to comprehend the task, as it has several unknown (domain-specific) details, which I couldn't clarify at the sufficient level.
I investigated the main elements (FIX, SOR, OMS), and decided to simply follow details.

Below are my questions and assumptions:
- Is FIX adapter is a part of the system? Assumed YES, as we need to convert messages to/from FIX.
- FIX protocol is async, so validation errors should be sent to FIX (in theory).
- How should we inform the client about order state changes? In theory the notification should be sent through FIX, alternatively through other channels (like email, callbacks etc). Assumed first option.
- No idea how orders are passed to SOR, nor execution reports are received.
- No mention how clients are validating. I'm assuming it's a separate system.
- No idea is there clientId is a part of the orders processing. It should be, but without deeper understanding I decided to exclude it from the design.
- no NFR requirements (users count; rpm/rps; peaks) makes impossible to make better decisions

Skipped next topics:
- idempotency
- SOR communications
- clientId as a part of orders processing

## High-level design

![High-level system diagram](FX_OMS_1.drawio.svg)

### FIX adapter

It's a stateful adapter FIX<->OMS, which also communicates with SOR (see above). It's probably better to split input and output flows into separate components, however without understanding communication details, I put it all into the same component.
Component's functionality:
- receive FIX requests, convert them and pass to the Order Processing Service (OPS).
- receive SOR requests from the system, convert them into FIX and pass to SOR.
- receive SOR requests from FIX, convert them and pass to OPS.


### Authentication service (optional)

If global authentication provider cannot be used for the system, system should contain own authentication provider. Responsibilites:
- authenticate UI users (OAuth)
- authenticate internal communications (OAuth)
- authenticate REST API requests (API key)

### REST API gateway
This is a simple gateway that allows to serve public API requests, such as CancelOrder. 

### UI gateway
This is a simple gateway that allows to serve UI requests, such as GetOrderAudit. 

### Order Processing Service (OPS)

It's a stateless service that allows to orchestrate requests for orders processing. Every request consists of several operations (validation and storing), and involves calling different services.  
- processing CreateOrder requests
- processing CancelOrder requests
- validating orders (calls to external systems)
This component is very important part of the system and probably the most loading part of it. It's important to reduce loading, for example by introducing reactive requests processing.  

### Orders Changes MQ Service (OMQ)
This is MQ topic(s), which allows processing OrderUpdated events.

### Order Service (OS)

This is a simple CRUD service which allows operating with orders. Input requests are synchronous
- processing create order requests
- processing update order requests
- generating OrderUpdated events for OMQ

### Order History Service (OHS)

This component allows storing the changes when processing the order: create, update, finish. Also, it serves UI requests, allowing to get audit data.
- read OrderUpdated events from OMQ
- store events in the database
- process getOrderAudit requests

### Order Notification Service (ONS)

This component analyses OrderUpdated events and notifies clients about major changes of orders.  
- read OrderUpdated events from OMQ
- calling FIX adapter to inform clients
- alternatively inform clients in some another way

### Client Proxy (CP)
This component allows communicating with external system for client validations. It should obtain receive of client's data that allows to validate it, but not expose sensitive information.
The external system can just return True/False (or some extended status), depending on whether the client exists or not, and is active.   

### Client Balance Proxy (CBP)
This component allows communicating with external system in order to validate that client has enough funds on the "from" currency account. 
The external system can return 202 Accepted or some of 4xx error, signalling that there are not enough funds.   

### Client Cache (CC)
It's a distributed cache that allows to store some client's information to reduce network calls. Any distributed cache can be used, for example Hazelcast.

## Design decisions

1. Most of the communications between components are async.
2. Some of the communications are synchronous, for example, processing of CreateOrder events. The whole operation must be atomic, and any unrecoverable error on any step should result in rollbacks of previous steps. Saga pattern can be used to make it async, but in this case it should be overkill.
3. In case of client validation, it's possible to use a cache to decrease the loading, as this data is rarely changed. Depending on the consistency requirements, short or long TTL can be used, and some feedback loops with external system would be used/introduced.
4. For MQ, it's possible to use RabbitMQ with routing or Kafka with several topics and partitions; that would allow reading the same messages by multiple recipients and filter messages at the same time. MQ should have DLQ/DLT for failed processing.
5. System should be permanently monitored; failed requests and processings (including DLQ/DLT) should trigger alarms of different severity. Existing systems can be used for monitoring and tracing; alternatively Prometheus is good for metrics, Grafana for dashboard, Splunk for logs and alerts. Service should log all exceptional situation in brief and concise way (<here is some log pattern>)
6. Validating the client's balance is not a simple thing. In the real life, such validation involves locking the deducted amount at the beginning and reducing balance and locked amount on every update. For this design, I use a single check for the balance, for brevity reason.
7. Load Balancers are not the part of the design here, but it's implied that a) there are load balancers before every service and b) services are scalable.
8. An important part of the system is authentication. No idea how FIX requests are authenticated, but for UI OAuth can be used, and REST API requests can be authenticated by API keys.
9. Authorization should be based on RBAC and client-based access. It's not enough just to secure access to endpoints by global roles, the system also should restrict access by clients. Because of this, clientId is a part of UI and REST APIs.
10. OPS, OS, CP and CBP can be designed as one service to reduce latency and increase the performance, instead of creating separate services. I put them on diagram as standalone components to express one of microservices idea: single responsibility.
11. Technical stack is depending on the existing stack. If it's possible, Java 21 LTS and SpringBoot 3.2.x should be used.
12. It's supposed to use kubernetes to manage microservices. CI/CD should utomatically deploy services on testing environments and automatically run e2e and system tests.
13. For MVP diamond testing approach should be used, with bias to component tests. Later, coverage for UT should be increased up to 70%. UI requests should be tested with e2e (Cypress), REST API - with system tests. Services communications should be tested with contract tests. Performance tests (Gatling) should test critical system's flows.


## Order History Service

This is a very brief description; the data definition contains only two example fields.

### Technical stack:

- Java 21 or existed)
- Maven
- Spring Boot 3.2 (Spring Data, Spring MVC, Spring Metrics, Spring Security).

### Models

There are two models in the service:
- OrderHistory
- OrderHistoryDTO

### Security

Access to the service is secured by:
- OAuth 2.0 JWT authentication
- RBAC & client authorization

### REST API

Rest API has next endpoints:
1. Get the order's audit data for the given order. This should return a collection or stream of OrderHistoryDTO.
```
GET /v1/ordershistory/clients/<client_id>/orders/<order_id>
Headers:
- bearer authorization JWT token
Status code: 200
Exceptions:
- 403 Unauthorized
Format: JSON
Returns: list of OrderHistoryDTO
```

### RabbitMQ

1. OrderUpdated listener

- Exchange name: order_updated
- Queue name: order_updated.order_history_service
- DLQ: order_updated.order_history_service.dlq
- Validations: as per model; invalid events should be rejected and go to DLQ

### Persistence

1. OrdersHistory

- Type: Elasticsearch
- Index: ordershistory
- Partitioning: sharding

Operations:
- index
- query

### Monitoring

Metrics:
- CPU
- Memory
- successful requests (rpm, count)
- unsuccessful requests (rpm, count)

Alerts:
1. When new items arrived into order_updated.order_history_service.dlq.
Severity: high
Frequency: 1 min
2. When error appears in the log
Severity: high
Frequency: 5 min
3. When Unauthorized error appears in the log
Severity: critical
Frequency: 1 min

Dashboards:
1. REST RPM success/unsuccess
2. Average REST response time (per last hour)
3. Average REST response time (per last week)
4. MQ events/sec
5. MQ queue size
6. DLQ count
7. Database size
