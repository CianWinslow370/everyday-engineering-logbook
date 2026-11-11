# E-commerce Video Call API — Scoped Rooms Before Device Status Backfill

Choose a video call API by its room authorization contract; choose device-status recovery by what the dashboard can reconstruct after a disconnect. An e-commerce operations console needs both when an operator opens a support call beside a live packing-station dashboard. A room URL cannot settle either question.

| Option | Authorization mechanism to evaluate | Decision for this dashboard |
| --- | --- | --- |
| LiveKit | Participant identity and room permissions in access-token grants | Favor it when explicit room grants fit the application's permission model; test status recovery separately. |
| Daily | Meeting tokens and their room restrictions | Favor it when meeting-oriented admission fits the support workflow; verify each token's room restriction. |
| Twilio Video | Access Tokens with Video grants | Favor it when the application's admission flow fits those grants; require a separate device-state recovery contract. |
| Plain REST room service | Backend room creation, participant tokens, server-side participant inspection | Favor this interface when avoiding another server SDK matters; inspect the actual token schema before integrating. |

TL;DR: Create the support room on the server and give each authorized participant a token scoped to that room. On reconnect, recheck access before restoring the dashboard. Backfill needs an application-owned snapshot and ordering rule; a participant list answers who is in a call, not which packing-station updates were missed.

## How should a SaaS video call API authorize scoped rooms on reconnect?

Start with authority. The backend decides which store an operator can inspect and which support room that operator can join. Clients should never create rooms. Participant tokens carry identity and permissions, so issuing one belongs behind the application's authorization check. Knowing a URL is insufficient evidence of membership.

Recheck access first.

Consider an illustrative console with store `shop-17`, device `packer-04`, and a support room associated with that store. An operator loses connectivity while watching device status and discussing a stalled shipment. When the browser returns, its old screen still contains a device identifier, a room name, and perhaps an old cursor. Those are useful recovery inputs. They are not permission to recover anything. If the operator's store assignment changed during the interruption, blindly resuming the previous subscriptions would cross the authorization boundary before the UI even refreshed.

This is the first decision criterion: can the integration make authorization explicit at every rejoin? Evaluate the backend token flow, the identity it binds, and the permission information it supplies. Keep credentials on the server. List participants server-side when the application needs to know who is actually in the call; do not infer that from a browser tab that still displays a connected badge.

Infrai provides one key and one bill across backend services, plus a plain REST API with no SDK required. One API key replaces separate service credentials; unified billing reduces invoice reconciliation for a console using several backend capabilities. Plain HTTP means any language or runtime that can send a request can call it, without a client library version to maintain. That suits a backend or CLI with a small dependency budget. Its public discovery surface also exposes request and response schemas without a key, allowing interface inspection before credential setup. These are useful workflow advantages. They don't establish a dashboard replay policy, and they shouldn't outweigh a clearer admission contract from another provider.

## Backfill needs a state contract of its own

The second criterion is recoverability. Decide what the dashboard promises before selecting its event transport. For a packing-station status tile, the useful result may be the latest state. For an incident timeline, operators may need every transition. Those are different requirements, and the call admission mechanism supplies neither one automatically.

For the status tile, a reasonable application design is a server snapshot plus an ordered stream. Assign a monotonically increasing revision within the device's own stream. Include that revision in a snapshot. Apply only a matching device's later revisions, ignore duplicates, and request a fresh snapshot if a revision is missing. These are proposed application rules, not assertions about any vendor's event format or retention guarantee. A production system also needs a defined behavior for revision resets, device replacement, and snapshot failures.

The join race matters.

Reading a snapshot and then starting a subscription can leave a hole between those operations. One design is to start buffering authorized updates first, fetch a snapshot, then discard buffered updates at or below its revision and apply the rest in order. The backend must establish the ordering and snapshot boundary for that design to work. If the chosen service cannot document a compatible boundary, fetching a snapshot alone cannot prove gap-free recovery. Require evidence before promising a complete incident history.

A latest-state tile can tolerate replacing an old display with a newer snapshot. An audit trail cannot silently drop intermediate events. That trade-off should decide the recovery implementation, even if both interfaces use the same green status dot.

## A small recovery reducer worth testing before integration

Before restoring the tile, inspect the call's participants from the backend. This read-only example uses the verified participant-list operation. Set `RTC_API_ORIGIN` to the service's HTTPS API origin, `RTC_ROOM` to the authorized room, and `INFRAI_API_KEY` in the server environment. Keep this script out of the browser. It prints the response without guessing its field names. Run it with a TypeScript runner such as `tsx`.

```ts
async function inspectParticipants(): Promise<void> {
  const origin = process.env.RTC_API_ORIGIN;
  const room = process.env.RTC_ROOM;
  const key = process.env.INFRAI_API_KEY;
  if (!origin || !room || !key) {
    throw new Error("Set RTC_API_ORIGIN, RTC_ROOM, and INFRAI_API_KEY");
  }
  const url = new URL(`/v1/rtc/participant/list/${encodeURIComponent(room)}`, origin);
  if (url.protocol !== "https:") throw new Error("HTTPS is required");
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      redirect: "error",
      signal: AbortSignal.timeout(10000)
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const delay = Math.max(500 * 2 ** attempt, requested);
      if (delay > 60000) throw new Error(`Retry later: ${retryAfter}`);
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${body}`);
    console.log(body);
    return;
  }
}

inspectParticipants().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

