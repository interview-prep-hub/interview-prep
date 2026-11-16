# Node.js Fundamentals: Runtime, Event Loop and Modules

> **Tier:** T1 (know cold) · **Related:** [Async JavaScript and the event loop](../03-languages/javascript/04-async-event-loop-promises.md), [Core modules, streams and errors](02-core-modules-streams-errors.md), [Express](03-express.md), [Hono](04-hono.md), [Production APIs](06-production-apis.md), [TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md), [Operating systems](../01-cs-fundamentals/05-operating-systems.md)
> **On your resume:** Node.js (Hono, oRPC) in your skills list; the TypeScript backend of the multi-tenant platform at Devntech (oRPC procedures, Rails sync APIs); the Next.js (TypeScript) APIs you built at RemoteDev, which run on Node.

## Quick revision
- Node.js = **V8** (compiles and runs JavaScript, garbage-collects) + **libuv** (event loop, non-blocking I/O, thread pool) + core modules (`fs`, `http`, `crypto`, `stream`) with C++ bindings.
- Your JavaScript runs on **one main thread**. Network I/O is handed to the OS (epoll, kqueue, IOCP). Work with no async OS API runs on libuv's **thread pool** (4 threads by default, `UV_THREADPOOL_SIZE`): `fs`, `dns.lookup`, `crypto.pbkdf2`/`scrypt`/`randomBytes`, `zlib`.
- Great for I/O-bound work (APIs, BFFs, proxies, websockets, streaming). Poor for CPU-bound work on the main thread unless you offload it.
- Loop phases: **timers** (`setTimeout`/`setInterval`), **pending callbacks**, idle/prepare (internal), **poll** (I/O callbacks; waits for I/O), **check** (`setImmediate`), **close callbacks**, then around again.
- After every callback (and after the main script) Node drains the **`process.nextTick` queue**, then the **promise microtask queue**. Since Node 11 this happens per callback, like browsers.
- `setTimeout(fn, 0)` vs `setImmediate(fn)`: order is **not guaranteed** in the main module; inside an I/O callback `setImmediate` **always** wins.
- `process.nextTick` beats promise callbacks in CommonJS; in an ES module, promise callbacks queued by top-level code run first. Recursive `nextTick` starves I/O.
- Anything slow and synchronous blocks every request: `*Sync` APIs in handlers, huge `JSON.parse`, CPU loops, catastrophic regexes. Fix: async APIs, chunking with `setImmediate`, `worker_threads`, or a job queue.
- CommonJS: `require`/`module.exports`, synchronous, dynamic. ESM: `import`/`export`, static, async loading, top-level `await`, always strict.
- `"type": "module"` makes `.js` ESM; `.mjs` is always ESM and `.cjs` always CommonJS.
- ESM imports CommonJS (`module.exports` becomes the default export). CommonJS loads ESM with `import()`, or with `require()` for synchronous ES modules (unflagged in Node 20.19+ and 22.12+).
- ESM has no `__dirname`: use `import.meta.dirname` (Node 20.11+) or `path.dirname(fileURLToPath(import.meta.url))`.
- package.json: `main` (legacy entry), `exports` (public entry points and conditions), `type`, `scripts`, `engines`, and `dependencies` vs `devDependencies` vs `peerDependencies`.
- Semver: `^1.4.2` allows `<2.0.0`, `~1.4.2` allows `<1.5.0`; for `0.x` the caret stays inside the minor (`^0.4.2` allows `<0.5.0`).
- Commit the lockfile; install with `npm ci` (or `pnpm install --frozen-lockfile`) in CI and Docker.
- `process.env` values are strings or `undefined`. Load files with `node --env-file=.env` (Node 20.6+) or dotenv, then validate once at startup with Zod.
- Production runs LTS releases only. Node 20 reached end of life in 2026; from Node 27 the project moves to one major release per year, each becoming LTS.
- TypeScript: `tsc` or a bundler for builds, `tsx` in development; Node 22.18+ and 23.6+ strip erasable types natively without type checking.
- Deno: secure by default, TypeScript and web APIs built in, npm compatible since Deno 2. Bun: JavaScriptCore engine plus package manager, bundler and test runner in one binary.

## 1. What Node.js is

Node.js is a runtime that runs JavaScript outside the browser. It is not a language and not a framework.

| Piece | Job |
|---|---|
| V8 | The JavaScript engine from Chrome: parses and JIT-compiles your code, manages the heap and garbage collection |
| libuv | C library behind the event loop: non-blocking network I/O through the OS, timers, a thread pool for blocking work |
| Bindings and core modules | C++ glue plus JavaScript APIs: `fs`, `http`, `net`, `crypto`, `stream`, `child_process`, `worker_threads` |
| Package ecosystem | npm registry packages installed into `node_modules` by npm, pnpm or Yarn |

