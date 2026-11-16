# Node.js Core Modules, Streams and Error Handling

> **Tier:** T1 (know cold) · **Related:** [Node.js fundamentals](01-nodejs-fundamentals.md), [Express](03-express.md), [Production APIs](06-production-apis.md), [Async JavaScript](../03-languages/javascript/04-async-event-loop-promises.md), [Web security](../04-web-fundamentals/07-web-security-owasp.md), [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md)
> **On your resume:** the Node.js (Hono, oRPC) side of the Devntech platform migration, where Rails sync APIs, webhooks, exports and error handling all rely on these modules; the Next.js (TypeScript) APIs at RemoteDev run on the same runtime.

## Quick revision
- Import core modules with the `node:` prefix (`node:fs/promises`): explicit, and it can never resolve to an npm package.
- `fs`: use `node:fs/promises`; `*Sync` only at startup or in scripts; stream large files with `createReadStream`.
- `path.join` joins segments; `path.resolve` produces an absolute path. Never concatenate paths by hand, and check user-supplied names for `..` traversal. Use `URL`/`URLSearchParams` for URLs (`url.parse` is legacy).
- `crypto`: `randomUUID()` for ids, `randomBytes` for tokens, SHA-256 for checksums, HMAC for signatures, `timingSafeEqual` to compare secrets, scrypt/argon2/bcrypt for passwords.
- Webhooks: HMAC over the **raw** body bytes, constant-time comparison, timestamp tolerance against replays, dedupe by event id, acknowledge fast.
- `emitter.emit()` is synchronous; an `'error'` event with no listener throws.
- `Buffer` holds bytes (it subclasses `Uint8Array`). String length counts UTF-16 code units, not bytes: `Buffer.byteLength('é')` is 2.
- Streams process data in chunks with constant memory: Readable, Writable, Duplex, Transform.
- Backpressure: `write()` returns `false` once the buffer passes `highWaterMark`; stop and wait for `'drain'`. `pipeline` does this for you.
- Prefer `pipeline` from `node:stream/promises` over `.pipe()`: it forwards errors, destroys every stream on failure and returns a promise.
- `child_process`: `spawn` streams with no shell; `exec` buffers output through a shell (injection risk); `execFile` runs a binary with an args array; `fork` runs a Node script with IPC.
- `worker_threads` for CPU work inside one process; multiple processes (cluster, PM2, or container replicas) to use every core.
- Error channels: sync code throws (try/catch); callbacks pass `err` first; promises reject (await in try/catch); emitters and streams emit `'error'`.
- Unhandled rejections crash the process (Node 15+). `uncaughtException` handlers log and exit; they never resume serving.
- Operational errors (bad input, timeouts, partner 503s) are expected and handled; programmer errors (TypeError, broken invariant) are bugs: crash, restart, fix.
- Custom error classes carry `statusCode`, `code` and `cause`, so one error handler maps them to responses.
- Graceful shutdown on SIGTERM: stop accepting, finish in-flight requests, close pools and queues, exit, with a forced exit after a timeout.
- Every outbound call gets a timeout (`AbortSignal.timeout(ms)`, combined with `AbortSignal.any`). Global `fetch` (undici, Node 18+) rejects only on network errors and aborts, so check `res.ok`.

## 1. Core modules at a glance

| Module | Use it for |
|---|---|
| `node:fs`, `node:fs/promises` | files and directories |
| `node:path` | building and parsing file paths |
| `node:os` | CPU count, memory, temp directory, hostname |
| `node:url` | `URL`, `fileURLToPath`, `pathToFileURL` |
| `node:crypto` | hashing, HMAC, random values, encryption |
| `node:http`, `node:https` | raw HTTP servers and clients |
| `node:events` | `EventEmitter`, `once`, `on` |
| `node:stream`, `node:stream/promises`, `node:zlib` | streaming data, `pipeline`, gzip and brotli |
| `node:child_process`, `node:worker_threads`, `node:cluster` | other programs, CPU work on threads, processes sharing a port |
| `node:util`, `node:timers/promises` | `promisify`, `inspect`; promise versions of `setTimeout` and `setImmediate` |
| `node:async_hooks`, `node:perf_hooks` | `AsyncLocalStorage` for request context; timings and event-loop delay |
| `node:test`, `node:assert` | built-in test runner and assertions |

## 2. fs, path, os and url

