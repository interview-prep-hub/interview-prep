# Firebase

> **Tier:** T3 (essentials) · **Related:** [Push notifications in Expo](../08-react-native-expo/05-push-notifications.md), [State, storage, auth and offline in React Native](../08-react-native-expo/04-state-storage-auth-offline.md), [Authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md), [MongoDB fundamentals](08-mongodb-fundamentals.md)
> **On your resume:** Firebase is in your skills list, and the Coloc resident app (Expo) ships push notifications: on Android, Expo's push service delivers through Firebase Cloud Messaging.

## Quick revision
- Firebase is Google's **backend-as-a-service**: Auth, Cloud Firestore, Realtime Database, Cloud Storage, Cloud Functions, Cloud Messaging (FCM), Hosting, plus Crashlytics, Analytics and Remote Config.
- **Firestore**: collections of documents (max 1 MiB each), documents can have subcollections; queries are shallow, index-backed, and have no joins, so you denormalize.
- Every query needs an index: single-field indexes are automatic, multi-field queries need **composite indexes** (the error message links to create one).
- **Real-time listeners** (`onSnapshot`) push changes; **offline persistence** caches data and queues writes (on by default in the mobile SDKs, opt-in on the web).
- **Realtime Database**: one big JSON tree, very low latency, simpler queries, billed by bandwidth. Firestore is the default for new apps.
- **Firebase Auth** issues short-lived **ID tokens** (JWTs, about an hour); backends verify them with the Admin SDK (`verifyIdToken`) or by checking the JWT against Google's public keys.
- **Security Rules** guard client access; they are **not filters**: a query that could return a forbidden document is rejected outright. The Admin SDK bypasses rules.
- **FCM** delivers pushes; Expo's push service sends Android notifications through FCM (you upload FCM credentials to Expo) and iOS through APNs.
- Firestore bills **per document read, write and delete**: chatty listeners, large result sets, offsets and rule `get()` calls all cost reads.
- Firebase gives speed to market and realtime for free; your own backend gives SQL, integrity, centralized logic, predictable cost and no lock-in.

## 1. Firebase as a BaaS

A backend-as-a-service lets client apps talk to managed services directly instead of to your own API: Firebase Auth signs users in, Firestore and the Realtime Database store data with realtime sync, Cloud Storage holds files, Cloud Functions run server-side code on events, and FCM delivers push messages. Security Rules decide what each client may touch. You trade control and portability for speed.

## 2. The Firestore data model

Paths alternate collection and document: `houses/{houseId}/tasks/{taskId}`. Documents hold typed fields (string, number, boolean, map, array, null, timestamp, geopoint, reference, bytes). Without joins, you choose between **subcollections** (one-to-many, queried per parent or across parents with collection group queries) and **duplicated fields** (store `assigneeName` next to `assigneeId` so a task list needs no second read). A single document should not be written more than about once per second sustained, so high-frequency counters use distributed (sharded) counters.

```ts
import { initializeApp } from "firebase/app";
import {
  getFirestore, collection, doc, addDoc, updateDoc, getDocs,
  query, where, orderBy, limit, serverTimestamp, increment, arrayUnion,
} from "firebase/firestore";

const app = initializeApp({ apiKey: "...", projectId: "coloc-demo" });  // config from the Firebase console
const db = getFirestore(app);

await addDoc(collection(db, "houses", houseId, "tasks"), {
  title: "Take out the bins",
  assigneeId: userId,
  done: false,
  createdAt: serverTimestamp(),             // set by the server, not the device clock
});

const openTasks = query(
  collection(db, "houses", houseId, "tasks"),
  where("done", "==", false),
  orderBy("createdAt", "desc"),
  limit(20)
);
const snap = await getDocs(openTasks);
const tasks = snap.docs.map((d) => ({ id: d.id, ...d.data() }));

await updateDoc(doc(db, "houses", houseId), { openTasks: increment(1), memberIds: arrayUnion(userId) });
```

Firestore also supports transactions (`runTransaction`) and atomic batched writes (`writeBatch`).

## 3. Queries and their limits