Compared with the browser there is no DOM, `window` or `document`; instead you get files, sockets, processes, `process.env` and `Buffer`. Both sides share the language, promises, `fetch` (global since Node 18), `URL`, `AbortController`, Web Streams and `structuredClone`.

"Node is single-threaded" is half true: **your JavaScript** runs on one thread, but the runtime uses more (libuv's pool, V8's garbage collector and compiler threads). Keeping those two ideas apart answers most scaling questions.

## 2. How one thread serves thousands of connections

A typical API request mostly waits: for Postgres, Redis, another HTTP API, the disk. A thread-per-request server (Puma threads in Rails, classic Java servlets) parks a thread on each waiting request. Node parks nothing. It starts the I/O, registers a callback and moves on to the next piece of JavaScript.

```ts
import { createServer } from 'node:http';
import { setTimeout as sleep } from 'node:timers/promises';

createServer(async (_req, res) => {
  await sleep(100); // stands in for a 100 ms database query: no thread is held while waiting
  res.end('done\n');
}).listen(3000);
// 1,000 concurrent requests finish in roughly 100 ms in total, not 100 seconds
```

How the waiting really happens:
- **Sockets** (HTTP, database drivers, Redis): libuv asks the OS to report readiness (epoll on Linux, kqueue on macOS, IOCP on Windows). No thread waits per connection.
- **Work the OS cannot do asynchronously**: file system calls, `dns.lookup`, CPU-heavy crypto (`pbkdf2`, `scrypt`, async `randomBytes`) and `zlib`. libuv runs these on its thread pool.
- When a result is ready, its callback is queued and the event loop runs it on the main thread once the call stack is empty.

The thread pool is a hidden bottleneck. Start eight async hashes and only four run at once; every `fs` call queues behind them too:

```ts
import { pbkdf2 } from 'node:crypto';

const start = Date.now();
for (let i = 1; i <= 8; i++) {
  pbkdf2('secret', 'salt', 200_000, 64, 'sha512', () => {
    console.log(`hash ${i} done after ${Date.now() - start} ms`); // two waves of four on a 4-thread pool
  });
}
// UV_THREADPOOL_SIZE=8 node hash.js: one wave (the variable must be set before the process starts)
```

Concurrency is not parallelism: Node interleaves many requests on one thread (concurrency) and runs JavaScript in parallel only when you add worker threads or more processes.

## 3. Where Node fits

| Good fit | Why | Poor fit | Why |
|---|---|---|---|
| JSON APIs, BFFs, API gateways | mostly waiting on I/O | video encoding, image processing at scale | CPU-bound: blocks the loop |
| Realtime: chat, notifications, websockets | idle connections are cheap | ML inference, heavy number crunching | better in a separate service (Python, Go, C++) |
| Streaming: uploads, CSV exports, proxies | streams with backpressure | big synchronous report generation inside a request | every other request waits |
| Full-stack TypeScript (Next.js, TanStack Start) | one language and shared types | CPU-heavy cron work inside the API process | move it to workers or a queue |

Node can do CPU work, just not on the event loop thread.

## 4. The event loop phases

The loop keeps running while something keeps it alive (a listening server, a pending timer, an open socket). Each iteration walks these phases, and each phase has a FIFO queue of callbacks:

| Phase | Runs | Examples |
|---|---|---|
| timers | callbacks whose delay has passed | `setTimeout`, `setInterval` |
| pending callbacks | some system I/O callbacks deferred from the previous iteration | certain TCP errors such as `ECONNREFUSED` |
| idle, prepare | internal use only | |
| poll | new I/O events and their callbacks; blocks here waiting for I/O when nothing else is scheduled | socket data, `fs.readFile` completion |
| check | `setImmediate` callbacks, right after poll | `setImmediate(fn)` |
| close callbacks | `'close'` handlers | `socket.on('close', ...)` |

Two queues sit outside the phases and are drained **after every single callback** and after the main script:
1. the **`process.nextTick` queue**, completely;
2. then the **promise microtask queue** (`then`/`catch`/`finally`, `await` continuations, `queueMicrotask`), completely. If microtasks queue new `nextTick` callbacks, Node goes back to step 1.

```text
main script -> nextTick queue -> microtasks
then repeat: timers -> pending -> poll -> check -> close
             (nextTick queue + microtasks drained after each callback)
```