```ts
import { mkdir, readdir, readFile, rm, stat, writeFile } from 'node:fs/promises';
import path from 'node:path';

const dir = path.join(import.meta.dirname, 'exports');   // correct separator on every OS
await mkdir(dir, { recursive: true });                   // no error if it already exists
const file = path.join(dir, 'report.json');
await writeFile(file, JSON.stringify({ ok: true }), 'utf8');
const text = await readFile(file, 'utf8');               // without an encoding you get a Buffer
const [entries, { size }] = await Promise.all([readdir(dir), stat(file)]); // ['report.json'], bytes
await rm(dir, { recursive: true, force: true });

try {
  await readFile('missing.json', 'utf8');
} catch (err) {
  if ((err as NodeJS.ErrnoException).code !== 'ENOENT') throw err; // only "not found" is expected here
}
```

| Style | Example | Use |
|---|---|---|
| Promises | `await readFile(p, 'utf8')` | the default |
| Callbacks | `readFile(p, 'utf8', (err, data) => {...})` | legacy code (error-first convention) |
| Sync | `readFileSync(p, 'utf8')` | startup config and CLI scripts, never per request |

- Error codes worth recognizing: `ENOENT` (missing), `EACCES`/`EPERM` (permissions), `EEXIST`, `EISDIR`, `EMFILE` (too many open files, usually from unbounded `Promise.all` over thousands of files).
- `existsSync()` followed by `readFile()` is a race; attempt the operation and handle `ENOENT`. `path.join('a', '../b')` normalizes to `b`; `path.resolve('uploads')` returns an absolute path from the current working directory; `basename`, `extname`, `dirname` and `relative` cover the rest.

Never trust a file name from a user:

```ts
const UPLOAD_ROOT = path.resolve('uploads');

export function safeUploadPath(userFileName: string): string {
  const full = path.resolve(UPLOAD_ROOT, userFileName);            // resolves any '..' segments
  if (!full.startsWith(UPLOAD_ROOT + path.sep)) throw new Error('Invalid file name'); // blocks ../../etc/passwd
  return full;
}
```

`os` gives you `availableParallelism()` (CPUs you can use), `totalmem()`, `freemem()`, `tmpdir()` and `hostname()`. For URLs, use the WHATWG classes:

```ts
const url = new URL('/listings?page=2', 'https://api.example.com');
url.searchParams.get('page');            // '2': query values are always strings
url.searchParams.append('tag', 'garden room');
url.toString();                          // 'https://api.example.com/listings?page=2&tag=garden+room'
```

## 3. crypto

```ts
import { createHash, createHmac, randomBytes, randomUUID } from 'node:crypto';

randomUUID();                                      // e.g. '3b241101-e2bb-4255-8caf-4136c566a962' (v4)
randomBytes(32).toString('base64url');             // an unguessable token for reset links or API keys
createHash('sha256').update('file contents').digest('hex');         // fingerprint or checksum
createHmac('sha256', 'shared-secret').update('payload').digest('hex'); // keyed signature
```

| Need | Use | Avoid |
|---|---|---|
| Unique ids | `randomUUID()` | `Math.random()` |
| Tokens (password reset, API keys) | `randomBytes(32)` | timestamps, sequential ids |
| Checksums and fingerprints | SHA-256 | MD5 or SHA-1 where security matters |
| Proving who sent a message (webhooks, signed URLs) | HMAC-SHA256 with a shared secret | `sha256(secret + body)` |
| Passwords | scrypt (built in), or argon2id/bcrypt libraries | any fast hash, even SHA-256 |
| Comparing signatures or tokens | `timingSafeEqual` | `===` (timing leaks) |

Fast hashes are wrong for passwords because attackers can try billions of guesses per second with GPUs. Password KDFs are deliberately slow and salted.

### Verifying webhook signatures

Providers sign the exact bytes they send. Parse-then-stringify changes whitespace and key order, so verify against the raw body:

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

