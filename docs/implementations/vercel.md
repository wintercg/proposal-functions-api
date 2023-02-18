## Vercel Edge Functions

Using an [example from Vercel](https://vercel.com/docs/concepts/functions/edge-functions):

```js

export const config = { runtime: 'edge' };

export default (request) => {
  return new Response(`Hello, from ${request.url}`);
};
```

### Related Links

* https://github.com/wintercg/admin/issues/44#issuecomment-1386274222

* https://vercel.com/docs/concepts/functions/edge-functions
