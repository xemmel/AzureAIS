# WORKING IT AIS TUTORIAL

## Table Of Content
- [A simple boilerplate Azure Function](#a-simple-boilerplate-azure-function)
- [function.json](#function-json)
- [Timer function](#timer-function)


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



[Back to top](#table-of-content)