// GitHub-style header: "sha256=<hex HMAC of the raw body>"
export function isValidSignature(rawBody: Buffer, header: string | undefined, secret: string): boolean {
  if (!header?.startsWith('sha256=')) return false;
  const expected = Buffer.from(`sha256=${createHmac('sha256', secret).update(rawBody).digest('hex')}`);
  const received = Buffer.from(header);
  // timingSafeEqual throws when lengths differ; the length itself is not secret
  return expected.length === received.length && timingSafeEqual(expected, received);
}
```

```ts
// Express: mount the raw parser on this route BEFORE any global express.json()
app.post('/webhooks/payments', express.raw({ type: 'application/json', limit: '1mb' }), async (req, res) => {
  const raw = req.body as Buffer;
  if (!isValidSignature(raw, req.get('x-signature-256'), env.WEBHOOK_SECRET)) {
    res.sendStatus(401);
    return;
  }
  const event = JSON.parse(raw.toString('utf8')) as { id: string; type: string };
  if (await alreadyProcessed(event.id)) { res.sendStatus(200); return; } // providers retry: stay idempotent
  await enqueueWebhookJob(event);                                        // acknowledge fast, process in a job
  res.sendStatus(200);
});
```

Schemes that sign `timestamp.body` (Stripe-style) also let you reject events older than a few minutes, which blocks replays.

## 4. A raw HTTP server

```ts
import { createServer } from 'node:http';

const server = createServer(async (req, res) => {
  const url = new URL(req.url ?? '/', 'http://localhost');
  if (req.method === 'GET' && url.pathname === '/health') {
    res.writeHead(200, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ ok: true }));
    return;
  }
  if (req.method === 'POST' && url.pathname === '/echo') {
    const chunks: Buffer[] = [];
    for await (const chunk of req) chunks.push(chunk as Buffer); // the request is a Readable stream
    const body = JSON.parse(Buffer.concat(chunks).toString('utf8')); // invalid JSON: rejected promise, process crash
    res.writeHead(201, { 'content-type': 'application/json' });
    res.end(JSON.stringify(body));
    return;
  }
  res.writeHead(404).end();
});

server.listen(3000);
```

The bug in the comment is the point: with no framework, a throw inside an async handler is an unhandled rejection, and nothing limits the body size. Frameworks add routing, body parsing with limits, async error handling, middleware, cookies, validation hooks and content negotiation. Everything in [Express](03-express.md) and [Hono](04-hono.md) is built on this primitive (Hono through the Web `Request`/`Response` adapter).

## 5. EventEmitter

```ts
import { EventEmitter, once } from 'node:events';

const orders = new EventEmitter();

orders.on('created', (order: { id: string }) => console.log('send confirmation for', order.id));
orders.once('created', () => console.log('runs for the first event only'));
orders.on('error', (err: Error) => console.error('order pipeline failed', err)); // without it, emit('error') throws

orders.emit('created', { id: 'o1' }); // listeners run synchronously, in registration order, before emit returns

const [next] = await once(orders, 'created'); // a promise for the next event (top-level await: ESM)
```

- `emit` is synchronous: a slow listener blocks the emitter, and an exception in a listener propagates to the code that called `emit`.
- Servers, sockets, streams and `process` are all emitters.
- Adding more than 10 listeners for one event prints `MaxListenersExceededWarning`, usually a leak from registering listeners per request. `events.on(emitter, 'name')` returns an async iterator for `for await` loops.
- In-process events are not durable. Work that must survive a crash (send email after signup) belongs in a job queue.

## 6. Buffer and encodings

```ts
const buf = Buffer.from('héllo', 'utf8');
buf.length;                                   // 6 bytes: 'é' takes 2 bytes in UTF-8
'héllo'.length;                               // 5 UTF-16 code units
Buffer.byteLength('héllo');                   // 6: use this for Content-Length and size limits
buf.toString('base64');                       // 'aMOpbGxv'
buf.toString('hex');                          // '68c3a96c6c6f'
Buffer.from('aMOpbGxv', 'base64').toString('utf8'); // 'héllo'
Buffer.alloc(16);                             // zero-filled
Buffer.allocUnsafe(16);                       // faster, may contain old memory: overwrite it fully
```

`base64url` (no `+`, `/` or padding) is the variant for tokens in URLs and JWTs. `new Buffer()` is deprecated. When you decode text from a stream, set an encoding (`setEncoding('utf8')` or the `encoding` option) so multi-byte characters split across chunks are decoded correctly.

## 7. Streams

Streams move data in chunks, so memory stays flat whether the payload is 1 KB or 10 GB, and the first bytes reach the client sooner.

| Type | Role | Examples |
|---|---|---|
| Readable | source | `fs.createReadStream`, an incoming request (`req`), `process.stdin` |
| Writable | sink | `fs.createWriteStream`, the server response (`res`), `process.stdout` |
| Duplex | readable and writable, independent sides | a TCP socket |
| Transform | duplex whose output is computed from its input | `zlib.createGzip()`, a CSV parser |

**Backpressure** is the flow-control signal from a slow consumer to a fast producer. Each writable buffers up to `highWaterMark`; past that, `write()` returns `false`. A producer that ignores it keeps buffering in memory until the process falls over. Doing it by hand:

```ts
import { once } from 'node:events';
import type { Writable } from 'node:stream';

