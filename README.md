# WORKING IT AIS TUTORIAL

## Table Of Content
- [A simple boilerplate Azure Function](#a-simple-boilerplate-azure-function)
- [function.json](#function.json)


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


### function.json

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
