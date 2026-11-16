# GraphQL

> **Tier:** T2 (working knowledge) · **Related:** [REST API design](04-rest-api-design.md), [Authentication and authorization](06-authentication-authorization.md), [Caching and performance](08-caching-and-performance.md), [N+1 queries in ActiveRecord](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md), [Data fetching with TanStack Query](../06-react/04-data-fetching-tanstack-query.md), [oRPC and type-safe APIs](../09-nodejs-backend/05-orpc-and-typesafe-apis.md)
> **On your resume:** GraphQL is in your skills list without a named project, so expect conceptual questions and comparisons with the REST and oRPC APIs you have built.

## Quick revision
- GraphQL is a query language and runtime for APIs: one endpoint, a strongly typed schema, and clients ask for exactly the fields they need.
- Types: scalars (`Int`, `Float`, `String`, `Boolean`, `ID`, custom ones like `DateTime`), object types, input types, enums, interfaces (shared fields), unions (one of several types), lists `[T]` and non-null `T!`.
- Operations: queries (read), mutations (write; top-level fields run in order), subscriptions (server pushes events, usually over WebSockets).
- Resolvers are per-field functions `(parent, args, context, info)`; the context carries the current user, data loaders and DB access for one request.
- **N+1:** resolving a field for each item of a list fires one query per item. **DataLoader** collects the keys requested in one tick, loads them in one batched query and caches them for the request.
- Pagination: Relay-style connections with `edges { cursor node }`, `pageInfo { hasNextPage endCursor }` and `first`/`after` arguments.
- Errors: usually HTTP 200 with an `errors` array next to partial `data`; expected validation errors are modeled as data in mutation payloads (`userErrors`).
- Authenticate at the HTTP layer into the context; authorize per object and field in resolvers or, better, in a shared business layer.
- Caching is harder than REST: POST to one URL defeats HTTP caches, so clients use normalized caches (Apollo `InMemoryCache`); **persisted queries** send a hash instead of query text, enabling GET plus CDN caching and an allowlist of known operations.
- No versions by convention: add fields freely, mark old ones `@deprecated(reason: "...")`, track usage, remove later.
- Security: depth and complexity limits, max page sizes, timeouts, cost-based rate limits, introspection restricted in production for private APIs.
- graphql-ruby: type classes with `field`, resolver methods, mutation classes, connection types and `GraphQL::Dataloader`. In JavaScript: Apollo Server and Client, plus GraphQL Code Generator for TypeScript types.
- GraphQL suits many clients with different data needs over a rich graph; REST suits CRUD, public integrations, webhooks, uploads and HTTP caching.

## 1. What GraphQL is and the problem it solves

A REST screen often needs several requests (`/houses/12`, `/houses/12/bills`, `/users/...`), each returning more fields than the screen uses (over-fetching) or not enough related data (under-fetching, so more round trips). GraphQL lets the client describe the exact shape it needs in one request:

```text
query HouseBills($houseId: ID!) {
  house(id: $houseId) {
    name
    openBills: bills(status: OPEN, first: 10) {      # "openBills" is an alias
      edges { node { ...BillFields } }
      pageInfo { hasNextPage endCursor }
    }
  }
}

fragment BillFields on Bill { id title amountCents payer { name } }
```

The client sends `POST /graphql` with JSON `{"query": "...", "variables": {"houseId": "12"}, "operationName": "HouseBills"}` and gets back data in the same shape:

```json
{ "data": { "house": { "name": "House 12", "openBills": {
  "edges": [{ "node": { "id": "981", "title": "Internet", "amountCents": 4500, "payer": { "name": "Sara" } } }],
  "pageInfo": { "hasNextPage": false, "endCursor": "Y3Vyc29yOjk4MQ==" } } } } }
```

The schema is typed and introspectable, so tools such as GraphiQL autocomplete queries and code generators produce client types.

## 2. Schema and types

