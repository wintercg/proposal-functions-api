## Cloudflare Workers/workerd

For [workerd/cloudflare workers](https://developers.cloudflare.com/workers/), we have two API models (one legacy and one preferred that we're actively moving to):

Legacy (service worker style):

```js
// global addEventListener
addEventListener('fetch', (event) ={
  // request is a property on event (e.g. request.event), as is `waitUntil`
  // bindings (additional resources/capabilities configured for the worker) are injected as globals
  event.respondWith(new Response("Hello World"));
});
```

New (ESM worker style):

```js
export default {
  async fetch(request, env, context) => {
    // bindings are available through `env`, `waitUntil` is available on `context`
    return new Response("Hello World");
  }
}
```

We use `console.log(...)` for all logging. We do have a more structured logging API configurable through bindings but it's more specific purpose for metrics.

Error reporting is pretty straightforward. There's really nothing fancy. We throw synchronous errors and report unhandled promise rejections to the `'unhandedrejection'` event handler using the global `addEventListener` with either worker model.

Overall, our strong preference is to stick with the ESM worker style as experience has shown us time and again that the service worker model is pretty limited.

The `Request` and `Response` objects here are the standard fetch APIs. In the service worker model, we use the standard `FetchEvent`. For the ESM worker model, `env` and `context` are platform-defined, non-standard APIs, although `context` mimics the `FetchEvent` API.

I guess there are a few main ways Cloudflare's interface is unusual here. Let me try to explain our reasoning.

# `env`
Cloudflare's `env` contains "environment variables", which we also often call "bindings". But our design here is quite different from "environment variables" in most systems. Cloudflare's bindings implement a capability-based security model for configuring Workers. This is a central design feature of the whole Workers platform.

Importantly, unlike most systems, environment variables are not just strings; they may be arbitrary objects representing external resources. For example, if the worker is configured to use a Workers KV namespace for storage, then the binding's type will be `KvNamespace`, which has methods `get(key)`, `put(key, value)`, and `delete(key)`. Another example of a complex binding is a "service binding", which points to another Worker. A service binding has a method `fetch()`, which behaves like the global fetch, except all requests passed to it are delivered to the target worker. (In the future, service bindings could have other methods representing other event types that Workers can listen for.)

So for example, if I want to load the key "foo" from my KV namespace, I might write:

```js
let value = await env.MY_KV.get("foo");
```

We expect to add more kinds of bindings over time, which could have arbitrary APIs. Obviously those APIs aren't going to be standardized here. But, I think the concept of `env` could be.

### Why not have `env` be just strings?
Most systems that have environment variables only support strings, and if the environment variable is meant to refer to an external resource, it must contain some sort of URL or other identifier for that resource. For example, you could imagine KV namespaces being accessed like:

```js
let ns = KvNamespace.get(env.KV_NAME);
let value = await ns.get("foo");
```

However, this opens a can of worms.

First, there is security: What limitations exist on `KvNamespace.get()`? Can the Worker pass in any old namespace name it wants? Does every Worker then have access to every KV namespace on the account? Do we need to implement a permissions model, whereby people can restrict which namespaces each Worker can access? Will any users actually configure these permissions or will they mostly just leave it unrestricted?

Second, there is the problem that this model seems to allow people to hard-code namespace names, without an environment variable at all. But when people do that, it creates a lot of problems. What if you want to have staging vs. production version of your Worker which use different namespaces? How do developers test the Worker against a test namespace? You really want to _force_ people to use environment variables for this because it sets them up for success.

Third, this model allows the system to _know_ which Workers are attached to which resources. You can answer the question, "What Workers are using this KV namespace?" If the user tries to delete a namespace that is in use, we can stop them. Relatedly, we can make sure that the user cannot typo a namespace name – when they configure the binding, we only let them select from valid namespaces.

By having the environment variable actually be _the object_ and not just an identifier for it, we nicely solve all these problems. Plus, the application code ends up shorter.

### Why not make `env` globally available?
A more common way to expose environment variables is via a global API, e.g. `process.env` in Node.

The problem with this approach is that is is not _composable_. Composability means: I should be able to take two Workers and combine them into a single Worker, without changing either workers' code, just placing a new wrapper around them. For example, say I have one worker that serves static assets and another that serves my API, and for whatever reason I decide I'd rather combine them into a single worker that covers both. I should be able to write something like:

```js
import assets from "static-assets.js";
import api from "api.js";

export default {
  async fetch(req, env, ctx) {
    let url = new URL(req.url);
    if (url.pathname.startsWith("/api/")) {
      return api.fetch(req, env, ctx);
    } else {
      return assets.fetch(req, env, ctx);
    }
  }
}
```

Simple enough! But what if the two workers were designed to expect different bindings, and the names conflict between them. For example, say that each sub-worker requires a KV namespace with the binding name `KV`, but these refer to different KV namespaces. No problem! I can just remap them when calling the sub-workers:

```js
    if (url.pathname.startsWith("/api/")) {
      return api.fetch(req, {KV: env.ASSETS_KV}, ctx);
    } else {
      return assets.fetch(req, {KV: env.API_KV}, ctx);
    }
```

But if `env` were some sort of global, this would be impossible!

Arguably, an alternative way to enable composability would be to wrap the entire module in a function that takes the environment as a parameter. But, I felt that passing it as a parameter to the event handler was "less weird".

# Exporting functions vs. objects
Many other designs work by exporting a top-level function, like Vercel's:

```
export default (request) ={ ... }
```

But Workers prefers to export an object with a `fetch` method:

```js
export default {
  async fetch(req, env, ctx) { ... }
}
```

Why?

In Workers, we support event types other than HTTP. For example, Cron Triggers deliver events on a schedule. Scheduled events use a different function name:

```js
export default {
  async scheduled(controller, env) { ... }
}
```

We also support queue and pubsub events, and imagine adding support in the future for things like raw TCP sockets, email, and so on. A single worker can potentially support multiple event types.

By wrapping the exports in an object, it becomes much easier to programmatically discover what event types a worker supports. A function is just a function, there's not much you can say about it. But an object has named properties which can be enumerated, telling us exactly what the worker supports. When you upload a Worker to Cloudflare, we actually execute the Worker's global scope once in order to discover what handler it exports, so that our UI can guide the user in configuring it correctly for those events.

### Why not use the export name for that? Why wrap in an object?
You could argue we should just use export names instead:

```
export async function fetch(req, env, ctx) {...};
```

The function is exported with the name `fetch`, therefore it is an HTTP handler.

The problem with this is that it necessarily means a Worker can only have _one_ HTTP handler. We actually foresee the need for multiple "named entrypoints". That is, in the future, we plan to support this:

```
export default {
  async fetch (req, env, ctx) { … }
}

export let adminInterface = {
  async fetch (req, env, ctx) { … }
}
```

Here, we have a worker that exports two different HTTP handlers. The default one probably serves an application's main web interface. The `adminInterface` export is an alternate entrypoint which serves the admin interface. This alternate entrypoint could be configured to sit behind an authorization layer like Cloudflare Access. This way, the application itself need not worry about authorizing requests and can just focus on its business logic.

# What is `context`?
It looks like several designs feature a "context" object, but the purpose of the object differs.

In Workers' case, the purpose of the context object is to provide control over the execution environment of the specific event. The most important method it provides is `waitUntil()`, which has similar meaning to the Service Workers standard `ExtendableEvent.waitUntil()`: it allows execution to continue for some time after the event is "done", in order to perform asynchronous tasks like submitting logs to a logging service.

All event types feature the same context type. This makes it not a great place to put metadata about the request, since metadata probably differs for different event types. For example, the client IP address (for HTTP requests) is not placed in `context`. Instead, we have a non-standard field `request.cf` which contains such metadata. (I am not extremely happy about `request.cf`, but repeated attempts to design something better have always led to dead ends so far.)

### Why not put `env` inside `context`?
We could, but this poses some challenges to composability. In order for an application to pass a rewritten environment to a sub-worker, it would need to build a new context object. Applications cannot construct the `ExecutionContext` type directly. They could create an alternate type that emulates its API and forwards calls to the original context, but that seems tedious. I suppose we could provide an API to construct `ExecutionContext` based on an original context with an alternate `env`. But it seemed cleaner to just keep these sepanate. `env` contains things defined by the application, `context` contains things that come from the platform.





### Related Links

* https://github.com/wintercg/admin/issues/44#issuecomment-1386249383
* https://github.com/wintercg/admin/issues/44#issuecomment-1387268875

* https://developers.cloudflare.com/workers/
