
# **Azion - Edge Functions - Functions API Specification**

## **1. Introduction**

Azion’s Runtime enables developers to build serverless applications on the edge by leveraging JavaScript functions and interacting with Azion’s extensive edge computing platform. This document outlines the standard API for writing JavaScript serverless functions on Azion, emphasizing interoperability, flexibility, and scalability while avoiding vendor lock-in.

---

## **2. Function Signature**

Azion’s Edge Functions support two function signature styles—an event-driven model and an ECMAScript module (ESM) style**, similar to other serverless platforms. These signatures ensure compatibility with various use cases while giving developers flexibility in handling events.

### **2.1 Event-Driven Model**

In the event-driven model, edge functions respond to incoming events such as HTTP requests or firewall events, allowing them to execute logic based on specific triggers. In Azion, two key types of events are commonly used:

Fetch Event: This event is used to handle incoming HTTP requests and runs within the Edge Application product.
Firewall Event: This event is used for security checks and runs within the Edge Firewall product, allowing the function to intercept and handle requests based on security policies.

#### **2.1.1 Fetch Event**

The Fetch Event API allows edge functions to process incoming HTTP requests, access the request metadata, and generate responses. This model runs on the Edge Application product, and it's typically used for dynamic content generation, caching, or modifying responses based on user requests.

```javascript
addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
    return new Response('Hello from Azion!');
}
```


#### **2.1.2 Firewall Event**

The Firewall Event API allows edge functions to execute security logic when a request passes through the Edge Firewall. This event provides access to important metadata, such as the remote IP address, and integrates with security tools like Azion's network lists. The function can take security actions such as denying requests based on conditions like IP address matching.



```javascript
addEventListener("firewall", (event) => {

      let ip = event.request.metadata["remote_addr"] // Accessing the remote address

      try {
        let found = Azion.networkList.contains(String(networkListId), ip); // Checking if the ip is in the list
        if (found) {
          event.deny(); // If it's in the list, deny the request
        }
      } catch (err) {
        event.console.error(`Error: `, err.stack);
      }
    });
```

### **2.2 ECMAScript Module Style**

This model allows functions to be exported as modules, enabling flexibility for developers who prefer this structure. The request, environment, and context objects are passed in, allowing the use of environment variables, background tasks, and bindings.

```javascript
export default async function main(event) {
  return handleRequest(event?.request, event?.args);
}

async function handleRequest(request, args) {
  return new Response('Hello from Azion!');

  return handle(request, { args });
}
```
*Atention: The ECMAScript Module style is only supported by the Azion CLI during build time. This method allows you to export functions as modules, which will then be processed and deployed via the CLI. If you are not using the Azion CLI for build and deployment, this style may not be compatible with your runtime environment.*

---

## **3. Event Handling**

Azion edge functions are primarily designed to handle HTTP requests via the Fetch Event API. However, they can also interact with other types of events, such as WebSockets or custom event triggers.

### **3.1 HTTP Request Handling**
Azion functions use the Fetch API to handle incoming HTTP requests. Functions can parse headers, access the request body, and return structured responses.

```javascript
async function handleRequest(request) {
    const { headers } = request;
    const userAgent = headers.get('User-Agent') || 'Unknown';
    return new Response('User Agent: ' + userAgent);
}
```

---

## **4. Environment Variables**

Azion functions can interact with environment variables to manage configurations, credentials, and other operational data. The `env` object exposes these variables within functions, ensuring secure access to sensitive information without hardcoding.

### **4.1 Accessing Environment Variables**
```javascript
const apiToken = Azion.env.get('API_SERVICE_TOKEN');
```


### **4.2 Support Process.env API**
```javascript
process.env.VAR_NAME	
```


Environment variables can store data such as:
- API keys
- Secrets
- Configuration options

---

## **5. Logging and Debugging**

Azion provides built-in logging through `console.log`, enabling developers to track execution flows, errors, and other events. Structured logs can also be set up using bindings, which provide additional control over log data.

### **5.1 Basic Logging**
```javascript
console.log('Handling request at the edge...');
console.error('Error processing request:', error);
```

---

## **6. Error Handling and Reporting**

Error handling in Azion functions follows JavaScript’s standard `try...catch` mechanism. All unhandled exceptions are automatically logged and reported to Azion's monitoring services.

### **6.1 Example of Error Handling**
```javascript
try {
    // Main logic
} catch (error) {
    console.error('An unexpected error occurred:', error);
}
```