Rules to memorize:
- Delays are minimums. `setTimeout(fn, 0)` is treated as 1 ms, and it fires only when the loop reaches the timers phase.
- `setImmediate` means "after this poll phase". `process.nextTick` is the one closer to "immediately": after the current operation, before promises.
- Each phase callback behaves like a browser task followed by a microtask checkpoint, so the [browser model](../03-languages/javascript/04-async-event-loop-promises.md) carries over.

### Going deeper: libuv 1.45 and Node 20
Since libuv 1.45 (Node 20), due timers are processed after the poll phase of each iteration, with one extra timers pass before the loop starts; older versions processed them at the start of each iteration. The cycle order (poll, check, close, timers) and the two `setImmediate` rules above did not change, but old blog posts may show slightly different fine-grained timing.

## 5. Ordering exercises

Outputs assume a CommonJS file (`.cjs`, or TypeScript compiled to CommonJS). Exercise 6 shows how ES modules differ.

### Exercise 1: the classic
```ts
console.log('A');
setTimeout(() => console.log('B'), 0);
setImmediate(() => console.log('C'));
Promise.resolve().then(() => console.log('D'));
process.nextTick(() => console.log('E'));
queueMicrotask(() => console.log('G'));
console.log('F');
```
**Output:** `A F E D G`, then `B C` or `C B`. Synchronous code first; then the nextTick queue (E); then microtasks in order (D, G). The timer-vs-immediate race in the main module depends on whether 1 ms has elapsed when the loop starts, so never rely on it.

### Exercise 2: inside an I/O callback
```ts
import { readFile } from 'node:fs';

readFile('package.json', () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
  process.nextTick(() => console.log('nextTick'));
  Promise.resolve().then(() => console.log('promise'));
});
```
**Output:** `nextTick promise immediate timeout`, every time. The callback runs in the poll phase; nextTicks and microtasks drain after it; check (immediate) comes before the loop wraps around to timers.

### Exercise 3: nextTick and promises scheduling each other
```ts
Promise.resolve().then(() => {
  console.log('p1');
  process.nextTick(() => console.log('tick from p1'));
});
Promise.resolve().then(() => console.log('p2'));
process.nextTick(() => {
  console.log('t1');
  Promise.resolve().then(() => console.log('promise from t1'));
});
process.nextTick(() => console.log('t2'));
```
**Output:** `t1 t2 p1 p2 promise from t1 tick from p1`. The whole nextTick queue drains first. The microtask queue then drains completely (V8 does not stop halfway), including the promise queued by t1. Only then does Node look at the nextTick queue again.

### Exercise 4: microtasks between timers (Node 11+)
```ts
setTimeout(() => {
  console.log('t1');
  Promise.resolve().then(() => console.log('p1'));
  process.nextTick(() => console.log('n1'));
}, 0);
setTimeout(() => console.log('t2'), 0);
```
**Output:** `t1 n1 p1 t2`. Queues drain after each callback, even within one phase. Node 10 and earlier printed `t1 t2 n1 p1`.

### Exercise 5: async functions and nextTick
```ts
async function main() {
  console.log('1');
  await null;
  console.log('2');
  process.nextTick(() => console.log('3'));
  await null;
  console.log('4');
}
main().then(() => console.log('5'));
process.nextTick(() => console.log('6'));
console.log('7');
```
**Output:** `1 7 6 2 4 5 3`. `main` runs synchronously until the first `await`. After the script, the nextTick queue runs (6). The microtask drain then runs the rest of `main` and its `.then` (2, 4, 5) before the tick queued inside it (3).

### Exercise 6: the same code as an ES module
Running Exercise 1 as `.mjs` prints `A F D G E` before the timer and immediate; Exercise 5 prints `1 7 2 4 5 6 3`. ESM top-level code is evaluated inside a promise job, so V8 drains the microtasks it queued before Node processes `nextTick` callbacks. If a puzzle mixes the two, ask which module system it assumes.

### Exercise 7: immediates scheduling immediates
```ts
setImmediate(() => {
  console.log('immediate 1');
  process.nextTick(() => console.log('tick'));
  Promise.resolve().then(() => console.log('promise'));
  setImmediate(() => console.log('immediate 3'));
});
setImmediate(() => console.log('immediate 2'));
```
**Output:** `immediate 1 tick promise immediate 2 immediate 3`. Queues drain between the two immediates; an immediate scheduled during the check phase waits for the next iteration.

## 6. Blocking the event loop

While your JavaScript runs, nothing else runs: no other request, no timer, no I/O callback.

```ts
const start = Date.now();
setTimeout(() => console.log('timer fired after', Date.now() - start, 'ms'), 10);
while (Date.now() - start < 200) {} // busy-wait: the 10 ms timer fires after about 200 ms
```

