# Consuming Messages from Streams

Consuming messages from a stream can be done using one of three different methods depending on you application needs.
You can access these methods from the consumer object you can create using JetStream context:

```csharp
await using var nats = new NatsConnection();
var js = new NatsJSContext(nats);

var consumer = await js.CreateConsumerAsync(stream: "orders", consumer: "order_processor");
```

## Next Method

Next method is the simplest was of retrieving messages from a stream. Every time you call the next method you get
a single message or nothing based on the expiry time to wait for a message. Once a message is received you can
process it and call next again for another.

```csharp
while (!cancellationToken.IsCancellationRequested)
{
    var next = await consumer.NextAsync<Order>();

    if (next is { } msg)
    {
        Console.WriteLine($"Processing {msg.Subject}: {msg.Data.OrderId}...");
        await msg.AckAsync();
    }
}
```

Next is the simplest and most conservative way of consuming messages since you request a single message from JetStream
server then acknowledge it before requesting more messages. However, next method is also the least performant since
there is not message batching.

## Fetch Method

Fetch method requests messages in batches to improve the performance while giving the application the control over how
fast it can process the messages without overwhelming the application process.

```csharp
while (!cancellationToken.IsCancellationRequested)
{
    // Consume a batch of messages (1000 by default)
    await foreach (var msg in consumer.FetchAllAsync<Order>())
    {
        // Process message
        await msg.AckAsync();
    }
}
```

## Consume Method

Consume method is the most performant method of consuming messages. Request for messages (a.k.a. pull requests) are
interleaved so that there is a constant flow of messages from the JetStream server. Flow is controlled by `MaxMsgs`
or `MaxBytes` and respective thresholds not to overwhelm the application and not to waste server resources.

```csharp
await foreach (var msg in consumer.ConsumeAllAsync<Order>())
{
    // Process message
    await msg.AckAsync();

    // loop never exits unless there is an error or a break
}
```

## Handling Exceptions

While consuming messages (using next, fetch or consume methods) there are several scenarios where exception might be
thrown by the client library, for example:

* Consumer is deleted by another application or operator
* Connection to NATS server is interrupted (mainly for next and fetch methods, consume method can recover)
* Client request for the next batch is invalid
* Account permissions have changed
* Cluster leader changed

A naive implementation might try to recover from errors assuming they are temporary e.g. the stream or the consumer
will be created eventually:

```csharp
while (!cancellationToken.IsCancellationRequested)
{
    try
    {
        await consumer.RefreshAsync(); // or try to recreate consumer
        await foreach (var msg in consumer.ConsumeAllAsync<Order>())
        {
            // Process message
            await msg.AckAsync();
        }
    }
    catch (NatsJSProtocolException e)
    {
        // log exception
    }
    catch (NatsJSException e)
    {
        // log exception
        await Task.Delay(1000); // or back off
    }
}
```

Depending on your application you should configure streams and consumers with appropriate settings so that the
messages are processed and stored based on your requirements.