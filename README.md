# A Syntax for Parallelism in ECMAScript: `await.all {...}`

This document proposes a new syntax for parallel promises in ECMAScript. Wherever `await` is available, the following syntax would also become available:

```javascript
await.all {
  statement;
  statement;
}
```

Inside the curly braces, all child statements (which may contain `await`) run in parallel. Execution continues only after all of those statements have completed.

## Motivations

Currently the only way to achieve parallelism with async/await is to use `Promise.all`. `Promise.all` works well for a set of homogeneous promises that will be handled the same way, like requests to the same API with different IDs; however, it is awkward for a set of promises where each one should be handled differently.

Let's say your website has an `initialize()` function that has to make a number of independent API requests and then do something with them. For argument's sake, let's say you use a `request` function that returns a Promise that resolves to something like `{ data: <json data> }`:

```javascript
async function initialize() {
  const foo = (await request('foo.json')).data;
  const bar = (await request('bar.json')).data;
  const baz = (await request('baz.json')).data;
  render(foo, bar, baz);
}
```

The above code waits until `request('foo.json')` finishes before it moves on to `request('bar.json')`. You want to make these requests happen in parallel so your website can load faster. You'd probably write something like this:

```javascript
async function initialize() {
  const [
    { data: foo }, // renaming response.data => foo using destructuring
    { data: bar },
    { data: baz },
  ] = await Promise.all([
    request('foo.json'),
    request('bar.json'),
    request('baz.json'),
  ]);
  render(foo, bar, baz);
}
```

The problem is that you end up maintaining "parallel lists" of requests and their corresponding response objects, which must stay in sync or else your app breaks.

Depending on how you structure your code, this method can also involve a lot of temporary variables: one for each promise, one for each result, and without destructuring, an additional array for the output of `Promise.all`.

More fundamentally, it forces you to reason about the promises in a very different way than when you were handling the requests in series. For serial requests, all you needed was the one keyword `await` to unbox them. To run them in parallel, you have to collect them all in an array, and then get their values out of another array.

This doesn't appear to be a conscious decision to force you to follow best code practices, it's just a limitation that falls naturally out of the existing syntax and which has not yet been addressed. Thus, we have an opportunity to shift some of the burden back to the language with new syntax.

<details>
<summary>Alternative Ways to Parallelize <code>initialize()</code> with Existing Syntax</summary>

### Alternative Ways to Parallelize `initialize()` with Existing Syntax

#### `Promise.all` with IIAFEs

Another way to restructure `initialize()` is to pass `Promise.all` a bunch of immediately-invoked async function expressions (IIAFEs), which set variables in the surrounding scope:

```javascript
async function initialize() {
  let foo, bar, baz;
  await Promise.all([
    (async () => { foo = (await request('foo.json')).data })(),
    (async () => { bar = (await request('bar.json')).data })(),
    (async () => { baz = (await request('baz.json')).data })(),
  ]);
  render(foo, bar, baz);
}
```

...this looks and feels a lot like the original `initialize()` function, but it requires a lot of brackets. It may also not the most obvious way to structure the program.

#### `await` twice

After a Promise has resolved, you can keep using `await` to get its value immediately. We can take advantage of this to get the values out of our Promise objects after using `Promise.all`:

```javascript
async function initialize() {
  const foo = request('foo.json');
  const bar = request('bar.json');
  const baz = request('baz.json');
  await Promise.all([foo, bar, baz]);
  render((await foo).data, (await bar).data, (await baz).data);
})
```

There are two downsides here: `await` always creates a microtask, even for resolved promises, so we're making the computer do _ever so slightly_ more work. Also, we can't get the value out of the `data` property as early as we did for the first couple solutions.
  
#### `Promise.allObject`