| Culprit | Example | Fix |
|---|---|---|
| Sync APIs in request paths | `readFileSync`, `pbkdf2Sync`, `gzipSync` inside a handler | async versions (`node:fs/promises`, callback or promisified crypto) |
| Huge JSON | parsing a 50 MB body, stringifying 100k rows | body size limits, pagination, streaming (NDJSON) |
| CPU work | image resizing, PDF generation, large CSV transforms | `worker_threads`, a job queue, a separate service |
| Catastrophic regex (ReDoS) | `/(a+)+$/` against untrusted input | safe patterns, input length limits, lint rules |
| In-memory crunching | sorting and aggregating a million rows in JavaScript | let the database aggregate, or chunk the work |
| Endless microtasks | recursive `process.nextTick` or promise loops | yield with `setImmediate` |

**Detect it.** Measure event-loop delay and alert when p99 grows, then profile with `node --cpu-prof` or `node --inspect` and Chrome DevTools:

```ts
import { monitorEventLoopDelay } from 'node:perf_hooks';

const delay = monitorEventLoopDelay({ resolution: 20 });
delay.enable();
setInterval(() => {
  console.log({ p99ms: delay.percentile(99) / 1e6 }); // the histogram records nanoseconds
  delay.reset();
}, 10_000).unref(); // unref: this timer alone does not keep the process alive
```

**Fix 1: chunk and yield.** Latency for other requests stays low, but the total CPU time is unchanged and still on the main thread.

```ts
import { setImmediate as yieldToLoop } from 'node:timers/promises';

export async function reindexAll(rows: { id: string; title: string }[]) {
  for (let i = 0; i < rows.length; i += 500) {
    for (const row of rows.slice(i, i + 500)) indexRow(row); // a chunk of CPU work
    await yieldToLoop(); // let pending I/O and other requests run first
  }
}
declare function indexRow(row: { id: string; title: string }): void;
```

**Fix 2: a worker thread.** Each worker has its own V8 isolate and event loop and talks to the main thread through messages.

```ts
// primes.worker.ts
import { parentPort, workerData } from 'node:worker_threads';

function countPrimes(limit: number): number {
  let count = 0;
  for (let n = 2; n <= limit; n++) {
    let isPrime = true;
    for (let d = 2; d * d <= n; d++) {
      if (n % d === 0) { isPrime = false; break; }
    }
    if (isPrime) count++;
  }
  return count;
}

parentPort?.postMessage(countPrimes(workerData as number));
```

```ts
// primes.ts (ESM): called from a route handler
import { Worker } from 'node:worker_threads';

export function countPrimesOffThread(limit: number): Promise<number> {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL('./primes.worker.js', import.meta.url), { workerData: limit });
    worker.once('message', resolve);
    worker.once('error', reject);
    worker.once('exit', (code) => {
      if (code !== 0) reject(new Error(`worker stopped with exit code ${code}`));
    });
  });
}
```

Starting a worker costs tens of milliseconds and some memory, so reuse a pool (the Piscina library is a common choice) for repeated jobs.

**Fix 3: a background job.** If the client does not need the result in the response (emails, exports, third-party syncs like the PriceCharting jobs you know from Rails), enqueue it (BullMQ, pg-boss), return `202 Accepted`, and process it in a separate worker process. See [production APIs](06-production-apis.md).

## 7. CommonJS vs ES modules

```ts
// CommonJS: what tsc emits for "module": "commonjs" and what many older packages ship
const path = require('node:path');               // synchronous; can run anywhere, even inside if blocks
function uploadPath(name: string) { return path.join(__dirname, 'uploads', name); }
module.exports = { uploadPath };
```

```ts
// ES module: "type": "module" in package.json, or a .mjs/.mts file
import path from 'node:path';

export function uploadPath(name: string) {
  return path.join(import.meta.dirname, 'uploads', name); // Node 20.11+
}
// Older Node: import { fileURLToPath } from 'node:url';
//             const __dirname = path.dirname(fileURLToPath(import.meta.url));
export const settings = await loadSettings(); // top-level await: ESM only
declare function loadSettings(): Promise<Record<string, string>>;
```

| | CommonJS | ES modules |
|---|---|---|
| Syntax | `require()`, `module.exports`, `exports.x` | `import`, `export` |
| Loading | synchronous, at call time | graph parsed first, loaded asynchronously |
| Dynamic loading | `require(anyExpression)` anywhere | static `import` at top level; `import()` returns a promise |
| Top-level `await` | no | yes |
| `__dirname`, `__filename` | available | `import.meta.dirname`, `import.meta.filename`, `import.meta.url` |
| Relative import extensions | optional (`./db`) | required (`./db.js`, even when the source file is `db.ts`) |
| Strict mode | opt-in | always |
| Tree-shaking | hard (dynamic) | easy (static) |

