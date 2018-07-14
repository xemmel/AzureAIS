# WORKING IT AIS TUTORIAL

## Table Of Content
- [A simple boilerplate Azure Function](#a-simple-boilerplate-azure-function)
- [function.json](#function-json)
- [Send to Queue](#send-to-queue)
- [Timer function](#timer-function)
- [Receiving from Queue](#receiving-from-queue)


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

[Back to top](#table-of-content)
