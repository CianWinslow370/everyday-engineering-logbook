# Node.js Multiplayer Lobby State API Admission Receipts for Video Sessions Under Partial Fanout

Short answer: For a Node.js multiplayer lobby that opens a media video room, make admission a server-owned room-state transition with a retrievable receipt. Treat fanout as notification delivery. A missed notification must not decide whether somebody joined, and a successful join must not claim that their media connection is ready.

The useful alternative to a bundled room framework starts with that contract. This note uses a media watch-party lobby: viewers gather before a screening, and an admitted participant receives permission to enter that screening's video session. The selection criterion is recovery after partial fanout. Framework feature counts can wait.

No benchmark results are claimed here. The numbers below define an example and a proposed test, so the decision can be measured instead of inferred from a working demo.

## What should a multiplayer lobby room state API promise after admission?

Consider an illustrative lobby with 24 viewers. The server accepts one admission, but only 17 viewers receive its notification before a connection drops. The transition has one committed result and several delivery outcomes. Calling the whole operation successful or failed hides the information a reconnecting client needs.

Separate the receipts. An admission receipt identifies the committed room revision. A delivery acknowledgement identifies a particular recipient's processing progress. Media readiness belongs to the connection lifecycle. The WebRTC specification defines peer connections and their state transitions; it does not define the application's multiplayer membership, token permissions, or admission transaction. Those are application contracts.

Keep them separate.

For this design, a scoped token grants a subject permission to join one lobby and its associated video session until an explicit expiry. The application must validate the token and resolve the subject before accepting a command. The room identifier in a request is not evidence of permission. Nor does an open signaling connection prove that the client has applied the current membership state.

This distinction changes the API choice. Evaluate whether an implementation can expose committed admission and let a reconnecting client recover it. A promise that resolves after sending a message is too ambiguous unless its documented meaning matches the contract.

## The constraint that changes the choice

The hard constraint is a viewer whose request committed just before their connection disappeared. On reconnect, that viewer must learn whether they were admitted without creating another membership entry or broadcasting another apparent join. Everyone else must also have a way to converge after missing the original event.

Use a client-generated command identifier that survives a retry. Bind it to the authenticated subject, lobby, and operation. Record its result together with the membership change. For an already-admitted subject using a new command identifier, return the existing admission revision; a second request should not invent a second membership transition.

That is a proposed application policy, not a transport guarantee.

The tempting shortcut is to count successful socket writes and return that count as proof of admission.

It answers the wrong question.

A write result alone cannot establish that a viewer rendered a new member, stored a revision, or completed a media connection. Define an application acknowledgement if recipient processing matters. Then specify precisely what the acknowledgement means.

For a watch-party membership display, choose a current snapshot as the recovery contract. A reconnecting viewer needs the current roster. If the product later requires every moderation action to be processed in order, a roster snapshot will be insufficient: that requirement needs a retained event history and a processing cursor. Accepting snapshot recovery deliberately gives up intermediate transitions.

## The smallest admission core worth testing

Start with the state transition. Keep transport configuration outside it. This TypeScript example is an executable in-memory admission core for one process; its receipt means that this process accepted the change. It does not promise persistence across a restart.

The authentication adapter supplies a verified grant. It must validate the actual token, including its intended audience, before constructing that grant. The core checks the grant's application scope and expiry again at the point of use. Time is an argument so expiry tests need no timers.

```ts
type VerifiedGrant = Readonly<{
  subject: string;
  lobby: string;
  videoSession: string;
  permission: "join";
  expiresAtMs: number;
}>;

type Receipt = Readonly<{
  commandId: string;
  admittedRevision: number;
}>;

class AdmissionCore {
  private revision = 0;
  private members = new Map<string, number>();
  private receipts = new Map<string, Receipt>();

  constructor(
    private readonly lobby: string,
    private readonly videoSession: string,
  ) {}

  admit(
    grant: VerifiedGrant,
    commandId: string,
    nowMs: number,
  ): Receipt {
    if (
      !grant.subject ||
      grant.lobby !== this.lobby ||
      grant.videoSession !== this.videoSession ||
      grant.permission !== "join" ||
      !Number.isFinite(grant.expiresAtMs) ||
      !Number.isFinite(nowMs) ||
      nowMs >= grant.expiresAtMs
    ) {
      throw new Error("Admission denied");
    }
    if (!commandId || commandId.length > 128) {
      throw new Error("Invalid command identifier");
    }

    const key = JSON.stringify([grant.subject, commandId]);
    const previous = this.receipts.get(key);
    if (previous) return previous;

    let admittedRevision = this.members.get(grant.subject);
    if (admittedRevision === undefined) {
      admittedRevision = ++this.revision;
      this.members.set(grant.subject, admittedRevision);
    }

    const receipt = Object.freeze({ commandId, admittedRevision });
    this.receipts.set(key, receipt);
    return receipt;
  }

  snapshot() {
    return {
      revision: this.revision,
      members: [...this.members.keys()],
    };
  }
}

const core = new AdmissionCore("screening-lobby", "screening-video");
const grant: VerifiedGrant = {
  subject: "viewer-17",
  lobby: "screening-lobby",
  videoSession: "screening-video",
  permission: "join",
  expiresAtMs: 60_000,
};
const first = core.admit(grant, "join-attempt-1", 1_000);
const retry = core.admit(grant, "join-attempt-1", 2_000);
if (first.admittedRevision !== retry.admittedRevision) {
  throw new Error("Retry changed admission");
}
```

