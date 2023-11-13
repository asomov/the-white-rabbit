
<img src=https://github.com/viartemev/the-white-rabbit/assets/23705041/1bd2825b-1241-49d8-94fc-550c381969de width="200" height="200">

# RabbitMQ Kotlin
[![CI](https://github.com/viartemev/rabbitmq-kotlin/actions/workflows/gradle.yml/badge.svg?branch=master)](https://github.com/viartemev/rabbitmq-kotlin/actions/workflows/gradle.yml)
[![Open Source Helpers](https://www.codetriage.com/viartemev/the-white-rabbit/badges/users.svg)](https://www.codetriage.com/viartemev/the-white-rabbit)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Gitter](https://badges.gitter.im/kotlin-the-white-rabbit/community.svg)](https://gitter.im/kotlin-the-white-rabbit/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

The White Rabbit is a [fast](https://github.com/viartemev/the-white-rabbit/issues/88#issuecomment-470461937) and asynchronous RabbitMQ (AMQP) client library based on Kotlin coroutines. Currently the following features are supported:
* Queue and exchange manipulations
* Message publishing with confirmation
* Message consuming with acknowledgment
* Transactional publishing and consuming
* RPC pattern

## Adding to project

## Usage notes and examples

Use one of the extension methods on `com.rabbitmq.client.Connection` to get a channel you need:

```kotlin
connection.channel {
    /*
    The plain channel with consumer acknowledgments, supports:
        -- queue and exchange manipulations
        -- asynchronous consuming
        -- RPC pattern
     */
}

connection.confirmChannel { //
    /*
    Channel with publisher confirmations, additionally supports:
        -- asynchronous message publishing
     */
}

connection.txChannel { // transactional support
    /*
    Supports transactional publishing and consuming.
     */
}
```

### Queue and exchange manipulations
#### Asynchronous exchange declaration
```kotlin
connection.channel.declareExchange(ExchangeSpecification(EXCHANGE_NAME))
```
#### Asynchronous queue declaration
```kotlin
connection.channel.declareQueue(QueueSpecification(QUEUE_NAME))
```
#### Asynchronous queue binding to an exchange
```kotlin
connection.channel.bindQueue(BindQueueSpecification(EXCHANGE_NAME, QUEUE_NAME))
```

### Asynchronous message publishing with confirmation
```kotlin
connection.confirmChannel {
    publish {
        val messages = (1..n).map { createMessage("Hello #$it") }
        publishWithConfirmAsync(coroutineContext, messages).awaitAll()
    }
}
```
or
```kotlin
connection.confirmChannel {
     publish {
        coroutineScope {
            val messages = (1..n).map { createMessage("Hello #$it") }
            messages.map { async { publishWithConfirm(it) } }
        }
    }
}
```

### Asynchronous message consuming with acknowledgement
Consume only n-messages:
```kotlin
connection.channel {
    consume(QUEUE_NAME, PREFETCH_COUNT) {
        (1..n).map { async { consumeMessageWithConfirm({ println(it) }) } }.awaitAll()
    }
}
```

### Transactional publishing and consuming

RabbitMQ and AMQP itself offer rather scarce support for transaction. When considering using transactions you should be aware that:
* a transaction could only span one channel and one queue;
* `com.rabbitmq.client.Channel` is not thread-safe;
* channel can be either in confirm mode or in transaction mode at a time;
* transactions cannot be nested into each other;

 The library provides a convenient way to perform transactional publishing and receiving based on `transaction` extension function. This function commits a transaction upon normal execution of the block and rolls it back if a `RuntimeException` occurs. Exceptions are always propagated further. Coroutines are not used for publishing though, since there are no any asynchronous operations involved.

```kotlin
connection.txChannel {
    transaction {
        val message = createMessage(queue = oneTimeQueue, body = "Hello from tx")
        publish(message)
    }
}
```

### RPC pattern
```kotlin
connection.channel {
    val message = RabbitMqMessage(MessageProperties.PERSISTENT_BASIC, "Hello world".toByteArray())
    coroutineScope {
        (1..10).map {
            async {
                rpc {
                    call(requestQueueName = "rpc_request", message = message)
                        .also { println("Reply: ${String(it.body)}") }
                }
            }
        }.awaitAll()
    }
}
```

## Links
* [Benchmarks](https://github.com/viartemev/the-white-rabbit/issues/88#issuecomment-470461937)