```text
scalar DateTime
enum BillStatus { OPEN PAID OVERDUE }

interface Node { id: ID! }

type House implements Node {
  id: ID!
  name: String!
  bills(status: BillStatus, first: Int = 20, after: String): BillConnection!
}

type Bill implements Node {
  id: ID!
  title: String!
  amountCents: Int!
  status: BillStatus!
  dueAt: DateTime
  payer: User               # nullable: the payer may have left the house
}

type User implements Node { id: ID!  name: String! }

union SearchResult = House | Bill | User

input CreateBillInput { houseId: ID!  title: String!  amountCents: Int! }
type UserError { field: String  message: String! }
type CreateBillPayload { bill: Bill  userErrors: [UserError!]! }

type Query {
  house(id: ID!): House
  search(term: String!): [SearchResult!]!
}
type Mutation { createBill(input: CreateBillInput!): CreateBillPayload! }
type Subscription { billCreated(houseId: ID!): Bill! }
```

- `!` means non-null; `[Bill!]!` is a non-null list of non-null bills. Nullability is part of the contract, and it controls how errors propagate (section 7).
- **Input types** are the only object types allowed as arguments; output types cannot be used as inputs.
- **Interfaces** declare fields that implementing types must have, so you can query `id` on any `Node`. **Unions** group types with no required shared fields, so clients use inline fragments: `search(term: "zelda") { ... on Bill { title } ... on House { name } }`.
- `ID` is serialized as a string, even when the database uses integers.

## 3. Queries, mutations and subscriptions

```text
mutation CreateBill($input: CreateBillInput!) {
  createBill(input: $input) {
    bill { id title amountCents }
    userErrors { field message }
  }
}
```

- **Queries** read; their fields may resolve in parallel.
- **Mutations** write. Top-level mutation fields execute one after another, and the payload returns the changed objects so client caches can update without refetching.
- **Subscriptions** keep a connection open and push results when an event fires. They usually run over WebSockets with the graphql-ws protocol (the older subscriptions-transport-ws protocol is deprecated) or over SSE; graphql-ruby can deliver them through Action Cable. Many teams use plain queries plus polling or a realtime channel instead, because subscriptions add infrastructure ([Realtime and WebSockets](09-realtime-websockets.md)).

## 4. Resolvers and context

Every field has a resolver: a function that receives the **parent** object (the result of the field above), the field's **arguments**, a per-request **context** and **info** (the query AST, useful for look-ahead optimizations). Fields without a custom resolver use a default one that reads the property with the same name from the parent. Execution walks the query top-down: `Query.house` returns a house, then `House.bills` runs with that house as its parent, then `Bill.payer` runs once per bill.

The context is built once per request: authenticate the request (cookie or bearer token), attach `currentUser`, create fresh DataLoaders, and pass database or service handles. Resolvers should stay thin and call the same services and policies your REST controllers use.

## 5. The N+1 problem and DataLoader

For 50 bills with `payer { name }`, a naive `Bill.payer` resolver runs one query for the bills and then one user query per bill: 51 queries. In REST you would fix it with `includes(:payer)`, but a GraphQL server does not know in advance which nested fields a client will ask for. **DataLoader** fixes it generically:

```ts
import DataLoader from "dataloader";
import { inArray } from "drizzle-orm";

export function createLoaders(db: Database) {
  return {
    user: new DataLoader<string, User | null>(async (ids) => {
      const rows = await db.select().from(users).where(inArray(users.id, [...ids])); // one query for all ids
      const byId = new Map(rows.map((u) => [u.id, u]));
      return ids.map((id) => byId.get(id) ?? null); // same length and order as the keys
    }),
  };
}

// resolver: Bill.payer = (bill, _args, ctx) => (bill.payerId ? ctx.loaders.user.load(bill.payerId) : null)
```

Every `load(id)` call made during the same tick of the event loop is queued; DataLoader then calls the batch function once with all unique keys and caches each result for the rest of the request. Create loaders per request in the context factory so the cache never leaks data between users. graphql-ruby has the same idea built in:

```ruby
class Sources::RecordById < GraphQL::Dataloader::Source
  def initialize(model_class)
    @model_class = model_class
  end

  def fetch(ids)
    records = @model_class.where(id: ids).index_by(&:id)
    ids.map { |id| records[id] } # results in the same order as ids
  end
end

# in Types::BillType (the schema must `use GraphQL::Dataloader`)
def payer
  object.payer_id && dataloader.with(Sources::RecordById, User).load(object.payer_id)
end
```