- Queries are **shallow**: reading a house does not read its `tasks` subcollection.
- Filters: `==`, `!=`, `<`, `<=`, `>`, `>=`, `in`, `not-in`, `array-contains`, `array-contains-any`; `in`-style filters accept a limited number of values.
- Historically, inequality filters were limited to a single field (and the first `orderBy` had to use it), and OR queries were not available; newer SDKs relax both. Check the current docs before designing around either limit.
- No joins and no native full-text search (teams use an extension that syncs to a search service such as Algolia or Typesense).
- Aggregations: newer SDKs offer server-side `count`, `sum` and `average` (for example `getCountFromServer`), cheaper than downloading documents.
- Paginate with cursors (`startAfter(lastDoc)`), not offsets.

## 4. Real-time listeners and offline persistence

```ts
import { onSnapshot } from "firebase/firestore";

const unsubscribe = onSnapshot(openTasks, (snapshot) => {
  snapshot.docChanges().forEach((change) => {
    console.log(change.type, change.doc.id);   // "added" | "modified" | "removed"
  });
});
// In React, return unsubscribe from the useEffect cleanup, or listeners pile up (and bill reads)
```

A listener first delivers all matching documents, then only changes. With **offline persistence**, the SDK serves cached data when the network drops, applies local writes immediately (snapshots report `hasPendingWrites`), and syncs queued writes on reconnect. The native Android and iOS SDKs (and React Native Firebase, which wraps them) enable it by default; the web SDK needs opting in, and that configuration API has changed between SDK versions, so check the docs for yours.

## 5. Composite indexes

A query such as `where("done", "==", false)` plus `orderBy("createdAt", "desc")` needs a composite index on `(done, createdAt desc)`. Missing indexes fail at runtime with an error containing a console link that creates the index. Keep indexes in source control in `firestore.indexes.json` and deploy them with the Firebase CLI (`firebase deploy --only firestore:indexes`), so every environment matches.

## 6. Realtime Database vs Firestore

| | Realtime Database | Cloud Firestore |
|---|---|---|
| Data model | One large JSON tree | Collections, documents, subcollections |
| Queries | One sort or filter per query; reading a node returns its whole subtree | Compound, index-backed, shallow queries |
| Offline | Mobile SDKs | Mobile and web SDKs |
| Scaling | Per-database connection and throughput limits; shard across several databases | Scales automatically; regional and multi-region locations |
| Pricing | Mostly bandwidth and storage | Per document operation plus storage |
| Best for | Presence ("who is online"), live cursors, small rapidly changing state | Most application data |

## 7. Firebase Auth and verifying ID tokens on a backend

The client signs in with the Firebase SDK and receives an **ID token**: a JWT signed by Google that expires after about an hour (the SDK refreshes it with a refresh token). Send it to your API as `Authorization: Bearer <token>` and verify it on every request.

```ts
import { initializeApp, applicationDefault } from "firebase-admin/app";
import { getAuth } from "firebase-admin/auth";

initializeApp({ credential: applicationDefault() });

export async function requireFirebaseUser(authorization?: string) {
  const token = authorization?.startsWith("Bearer ") ? authorization.slice(7) : null;
  if (!token) throw new Error("Missing bearer token");
  const decoded = await getAuth().verifyIdToken(token);   // signature, expiry, audience, issuer
  return { uid: decoded.uid, email: decoded.email, role: decoded.role as string | undefined };
}
```

- Roles: set **custom claims** server-side (`getAuth().setCustomUserClaims(uid, { role: "admin" })`); they appear in the user's next refreshed token.
- Revocation: `verifyIdToken(token, true)` also checks whether the user's tokens were revoked (an extra lookup).
- **Rails or other stacks without an official Admin SDK**: verify the JWT yourself with a JWT library: RS256 signature against Google's published certificates for Firebase Auth (cache them as their HTTP cache headers allow), `aud` equal to your project id, `iss` equal to `https://securetoken.google.com/<project-id>`, a future `exp`, and a non-empty `sub` (the user's uid). Your own `users` table can then be keyed by the Firebase uid.

## 8. Cloud Messaging (FCM) and Expo push

FCM routes push notifications and data messages to devices: directly on Android, through APNs on iOS, and to browsers. With **Expo**:
- The app obtains an **Expo push token** and your server sends notifications to Expo's push service, which delivers through **FCM on Android** and APNs on iOS. That is why Android setup asks for a `google-services.json` in the app config and FCM (HTTP v1) service account credentials uploaded to Expo; Google retired the legacy FCM send API in favor of HTTP v1.
- Alternatively, get the native device token (`getDevicePushTokenAsync`) and send through FCM yourself with the Admin SDK (`getMessaging().send({ token, notification, data })`).
- Notification messages are shown by the OS when the app is in the background; data messages are handled by your code.
- Recent Expo SDKs require a development build (not Expo Go) for push notifications on Android; check the current Expo docs. Details in [push notifications](../08-react-native-expo/05-push-notifications.md).