export async function writeLines(out: Writable, lines: Iterable<string>) {
  for (const line of lines) {
    if (!out.write(line + '\n')) await once(out, 'drain'); // buffer full: wait until it empties
  }
  out.end();
}
```

**`pipe` vs `pipeline`.** `a.pipe(b)` handles backpressure but not errors: if `a` fails, `b` stays open (leaked file descriptors, hung responses). `pipeline` forwards errors, destroys every stage and returns a promise:

```ts
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

await pipeline(
  createReadStream('access.log', { encoding: 'utf8' }),
  async function* (source: AsyncIterable<string>) {   // an async generator works as a transform stage
    for await (const chunk of source) yield chunk.replaceAll(/token=\w+/g, 'token=[redacted]');
  },
  createGzip(),
  createWriteStream('access.log.gz'),
);
```

For line-by-line imports, `readline.createInterface({ input: createReadStream(file), crlfDelay: Infinity })` gives you `for await (const line of rl)`. Node and Web Streams interconvert with `Readable.toWeb()` and `Readable.fromWeb()`, useful when a `fetch` body or a Hono `Response` meets a Node stream.

### Streaming a large CSV export

The naive version loads every row, builds one giant string and sends it: memory spikes and the request may time out. Stream it instead, pulling rows in batches with keyset pagination (Drizzle shown; any driver works):

```ts
import type { ServerResponse } from 'node:http';
import { Readable } from 'node:stream';
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';
import { and, asc, eq, gt } from 'drizzle-orm';
import { db } from '../db/client';
import { orders } from '../db/schema';

const csvCell = (value: unknown) => `"${String(value ?? '').replaceAll('"', '""')}"`; // CSV quoting rules

async function* orderCsv(orgId: string) {
  yield 'id,customer,total_cents,created_at\n';
  let lastId = 0;
  while (true) {
    const batch = await db.select().from(orders)
      .where(and(eq(orders.orgId, orgId), gt(orders.id, lastId)))  // keyset: no slow OFFSET scans
      .orderBy(asc(orders.id))
      .limit(1000);
    if (batch.length === 0) return;
    for (const o of batch) {
      yield [o.id, o.customerName, o.totalCents, o.createdAt.toISOString()].map(csvCell).join(',') + '\n';
    }
    lastId = batch[batch.length - 1]!.id;
  }
}

export async function streamOrdersCsv(orgId: string, res: ServerResponse) {
  res.setHeader('Content-Type', 'text/csv; charset=utf-8');
  res.setHeader('Content-Disposition', 'attachment; filename="orders.csv"');
  res.setHeader('Content-Encoding', 'gzip');
  await pipeline(Readable.from(orderCsv(orgId)), createGzip(), res); // backpressure end to end
}
```

- Memory stays at about one batch, however many rows the org has.
- If the client disconnects, `pipeline` destroys the source, the generator stops, and the database stops being queried.
- Headers are sent before the first row, so a failure halfway cannot become a clean 500: log it, and the client sees a truncated download. For very slow exports, build the file in a background job, upload it to S3 and send a link. Express's `res` extends `ServerResponse`, so this function plugs straight into a route.

## 8. child_process

| | `spawn` | `exec` | `execFile` | `fork` |
|---|---|---|---|---|
| Shell | no (unless `shell: true`) | yes | no | no |
| Output | streams | buffered (1 MB `maxBuffer` by default) | buffered | IPC messages plus stdio |
| Use for | long-running or large output (ffmpeg, pg_dump) | short one-liners with trusted input | a binary with arguments, safely | another Node script that exchanges messages |

```ts
import { execFile, spawn } from 'node:child_process';
import { promisify } from 'node:util';

const execFileAsync = promisify(execFile);
const { stdout } = await execFileAsync('git', ['rev-parse', 'HEAD']); // arguments are never parsed by a shell

