# Scoped Realtime Notification API Access for Video Rooms Without Socket Server Trust

TL;DR: Issue a short-lived token for one order room and one role, keep signing and notification publishing on the server, and treat every room identifier supplied by a browser as hostile input. A managed realtime API can remove socket operations from your workload. It cannot remove authorization design. For a buyer-seller video session, token scope is the decision that matters first; price and SDK polish come later.

This is the constraint that changes the build: the browser needs enough authority to join a call and receive updates, but never enough authority to select another order, mint a new identity, or publish server-trusted events. WebRTC covers peer connection behavior, not application authorization. The W3C specification explicitly leaves signaling outside its scope, so the application still owns the channel that exchanges session descriptions and ICE candidates.

## What should the browser be allowed to trust?

Very little. The client may present an authenticated session and an order ID. The server must derive the user ID and role from its own session, load the order, verify that the user is a participant, and construct the room name itself. Do not accept a client-provided channel pattern, role, or publish permission.

For an order such as `ord_72KQ`, separate capabilities instead of handing both parties a broad token. A buyer can subscribe to session state and send narrowly defined signaling messages for that room. A seller gets the same room boundary with a different subject. System events such as `payment_confirmed` or `session_closed` originate on a trusted backend path. They are not browser publish permissions disguised as convenience.

The distinction is easy to miss because the first demo works either way. Then someone changes `ord_72KQ` to another valid identifier in DevTools. If the token authorizes a wildcard or the publish endpoint trusts that string, the room boundary never existed.

No wildcard.

**The room ID is an authorization claim, not routing metadata.**

## The smallest implementation I would ship

Keep the adapter boring. A small interface prevents the rest of the application from depending on a provider's token vocabulary, while the authorization policy stays visible in application code. This example intentionally omits provider-specific claims. `TokenSigner` maps these generic capabilities to the selected service.

```ts
interface SessionUser {
  id: string;
}

type RoomRole = "buyer" | "seller";

interface OrderParticipant {
  userId: string;
  role: RoomRole;
}

interface RoomGrant {
  subject: string;
  room: string;
  role: RoomRole;
  capabilities: readonly ["subscribe", "signal"];
  expiresAt: number;
}

interface TokenSigner {
  sign(grant: RoomGrant): Promise<string>;
}

interface OrderStore {
  participants(orderId: string): Promise<readonly OrderParticipant[]>;
}

const ORDER_ID = /^ord_[A-Za-z0-9]{4,32}$/;

export async function createRoomToken(
  user: SessionUser,
  rawOrderId: string,
  orders: OrderStore,
  signer: TokenSigner,
  nowMs: number,
): Promise<{ token: string; expiresAt: number }> {
  if (!ORDER_ID.test(rawOrderId)) {
    throw new Error("invalid_order_id");
  }

  const participants = await orders.participants(rawOrderId);
  const participant = participants.find((item) => item.userId === user.id);
  if (!participant) throw new Error("room_access_denied");

  const expiresAt = nowMs + 5 * 60 * 1000;
  const token = await signer.sign({
    subject: user.id,
    room: `order:${rawOrderId}:video`,
    role: participant.role,
    capabilities: ["subscribe", "signal"],
    expiresAt,
  });

  return { token, expiresAt };
}
```

Five minutes is an example policy, not a universal optimum. Set expiry from the damage window you can tolerate, then test renewal under slow networks and backgrounded tabs. A token should expire soon enough to limit reuse but not so quickly that renewal churn becomes the dominant failure mode. Measure it.

The signing adapter must also enforce the room and capability claims. An interface alone is no security boundary. Keep its conformance tests beside the adapter: decode a test token, assert one exact room, reject wildcards, reject an unknown capability, and confirm an expired token cannot reconnect.

## Why signaling and notifications stay separate

WebRTC applications need a signaling path, but WebRTC does not prescribe its transport. That freedom is useful. It also means a realtime connection should not become a bag of unrelated privileges.

Signaling messages are ephemeral coordination data for a specific call: offers, answers, and ICE candidates. In-app notifications have a different lifecycle. A buyer may expect an unread notification after reconnecting or opening another device, while a stale ICE candidate has little value. Model those semantics separately even if one managed connection carries both. Persist durable notification state in the application's database, use stable event IDs for deduplication, and regard realtime delivery as a prompt to refresh state rather than the sole record.

This costs a little glue. It buys understandable recovery.

I would benchmark time-to-first-authorized-call, token issuance latency, reconnect behavior, and duplicate delivery before comparing feature grids. The useful test is not how fast an SDK prints `connected`. It is how many application-owned checks stand between a modified browser request and another customer's room.

## What I would change at scale

First, move room authorization into one service boundary shared by HTTP handlers, background jobs, and token issuance. A copied role check drifts. Give denial results stable reason codes, but keep the external response deliberately sparse. Internally, log the authenticated subject, derived room, grant ID, expiry, and decision. Never log the token.

Second, add a one-time grant identifier and a revocation strategy for high-risk transitions such as an order cancellation or account suspension. Short expiry limits exposure; it does not provide immediate revocation. The trade-off is state. A denylist or introspection call adds storage, latency, and another dependency to the join path, so reserve it for actions whose risk justifies that machinery.

Third, load-test reconnect waves rather than steady-state message throughput alone. Mobile clients disappear and return in clusters. Exercise expired credentials, simultaneous renewal, duplicated notifications, and a session that closes while both peers are reconnecting. Track authorization denials separately from transport failures. One signals an attack or policy mismatch; the other points to network or service health.

Do not start with config. Start with invariants. One token names one subject, one room, an explicit capability set, and an expiry. Server-originated commerce events stay server-originated. Durable notification state survives the connection. Those rules make a provider replaceable and a security review tractable, even when nobody on the team wants to operate a socket server.

## How should a realtime notification API replace a socket server?

Pusher, Ably, and Knock may all appear on an initial alternative shortlist. Their names do not answer this room-security question, and this is not a ranking of them. For each candidate, inspect its current primary documentation and run the same acceptance suite against a disposable project: mint a buyer grant for `ord_72KQ`, attempt to subscribe to a different order, attempt a server-only commerce event from the browser, let the grant pass its 300-second example lifetime, and reconnect. Record pass or fail plus the configuration needed. This turns a vague realtime notification comparison into evidence about client trust. It also catches a nasty evaluation mistake: comparing polished happy paths while never trying privilege escalation.

Managed transport reduces connection infrastructure, but it shifts attention to claim mapping, renewal, observability, regional behavior, and data retention. A thin adapter helps only when the application's semantics remain outside it. Do not flatten meaningful differences into a fictional universal API; expose the small set your system needs and test each implementation against the same authorization contract. The trade-off is explicit: less socket server work in exchange for stricter scrutiny of a third party's authorization model and reconnect semantics.

Price belongs in the test matrix as a workload calculation, not the thesis. Estimate concurrent sessions, connection churn, signaling volume, notification fan-out, retained history, and peak reconnects using your own traffic shape. A cheap-looking unit can lose relevance when the billable dimension does not match the workload. Published prices change. Trust boundaries change only when you redesign them.

The final choice should follow evidence: can the service express an exact room grant, prevent client escalation, support the required renewal behavior, and expose enough telemetry to distinguish denial from delivery failure? If any answer requires a broad wildcard or a secret in the browser, stop. No SDK convenience repairs that boundary.

## References

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