## 9. Storage and Cloud Functions

**Cloud Storage for Firebase** lets clients upload files directly to a Google Cloud Storage bucket, with resumable uploads and access controlled by Storage Security Rules; `getDownloadURL` returns a shareable URL. A Rails backend would more often use Active Storage with S3 or GCS.

**Cloud Functions** run server code on events: HTTPS requests, callable functions, Firestore document changes, Auth events, Storage uploads, schedules.

```ts
import { initializeApp } from "firebase-admin/app";
import { getFirestore, FieldValue } from "firebase-admin/firestore";
import { onDocumentCreated } from "firebase-functions/v2/firestore";

initializeApp();

export const onTaskCreated = onDocumentCreated("houses/{houseId}/tasks/{taskId}", async (event) => {
  await getFirestore()
    .doc(`houses/${event.params.houseId}`)
    .update({ openTasks: FieldValue.increment(1) });
});
```
Event triggers can be delivered more than once, so make handlers idempotent; expect cold starts on infrequently used functions.

## 10. Security Rules

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, update: if request.auth != null && request.auth.uid == userId;
    }

    match /houses/{houseId}/tasks/{taskId} {
      function isMember() {
        return request.auth != null &&
          request.auth.uid in get(/databases/$(database)/documents/houses/$(houseId)).data.memberIds;
      }
      allow read: if isMember();
      allow create: if isMember()
        && request.resource.data.title is string
        && request.resource.data.title.size() <= 200
        && request.resource.data.done == false;
      allow update: if isMember()
        && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['done']);
    }
  }
}
```

- `request.auth` is the signed-in user; `resource.data` is the stored document; `request.resource.data` is the document as it would be after the write.
- **Rules are not filters**: a query for all tasks in a house the user does not belong to fails entirely instead of returning an empty list; the query itself must be constrained to allowed documents.
- Never ship the default test-mode rules (open access until a date). Test rules with the Firebase Emulator Suite.
- Code using the Admin SDK (Cloud Functions, your server) bypasses rules entirely, so validate there too.

## 11. Pricing gotchas
- Every document returned by a query or delivered by a listener is a billed read; a busy collection with many listening clients multiplies reads quickly, and a listener reconnecting after a long disconnection (30 minutes or more) is billed like a fresh query.
- `useEffect` bugs that re-subscribe on every render (or never unsubscribe) create read storms.
- Offsets still bill the skipped documents; use cursors.
- `get()` and `exists()` calls inside rules bill reads; aggregation queries bill in batches of index entries.
- The free tier has daily quotas, and budget alerts notify but do not cap spending. Check current pricing before estimating.
- Cut costs by denormalizing (fewer reads per screen), storing counters instead of counting documents, paginating, and listening only to what is on screen.

## 12. Firebase vs your own backend

| | Firebase | Your own backend (Rails or Node with Postgres) |
|---|---|---|
| Time to first version | Very fast: auth, database, storage, push in an afternoon | Slower: you build and host the API |
| Realtime and offline | Built in | Build it (Action Cable or WebSockets, local caching) |
| Queries and reporting | Constrained queries, no joins | Full SQL, analytics, ad hoc reports |
| Business logic | Split across clients, rules and functions | Centralized and testable on the server |
| Integrity | Rules plus app code | Constraints, foreign keys, transactions |
| Cost at scale | Per operation; can spike with usage patterns | Infrastructure-based, more predictable |
| Lock-in | High (proprietary APIs and data model) | Low |

Many products mix them: their own API and Postgres for core data, plus Firebase for FCM, Crashlytics, Analytics, Remote Config or Auth.

## Common pitfalls
- Shipping test-mode or overly broad security rules.
- Designing Firestore like SQL (normalized collections joined on the client), causing many reads per screen.
- Hot documents written many times per second (global counters).
- Trusting client-sent user ids instead of the verified token's `uid` on the backend.
- Missing composite indexes discovered only in production because the emulator or development data never hit that query.
- Calling the legacy FCM API or forgetting to configure FCM credentials for Android Expo builds.

## How to talk about this in an interview
- Your strongest Firebase link is push notifications in the Coloc resident app: explain how Expo's push service relies on FCM for Android and why the FCM credentials and `google-services.json` matter.
- For the rest, be honest: Firebase is in your skills list; describe what you built or explored with it, and frame Firestore through the document-modeling ideas you know from MongoDB.
- Show judgment on Firebase vs a custom backend: the Devntech products you work on (rental applications, rent payments, multi-store catalogs) are relational and run on Rails and Postgres, a good example of when a custom backend fits.

> **Fill in:** what you have built with Firebase (a university project, a prototype, push setup for the Coloc app) and which products you used (Auth, Firestore, FCM, Storage).

> **Fill in:** how push notifications are sent in the Coloc app: Expo push tokens through Expo's service, or FCM directly, and from where (the Rails backend or a job).

## Interview questions

**Q1. What is Firebase, and when would you use it?**
A Google backend-as-a-service providing auth, a realtime document database (Firestore), file storage, serverless functions and push messaging, usable directly from client apps. Use it to ship an MVP fast, for realtime or offline-heavy mobile apps, or for specific services (FCM, Crashlytics, Auth) alongside your own backend. Avoid it as the core store when you need complex queries, reporting or strict relational integrity.

**Q2. What is the difference between Firestore and the Realtime Database?**
Realtime Database stores one JSON tree with simple queries and bandwidth-based pricing, great for presence and small fast-changing state. Firestore stores documents in collections with compound, index-backed queries, offline support on web and mobile, automatic scaling and per-operation pricing. Firestore is the default for new apps.

**Q3. How do you model data in Firestore without joins?**
Model around screens: put what one screen needs into one or a few documents, duplicate small fields (names, avatars) instead of joining, use subcollections for one-to-many data (house tasks), collection group queries across them, and store counters instead of counting documents. Accept that duplicated fields need update logic (often in a Cloud Function).

**Q4. Why don't Security Rules work as filters?**
Rules are evaluated against the query as a whole: Firestore must be able to prove every possible result is allowed, otherwise the entire query is rejected. So a query must include constraints matching the rules (for example `where("memberIds", "array-contains", uid)`) rather than relying on rules to drop forbidden documents.

**Q5. How do you verify a Firebase user on your backend?**
The client sends its ID token as a bearer token. In Node, call `getAuth().verifyIdToken(token)` from the Admin SDK, which checks the signature, expiry, audience and issuer, then use `decoded.uid` and custom claims. In Rails, verify the JWT with a library: RS256 against Google's public certificates, `aud` equal to the project id, `iss` equal to `https://securetoken.google.com/<project-id>`, a valid `exp`. Never trust a uid sent in the request body.