const ffmpeg = spawn('ffmpeg', ['-i', 'input.mp4', '-vf', 'scale=640:-1', 'out.mp4']);
ffmpeg.stderr.on('data', (chunk: Buffer) => process.stdout.write(chunk)); // progress output streams live
ffmpeg.on('close', (code) => console.log('ffmpeg exited with', code));
```

`exec(`convert ${fileName} out.png`)` with a user-controlled `fileName` such as `x; rm -rf ~` runs the injected command. Pass arguments as an array to `execFile` or `spawn` instead.

## 9. worker_threads, cluster and process managers

| | `worker_threads` | `child_process` | `cluster` | Container replicas |
|---|---|---|---|---|
| Unit | thread inside the process | separate OS process | Node processes sharing one port | processes on one or many machines |
| Memory | separate heaps; can share `SharedArrayBuffer` and transfer `ArrayBuffer`s | separate | separate | separate |
| Communication | `postMessage` | IPC, stdio | IPC through the primary | network (HTTP, Redis, queues) |
| Use for | CPU-heavy functions (parsing, hashing, image work) | other programs, isolating risky work | all cores on one VM without an orchestrator | production scaling on Docker, Kubernetes, ECS |

A worker example lives in [Node.js fundamentals](01-nodejs-fundamentals.md); use a pool (Piscina) instead of one worker per call. A cluster sketch:

```ts
import cluster from 'node:cluster';
import { availableParallelism } from 'node:os';

if (cluster.isPrimary) {
  for (let i = 0; i < availableParallelism(); i++) cluster.fork();
  cluster.on('exit', (worker) => {
    console.error(`worker ${worker.process.pid} died, starting a new one`);
    cluster.fork();
  });
} else {
  startServer(); // your app's listen(): every worker uses the same port; the primary hands out connections
}
```

On a plain VM, PM2's cluster mode does this plus restarts and log handling. In containers, run one Node process per container and let the orchestrator restart and scale replicas. Either way, processes share no memory: sessions, rate-limit counters, caches and websocket fan-out need Redis or the database.

## 10. Error handling

| Where | How errors arrive | How you handle them |
|---|---|---|
| Synchronous code | `throw` | `try/catch` |
| Callback APIs | first argument: `(err, data)` | check `err` before using `data` |
| Promises and async functions | rejection | `await` inside `try/catch`, or `.catch()` |
| EventEmitters and streams | `'error'` event | a listener, or `pipeline` |
| Anything missed | `unhandledRejection`, `uncaughtException` | log and exit |

A try/catch cannot catch errors from a callback that runs later:

```ts
import { readFile } from 'node:fs';

try {
  readFile('config.json', 'utf8', (err, data) => {
    if (err) throw err;   // thrown later, outside the try block: crashes the process
    JSON.parse(data);
  });
} catch {
  console.error('never reached for asynchronous errors');
}
// Fix: const config = JSON.parse(await readFile('config.json', 'utf8')) inside try/catch (node:fs/promises)
```

**Global handlers are a last resort**, not error handling:

```ts
process.on('unhandledRejection', (reason) => {
  logger.fatal({ err: reason }, 'unhandled promise rejection');
  process.exit(1); // registering this handler disables Node's default crash, so exit explicitly
});