I experimented with a function similar to `Promise.all`, but which accepts a JS object whose values are promises, as discussed on [this email thread](https://esdiscuss.org/topic/modify-promise-all-to-accept-an-object-as-a-parameter):

```javascript
// note: this function is not robust, but is good enough for this experiment
Promise.allObject = async function(inObj) {
  const keys = Object.keys(inObj);
  const valuePromises = keys.map(key => inObj[key]);
  const values = await Promise.all(valuePromises);
  const outObj = {};
  for(let i = 0; i < keys.length; i++) {
    outObj[keys[i]] = values[i];
  }
  return outObj;
}

async function initialize() {
  const {
    foo: { data: foo },
    bar: { data: bar },
    baz: { data: baz },
  } = await Promise.allObject({
    foo: request('foo.json'),
    bar: request('bar.json'),
    baz: request('baz.json'),
  });
  render(foo, bar, baz);
}
```

It didn't seem to reduce the syntactic or mental overhead from using `Promise.all` (at least in this scenario). Parallel objects are nearly as difficult to maintain as parallel arrays. While you no longer need to worry about property order, you do need to write each property name at least twice (in the above case, 3 times because of destructuring).

</details>

## Proposed New Syntax

Under the proposed syntax, the above `initialize()` function could be rewritten like this:

```javascript
async function initialize() {
  let foo, bar, baz;
  await.all {
    foo = (await request('foo.json')).data;
    bar = (await request('bar.json')).data;
    baz = (await request('baz.json')).data;
  }
  render(foo, bar, baz);
}
```

`await.all` executes each statement (direct child of the curly braces) in parallel, respecting `await`s within them, and waits until they've all resolved. In more concrete terms, each statement behaves as if it were wrapped in an immediately-invoked async function expression (IIAFE), and `await.all` behaves like `await Promise.all` on those IIAFEs.

It could be extended to include `await.race`, however the onus would be on the develper to figure out which statement had resolved (probably by seeing which variable had been set or `||`-ing them together). Since async/await is compatible with `try {} catch {}`, I don't feel strongly about whether it makes sense to include `await.allSettled`.

One could use block statements within `await.all` for sequential code:

```javascript
async function() {
  await.all {
    {
        const value1 = await promise1;
        console.log(value1);
    }
    await promise2;
  }
}
```

`return` statements should probably be illegal inside `await.all` and its substatements.

This syntax _would_ make it easier to write code that has invalid dependencies on race conditions. This is a reality of working with parallelism in most capacities, and is already possible in JS. This is probably something that would have to be left to a linter to catch.

### Reasoning for This Syntax

`await.all` is a non-breaking syntax because `await` becomes a reserved word inside async functions. The impending [Top-Level Await](https://github.com/tc39/proposal-top-level-await) proposal [appears](https://tc39.es/proposal-top-level-await/#sec-async-function-definitions) to maintain the invariant that `await` is never a valid identifier and operator in the same place, so it should not break `await.all`. (note: Chrome's console currently allows both TLA and `await` as an identifier but I believe that's a deviation from the spec that may be reversed once TLA is adopted.)

`async.all` or `await async.all` are breaking syntax since [`async` is not a reserved word](https://www.ecma-international.org/ecma-262/10.0/index.html#prod-Keyword) and may be used as an identifier, even inside an async function. Thus, `async.all \n { ... }` is currently valid syntax, and is parsed as a property access followed by an unlabeled block statement.

The reason for using curly braces over parentheses is to make it more obvious that `await.all` is syntax, as opposed to a function. If `await.all` is indistinguishable from a function, it encourages users to only evaluate expressions (as opposed to non-expression statements) inside it.

Alternatively, it could use square brackets. This would have the added benefit of not creating a block scope (so variables could be defined inside the brackets and used outside of them, saving a line of declarations). In my discussions with developers, square brackets made them interpret the construct as an array, leading them to only include expressions in the brackets.

An alternative syntax might be `await* { stmt, stmt, stmt }`, although this doesn't naturally extend to other methods like `Promise.race` the way `await.all` does. This could also cause confusion since it's unrelated to the similar-looking `yield*`.

## Examples

An initialize function that requests the first page of comments on an article:

```javascript
async function initialzeArticle(artId) {
  let article, newestComments;
  await.all {
    article = await (await fetch(`/api/article/${artId}`)).json();
    newestComments = await (await fetch(`/api/article/${artId}/comments?page=1`)).json();
  }
  render(article, newestComments);
}
```

Fetching from multiple third-party APIs whose data can't be combined on the backend:

```javascript
async function getTimeAndWeatherAt(lat, lon) {
  let time, weatherData;
  await.all {
    time = await (await fetch(`time.example.com?lat=${lat}&lon=${lon}`)).text();
    weatherData = await (await fetch(`weather.example.com?lat=${lat}&lon=${lon}`)).json();
  }
  return `It is ${time} and the weather is ${weatherData.temperature} degrees.`;
}
```

Parallel filesystem access:

```javascript
import { promises as fs } from 'fs';
import NodeRSA from 'node-rsa';

async function decryptFile(keypath, filepath) {
  let keyData, encrypted;
  await.all {
    keyData = await fs.readFile(keypath); // secure!
    encrypted = await fs.readFile(filepath);
  }
  const key = new NodeRSA(keyData);
  return key.decrypt(encrypted);
}
```

A slightly sillier musical example. This one isn't any simpler than its `Promise.all` equivalent, but it's nice to do it all with syntax:

```javascript
async function playChords() {
  await.all {
    await playNote('C4');
    await playNote('E4');
    await playNote('G4');
  }
  await rest();
  await.all {
    await playNote('C4');
    await playNote('F4');
    await playNote('A4');
  }
}
```

## Alternative Solution: `async do {}`

This is based on [a discussion](tc39/proposal-do-expressions#4) in the do-expressions repo. The above `initialize()` function might look something like this:

```javascript
async function initialize() {
  let foo, bar, baz;
  await Promise.all([
    async do { foo = (await request('foo.json')).data },
    async do { bar = (await request('bar.json')).data },
    async do { baz = (await request('baz.json')).data },
  ]);
  render(foo, bar, baz);
}
```

This is nearly identical to the "`Promise.all` with IIAFEs" solution (collapsed under the section "Alternative Ways to Parallelize `initialize()` with Existing Syntax"), but it requires 6 fewer brackets for each request. It is compatible with other functions beyond `Promise.all`, including user-defined ones.

This would depend on the [do-expressions proposal](https://github.com/tc39/proposal-do-expressions), which is still at Stage 1. This also makes some assumptions into how async do-expresisons would work, which I haven't verified.

My biggest concern with this approach is that, even with this syntax available, the structure in the above example wouldn't be the most obvious solution to a developer.

## Prior Art

None of the languages listed on the [Wikipedia article on async/await](https://en.wikipedia.org/wiki/Async/await#Benefits_and_criticisms) appear to have a comparable syntax solution for parallelism. Most (if not all) of them have something similar to `Promise.all`.

As brought up in [this thread](https://github.com/tc39/ecmascript-asyncawait/issues/25) in the async/await proposal repo, [IcedCoffeeScript](https://maxtaco.github.io/coffee-script/#iced_control) does have a similar construct:

```coffee
await
  $.get myurl1, defer result1
  douserinput defer result2
  for j in [0...10]
    $.get myurlarray[j], defer resultarray[j]
answer = result1+ result2 + sum(resultarray)
```

Here, `A defer B` spawns a promise that, when `A` resolves, declares `B` and sets it to the result of `A`. Since assigning the value to a variable is baked into the construct, the `defer` function is locally non-blocking and can be used inside a for-loop. Once all of the `defer`s in the `await` block resolve, the function resumes execution.

`await.all` is, in some sense, more flexible: because it's not limited to setting the value of a promise to a variable, it is easier to use with APIs like `fetch` that require chained promises, or any other post-processing (like the `response.data` example). However, since each statement in `await.all` has normal execution rules internally, a for-loop would not run in parallel, so it is not a good solution for iterating over an array of homogeneous promises (there's [another proposal](https://esdiscuss.org/topic/stage-0-proposal-specifying-concurrency-for-for-of-loops-potentially-containing-await-statements) for concurrent for-loops in JS).

## Relevant Discussions

- GitHub: async/await proposal repo
  - [Section about `await*` in an early version of the proposal](https://github.com/tc39/ecmascript-asyncawait/blob/73295fb2a1968d402f02e09e8c7a688ad8adee92/README.md#await-and-parallelism)
  - [Discussion about removing `await*`](https://github.com/tc39/ecmascript-asyncawait/issues/61)
  - [Discussion about an alternative syntax for parallelism](https://github.com/tc39/ecmascript-asyncawait/issues/25)
- [GitHub: async do-expressions](https://github.com/tc39/proposal-do-expressions/issues/4)
  - [Related discussion in gist comments](https://gist.github.com/dherman/1c97dfb25179fa34a41b5fff040f9879#gistcomment-2013114)
- ESDiscuss threads
  - [Modify Promise.all() to accept an Object as a parameter](https://esdiscuss.org/topic/modify-promise-all-to-accept-an-object-as-a-parameter)
  - [Specifying concurrency for "for...of" loops potentially containing "await" statements](https://esdiscuss.org/topic/stage-0-proposal-specifying-concurrency-for-for-of-loops-potentially-containing-await-statements)
  - [Discussion of a block in which all promises are automatically awaited](https://esdiscuss.org/topic/awaiting-block-expression)
  - [Wrap Array return values with Promise.all](https://esdiscuss.org/topic/await-enhancement-proposal-wrap-array-return-values-with-promise-all)