**Interop:**
- ESM importing CommonJS: `import pkg from 'cjs-lib'` gives `module.exports`; named imports work when Node can detect them statically.
- CommonJS loading ESM: `await import('esm-lib')` always works. `require()` of an ES module works without a flag in Node 20.19+, 22.12+ and 23+ (stable in 25.4), as long as its graph has no top-level `await` (otherwise `ERR_REQUIRE_ASYNC_MODULE`). Before that, ESM-only packages caused `ERR_REQUIRE_ESM` in CommonJS apps.
- Dual packages ship both formats through `exports` conditions. The "dual package hazard" is loading both copies at once, so module state (singletons, `instanceof` checks) exists twice.
- In TypeScript, use `"module": "nodenext"` for code that Node runs directly and `"moduleResolution": "bundler"` for code a bundler consumes. Details in [TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md).

## 8. package.json, package managers and versions

```json
{
  "name": "@acme/platform-api",
  "version": "1.4.0",
  "type": "module",
  "main": "./dist/index.js",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js", "require": "./dist/index.cjs" },
    "./package.json": "./package.json"
  },
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc -p tsconfig.build.json",
    "start": "node dist/server.js",
    "test": "vitest run"
  },
  "engines": { "node": ">=22" },
  "dependencies": { "hono": "^4.0.0", "zod": "^4.0.0" },
  "devDependencies": { "tsx": "^4.0.0", "typescript": "^5.0.0", "vitest": "^3.0.0" }
}
```
(Version numbers are illustrative.)

