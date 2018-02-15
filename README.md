# kafka-event-sourcing-demo

## Project Description ###

#### short ####
a demo of how one might use event-sourcing to simplify cross-team functionality at scale

#### long ####
event-sourcing is a powerful methodology for managing behavioral complexity across a distributed system, 
providing means for you to:
- maintain an **audit trail of logs** for each service
- **simplify inter-team coordination** by distributing and decoupling non-sequential behavior
- compile and organize meaningful **data for analytics** (e.g. usage stats) 

it's sometimes difficult to explain without an example, so here's one.

## Explanation ##

the example being used for this repository is **processing a return** of a physical device within an
  Enterprise Resource Planning (ERP) system. 
  
both the `traditional` and `event-sourced` workflows start when the **User Support (US)** team receives
  a return request from an external user. 
  
### Traditional Model ### 
in the more `traditional` model **US** must manually and directly manage _explicit calls to each team's endpoints_:
1. **US** _creates_ a new `OrderRefundRequest` with **Sales** to refund the customer's purchase
2. **US** _creates_ a new `ShipmentRequest` with **Inventory** to ship the user a replacement device
3. **US** _registers_ a new `DissassemblyOrder` with **Manufacturing** to get them to diagnose and disassemble the item
4. **US** _charges_ a new `AssetDisposal` with **Accounting** track the added cost to the company’s bottom line

### Event-Sourced Model ###
in the `event-sourced` model, **US** only has to manage the publication of the initial event and is abstracted from the 
  effect that event has on its subscribers
1. **US** _publishes_ a `RefundRequest` to an established topic with only the information necessary to _identify_ each piece
2. _each subscriber_ responds asynchronously to the event
    - **Sales** responds to the `Order` information
    - **Inventory** responds to the `Model` and `Shipment` information
    - **Manufacturing** responds to the `DefectiveItem` and its `Location`
    - **Accounting** responds to the `Cost` information

### Benefits ###
- the _publisher_ is only responsible for one repeatable call (creating the event)
- the responsibility for the _rest_ is handled instead by each _subscriber_, preserving bounded context for
  - the _data_ relevant to their domain
  - the _conceptual organization_ of how that data effects that team (e.g. **US** needs no concept of `AssetDisposal`)
  - the _implementation_ of any particular stepwise actions (e.g. _registering_ a `DisassemblyOrder`)
- the processing of each event can be handled _independently_ and _asynchronously_
  - outages or latency of one system does not propogate to others
- changes can be made with minimal coordination with the publisher
  - existing subscribers can easily update their response behavior
  - new subscribers can begin responding or logging events
- system-level occurences are plainly stored _as events_ and not just as _specific method calls_

### Drawbacks ###
- you must serve and maintain a separate system for passing and responding to message queues 
- you have to manage the growth and coordination of "events" and "topic streams"
- adds a layer of abstraction that may not be necessary for systems that aren't complex enough to warrant its use

## Usage ## 

Following the [kafka quickstart documentation](https://kafka.apache.org/quickstart):

1. Start Zookeeper server (and keep running)
``` 
kafka_2.11-1.0.0$ bin/zookeeper-server-start.sh config/zookeeper.properties
...
INFO binding to port 0.0.0.0/0.0.0.0:2181 (org.apache.zookeeper.server.NIOServerCnxnFactory)
```

2. Start the Kafka server (and keep running)
```
kafka_2.11-1.0.0$ bin/kafka-server-start.sh config/server.properties
...
INFO [KafkaServer id=0] started (kafka.server.KafkaServer)
```

3. Create a topic
```
kafka_2.11-1.0.0$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic topic1
Created topic "returnOrders".
```

4. Create producer (on a given topic) and populate with a few lines
```
kafka_2.11-1.0.0$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic returnOrders
> This is the first message
> This is the second message
```

5. Create consumer/s (on the same topic) and watch the lines become updated in turn
```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
> This is the first message
> This is the second message
```