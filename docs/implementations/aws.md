## AWS Lamda Functions input/output model:

From: https://docs.aws.amazon.com/lambda/latest/dg/nodejs-handler.html function signatures are:

* async function(event, context) or
* function (event, context, callback)

Parameters are:

* event - The invoker passes this information as a JSON-formatted string when it calls [Invoke](https://docs.aws.amazon.com/lambda/latest/dg/API_Invoke.html), and the runtime converts it to an object.
* The second argument is the [context object](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-context.html), which contains information about the invocation, function, and execution environment. In the preceding example, the function gets the name of the [log stream](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-logging.html) from the context object and returns it to the invoker.
* The third argument, callback, is a function that you can call in [non-async handlers](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-handler.html#nodejs-handler-sync) to send a response. The callback function takes two arguments: an Error and a response. When you call it, Lambda waits for the event loop to be empty and then returns the response or error to the invoker. The response object must be compatible with JSON.stringify.

For asynchronous function handlers, you return a response, error, or promise to the runtime instead of using callback.

Response is object must be compatible with JSON.stringify as JSON is returned

## Logging
Logging uses console.log, err

## Error handling
https://docs.aws.amazon.com/lambda/latest/dg/nodejs-exceptions.html still returns JSON


### Related Links

https://github.com/wintercg/admin/issues/44#issuecomment-1396204170

https://aws.amazon.com/lambda/#:~:text=AWS%20Lambda%20is%20a%20serverless,pay%20for%20what%20you%20use.
