## Lagon

To add another example, the syntax for [Lagon](https://lagon.app/) is similar to [Vercel](./vercel.md), except that the function is a named export:

```js
export function handler(request) {
  return new Response("Hello World");
}
```

Logging is the same (using `console.log` / `console.debug` / `console.warn` / `console.error`). I believe this syntax is already used by most (if not all) of the runtimes out there.

### Related Links

https://github.com/wintercg/admin/issues/44#issuecomment-1386620726

https://lagon.app/