- **`main`**: the entry for `require('pkg')` in tools that ignore `exports`.
- **`exports`**: the public entry points. Unlisted subpaths become unimportable (encapsulation). Conditions are matched in order: put `types` first and `default` last.
- **`type`**: how `.js` files are treated (`module` or `commonjs`, the default).
- **`scripts`**: `npm run dev`; `pre`/`post` hooks; `npm test` and `npm start` need no `run`.
- **`engines`**: the supported Node range. npm only warns unless `engine-strict` is set.
- **`dependencies`** are needed at runtime; **`devDependencies`** only for building and testing (`npm ci --omit=dev` skips them); **`peerDependencies`** are provided by the host app (a React component library declares `react` as a peer so the app's single copy is used).

| | npm | pnpm | Yarn (Berry) |
|---|---|---|---|
| Ships with Node | yes | separate install | separate install |
| `node_modules` | flat, hoisted | content-addressed store plus symlinks; no phantom dependencies | `node_modules` or Plug'n'Play |
| Lockfile | `package-lock.json` | `pnpm-lock.yaml` | `yarn.lock` |
| Reproducible install | `npm ci` | `pnpm install --frozen-lockfile` | `yarn install --immutable` |
| Monorepo workspaces | yes | yes, popular for TypeScript monorepos | yes |

**Semver** is `MAJOR.MINOR.PATCH`: breaking, feature, fix.

| Range | Allows | Note |
|---|---|---|
| `1.4.2` | exactly 1.4.2 | |
| `^1.4.2` | `>=1.4.2 <2.0.0` | npm's default when saving |
| `~1.4.2` | `>=1.4.2 <1.5.0` | patches only |
| `^0.4.2` | `>=0.4.2 <0.5.0` | in 0.x, minor bumps may break |
| `>=1.2 <2` | explicit range | |

**Lockfiles** record the exact resolved version and integrity hash of every package in the tree. Commit them. `npm ci` deletes `node_modules`, installs exactly what the lockfile says and fails if it disagrees with `package.json`; `npm install` may update the lockfile. Use `npm outdated`, `npm audit` and Renovate or Dependabot to upgrade deliberately.

**npx** runs a package's binary: a local one from `node_modules/.bin` first, otherwise a temporary download (`npx drizzle-kit generate`, `npx tsc --noEmit`).

## 9. Configuration and environment variables

The twelve-factor rule: config that changes between environments (URLs, credentials, feature flags) lives in environment variables, not in code.

```bash
node --env-file=.env dist/server.js                         # Node 20.6+: loads the file into process.env
node --env-file=.env --env-file=.env.local dist/server.js   # later files override earlier ones
# variables already set in the real environment win over values in the files
```

The dotenv package (`import 'dotenv/config'`) does the same on any Node version. Either way, parse once at startup:

```ts
// src/config/env.ts
import { z } from 'zod';

const Env = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().int().default(3000), // env values are strings: coerce
  DATABASE_URL: z.string().url(),
  FEATURE_NEW_CHECKOUT: z.enum(['true', 'false']).default('false').transform((v) => v === 'true'),
});

export const env = Env.parse(process.env); // crash at boot with a clear message, not on the first request
```

- `if (process.env.FEATURE_X)` is true for the string `"false"`: parse booleans explicitly, as above.
- Commit `.env.example`, never `.env`. In production, inject secrets from the platform or a secret manager.
- `NODE_ENV=production` changes library behavior (Express caches views and hides error details) and is expected by many tools.

## 10. Versions and LTS

- Each major release starts as **Current** for six months. Historically, even-numbered majors then became **LTS** (Active, later Maintenance, about 30 months in total) and odd-numbered majors simply ended.
- Production should run Active or Maintenance LTS only. Node 20 reached end of life in 2026, so new services target a newer LTS line such as 22 or 24.
- Starting with Node 27, the project moves to one major per year, and every major becomes LTS after its Current phase. Check the release schedule on nodejs.org before quoting dates.
- Pin the version everywhere: `engines`, `.nvmrc` or `.node-version` (nvm, fnm, Volta), the CI matrix and the Docker base image (`node:24-slim`, for example). "Works locally, fails in CI" is often a Node version mismatch.

## 11. Running TypeScript

| Option | How | Type checks | Typical use |
|---|---|---|---|
| `tsc` | compile to `dist/`, run `node dist/server.js` | yes | production builds of Node services |
| Bundler (esbuild, tsup) | bundle to one JS file | no: run `tsc --noEmit` in CI | fast builds, serverless functions |
| `tsx` | `tsx watch src/server.ts` | no | development and scripts |
| Node type stripping | `node src/server.ts` | no | scripts and simple services on recent Node |

Node's built-in type stripping arrived behind a flag in 22.6, became the default in 23.6 and 22.18, and was marked stable in 24.12 and 25.2. It replaces type annotations with whitespace, so:
- it never type-checks, and it ignores `tsconfig.json` (no `paths` aliases);
- syntax that needs code generation fails: `enum`, namespaces with runtime code, parameter properties (`constructor(private db: Db)`), legacy decorators;
- imports need real extensions (`import { db } from './db.ts'`).

TypeScript's `erasableSyntaxOnly` (5.8+) and `verbatimModuleSyntax` options make the compiler reject anything Node cannot strip.

## 12. Node vs Deno vs Bun

| | Node.js | Deno | Bun |
|---|---|---|---|
| Engine | V8 | V8 | JavaScriptCore |
| Implementation | C++ and libuv | Rust and Tokio | Zig |
| TypeScript | tsc/tsx; type stripping in recent versions | built in | built in |
| Security | full access (opt-in permission model in recent versions) | no file, network or env access without `--allow-*` flags | full access |
| Packages | npm, pnpm, Yarn | `npm:` specifiers, JSR, `package.json` support (Deno 2) | `bun install`, npm compatible and fast |
| Tooling | test runner built in, the rest from npm | formatter, linter, test runner, compiler built in | bundler, test runner, package manager built in |
| Strength | maturity, ecosystem, hosting support | secure defaults, web standards | speed, all-in-one developer experience |

Frameworks built on web standards (Hono, oRPC's fetch adapter) run on all three, which keeps the choice reversible.

## Common pitfalls
- Assuming `async` makes CPU work non-blocking. An async function that loops for two seconds still blocks; async only helps while awaiting I/O.
- Calling `*Sync` APIs inside request handlers. They are fine at startup (reading config once), not per request.
- Relying on `setTimeout(fn, 0)` vs `setImmediate` order in the main module.
- Recursive `process.nextTick` or promise loops that never let the poll phase run.
- Forgetting that `fs`, `dns.lookup`, heavy crypto and `zlib` share four pool threads, so heavy hashing slows file reads.
- ESM mix-ups: missing `.js` extensions in relative imports, `__dirname` is not defined, `require()` of an ESM-only package on an older Node.
- Treating env vars as typed values: `"false"` is truthy and `"3000"` is a string.
- Running `npm install` in CI or Docker, so the lockfile drifts; not committing the lockfile at all.
- Expecting `^0.3.1` to pick up `0.4.0`.
- Different Node versions locally, in CI and in the Docker image.

## How to talk about this in an interview
- Lead with the one-sentence model: "My JavaScript runs on one thread; libuv hands network I/O to the OS and blocking work to a small thread pool, and the event loop runs callbacks when results arrive. That makes Node cheap for I/O-heavy APIs and bad at CPU-heavy work on the main thread."
- Contrast it with what you know from Rails: Puma serves requests with threads and Sidekiq runs jobs in separate processes; Node serves many requests on one thread and moves CPU or slow work to workers or queues. This shows depth across stacks.
- Connect it to the multi-tenant TypeScript platform at Devntech, where your skills list Node.js with Hono and oRPC.
  > **Fill in:** which Node version the platform runs, whether it is ESM or CommonJS, how TypeScript runs in development and production (tsx, tsc, a bundler), and which package manager the repo uses.
  > **Fill in:** any slow endpoint or blocking issue you investigated, and how you found the cause.
- For RemoteDev: Next.js route handlers run on the Node.js runtime by default, so the same event-loop rules applied to the APIs you built there. Details: [RemoteDev deep dive](../17-projects-deep-dive/06-remotedev-nextjs-mongodb.md).
- If an ordering puzzle comes up, think aloud: synchronous code, then nextTick, then microtasks, then the phase order, and say which module system you are assuming.

## Interview questions

**Q1. What is Node.js, and how does it differ from JavaScript in the browser?**
A runtime that runs JavaScript outside the browser: V8 executes the code and libuv provides the event loop, async I/O and a thread pool, with core modules for files, networking, processes and crypto. There is no DOM; instead you get `fs`, `http`, `process`, `Buffer` and streams. Promises, `fetch`, `URL` and `AbortController` exist in both.
*Follow-up:* Is Node a framework? No. Express, Hono and NestJS are frameworks that run on Node.

**Q2. Node is single-threaded. How does it handle thousands of concurrent requests?**
Your JavaScript runs on one thread, but waiting does not occupy it. Sockets are watched by the OS (epoll, kqueue, IOCP), blocking work goes to libuv's thread pool, and callbacks run on the main thread when results are ready. Each request uses the thread only for the short bursts of JavaScript between awaits.
*Follow-up:* When does that model break? When one request does long CPU work, because every other request waits for it.

**Q3. What is libuv, and which operations use its thread pool?**
The C library behind Node's event loop and async I/O. Its pool (4 threads by default, `UV_THREADPOOL_SIZE`) runs file system calls, `dns.lookup`, `crypto.pbkdf2`, `scrypt`, async `randomBytes` and `zlib`. Network sockets do not use the pool; they use OS readiness notifications.
*Follow-up:* Symptom of an exhausted pool? File reads and hashing get slow under load even though CPU usage on the main thread looks fine.

**Q4. Walk me through the phases of the event loop.**
Timers (`setTimeout`, `setInterval`), pending callbacks (deferred system errors), idle/prepare (internal), poll (fetch I/O events, run their callbacks, wait for I/O if nothing else is due), check (`setImmediate`), close callbacks, then around again. After every callback, Node drains the nextTick queue, then the promise microtask queue.
*Follow-up:* Where does the process sit when idle? In the poll phase, waiting on the OS for I/O or the next timer.

**Q5. What is the difference between `process.nextTick`, `setImmediate` and `setTimeout(fn, 0)`?**
`nextTick` runs after the current operation, before promise callbacks and before the loop continues. `setImmediate` runs in the check phase after the next poll. `setTimeout(fn, 0)` runs in the timers phase after at least 1 ms. Main module: timeout vs immediate order is random; inside an I/O callback, immediate always wins.
*Follow-up:* Which should library code prefer for deferring? Usually `setImmediate` or `queueMicrotask`; `nextTick` can starve I/O if it recurses.

**Q6. What does this print in a CommonJS file?**
```ts
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('tick'));
console.log('sync');
```
`sync`, `tick`, `promise`, `timeout`. Synchronous code first, then the nextTick queue, then microtasks, then the timers phase.
*Follow-up:* And as an ES module? `sync`, `promise`, `tick`, `timeout`, because ESM top-level code runs inside a promise job.

**Q7. What can block the event loop, and how do you find and fix it?**
Synchronous APIs in handlers, huge `JSON.parse`/`stringify`, CPU loops, ReDoS-prone regexes, recursive microtasks. Detect it with event-loop delay metrics (`monitorEventLoopDelay`) and CPU profiles (`--cpu-prof`, `--inspect`). Fix it with async APIs, chunking with `setImmediate`, worker threads, or moving the work to a job queue or another service.
*Follow-up:* Does wrapping CPU work in a promise help? No: the executor still runs on the main thread.

**Q8. An endpoint that builds a large PDF report slows down the whole API. What do you do?**
Confirm with a profile or loop-delay metric. If the user can wait, move it to a background job: return `202 Accepted` with a job id, build the file in a worker process, upload it to S3 and notify or let the client poll. If the response must be synchronous, run the rendering in a worker thread pool and cap concurrency.
*Follow-up:* Why not just add more instances? It spreads the pain, but each instance still freezes while rendering.

**Q9. When would you not choose Node?**
For CPU-bound services (media processing, ML inference, heavy computation) and when a team or ecosystem fit points elsewhere (data science in Python, existing Rails expertise). Node is strongest for I/O-heavy APIs, realtime features and full-stack TypeScript.
*Follow-up:* Can Node still be part of such a system? Yes, as the API layer that delegates heavy work to a queue or service.

**Q10. CommonJS vs ES modules: what is the difference, and how do they interoperate?**
CommonJS uses `require`/`module.exports` and loads synchronously at call time; ESM uses static `import`/`export`, loads asynchronously, supports top-level `await` and tree-shaking, and needs file extensions in relative imports. ESM can import CommonJS as a default export; CommonJS can use `import()`, and recent Node versions can `require()` ES modules without top-level `await`.
*Follow-up:* What decides how a `.js` file is parsed? The nearest package.json `type` field; `.mjs` and `.cjs` override it.

**Q11. How do you get `__dirname` in an ES module, and why is it missing?**
ESM modules are identified by URLs, not file paths, so the CommonJS wrapper variables do not exist. Use `import.meta.dirname` and `import.meta.filename` (Node 20.11+) or `path.dirname(fileURLToPath(import.meta.url))`.
*Follow-up:* And `require` inside ESM? `createRequire(import.meta.url)` from `node:module`.

**Q12. What do `main`, `exports` and `type` do in package.json?**
`type` sets how `.js` files are parsed. `main` is the legacy entry point. `exports` defines the public entry points with conditions (`types`, `import`, `require`, `default`) and blocks deep imports of anything not listed.
*Follow-up:* Why might adding `exports` be a breaking change? Consumers importing unlisted subpaths start getting `ERR_PACKAGE_PATH_NOT_EXPORTED`.

**Q13. Explain `^` vs `~`, and why lockfiles and `npm ci` matter.**
`^1.4.2` accepts any 1.x from 1.4.2; `~1.4.2` accepts only 1.4.x patches; in 0.x the caret stays inside the minor. The lockfile pins the exact resolved tree with integrity hashes. `npm ci` installs exactly that and fails on mismatch, giving reproducible CI and Docker builds; `npm install` may update the lockfile.
*Follow-up:* Should libraries commit lockfiles? Yes for their own CI, but consumers resolve from the library's ranges, not its lockfile.

**Q14. dependencies vs devDependencies vs peerDependencies?**
Runtime packages vs build and test tools vs packages the host application must provide. A plugin or component library declares `react` or `hono` as a peer so the app's single copy is shared instead of bundling a second one.
*Follow-up:* What does `npm ci --omit=dev` do? Installs only runtime dependencies, which keeps production images small.

**Q15. How do you manage configuration and secrets in a Node app?**
Environment variables, loaded locally with `--env-file` or dotenv and injected by the platform in production. Parse and validate them once at startup with a schema (Zod), export a typed config object and never read `process.env` elsewhere. Secrets never go into Git, logs or client bundles.
*Follow-up:* Why validate at boot? A missing `DATABASE_URL` should stop the deploy, not fail the first customer request.

**Q16. Which Node version would you run in production, and why?**
An LTS line, because it gets security and bug fixes for its whole support window. Pin it in `engines`, `.nvmrc`, CI and the Docker image, and plan upgrades before end of life (Node 20 went end of life in 2026).
*Follow-up:* What changes from Node 27? One major per year, and every major becomes LTS after its Current phase.

**Q17. How do you run TypeScript on Node?**
Production: compile with `tsc` (or bundle with esbuild/tsup) and run JavaScript. Development: `tsx watch`. Recent Node versions also strip types natively, but only for erasable syntax, without type checking, so CI still runs `tsc --noEmit`.
*Follow-up:* Why might `enum` break native type stripping? Enums emit runtime code, which stripping cannot produce.

**Q18. How do Node, Deno and Bun compare?**
Node is the mature default with the largest ecosystem. Deno is secure by default (explicit permissions), has TypeScript and tooling built in and is npm compatible since Deno 2. Bun uses JavaScriptCore and bundles a fast package manager, bundler and test runner. Web-standard frameworks such as Hono run on all three.
*Follow-up:* Would you move a production API to Bun for speed? Only after benchmarking real workloads and checking compatibility of every dependency.

## Further reading
- [Node.js: The event loop, timers and process.nextTick()](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [Node.js: Don't block the event loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)
- [Node.js: Modules (ECMAScript and CommonJS)](https://nodejs.org/api/esm.html)
- [Node.js: Running TypeScript natively](https://nodejs.org/api/typescript.html)
- [Node.js: Release schedule](https://nodejs.org/en/about/previous-releases)
