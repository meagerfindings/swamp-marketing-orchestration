# @mgreten/marketing-orchestration

A self-contained Swamp model and report for approval-gated marketing releases.
It captures bounded ideation, freezes an approved direction, stages
channel-neutral content, binds exact release approval to a SHA-256 digest,
prepares a single configured publisher claim only when due, and records intended
versus observed delivery.

## Configuration

Model global arguments define authorized approval principal/role pairs, channel
dispatch bindings, intended/success states, and optional observed-state names.
Channels and publisher model names are consumer vocabulary—not package
constants.

```yaml
approvalPrincipals:
  - principal: alex@example.test
    role: marketing-owner
ideationPolicy:
  minIdeas: 12
  maxIdeas: 40
  minGroups: 3
dispatchBindings:
  - channel: newsletter
    model: newsletter-publisher
    method: send
    intendedState: send
    successfulObservedStates: [delivered]
states:
  initialObserved: idle
  queuedObserved: accepted
  failedObserved: error
  canceledObserved: withdrawn
```

Principal/role matching records the approval identity claimed by the method
caller; it does not authenticate a human. The consumer workflow must bind these
arguments to trusted approval evidence from its authentication and authorization
boundary rather than accepting untrusted caller-supplied identity strings.

## Consumer workflow contract

This package intentionally includes no publisher adapter workflows. A consumer
workflow should:

1. Build each complete publication request (`itemId`, `model`, `method`, and
   `arguments`), SHA-256 digest the canonical ordered request array, and include
   that digest in `exactApproval.publicationRequestsDigest`. `approveRelease`
   rejects any route or arguments that differ from this exact approved packet.
2. Call `prepareDispatch` with campaign, release, item, an explicit `asOf`, and
   a caller-generated safe `attemptId`. The emitted publisher arguments are
   canonically identical to the approved arguments: preparation never injects or
   mutates an argument. `attemptId` and `claimId` remain separate dispatch
   metadata. Invoke a publisher only when this call returns a newly-created
   dispatch handle; replay returns no handle and therefore cannot authorize
   another send. Preparation first persists the schedule's `dispatching` status,
   attempt ID, and independently computed claim ID. That durable binding
   serializes dispatch: only repair or replay of the bound attempt is accepted
   until delivery reconciliation.
3. Read that dispatch claim and invoke its configured `model`, `method`, and
   `arguments`. Consumers must use `claimId` as the transport idempotency key
   without adding it to or otherwise mutating the approved arguments. The method
   fails closed before approval or due time.
4. Call `recordDelivery` with the approved release digest, returned
   `dispatchClaimId`, and matching `attemptId`. The method reconciles that exact
   attempt claim while recording `scheduled`, `published`, `failed`, or
   `canceled`. Supply a platform reference for successful publication.

The consumer remains responsible for mapping publisher output into configured
observed states. Never construct a publisher request from mutable draft content;
use only the dispatch resource emitted by `prepareDispatch`.

Swamp holds an exclusive lock for the entire method invocation on each model
instance. That runtime guarantee serializes `prepareDispatch` and
`recordDelivery` across processes and workflow workers. Consumers must operate
on one shared model instance; separate model instances intentionally have
independent schedules and cannot coordinate the same release.

## Guarantees and bounds

- Direction and release approvals are principal/role authorized and bind the
  complete publication model, method, and argument packet by digest.
- Approved releases are create-once-or-verify; replay cannot overwrite them.
- Stage/approval retries verify immutable records and repair missing schedule
  entries after partial writes.
- Releases contain 1–100 items. Ideation policy defaults to 20–60 ideas across
  4+ groups, is consumer-configurable, and remains capped at 100 ideas
  package-wide.
- Dispatch preparation validates release digest, schedule state, and due time.
  If claim persistence fails after the schedule binding is written, replaying
  that attempt repairs the missing claim and emits its one fresh handle.
  Approved items permit an initial attempt; failed items permit a retry only
  under a new attempt ID. A reconciled failed attempt cannot be re-emitted.
- Delivery transitions are explicit: a dispatching attempt may immediately
  publish or fail, or become scheduled and later publish/fail using the same
  schedule-bound, independently recomputed claim. Terminal states cannot be
  dispatched.
- Calendar snapshots choose the latest version at `generatedAt`, honor latest
  deletions, and reject inverted windows.
- Resource garbage-collection bounds are declared by the model.

## Verification

```sh
~/.swamp/deno/deno fmt --check
~/.swamp/deno/deno check extensions/models/marketing_orchestration.ts extensions/reports/marketing_content_calendar.ts
~/.swamp/deno/deno lint
~/.swamp/deno/deno test
swamp extension fmt extensions/manifest.yaml --check --json
swamp extension quality extensions/manifest.yaml --json
swamp extension push extensions/manifest.yaml --dry-run --json
```

## License

MIT. See [LICENSE](LICENSE).