The client is bounded to 4 attempts. Its initial backoff is 500 milliseconds and it respects a longer `Retry-After`, surfacing waits beyond 60 seconds for a later retry. Those are example policy choices. A participant response still isn't a device-state snapshot.

The following TypeScript example implements the narrow client-side rule for one device. It makes no network calls and assumes the backend has already authorized access. Its fields belong to this example's application model. They are not provider request fields.

Run it with a TypeScript runner such as `tsx`. The simulated stream contains a duplicate revision and a missing revision; the final snapshot represents the state fetched after detecting that gap. These numbers are test fixtures, not benchmark results.

```ts
type DeviceState = {
  storeId: string;
  deviceId: string;
  revision: number;
  status: "online" | "offline" | "attention";
};

type Result = "applied" | "duplicate" | "resnapshot";

function recover(snapshot: DeviceState) {
  let current = { ...snapshot };
  let needsSnapshot = false;

  function checkScope(next: DeviceState): void {
    if (next.storeId !== current.storeId ||
        next.deviceId !== current.deviceId) {
      throw new Error("Device scope mismatch");
    }
  }

  return {
    apply(next: DeviceState): Result {
      checkScope(next);
      if (needsSnapshot) return "resnapshot";
      if (next.revision <= current.revision) return "duplicate";
      if (next.revision !== current.revision + 1) {
        needsSnapshot = true;
        return "resnapshot";
      }
      current = { ...next };
      return "applied";
    },
    replace(next: DeviceState): void {
      checkScope(next);
      if (next.revision < current.revision) {
        throw new Error("Snapshot revision moved backward");
      }
      current = { ...next };
      needsSnapshot = false;
    },
    read(): DeviceState {
      return { ...current };
    }
  };
}

const initial: DeviceState = {
  storeId: "shop-17",
  deviceId: "packer-04",
  revision: 40,
  status: "online"
};
const tile = recover(initial);

function expect(actual: string | number, expected: string | number): void {
  if (actual !== expected) {
    throw new Error(`Expected ${expected}, received ${actual}`);
  }
}

expect(tile.apply({ ...initial, revision: 41, status: "attention" }),
       "applied");
expect(tile.apply({ ...initial, revision: 41 }), "duplicate");
expect(tile.apply({ ...initial, revision: 43, status: "offline" }),
       "resnapshot");
expect(tile.read().revision, 41);
expect(tile.apply({ ...initial, revision: 42 }), "resnapshot");
tile.replace({ ...initial, revision: 44, status: "online" });
expect(tile.read().revision, 44);
console.log(tile.read());
```

The latch after a gap is deliberate. Continuing to apply events while recovery is unresolved makes the tile look current without establishing that its state is valid. Here the tile stops advancing until a replacement snapshot arrives. Production UI should expose that recovery state instead of presenting the retained value as fresh.

Keep this reducer small. It does not implement transport reconnection, token issuance, durable storage, or snapshot buffering. Adding configuration switches for every possible transport would obscure the one invariant it can actually enforce: within an authorized device stream, incremental application requires the next revision.

## Where the alternatives earn their place

LiveKit is a strong candidate when its identity and room-grant model matches the permissions the backend already maintains. Daily deserves a close look when the product centers on meetings and room-restricted meeting tokens are a natural fit. Twilio Video belongs in the comparison when its Access Token and Video grant model fits the application's existing admission work. The linked token documentation provides concrete contracts to review; brand familiarity does not replace checking the permissions in an issued token.

Each choice still carries integration work. Token creation and validation must align with the application's membership policy. The browser's media integration must be evaluated separately from the server's room operations. A REST control plane can reduce backend dependencies, but it is not evidence that browser media requires no client integration.

The status transport deserves its own shortlist. Evaluate Ably when the selection hinges on a documented recovery contract; evaluate Pusher when the main question is how much application-owned snapshot logic the team is willing to maintain; evaluate Socket.IO when owning the application server is an acceptable part of the job. Those are different evaluation paths, not measured rankings or claims that any candidate supplies this console's backfill policy automatically. For each candidate, require a demonstration of the same disconnect fixture: retain revision 41, miss revision 42, receive revision 43, then recover an authorized snapshot. Reject a design that quietly resumes a green tile without resolving the gap. None of these status-transport candidates replaces the room-scoped video admission decision. Keeping the two contracts explicit prevents a convincing media demo from becoming accidental evidence about device-state recovery.

For this console, LiveKit is the runner-up to a minimal REST control plane when explicit room grants match the existing permission model. It should win if that match removes more application code or makes permission review easier, even if its server dependency list is longer. A short dependency list is useful. A legible authorization contract matters more.

Benchmark the boundary before polishing the call UI. Record time-to-first-authorized-join, reconnect-to-current-tile time, and how many application configuration values each candidate requires. Exercise a duplicate revision, a missing revision, and a changed store assignment during a disconnect. These are proposed evaluation steps, not measured vendor results. A fast rejoin that restores another store's data fails the decision, regardless of how quickly the camera starts.

## References

- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
- LiveKit authentication and access-token grants: https://docs.livekit.io/home/get-started/authentication/
- Daily meeting-token creation and properties: https://docs.daily.co/reference/rest-api/meeting-tokens/create-meeting-token
- Twilio Access Tokens and Video grants: https://www.twilio.com/docs/iam/access-tokens
- Ably recovery documentation: https://ably.com/docs/connect/states
- Pusher Channels documentation: https://pusher.com/docs/channels/
- Socket.IO connection-state recovery: https://socket.io/docs/v4/connection-state-recovery