Recent graphql-ruby versions also ship ready-made ActiveRecord sources; check the docs for your version. Where the query shape is predictable, preloading in the root resolver (with look-ahead) also works.

## 6. Pagination with connections

```text
type BillConnection { edges: [BillEdge!]!  pageInfo: PageInfo! }
type BillEdge { cursor: String!  node: Bill! }
type PageInfo { hasNextPage: Boolean!  hasPreviousPage: Boolean!  startCursor: String  endCursor: String }
```

The Relay connection convention is cursor pagination expressed in the schema: request `bills(first: 10, after: $endCursor)`, render `edges[].node`, and pass `pageInfo.endCursor` back for the next page. Cursors are opaque and usually encode the keyset values, so the same trade-offs as REST apply ([REST API design](04-rest-api-design.md)). graphql-ruby generates these types with `Types::BillType.connection_type` and paginates ActiveRecord relations automatically; always set a maximum page size. Simple `limit`/`offset` arguments also work for small admin lists.

## 7. Error handling

```json
{
  "data": { "house": null },
  "errors": [{ "message": "Not authorized to view this house", "path": ["house"],
               "locations": [{ "line": 2, "column": 3 }], "extensions": { "code": "FORBIDDEN" } }]
}
```

- Responses can be partially successful: some fields fail while others return data.
- When a **non-null** field errors, null propagates up to the nearest nullable parent, so marking everything non-null can blank out a whole response. Make fields that can fail independently nullable.
- Put unexpected and authorization errors in `errors` with an `extensions.code` (conventional codes include `UNAUTHENTICATED`, `FORBIDDEN` and `BAD_USER_INPUT`). Model expected domain errors, such as validation failures, as data in the mutation payload (`userErrors`), so clients get typed results.
- HTTP status is traditionally 200 even when `errors` is present. Under the GraphQL over HTTP spec with the `application/graphql-response+json` media type, requests that fail before execution (unparseable or invalid queries) get 4xx codes. Either way, monitoring must look inside response bodies, not just at status codes.

## 8. Authentication and authorization

- **Authenticate** once per request while building the context, exactly as a REST API would (session cookie or bearer token), and attach `currentUser`.
- **Authorize** every object and sensitive field, not only root fields: `house { bills { payer { email } } }` reaches users through relationships, and generic `node(id:)` fields bypass root-level checks.
- Keep rules in one place that REST and GraphQL share: Pundit-style policies in Rails, a permissions module in TypeScript. graphql-ruby also offers hooks such as `self.authorized?(object, context)` on types (returning false makes the object resolve to null by default) and `authorized?` on mutations.
- In multi-tenant apps, scope every lookup to the current organization ([Authentication and authorization](06-authentication-authorization.md)).

## 9. Caching and persisted queries

