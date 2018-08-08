# WORKING IT AIS TUTORIAL

## Table Of Content
- [A simple boilerplate Azure Function](#a-simple-boilerplate-azure-function)
- [function.json](#function-json)
- [Send to Queue](#send-to-queue)
- [Timer function](#timer-function)
- [Receiving from Queue](#receiving-from-queue)
- [Receiving from Topic](#receiving-from-topic)
- [Write to Storage Queue](#write-to-storage-queue)
- [Write to Storage Table](#write-to-storage-table)

## A simple boilerplate Azure Function

```csharp

using System.Net; 
using System.Text;

public static async Task<HttpResponseMessage> Run(HttpRequestMessage req, TraceWriter log)
{
    string body = await req.Content.ReadAsStringAsync();
    string result = $"Hello there, you typed {body}";
    var response = new HttpResponseMessage(HttpStatusCode.OK) {
        Content = new StringContent(result, Encoding.UTF8, "text/plain")
    };
  
    return response;
 
}

``` 
[Back to top](#table-of-content)


### function json

this is a basic set up with an httpTrigger in that returns a response

```json

{
  "bindings": [
    {
      "authLevel": "function",
      "name": "req",
      "type": "httpTrigger",
      "direction": "in",
      "methods": [
        "get",
        "post"
      ]
    },
    {
      "name": "$return",
      "type": "http",
      "direction": "out"
    }
  ],
  "disabled": false
}

```
[Back to top](#table-of-content)

### Send To Queue

When a connectionString to a *Service Bus* queue has been set up in the Function App Settings the following can be appended to the **function.json** configuration 

```json

  {
      "type": "serviceBus",
      "connection": "workingitsbsender",
      "name": "outputSbMsg",
      "queueName": "outqueue",
      "accessRights": "Send",
      "direction": "out"
    }

```

A setting named **workingitsbsender** must be created first 

Now to write to the queue *the output param (outputSbMsg) just created* you need to do the following:

```csharp

using System.Net; 
using System.Text;

public static async Task<HttpResponseMessage> Run(HttpRequestMessage req, TraceWriter log, 
IAsyncCollector<string> outputSbMsg)
{

    string xslt = req.Headers.GetValues("xslt").FirstOrDefault();
    string body = await req.Content.ReadAsStringAsync();
    string result = $"Hello there, you typed {body}. xslt: {xslt}";

    //Submit to queue
    await outputSbMsg.AddAsync(result);
    var response = new HttpResponseMessage(HttpStatusCode.OK) {
        Content = new StringContent(result, Encoding.UTF8, "text/plain")
    };
  
    return response;
 
}


```

> Note: a parameter **IAsyncCollector<string>** was added and given the name *outputSbMsg* (Required)
> If the functions had not been an *async* function, *ICollector<T>* or simply *out T outputSbMsg* should have been used.
> out **cannot** be used in an async function, therefore in our case we need to use *IAsyncCollector*

[Back to top](#table-of-content)

## Timer function

The basic set up is as follows (using CRON)

```json

{
  "bindings": [
    {
      "name": "myTimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 */1 * * * *"
    }
  ],
  "disabled": false
}

```

```csharp

using System;

public static void Run(TimerInfo myTimer, TraceWriter log)
{
    log.Info($"C# Timer trigger function executed at: {DateTime.Now}");
}


```

### A timer example sending to a SB Queue

Lets join the things we have learned

```json

{
  "bindings": [
    {
      "name": "myTimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 */1 * * * *"
    },
    {
      "type": "serviceBus",
      "connection": "workingitsbsender",
      "name": "outputSbMsg",
      "queueName": "orders",
      "accessRights": "Send",
      "direction": "out"
    }
  ],
  "disabled": false
}

```

```csharp

using System;

public static void Run(TimerInfo myTimer, TraceWriter log, ICollector<string> outputSbMsg)
{
    log.Info($"C# Timer trigger function executed at: {DateTime.Now}");
    outputSbMsg.Add(DateTime.Now.ToString());
}


```

> Note that the timer is not *async* so ICollector is used.


[Back to top](#table-of-content)

## Receiving from Queue

When receiving from an artifact such as a **Service Bus Queue** the queue connection needs to be set up as *input* instead of *output*

```json

{
  "bindings": [
    {
      "name": "myQueueItem",
      "type": "serviceBusTrigger",
      "direction": "in",
      "queueName": "orders",
      "connection": "workingitsblistener",
      "accessRights": "Listen"
    }
  ],
  "disabled": false
}

```

```csharp

using System;

public static void Run(string myQueueItem, TraceWriter log)
{
    log.Info($"C# ServiceBus queue trigger function processed message: {myQueueItem}");
}


```

The example above just gets the content of the queue as a string.

A more advanced approach would be to use the **BrokeredMessage** class instead

```csharp

#r"Microsoft.ServiceBus"
using System;
using System.IO;
using Microsoft.ServiceBus.Messaging;

public static void Run(BrokeredMessage myQueueItem, TraceWriter log)
{
    Stream stream = myQueueItem.GetBody<Stream>();
    StreamReader reader = new StreamReader(stream);
    string s = reader.ReadToEnd();
    log.Info($"C# ServiceBus queue trigger function processed message: {s}. Property Count: {myQueueItem.Properties.Count}");

    foreach(var p in myQueueItem.Properties) {
        log.Info($"Property: {p.Key} Value: {p.Value}. Type: {p.Value.GetType().ToString()}");
    }
}

```

[Back to top](#table-of-content)

## Receiving from Topic

Same as Queue just use this *function.json* instead

```json

{
  "bindings": [
    {
      "name": "myQueueItem",
      "type": "serviceBusTrigger",
      "direction": "in",
      "topicName": "topic",
      "subscriptionName": "us",
      "connection": "workingitsblistener",
      "accessRights": "Manage"
    }
  ],
  "disabled": false
}


```

[Back to top](#table-of-content)

## Write to Storage Queue


```json

{
      "type": "queue",
      "name": "outputQueueItem",
      "queueName": "orders",
      "connection": "witstorage",
      "direction": "out"
}

```

Now we can submit messages to the queue (orders)
> Note Storage Queues DO support Async

```csharp

using System.Net; 
using System.Text;

public class Order {
    public string CustomerId {get; set;}
    public string Item {get; set;}
    public int Qty {get; set;}
    
}

public static async Task<HttpResponseMessage> Run(HttpRequestMessage req, 
TraceWriter log,
IAsyncCollector<Order> outputQueueItem)
{
    string body = await req.Content.ReadAsStringAsync();
    string result = $"Hello there, you typed {body}";
    
    //Write to queue Storage
    Order o = new Order() {CustomerId = "IBM", Item = "Matt", Qty = 17};
    await outputQueueItem.AddAsync(o);
    var response = new HttpResponseMessage(HttpStatusCode.OK) {
        Content = new StringContent(result, Encoding.UTF8, "text/plain")
    };
  
    return response;
 
}


```

[Back to top](#table-of-content)


## Write to Storage Table

```json

  {
      "type": "table",
      "name": "outputQueueItem",
      "tableName": "orders",
      "connection": "witstorage",
      "direction": "out"
    }

```

```csharp

#r "Microsoft.WindowsAzure.Storage"
using System.Net; 
using System.Text;
using Microsoft.WindowsAzure.Storage.Table;

public class Order : TableEntity {
    public string CustomerId {get; set;}
    public string Item {get; set;}
    public int Qty {get; set;}
    
}

public static async Task<HttpResponseMessage> Run(HttpRequestMessage req, 
TraceWriter log,
IAsyncCollector<Order> outputQueueItem)
{
    string body = await req.Content.ReadAsStringAsync();
    string result = $"Hello there, you typed {body}";
    
    //Write to queue Storage
    Order o = new Order() {CustomerId = "IBM", Item = "Matt", Qty = 17, PartitionKey = "Functions",
        RowKey = Guid.NewGuid().ToString()};

    await outputQueueItem.AddAsync(o);
    var response = new HttpResponseMessage(HttpStatusCode.OK) {
        Content = new StringContent(result, Encoding.UTF8, "text/plain")
    };
  
    return response;
 
}




```

#### Submit Object as JSON

```csharp 

#r "Microsoft.WindowsAzure.Storage"
using System.Net; 
using System.Text;
using Microsoft.WindowsAzure.Storage.Table;

public class Order : TableEntity {
    public string CustomerId {get; set;}
    public string Item {get; set;}
    public int Qty {get; set;}
    public string Desc {get; set;}
    
}

public static async Task<HttpResponseMessage> Run(HttpRequestMessage req, 
TraceWriter log,
IAsyncCollector<Order> outputQueueItem)
{
    Order order = await req.Content.ReadAsAsync<Order>();
    
    //Write to queue Storage
    order.PartitionKey = "Functions";
    order.RowKey = Guid.NewGuid().ToString();
  

    await outputQueueItem.AddAsync(order);
    var response = new HttpResponseMessage(HttpStatusCode.OK) {
        Content = new StringContent($"Order submitted to table..:", Encoding.UTF8, "text/plain")
    };
  
    return response;
 
}


```

[Back to top](#table-of-content)



```json

{
      "type": "blob",
      "name": "outputQueueItem",
      "connection": "witstorage",
      "direction": "out",
      "path": "orders/{rand-guid}"
    }
    

```



```csharp

using System.Net; 
using System.Text;



public static HttpResponseMessage Run(HttpRequestMessage req, 
TraceWriter log,
out string outputQueueItem)
{
    
    //Write to queue Storage
    
  

    outputQueueItem = "Hello Blob";
    var response = new HttpResponseMessage(HttpStatusCode.OK) {
        Content = new StringContent($"Order submitted to table..:", Encoding.UTF8, "text/plain")
    };
  
    return response;
 
}

```