process.on('uncaughtException', (err, origin) => {
  logger.fatal({ err, origin }, 'uncaught exception');
  process.exit(1); // state may be corrupt: let the orchestrator start a clean process
});
```

**Custom error classes** let one error handler (Express error middleware, Hono `onError`, oRPC interceptors) turn domain failures into HTTP responses:

```ts
export class AppError extends Error {
  readonly statusCode: number;
  readonly code: string;
  constructor(message: string, statusCode = 500, code = 'INTERNAL', options?: ErrorOptions) {
    super(message, options);       // options.cause keeps the original error and its stack
    this.name = new.target.name;   // 'NotFoundError' for subclasses
    this.statusCode = statusCode;
    this.code = code;
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

// Wrap low-level failures without losing them:
// throw new AppError('Pricing provider unavailable', 502, 'UPSTREAM', { cause: err });
```

| | Operational errors | Programmer errors |
|---|---|---|
| What | expected problems in a correct program | bugs |
| Examples | invalid input, record not found, DB timeout, partner returns 503, client disconnects | `undefined is not a function`, wrong argument types, forgotten `await`, broken invariant |
| Response | handle: 4xx response, retry with backoff, fallback, log at warn | fix the code; crash and restart is acceptable; log at error or fatal and alert |

## 11. Graceful shutdown

Deploys, autoscaling and restarts send `SIGTERM`. Exiting immediately drops in-flight requests and can leave half-finished work. Instead:

```ts
import type { Server } from 'node:http';

export function registerShutdown(server: Server, cleanup: () => Promise<void>, timeoutMs = 10_000) {
  let shuttingDown = false;

  const shutdown = (signal: string) => {
    if (shuttingDown) return;
    shuttingDown = true;
    logger.info({ signal }, 'shutting down');
    markNotReady();                                  // readiness probe starts returning 503

    setTimeout(() => {
      logger.error('shutdown timed out, forcing exit');
      process.exit(1);
    }, timeoutMs).unref();                           // unref: this timer alone keeps nothing alive

    server.close(async (err) => {                    // stop accepting; runs after in-flight requests finish
      try {
        await cleanup();                             // close the DB pool, Redis, queue workers, flush logs
        process.exit(err ? 1 : 0);
      } catch (e) {
        logger.error({ err: e }, 'cleanup failed');
        process.exit(1);
      }
    });
    server.closeIdleConnections();                   // idle keep-alive sockets would otherwise hold close() open
  };

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
} // logger and markNotReady come from your app (pino logger, readiness flag)
```

- `docker stop` sends SIGTERM and waits 10 seconds before SIGKILL; Kubernetes waits `terminationGracePeriodSeconds` (30 by default). Keep your timeout below that.
- Use the exec form `CMD ["node", "dist/server.js"]`. Wrapping Node in `npm start` or a shell can swallow the signal. Node as PID 1 only reacts to SIGTERM because you registered a handler; `docker run --init` adds a minimal init. Queue workers should stop taking new jobs and finish (or release) the current one.

## 12. AbortController and timeouts

A partner API that hangs forever ties up your sockets and memory unless you give up first.

```ts
export async function fetchJson(url: string, opts: { signal?: AbortSignal; timeoutMs?: number } = {}) {
  const timeout = AbortSignal.timeout(opts.timeoutMs ?? 5_000);        // rejects with a TimeoutError
  const signal = opts.signal ? AbortSignal.any([opts.signal, timeout]) : timeout; // Node 20.3+
  const res = await fetch(url, { signal });
  if (!res.ok) throw new Error(`GET ${url} failed with status ${res.status}`);
  return (await res.json()) as unknown;                                 // validate with Zod before trusting it
}
```

```ts
// Stop expensive work when the client disconnects (Express)
app.get('/reports/heavy', async (req, res) => {
  const controller = new AbortController();
  res.on('close', () => { if (!res.writableFinished) controller.abort(); });
  const report = await buildReport({ signal: controller.signal }); // pass the signal down to fetch, timers, queries
  res.json(report);
});
```

- Many APIs accept a `signal`: `fetch`, `fs.readFile`, `events.once`, `child_process.spawn`, `setTimeout` from `node:timers/promises`. Retry only idempotent calls and retryable failures (timeouts, 429, 502/503/504), with exponential backoff and jitter, and a cap on attempts.
- On the server side, Node's HTTP server has `requestTimeout`, `headersTimeout` and `keepAliveTimeout`; [production APIs](06-production-apis.md) covers the load balancer interaction.

## 13. Built-in fetch

```ts
const res = await fetch(`${env.PRICING_API_URL}/items/${itemId}`, {
  headers: { Authorization: `Bearer ${env.PRICING_API_TOKEN}`, Accept: 'application/json' },
  signal: AbortSignal.timeout(5_000),
});
if (res.status === 404) {
  await res.body?.cancel();                      // release the connection when you skip the body
  return null;
}
if (!res.ok) throw new AppError(`pricing API returned ${res.status}`, 502, 'UPSTREAM');
const data: unknown = await res.json();
```

- `fetch` does not reject on 4xx or 5xx: check `res.ok`. Read or cancel every body; unconsumed bodies keep connections busy.
- No cookie jar and no automatic retries; streaming a request body requires `duplex: 'half'`. For connection pooling settings, proxies or mocking, use the `undici` package directly (Agent, MockAgent), or a client such as axios or ky if the team prefers.

## Common pitfalls
- Using `.pipe()` without error handling, leaving files or responses open after a failure; ignoring `write()`'s return value and buffering a whole export in memory.
- Reading whole files or uploads into memory (`readFile` on a 2 GB file, in-memory upload storage for large files).
- Verifying webhook HMACs over re-serialized JSON, comparing signatures with `===`, or registering `express.json()` globally before the webhook route so the raw body is gone.
- `exec` with user input (shell injection).
- Throwing inside callbacks or event listeners and expecting a surrounding try/catch to help; swallowing errors with empty `catch {}` blocks.
- Continuing to serve after `uncaughtException`.
- Exiting on SIGTERM without draining, or draining with no timeout so shutdown hangs until SIGKILL.
- Outbound calls without timeouts, and retries on non-idempotent POSTs.
- Unbounded concurrency (`Promise.all` over 10,000 files or API calls) causing `EMFILE` or rate limiting; batch it or use a concurrency limiter.

## How to talk about this in an interview
- Webhooks and sync endpoints are where these modules show up most. Explain HMAC over the raw body, constant-time comparison, replay windows, idempotency by event id and fast acknowledgement with async processing, then connect it to the Rails sync APIs in the platform migration at Devntech.
  > **Fill in:** how the Rails monolith and the TypeScript platform authenticated sync calls (shared-secret HMAC, API keys, something else) and how retries and duplicate deliveries were handled.
- For streams, use the CSV export story: constant memory, backpressure, cleanup on client disconnect, and when to switch to a background job.
  > **Fill in:** any export, import or file-processing feature you built, and its rough data size.
- For error handling, describe layers: domain errors as classes, one central mapper to HTTP or oRPC errors, structured logs with a request id, and global handlers only for logging and exiting.
- If asked about scaling across cores, say you would run several container replicas behind a load balancer rather than hand-rolled `cluster`, and keep the processes stateless.

## Interview questions

**Q1. When is it acceptable to use synchronous `fs` APIs?**
At startup (reading config or certificates once), in CLI scripts and in build tooling. Never on the request path: a sync call blocks the event loop and every concurrent request waits for the disk.
*Follow-up:* What about `fs.promises`? It runs on libuv's thread pool, so it does not block the main thread.

**Q2. What is the difference between `path.join` and `path.resolve`, and how do you stop path traversal?**
`join` concatenates segments with the right separator and normalizes `..`; `resolve` returns an absolute path, starting from the working directory. For user-supplied names, resolve against an allowed root and reject results that do not start with that root plus a separator.
*Follow-up:* Is stripping `../` with a regex enough? No: encodings and absolute paths slip through; check the resolved path.

**Q3. What are streams, and what are the four types?**
Abstractions for processing data in chunks: Readable (source), Writable (sink), Duplex (both sides, like a socket) and Transform (output derived from input, like gzip). They keep memory constant and start sending data before the whole payload exists.
*Follow-up:* Name streams you meet in an HTTP server. The request is Readable and the response is Writable.

**Q4. What is backpressure, and how does Node handle it?**
A slow consumer telling a fast producer to pause. A writable's `write()` returns `false` when its buffer exceeds `highWaterMark`; the producer should wait for `'drain'`. `pipe` and `pipeline` do this automatically.
*Follow-up:* Symptom of ignoring it? Memory grows until the process slows down or crashes during large transfers.

**Q5. Why prefer `pipeline` over `pipe`?**
`pipe` does not forward errors or clean up: a failing source leaves the destination open. `pipeline` (from `node:stream/promises`) destroys every stage on error, handles backpressure and returns a promise you can await in try/catch.
*Follow-up:* Can a pipeline stage be a function? Yes, an async generator that consumes the previous stage.

**Q6. How would you export two million rows as CSV without running out of memory?**
Stream it: fetch rows in batches with keyset pagination or a database cursor, turn each batch into CSV lines in an async generator, and `pipeline` it through gzip into the response. Memory stays at one batch and client disconnects stop the queries. For exports that take minutes, build the file in a background job and email or show a download link.
*Follow-up:* Why not OFFSET pagination? Deep offsets scan and discard rows, getting slower page by page.

**Q7. How do you verify a webhook signature?**
Compute an HMAC (usually SHA-256) of the raw request body with the shared secret, compare it with the header using `timingSafeEqual` after checking lengths, reject stale timestamps, and dedupe by event id because providers retry. Use the raw bytes, since re-serialized JSON differs.
*Follow-up:* Why constant-time comparison? `===` returns early at the first differing byte, which can leak the signature through timing.

**Q8. How should passwords be stored, and why not SHA-256?**
With a slow, salted key-derivation function: scrypt (built into `node:crypto`), argon2id or bcrypt. SHA-256 is designed to be fast, so leaked hashes can be brute-forced at billions of guesses per second.
*Follow-up:* Where should hashing run? Async APIs or a library that uses the thread pool, so logins do not block the event loop.

**Q9. Is `EventEmitter.emit` synchronous? What happens if nobody listens to `'error'`?**
Yes: listeners run synchronously in registration order before `emit` returns. An `'error'` event with no listener is thrown as an exception, which usually crashes the process.
*Follow-up:* What does `MaxListenersExceededWarning` usually mean? You add a listener per request and never remove it.

**Q10. Compare `spawn`, `exec`, `execFile` and `fork`.**
`spawn` starts a process without a shell and streams its output. `exec` runs a command string in a shell and buffers output (injection risk, 1 MB default buffer). `execFile` runs a binary with an argument array, no shell. `fork` starts a Node script with an IPC channel for messages.
*Follow-up:* Which for a large `pg_dump`? `spawn`, piping stdout to a file or S3 upload.

**Q11. `worker_threads` vs `child_process` vs `cluster`: when do you use each?**
Worker threads for CPU-heavy JavaScript inside one service (shared process, message passing). Child processes for running other programs or isolating risky work. Cluster for using every core on one machine with one port, though in containers you usually run more replicas instead.
*Follow-up:* Can workers share memory? Yes, through `SharedArrayBuffer` and `Atomics`, or by transferring `ArrayBuffer`s.

**Q12. Why does try/catch not catch an error thrown inside a callback, and how do you handle async errors?**
The callback runs later on a fresh call stack, after the try block has finished. Use promise APIs with `await` inside try/catch, check `err` in error-first callbacks, attach `'error'` listeners to emitters, and use `pipeline` for streams.
*Follow-up:* What happens to a rejected promise nobody awaits? It is an unhandled rejection, which crashes Node 15+ by default.

**Q13. What should `unhandledRejection` and `uncaughtException` handlers do?**
Log the error with as much context as possible, then exit with a non-zero code so the orchestrator starts a clean process. After an uncaught exception, in-memory state may be inconsistent, so continuing to serve risks corrupt data.
*Follow-up:* Then why register them at all? For structured logging and alerting before the process dies.

**Q14. What is the difference between operational and programmer errors?**
Operational errors are expected runtime conditions in correct code (invalid input, timeouts, a partner's 503) and are handled with 4xx responses, retries or fallbacks. Programmer errors are bugs (TypeErrors, broken invariants) and are fixed in code; crashing and restarting is acceptable.
*Follow-up:* How do they map to HTTP? Operational errors become meaningful 4xx/5xx codes; programmer errors become a generic 500 with details only in logs.

**Q15. How do you shut down a Node HTTP server gracefully?**
On SIGTERM: fail the readiness check, call `server.close()` to stop accepting connections, close idle keep-alive sockets, wait for in-flight requests, then close database pools, Redis and queue workers and exit. A timer forces exit before the platform's SIGKILL deadline.
*Follow-up:* Why might the process never receive SIGTERM in Docker? A shell or `npm start` as PID 1 not forwarding it; use the exec form `CMD ["node", ...]`.

**Q16. How do you add timeouts and cancellation to outbound HTTP calls?**
Pass a signal: `fetch(url, { signal: AbortSignal.timeout(5000) })`, combining it with a request-scoped signal through `AbortSignal.any`. Check `res.ok`, retry only idempotent calls on retryable errors with exponential backoff and jitter, and cap attempts.
*Follow-up:* Does `fetch` reject on a 500 response? No, only on network failures and aborts.

## Further reading
- [Node.js: Stream API](https://nodejs.org/api/stream.html)
- [Node.js: Backpressuring in streams](https://nodejs.org/en/learn/modules/backpressuring-in-streams)
- [Node.js: Crypto](https://nodejs.org/api/crypto.html)
- [Node.js: Errors](https://nodejs.org/api/errors.html)
- [Node.js: Process events](https://nodejs.org/api/process.html)