The grant literal is test data.

Never accept an equivalent object straight from a request body. A TypeScript type isn't token verification. The example limits command identifiers to 128 UTF-16 code units; that is an application input limit, not a token standard.

The core deliberately has no network call between checking an existing receipt and storing a new one. That makes the example's local transition easy to inspect. After it returns, a transport adapter may notify authorized subscribers that a newer room revision exists. The snapshot method also belongs behind authorization; it is not a public roster endpoint.

Notice what the receipt contains: the revision at which admission occurred. A later snapshot may have a larger revision because other viewers joined. Those values answer different questions. Do not overwrite an old admission receipt with the latest snapshot revision just to make the numbers match.

## Benchmark the interrupted path

Time-to-first-call is useful, but a successful localhost join is a weak selection test. Use the same command contract against each candidate implementation. Measure time to admission receipt, time to an authorized snapshot, and time until a reconnected client displays the current revision. Report their latency distributions separately, together with the participant count and the interruption pattern.

For the 24-viewer example, propose three controlled interruptions: drop the request response after commit, suppress the notification for seven viewers, and reconnect one viewer after several more admissions. These are test inputs, not measured outcomes. The first test should recover the same admission revision with the same command identifier. The second should recover the current membership through a snapshot. The third should avoid presenting an older snapshot as newer state. Make that last case concrete: hold the earlier snapshot response, admit another viewer, let the client apply the newer response, and then release the held response. Within the same room epoch, the displayed revision must not move backward. Repeat with injected pauses of 100 ms and 1,000 ms; these are scheduling parameters, not performance claims. Preserve the command identifier across the lost-response test, but generate a new identifier for the already-admitted test. Those two requests should exercise different receipt paths while leaving membership unchanged. A harness that silently generates a fresh identifier on every retry cannot verify the original-command recovery contract.

Don't skip retries.

Test permission failures as well. Try a valid grant for a different lobby, an expired grant, and an unauthenticated snapshot request. None should disclose the roster or mutate membership. A retry with expired authorization must go through renewal and verification; possessing a command identifier is not a capability to bypass access checks.

Record failures by phase: authorization, admission commit, receipt retrieval, notification, snapshot recovery, and media connection. Otherwise a media negotiation problem can inflate the apparent admission failure rate, and a fast receipt can conceal slow convergence across viewers.

Keep the SDK small. One admission operation and one authorized recovery operation make a useful starting surface. Configuration should express actual application policy, such as authorization and recovery retention, rather than expose every internal timer to the caller.

## What I would change at scale

Before multiple processes can accept commands for the same lobby, replace the local maps with a storage design that atomically records admission, its receipt, and pending notification work. Define one revision authority per lobby. A worker can retry notifications from committed work, while clients tolerate repeated notices by fetching state or rejecting stale revisions. Test a worker crash after commit and before send.

The trade-off is storage and operational work in exchange for a recoverable boundary between state changes and fanout. A limitation of the current example is that its maps disappear on restart. It is not suitable for durable admission across deployments. Its maps also need an explicit retention policy before sustained traffic. Receipt expiry must be part of the retry contract: after retention ends, the service cannot promise to return the original receipt.

| Recovery approach | Useful contract | Limitation |
| --- | --- | --- |
| Current roster snapshot | Rebuild the membership display | Omits intermediate transitions |
| Retained event history | Recover ordered application actions | Requires retention and cursor policy |
| Local admission maps | Exercise retry behavior in one process | Lose receipts on restart |

Deployment needs a recovery identity too. If revisions restart after lost state, clients need a new room epoch or an equivalent way to recognize a fresh history. Authorization renewal must recheck access to both the lobby and its associated media session. Log command identifiers and revision progress; keep raw bearer tokens out of those logs.

The decision rule is narrow: choose the implementation whose interrupted admission can be explained and tested. For this media lobby, a viewer should recover a committed membership even when fanout was partial, while video readiness remains separately observable. Measure that recovery before expanding the SDK.

## References

The media boundary follows the W3C peer-connection model. Admission receipts, snapshot recovery, and the test plan above are proposed application contracts, not guarantees supplied by that specification.

## Further reading

- W3C WebRTC 1.0 specification: https://www.w3.org/TR/webrtc/