- **HTTP caching** mostly disappears: every operation is a POST to `/graphql`, so browsers and CDNs cannot cache by URL.
- **Client caches** are normalized: Apollo `InMemoryCache` (and Relay or urql's graphcache) store each object once under `__typename` plus `id`, so a mutation that returns the updated bill refreshes every view showing it.
- **Server side:** per-request DataLoader caching, application caching inside resolvers (Redis) and, carefully, response caching keyed by operation plus variables (never across users for private data).
- **Persisted queries:** the client sends a hash that the server maps to a stored query. Automatic persisted queries (APQ) register unknown hashes on first use and mainly shrink requests; **trusted documents** (an allowlist extracted from client code at build time) also block arbitrary queries in production. Both make GET requests, and therefore CDN caching, practical.

## 10. Schema evolution and deprecation

```text
type Bill {
  amount: Float @deprecated(reason: "Use amountCents (integer minor units).")
  amountCents: Int!
}
```

GraphQL APIs usually evolve one schema instead of versioning. Adding types, fields and optional arguments is safe; removing or renaming fields, changing types, making an argument required or making a non-null output field nullable breaks clients. The process: add the replacement, deprecate the old field (tools show it struck through), measure field usage per client (schema registries and analytics do this), and remove it only when nobody queries it. Installed mobile app versions make that last step slow, exactly as with REST.

## 11. Security

- **Depth limits** stop recursive queries such as `house { members { houses { members { ... } } } }`; **complexity or cost analysis** weights each field, multiplies by requested list sizes and rejects queries over a budget. Rate limit by cost, not just request count.
- **Aliases and batching** let one request run many operations (`a1: login(...) a2: login(...)`), so brute-force protection must count operations, not HTTP requests.
- **Max page sizes, timeouts and request size limits** cap the damage of any single query; authorize at every node and keep internals out of error messages.
- **Introspection:** restrict or disable it in production for private APIs, but never treat hiding the schema as the security boundary. Trusted documents allow only known operations.

## 12. graphql-ruby basics

`bundle add graphql` and `bin/rails generate graphql:install` create `app/graphql` (schema, types, mutations) and a `GraphqlController#execute` that passes `context: { current_user: current_user }`.

```ruby
class AppSchema < GraphQL::Schema
  query Types::QueryType
  mutation Types::MutationType
  use GraphQL::Dataloader

  max_depth 12
  max_complexity 300
  default_max_page_size 50
  disable_introspection_entry_points if Rails.env.production?
end

module Types
  class HouseType < Types::BaseObject
    field :id, ID, null: false
    field :name, String, null: false
    field :bills, Types::BillType.connection_type, null: false # paginated automatically

    def self.authorized?(object, context)
      super && context[:current_user]&.member_of?(object)
    end
  end
end

module Mutations
  class CreateBill < Mutations::BaseMutation
    argument :house_id, ID, required: true
    argument :title, String, required: true
    argument :amount_cents, Integer, required: true

    field :bill, Types::BillType, null: true
    field :user_errors, [Types::UserErrorType], null: false

    def resolve(house_id:, title:, amount_cents:)
      house = context[:current_user].houses.find(house_id)
      bill = house.bills.new(title: title, amount_cents: amount_cents)
      return { bill: bill, user_errors: [] } if bill.save

      { bill: nil, user_errors: bill.errors.map { |e| { field: e.attribute.to_s, message: e.message } } }
    end
  end
end
```

- snake_case Ruby names become camelCase in the schema (`amount_cents` is exposed as `amountCents`).
- Inside a type, `object` is the parent record and `context` is the per-request hash; a method named like the field acts as its resolver.
- Test by executing operations directly: `AppSchema.execute(query, variables: { "houseId" => house.id }, context: { current_user: user })`. `AppSchema.to_definition` prints the SDL, which you can commit and diff in CI to catch breaking changes.

## 13. Apollo Server and Apollo Client basics

```ts
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";

// typeDefs: the SDL as a string; resolvers: { Query: { bills: (_parent, args, ctx) => ctx.services.billsFor(ctx.user, args.houseId) } }
const server = new ApolloServer<Context>({ typeDefs, resolvers });
const { url } = await startStandaloneServer(server, {
  context: async ({ req }) => buildContext(req.headers.authorization), // user plus fresh DataLoaders per request
  listen: { port: 4000 },
});
```

```tsx
import { ApolloClient, HttpLink, InMemoryCache, gql } from "@apollo/client";
import { useQuery } from "@apollo/client/react";

export const client = new ApolloClient({ link: new HttpLink({ uri: "/graphql" }), cache: new InMemoryCache() });

const BILLS = gql`
  query Bills($houseId: ID!) { bills(houseId: $houseId) { id title amountCents } }
`;
type BillsData = { bills: { id: string; title: string; amountCents: number }[] };

export function BillList({ houseId }: { houseId: string }) {
  const { data, loading, error } = useQuery<BillsData>(BILLS, { variables: { houseId } });
  if (loading) return <p>Loading...</p>;
  if (error) return <p>Could not load bills.</p>;
  return <ul>{data?.bills.map((b) => <li key={b.id}>{b.title}</li>)}</ul>;
}
```

The app is wrapped in `<ApolloProvider client={client}>`. This is the Apollo Server 4+ package layout; production servers usually mount Apollo inside an HTTP framework through an integration rather than the standalone server. Apollo Client 4 serves React APIs from `@apollo/client/react` (version 3 also re-exported them from `@apollo/client`), so check the version in your project. Lighter alternatives are urql, or graphql-request combined with TanStack Query.

## 14. TypeScript code generation

```ts
// codegen.ts
import type { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
  schema: "http://localhost:3000/graphql",
  documents: ["src/**/*.{ts,tsx}"],
  generates: {
    "./src/gql/": { preset: "client" }, // typed graphql() function and typed documents for every operation
  },
};
export default config;
```

Run `npx graphql-codegen` (or with `--watch`). With the client preset you write operations through the generated `graphql()` function, and `useQuery` infers both result and variables types, so a renamed field breaks the build instead of production. On the server, the `typescript-resolvers` plugin types resolver signatures. This is how GraphQL gets the end-to-end safety that oRPC or tRPC give a TypeScript monorepo through inference.

## 15. GraphQL vs REST

| | GraphQL | REST |
|---|---|---|
| Shape of responses | defined by the client per query | defined by the server per endpoint |
| Data for one screen | one request, exact fields | several requests or a bespoke endpoint; over-fetching |
| Types and docs | schema and introspection built in | OpenAPI, optional |
| Caching | normalized client caches; HTTP caching is hard | HTTP and CDN caching are natural |
| Evolution | deprecate fields in one schema | often URI or header versions |
| Errors | `errors` array, usually HTTP 200 | HTTP status codes |
| Performance risks | N+1 in resolvers, expensive client queries | over-fetching, chatty clients |
| File uploads | awkward (multipart extension or presigned URLs) | natural |
| Best fit | many clients with different needs, rich graphs, aggregating services | CRUD, public APIs, webhooks, simple mobile backends |

## Common pitfalls
- Resolvers that query the database per item with no DataLoader (N+1), or DataLoaders shared across requests, leaking cached data between users.
- Authorization only on root fields, so nested relationships expose data.
- Everything non-null, so one failing field nulls the whole response.
- No depth, complexity or page-size limits on a public endpoint.
- Treating HTTP 200 as success in monitoring while `errors` is full.
- Removing fields without deprecation and usage tracking; old mobile apps break.
- Business logic inside resolvers instead of shared services, duplicating REST code paths.

## How to talk about this in an interview
- Be upfront: GraphQL is in your skills list, but your production APIs on the resume are REST (Rails, DRF, Next.js) and oRPC. Then show you understand the trade-offs and the classic problems: N+1, authorization at every node, caching and query cost.
- Bridge from what you know: the GraphQL N+1 problem is the same one you fix in Rails with `includes`, and DataLoader is per-request batching. Connections are keyset pagination in schema form.
- When asked "REST or GraphQL for this project?", answer from the clients' needs: one web app plus one mobile app with stable screens often fits REST or a typed RPC layer; many clients with diverse data needs over a rich graph fit GraphQL.

> **Fill in:** where you used GraphQL (personal project, coursework, a client codebase), which server and client libraries, and one problem you solved (N+1, pagination, auth). If you have not used it hands-on, say how you learned it.

## Interview questions

**Q1. What is GraphQL, and what problems does it solve?**
A query language and runtime where the server publishes a typed schema and clients request exactly the fields they need from a single endpoint. It addresses over-fetching, under-fetching and multiple round trips per screen, and gives typed, introspectable APIs.
*Follow-up:* What does it cost? Resolver complexity, N+1 risks, harder HTTP caching and the need for query cost limits.

**Q2. GraphQL vs REST: when would you pick each?**
GraphQL when many clients need different shapes of related data, or when aggregating several backends behind one graph. REST for CRUD resources, public APIs and integrations, webhooks, file uploads and anything that benefits from HTTP and CDN caching. Small teams with one or two clients often move faster with REST or a typed RPC layer.
*Follow-up:* Can you mix them? Yes: REST for webhooks and uploads, GraphQL for app screens is common.

**Q3. Explain the N+1 problem in GraphQL and how DataLoader fixes it.**
Nested field resolvers run once per parent, so a list of 50 bills with payers triggers 50 user queries. DataLoader queues every `load(key)` call made in one tick, calls a batch function once with all keys (one `WHERE id IN (...)` query), returns results in key order and caches them for the request.
*Follow-up:* Why per-request loaders? The cache holds data loaded with one user's permissions; sharing it across requests leaks data and serves stale values.

**Q4. Queries vs mutations vs subscriptions?**
Queries read and may resolve fields in parallel. Mutations write; their top-level fields run serially and return payloads with the changed objects. Subscriptions hold a connection (usually WebSockets with graphql-ws) and push results when events occur.
*Follow-up:* Alternatives to subscriptions? Polling, refetching on push notifications, or a separate realtime channel.

**Q5. How do you paginate in GraphQL?**
Relay-style connections: `first`/`after` arguments, `edges` with `cursor` and `node`, and `pageInfo` with `hasNextPage` and `endCursor`. Cursors are opaque keyset positions, so pages stay fast and stable; set a maximum page size.
*Follow-up:* Does graphql-ruby help? `connection_type` builds the types and paginates relations automatically.

**Q6. How does error handling work, and why do GraphQL APIs often return 200 for errors?**
A response can contain partial `data` plus an `errors` array with message, path and `extensions.code`; because one request can partly succeed, the transport status is traditionally 200. Expected domain errors are better modeled as typed `userErrors` in mutation payloads. Non-null fields that fail null out their nearest nullable parent.
*Follow-up:* What does that mean for monitoring? Alert on `errors` in bodies and on resolver exceptions, not just HTTP status codes.

**Q7. Where do authentication and authorization happen?**
Authentication happens once per request when building the context (cookie or token). Authorization happens for every object and sensitive field, ideally through shared policies used by both REST and GraphQL, because nested relationships and `node(id:)` lookups can reach data without passing root fields.
*Follow-up:* What happens in graphql-ruby when `authorized?` returns false? The object resolves to null by default, and the behavior can be customized.

**Q8. Why is caching harder in GraphQL, and what are persisted queries?**
Operations are POSTs to one URL with arbitrary bodies, so HTTP caches cannot key on them. Clients rely on normalized caches keyed by type and ID; servers use DataLoader and resolver-level caching. Persisted queries replace the query text with a hash the server already knows, enabling GET requests, CDN caching, smaller payloads and, with trusted documents, an allowlist.
*Follow-up:* APQ vs trusted documents? APQ registers any query on first use (performance); trusted documents only accept operations extracted at build time (security).

**Q9. How do you evolve a schema without versions?**
Add new fields and types (safe), deprecate old fields with `@deprecated(reason:)`, track which clients still query them, and remove them once unused. Avoid breaking changes such as renames, type changes or new required arguments.
*Follow-up:* Why is removal slow for mobile apps? Old app versions keep sending old queries for months.

**Q10. How do you protect a GraphQL API from expensive or malicious queries?**
Depth limits, complexity or cost analysis, maximum page sizes, timeouts, request size limits, cost-based rate limiting, brute-force protection that counts aliased operations, introspection restricted in production for private APIs, and trusted documents where possible.
*Follow-up:* Example of an abusive query? Deeply nested cyclic relationships, or hundreds of aliases calling an expensive field.

**Q11. How do you get type safety from the schema to React components?**
GraphQL Code Generator reads the schema and the operations in your code and generates TypeScript types and typed documents (the client preset), so `useQuery` knows the result and variables types and schema changes break the build. Server resolvers can be typed with the resolvers plugin.
*Follow-up:* How does that compare to oRPC? oRPC infers types directly from TypeScript procedures without code generation, but only for TypeScript clients.

## Further reading
- [GraphQL: Learn](https://graphql.org/learn/)
- [graphql-ruby guides](https://graphql-ruby.org/)
- [Apollo documentation](https://www.apollographql.com/docs/)
- [GraphQL Code Generator](https://the-guild.dev/graphql/codegen) and [DataLoader](https://github.com/graphql/dataloader)
