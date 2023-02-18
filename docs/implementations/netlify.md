## Netlify Edge Functions

The syntax for [Netlify Edge Functions](https://docs.netlify.com/edge-functions/overview/) is very similar to [Vercel](./vercel.md) and [Lagon](./lagon.md), and both of those examples would work unchanged on Netlify.

This is the standard signature for Netlify:

```ts
export default async function handler(request: Request, context: Context) {
    return new Response("Hello world")
}
```

The `Request` and `Response` are standard Deno objects. The `Context` object is optional, and provides things like `geo` and `ip`, as well as a `next()` function (which itself returns a `Response`). We've tried to keep everything as standard as possible, putting any non-standard fields on the context object instead of adding anything to the request or response. `console` works as expected.

Netlify also supports an optional `config` export similar to Vercel but in our case it's currently just used for mapping the function to paths.

We're open to adding more fields to `Request` if they are standardised but would suggest that we avoid using these objects for non-standard extensions if possible.


### Related Links

* https://github.com/wintercg/admin/issues/44#issuecomment-1386751428

* https://docs.netlify.com/edge-functions/overview/
