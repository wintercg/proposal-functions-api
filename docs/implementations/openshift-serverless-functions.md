## Openshift Serverless Functions

Here is the current syntax for an [Openshift Serverless Function](https://docs.openshift.com/container-platform/4.12/serverless/functions/serverless-functions-getting-started.html) for a "normal" function

```js
/**
 * Your HTTP handling function, invoked with each request. This is an example
 * function that echoes its input to the caller, and returns an error if
 * the incoming request is something other than an HTTP POST or GET.
 *
 * In can be invoked with 'func invoke'
 * It can be tested with 'npm test'
 *
 * @param {Context} context a context object.
 * @param {object} context.body the request body if any
 * @param {object} context.query the query string deserialized as an object, if any
 * @param {object} context.log logging object with methods for 'info', 'warn', 'error', etc.
 * @param {object} context.headers the HTTP request headers
 * @param {string} context.method the HTTP request method
 * @param {string} context.httpVersion the HTTP protocol version
 * See: https://github.com/knative-sandbox/kn-plugin-func/blob/main/docs/guides/nodejs.md#the-context-object
 */
const handle = async (context) => {
  // YOUR CODE HERE
  context.log.info(JSON.stringify(context, null, 2));

  // If the request is an HTTP POST, the context will contain the request body
  if (context.method === 'POST') {
    return {
      body: context.body,
    }
  // If the request is an HTTP GET, the context will include a query string, if it exists
  } else if (context.method === 'GET') {
    return {
      query: context.query,
    }
  } else {
    return { statusCode: 405, statusMessage: 'Method not allowed' };
  }
}

// Export the function
module.exports = { handle };
```
The only parameter here is the context object, which provides a few pieces of information


If you need a function that can also handle [CloudEvents](https://cloudevents.io/),  then an extra `event` param is used:

```js
const { CloudEvent, HTTP } = require('cloudevents');

/**
 * Your CloudEvent handling function, invoked with each request.
 * This example function logs its input, and responds with a CloudEvent
 * which echoes the incoming event data
 *
 * It can be invoked with 'func invoke'
 * It can be tested with 'npm test'
 *
 * @param {Context} context a context object.
 * @param {object} context.body the request body if any
 * @param {object} context.query the query string deserialzed as an object, if any
 * @param {object} context.log logging object with methods for 'info', 'warn', 'error', etc.
 * @param {object} context.headers the HTTP request headers
 * @param {string} context.method the HTTP request method
 * @param {string} context.httpVersion the HTTP protocol version
 * See: https://github.com/knative-sandbox/kn-plugin-func/blob/main/docs/guides/nodejs.md#the-context-object
 * @param {CloudEvent} event the CloudEvent
 */
const handle = async (context, event) => {
  // YOUR CODE HERE
  context.log.info("context");
  context.log.info(JSON.stringify(context, null, 2));

  context.log.info("event");
  context.log.info(JSON.stringify(event, null, 2));

  return HTTP.binary(new CloudEvent({
    source: 'event.handler',
    type: 'echo',
    data: event
  }));
};

module.exports = { handle };
```

### Related Links

* https://github.com/wintercg/admin/issues/44#issuecomment-1387160922

* https://docs.openshift.com/container-platform/4.12/serverless/functions/serverless-functions-getting-started.html