**Q6. Why can Firestore bills surprise you, and how do you control them?**
Billing is per document read, write and delete, so many clients listening to busy collections, large result sets, offsets, re-subscribing listeners and rule `get()` calls multiply reads. Control it with denormalized documents (fewer reads per screen), cursor pagination, narrow listeners with cleanup, stored counters, caching, and budget alerts.

**Q7. What role does FCM play when an Expo app sends push notifications?**
Expo's push service is a relay: for Android devices it delivers through FCM (using the FCM v1 credentials you upload), and for iOS through APNs. You can also skip the relay and send with FCM directly using native device tokens and the Admin SDK.

**Q8. How does Firestore offline persistence work?**
The SDK keeps a local cache of documents you have read and queues writes made offline. Reads come from the cache when offline, local writes appear immediately with pending-write metadata, and the queue syncs on reconnect. It is on by default in the native mobile SDKs and opt-in on the web.

**Q9. Would you choose Firebase or your own backend for a new product?**
Firebase for an MVP, a realtime collaboration feature or a small team without backend capacity. A custom backend (Rails or Node with Postgres) when the domain is relational (bookings, payments, inventory), needs reporting, complex authorization or predictable costs. Often a mix: own API for core data, Firebase for push, crash reporting and possibly auth.

## Further reading
- Cloud Firestore: https://firebase.google.com/docs/firestore
- Verify ID tokens: https://firebase.google.com/docs/auth/admin/verify-id-tokens
- Security Rules: https://firebase.google.com/docs/rules
- Expo push notifications: https://docs.expo.dev/push-notifications/overview/
