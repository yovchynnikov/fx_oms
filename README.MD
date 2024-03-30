## Investigation details

It was hard to comprehend the task, as it has several unknown (domain-specific) details, which I couldn't clarify at the sufficient level.
I investigated the main elements (FIX, SOR, OMS), and decided to simply follow details.

Below are my questions and assumptions:
- Is FIX adapter is a part of the system? Assumed YES, as we need to convert messages to/from FIX.
- FIX protocol is async, so validation errors should be sent to FIX (in theory).
- How should we inform the client about order state changes? In theory the notification should be sent through FIX, alternatively through other channels (like email, callbacks etc). Assumed first option.
- No idea how orders are passed to SOR, nor execution reports are received.
- No mention how clients are validating. I'm assuming it's a separate system.

Skipped next topics:
- idempotency
- SOR communications

## High-level design

![High-level system diagram](FX_OMS_1.drawio.svg)

### FIX adapter

It's a stateful adapter FIX<->OMS, which also communicates with SOR (see above). It's probably better to split input and output flows into separate components, however without understanding communication details, I put it all into the same component.
Component's functionality:
- receive FIX requests, convert them and pass to the Order Processing Service (OPS).
- receive SOR requests from the system, convert them into FIX and pass to SOR.
- receive SOR requests from FIX, convert them and pass to OPS.


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
It's a distributed cache that allows to store some client's information to reduce network calls.

## Design decisions

1. Most of the communications between components are async.
2. Some of the communications are synchronous, for example, processing of CreateOrder events. The whole operation must be atomic, and any unrecoverable error on any step should result in rollbacks of previous steps. Saga pattern can be used to make it async, but in this case it should be overkill.
3. In case of client validation, it's possible to use a cache to decrease the loading, as this data is rarely changed. Depending on the consistency requirements, short or long TTL can be used, and some feedback loops with external system would be used/introduced.
4. All the internal synchronous calls should be secured by authorization/authentication/RBAC (zero trust security). I believe the FX system already has some authentication provider. To secure connections, a system account can be used. 
5. For MQ it's possible to use RabbitMQ with routing or Kafka with several topics and partitions; that would allow reading the same messages by multiple recipients and filter messages at the same time. MQ should have DLQ/DLT for failed processing.  
6. System should be permanently monitored; failed requests and processings (including DLQ/DLT) should trigger alarms of different severity.
7. Validating the client's balance is not a simple thing. In the real life such validation involves locking the deducted amount at the beginning and reducing balance and locked amount on every update. For this design I use a single check for the balance, for brewity reason.
8. Load Balancers are not the part of the design here, but it's implied that a) there are load balancers before every service and b) services are scalable. 


## Order History Service

This is a very brief description; the data definition contains only two example fields.

Tech stack: Java 17, Maven, Spring Boot 3.2 (Spring Data, Spring MVC, Spring Metrics, Spring Security).

### Models
```
OrderHistory {
    @NonEmpty String orderId,
    @NotNull Instant timestamp
}

OrderHistoryDTO {
    String orderId,
    Instant timestamp
}
```

### Security

Authorization & RBAC.

### REST API

Rest API has next endpoints:
1. Get the order's audit data for the given order. This should return a collection or stream of OrderHistoryDTO.
```
GET /v1/ordershistory/<order_id>
Headers:
- bearer authorization token
Roles:
- FX_OMS_AUDIT_READ
Status code: 200
Exceptions:
- 403 Unauthorized
Format: JSON
```
Example:
```
[
    {
        "order_id": 12345,
        "timestamp": "2024-03-30T18:26:47Z"
    },
    {
        "order_id": 12345,
        "timestamp": "2024-03-30T18:21:01Z"
    }
]
```

### RabbitMQ

1. OrderUpdated listener

- Exchange name: order_updated
- Queue name: order_updated.order_history_service
- DLQ: order_updated.order_history_service.dlq
- Validations: as per model; invalid events should be rejected and go to DLQ

### Databases

1. OrdersHistory

- Type: Cassandra
- Keyspace: ordershistory
- Table: ordershistory
- Keys: partition key order_id, cluster key timestamp

Operations:
- insert
- select * where order_id = <order_id> order by timestamp descending

### Monitoring

Service should log all exceptional situation in brief and concise way. The pattern for the whole system will be described in system documentation.

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