Azion also supports structured error reporting through GraphQL. Access the Documentation [here](https://www.azion.com/en/documentation/devtools/graphql-api/queries/).

---

## **7. Edge Storage and SQL API**

Azion provides APIs for interacting with persistent storage and executing SQL queries directly from Edge Functions.

### **7.1 Edge Storage API**
The Azion Edge Storage API allows functions to interact with buckets, storing and retrieving data at the edge.

```javascript
async Storage.put(key, value, options)
```



```js
import Storage from "azion:storage";

async function handleRequest(event) {
    try{
        const bucket = "mybucket";
        const storage = new Storage(bucket);
        const key = "test";
        const data = JSON.stringify({
            name:"John",
            address:"Abbey Road"
        });
        const buffer = new TextEncoder().encode(data);
        await storage.put(key, buffer);
        return new Response("OK");
    }catch(error){
        return new Response(error, {status:500});
    }
}

addEventListener("fetch", (event) => {
    event.respondWith(handleRequest(event));
});

```

#### **7.1.1 Parameters**


| Parameter | Type | Description |
| - | - | - |
| `key` | string | Identifier that allows the search for an object in the storage |
| `value` | ArrayBuffer ou ReadableStream | Content of the object being stored. In case `value` implements a [ReadableStream](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream), the Stream APIs can be used, and the option `content-lenght` is required |
| `options` | object | The `options` attributes are described in the table below |


### **7.2 Edge SQL API**
Azion Edge Functions can also execute SQL queries on the Azion Edge SQL API, which provides an interface for interacting with a database.

```javascript
import { Database } from "azion:sql";

async function db_query() {
  let connection = await Database.open("mydatabase");
  let rows = await connection.query("select * from users");
  let column_count = rows.columnCount();
  let column_names = [];
  for (let i = 0; i < column_count; i++) {
    column_names.push(rows.columnName(i));
  }
  let response_lines = [];
  response_lines.push(column_names.join("|"));
  let row = await rows.next();
  while (row) {
    let row_items = [];
    for (let i = 0; i < column_count; i++) {
      row_items.push(row.getString(i));
    }
    response_lines.push(row_items.join("|"));
    row = await rows.next();
  }
  const response_text = response_lines.join("\n");
  return response_text;
}

async function handle_request(request) {
  if (request.method != "GET") {
    return new Response("Method not allowed", { status: 405 });
  }
  try {
    return new Response(await db_query());
  } catch (e) {
    console.log(e.message, e.stack);
    return new Response(e.message, { status: 500 });
  }
}

addEventListener("fetch", (event) =>
  event.respondWith(handle_request(event.request))
);
```

---

## **8. Network and Metadata API**

Azion functions provide access to network-specific and metadata information, such as GeoIP data, client IP addresses, and TLS details, enhancing security and performance.

### **8.1 Network List API**
The `Azion.networkList.contains` interface allows functions to check if an IP address belongs to a specified network list.

```javascript
if (Azion.networkList.contains('192.168.1.1')) {
    return new Response('IP is within the network');
}
```

### **8.2 Metadata API**
Edge Functions can access request metadata, such as GeoIP, TLS certificates, and client information.

```javascript
    let ip = event.request.metadata["remote_addr"] // Accessing the remote address
```

|  Name                            | Description                                                    |
|----------------------------------|----------------------------------------------------------------|
|  geoip_asn                       | Autonomous system number                                       |
|  geoip_city                      | City code                                                      |
|  geoip_city_continent_code       | City continent code information                                |
|  geoip_city_country_code         | City country code                                              |
|  geoip_city_country_name         | City country name                                              |
|  geoip_continent_code            | Continent code                                                 |
|  geoip_country_code              | Country code                                                   |
|  geoip_country_name              | Country name                                                   |
|  geoip_region                    | Region code                                                    |
|  geoip_region_name               | Region name                                                    |
|  remote_addr                     | Remote (client) IP address                                     |
|  remote_port                     | Remote (client) TCP port                                       |
|  remote_user                     | User informed in the URL. Example: user in http://user@site.com/|
|  server_protocol                 | Protocol being used in the request. Example: HTTP/1.1          |
|  ssl_cipher                      | TLS cipher used                                                |
|  ssl_protocol                    | TLS protocol used                                              |
---

Access the full list of metadata APIs in the [official documentation](https://www.azion.com/en/documentation/products/edge-application/edge-functions/runtime/api-reference/metadata/).

## **9. Conclusion**

The Functions API is extensible, allowing for integration with various JavaScript frameworks and custom workflows while providing powerful tools like Edge Storage, Edge SQL, and Metadata APIs and others DevTools.

## **10. References**

[API Reference](https://www.azion.com/en/documentation/products/edge-application/edge-functions/runtime/api-reference/).

[Environment Variables](https://www.azion.com/en/documentation/products/edge-application/edge-functions/runtime/api-reference/environment-variables/)

[Network List](https://www.azion.com/en/documentation/products/edge-application/edge-functions/runtime/api-reference/network-list/)

[Edge Storage](https://www.azion.com/en/documentation/runtime/api-reference/storage/)

[Edge SQL](https://www.azion.com/en/documentation/runtime/api-reference/edge-sql/)

[Metadata](https://www.azion.com/en/documentation/products/edge-application/edge-functions/runtime/api-reference/metadata/)

[Edge Functions For Edge Application](https://www.azion.com/en/documentation/products/edge-application/edge-functions/)

[Edge Functions For Edge Firewall](https://www.azion.com/en/documentation/products/secure/edge-firewall/edge-functions/)

