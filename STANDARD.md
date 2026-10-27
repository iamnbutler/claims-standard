# The Claims Standard

Version 0.1, draft (2026-10-26).

A **claim** is a statement about what a person can do in a piece of software and what follows.
It has a durable id, a plain-language half for people, and a formal half that tools check. This
standard defines claims independently of any language or UI framework, so that implementations in Rust,
TypeScript, Swift, Lean or anything else can issue, check, publish and read claims that mean
the same thing. Tools can then interoperate across all of them.

Formal verification is not required. The standard has three conformance levels:
- **Ledger**: ids and their history.
- **Checked**: claims enforced at runtime and exercised by tests.
- **Proved**: claims proved for every input.

Most implementations will stop at Checked. Proved is an optional target for implementations whose
language can carry machine-checked proofs.

The key words MUST, MUST NOT, REQUIRED, SHOULD, SHOULD NOT, RECOMMENDED, MAY and OPTIONAL are to
be read as described in RFC 2119 and RFC 8174 when, and only when, they appear in capitals.
Sections marked *informative* are not normative.

## 1. Why claims (informative)

Tests check the cases someone wrote down. A Playwright test finds a button by a test id, clicks
it, and expects some text. The test id is a private string. The expectation lives in a file nobody
reads alongside the button. And nothing stops a button from shipping with no test at all.

Claims turn that around. Every action a person can take is declared up front as a claim with:
- a permanent id (`^2`);
- what the person does and what they see;
- when the action is offered;
- what it promises about the result.

The UI is built from the claims, so an action with no claim can't exist. The id is public: it
appears in the UI's accessibility tree, in bug reports, in test selectors and on a published
claim page. It means the same thing in every later version, because ids are never reused and a
change in meaning needs a new id.

Evidence then attaches to the id:
- a proof, where the language can carry one;
- a fuzzer's or property test's random runs;
- an end-to-end test of the running app;
- a screenshot.

Each kind of evidence rules out different things, and a claim keeps all of them.

## 2. Terms

- **Package**: the unit that issues claims and is versioned and published together, such as a
  crate, an npm package or a Swift package.
- **Scope**: a package's claim namespace. It is the package's name.
- **Claim id**: a positive integer, unique within a scope for all time.
- **Ref**: a claim id written `^N`. Its **qualified** form is `scope^N`. A **keyed ref**
  `^N#K` names one instance of a keyed claim.
- **State**: the application's model: everything the UI is a function of.
- **Action**: one thing a person did, addressed to a claim: a trigger, a ref, and an argument
  for some triggers.
- **Effect**: a request from the application to the outside world, made as data (an HTTP request,
  a timer, a clipboard write) and run by a host.
- **Machine**: an initial state, plus the set of moves that make up the application's update
  function.
- **Ledger**: the append-only file of every id a scope has issued (§6).
- **Export**: a package's claims as JSON (§9).
- **Evidence**: a record that some check passed, failed or wasn't run for a claim (§8).
- **Implementation**: a library, macro set, compiler plugin or tool that lets a package declare
  claims and that produces the ledger and export.

## 3. Identity

### 3.1 Scopes

A scope is the package's name, written the way its ecosystem writes it: `my-app`, `my_app`,
`My.App` and `@org/app` are all valid scopes. The standard doesn't pick a naming convention,
because ecosystems disagree: some refuse `_`, and others refuse `-`.

- A scope MUST be non-empty and MUST NOT contain `^`, `#`, whitespace or control characters,
  because those delimit refs (§3.3) and ledger fields (§6.1).
- Two scopes are the **same scope** when their normalized forms are equal. To normalize a scope,
  lowercase it, then replace every run of `-`, `_` and `.` with a single `-`. Under this rule
  `My_App`, `my-app` and `my.app` are the same scope. Implementations MUST compare scopes by
  their normalized form. They MUST write a scope as it was declared, and never as its normalized
  form.
- A package has exactly one scope. Implementations MUST refuse to build when the scope declared
  for claims isn't the same scope as the package's name in its manifest.

Implementations MUST NOT supply a default scope that another package could also get by default.
One safe choice is to default to the implementation's own scope. A package that forgot to set its
scope then collides with the implementation's ids at build time, rather than shipping with them.

### 3.2 Ids

- An id is an integer from 1 to 2⁵³−1, so it survives a JSON number.
- **An id is issued once per scope, forever.** It MUST NOT be reused, renumbered or deleted.
  When a claim ends, it is retired (§5.3); its id stays issued.
- Ids need not be contiguous. Packages MAY reserve ranges for areas, such as ^9xx for assumptions
  about the platform.
- An implementation MUST reject, at build time, two live claims with the same id in one scope.
  The two may be in modules or files that are never loaded together. In that case the ledger
  check (§6.3) MUST catch it.

### 3.3 Refs

| Form | Means |
|---|---|
| `^N` | claim N in the current scope |
| `scope^N` | claim N in a named scope |
| `^N#K` | the instance of keyed claim N with key K |
| `scope^N#K` | the same, qualified |

In source code and inside one package, a bare `^N` always means the package's own scope.
Messages, logs, exports and anything read outside the package MUST use the qualified form.

A key K is a non-negative integer chosen by the application. It identifies one item of a
collection, such as a row's checkbox, and MUST be stable for that item while it exists.
Implementations MAY map a non-integer item identity to keys. If they do, the mapping MUST be
injective over the live items.

## 4. Claims

Every claim has an **id**, a **kind**, a **title** (one line, no tabs or newlines), a **does**
(prose) and the kind-specific parts below. A claim MAY name a **place** (`in ^P`): a live place
claim in the same scope.

The formal parts are written in the implementation's language. This standard says what each
part means and what has to be checked about it. It doesn't say how the part is spelled.

### 4.1 move

A move is one thing a person can do: a button, a text field, a menu item, a gesture.

| Part | Required | Meaning |
|---|---|---|
| `does` | yes | what the person does ("Press Reset.") |
| `sees` | yes | what follows, in the person's terms ("The count is zero, and Reset disappears.") |
| `keyed` | no | whether the move has one control per item, addressed `^N#K` |
| `enabled` | no | a total function from state (and key) to a boolean: when the move is offered. The default is always. |
| handlers | at least one | one per trigger the move answers (§4.1.1) |

Each **handler** has:

| Part | Required | Meaning |
|---|---|---|
| `trigger` | yes | one of §4.1.1 |
| `apply` | yes | a total, deterministic function `(state, key?, arg?) → (state′, effects)` |
| `post` | yes | a predicate over `(state, key?, arg?, state′)`: what the handler promises about the result |
| `sound` | Proved level only | a proof that `enabled(s, k) ⇒ post(s, k, a, apply(s, k, a).state)` for all `s`, `k`, `a` |

`apply` MUST NOT read anything except its arguments: no clock, no randomness, no I/O. Anything
from outside comes in as an action or a message (§7.2) and anything going out leaves as an effect.
That makes it possible to replay, fuzz and prove a move.

`post` constrains state only. Which effects a handler emits is not part of `post` in this version
(§13).

#### 4.1.1 Triggers

| Trigger | Argument | Delivered when |
|---|---|---|
| `press` | none | the control is activated (click, tap, Space or Return on focus, an accessibility press) |
| `doublePress` | none | the control is activated twice in quick succession |
| `edit` | a text operation (§4.1.2) | the person changes a text control's content, caret or selection |
| `submit` | none | the person commits a text control (Return) |
| `cancel` | none | the person backs out (Escape) |
| `blur` | none | the control loses focus |
| `appear` | none | the control comes on screen, when it opted in, as for "load more" at the end of a list |

A move has at most one handler per trigger. An implementation MAY add triggers. An added trigger's
name MUST contain a `.` (such as `acme.longPress`), so it can't collide with names a later
version of this standard adds.

#### 4.1.2 Text operations

An `edit` handler receives one of: `insert(text)`, `deleteBackward`, `deleteForward`,
`deleteWordBackward`, `deleteWordForward`, `moveLeft(extend)`, `moveRight(extend)`,
`moveWordLeft(extend)`, `moveWordRight(extend)`, `moveHome(extend)`, `moveEnd(extend)`,
`setCaret(index, extend)`, `selectAll`, `setMarked(text, selStart, selLen)`, `unmark`, `cut`,
`copy`, `paste(text)`, `undo`, `redo`.

Indices count Unicode scalar values. An implementation MAY deliver only a subset of these. Its
export MUST NOT claim a flow step uses one it can't deliver.

### 4.2 flow

A flow is a path a person can take through the application to reach a goal.

| Part | Required | Meaning |
|---|---|---|
| `does`, `sees` | yes | the path and its outcome, in the person's terms |
| `machine` | yes | the machine the flow is about |
| `goal` | yes | a predicate over state |
| `witness` | yes | a finite list of actions, each a trigger, a ref in this scope (keyed if needed) and its argument |

A flow holds when two things are true:
- **live**: replaying the witness from the machine's initial state, every step is answered by an
  enabled handler of the move it names. No step is a no-op.
- **reaches**: the state after the last step satisfies `goal`.

Both are decidable by running the witness. A flow is therefore checkable at every level, and at
the Checked level it MUST be checked on every build (§10.2).

### 4.3 invariant

An invariant is something true in every state the application can reach.

| Part | Required | Meaning |
|---|---|---|
| `does` | yes | the property, in words |
| `machine` | yes | the machine it holds for |
| `holds` | yes | a predicate over state. It SHOULD be decidable, so it can be checked at runtime. |

An invariant holds when it is true of:
- **init**: the initial state;
- **step**: the result of every enabled handler, from any state where it is true;
- **receive**: the result of every message from the outside world (§7.2), from any state where
  it is true. This applies only if the machine receives messages.

Together these give **reachable**: the invariant holds in every state reached by any sequence
of actions and messages.

### 4.4 assume

An assumption is something the claims rely on and the implementation cannot check by
construction. Most assumptions are about the host, the platform, or another package: "the host
reports clicks where they happen", "the clock never goes backwards".

| Part | Required | Meaning |
|---|---|---|
| `does` | yes | the assumption, in words |
| `statement` | no | a formal statement of it |
| `check` | no | the name of a runnable check that gathers evidence for it. It MUST NOT contain whitespace, `^` or `#`. |

An assumption is never proved. An implementation at the Proved level MUST NOT make an
assumption's statement an axiom. A theorem that relies on it takes it as a hypothesis, so every
use is visible in the theorem's statement (§10.3).

### 4.5 place

A place is a named region where moves appear: a window, a sheet, a sidebar, a page.

| Part | Required | Meaning |
|---|---|---|
| `does` | yes | what and where it is |

A move, flow or place MAY name a place with `in ^P`. The implementation MUST check that `^P` is a
live place in the same scope.

### 4.6 Tombstones

A retired claim is replaced in the source by a tombstone. The tombstone has the claim's id, its
kind, its last title and a **reason** (prose, REQUIRED). The reason SHOULD name the claims that
replace it ("Split into ^14 and ^15.").

### 4.7 What every implementation checks at build time

An implementation MUST refuse to build a package, naming the claim, when any of these fail:

1. Every claim has an id, a kind, a non-empty title and a non-empty `does`. A move also has `sees`.
2. No two live claims in the scope share an id. No live claim reuses an id the ledger records as
   retired or moved.
3. Every `in ^P` names a live place in the scope.
4. Every move has at least one handler, at most one per trigger, and each handler has `apply`
   and `post`.
5. Every flow's witness names existing moves in the scope, each with a handler for that step's
   trigger. A keyed move's step carries a key, and an `edit` step carries an operation.
6. Every tombstone has a reason.
7. **No behaviour without a claim**: the application's update function is assembled from its
   moves and nothing else (§7.1). If the implementation can't enforce this structurally, it MUST
   enforce it at runtime and report an unclaimed action as an error in development builds.

## 5. Lifecycle

### 5.1 States

| Status | Meaning |
|---|---|
| `active` | the claim is live in this scope |
| `retired` | the claim ended. The id stays issued and MUST NOT come back as a live claim. |
| `moved:<scope>` | the claim left with code that moved to another package, which keeps the same number. The id stays issued here and MUST NOT come back in this scope. |

### 5.2 Meaning

A claim's meaning is its kind, its formal parts and its prose. Changing what a claim promises
needs a new id: retire the old one and issue the new one.

Implementations detect a possible change with the **statement fingerprint** (§6.2), and also flag
a changed title. A changed fingerprint or title is a warning, not an error, because a fingerprint
can't tell a refactor from a change of meaning. A person decides, and records the decision in the
ledger (`accept`). The change then shows up in review as a diff of the ledger.

### 5.3 Retiring and versioning

A released claim is a promise to everyone who relies on it, so ending one is a breaking change.
Packages use semantic versioning. A claim that the package's last release had `active` MAY become
`retired` or `moved:*` only in:
- a new major version, or
- a new minor version while the major version is 0, as Cargo treats 0.x.

A claim added since the last release MAY be retired in any version.

The last release is the highest published final version that is not greater than the current
one. Pre-releases don't count. The ledger check (§6.3) and the registry (§11) MUST both enforce
this rule.

Adding claims is never breaking. A changed fingerprint that was accepted is not breaking by this
rule. Reviewers SHOULD treat a weakened `post` as breaking, but no tool can tell.

## 6. The ledger

### 6.1 Format

The ledger is a UTF-8 text file named `claims.lock` at the package root. Tools write it, and
people never edit it by hand. Lines end with `\n`.

```
# claims.lock: every claim id ever issued. Written by `claims update`; never edit by hand.
# id	kind	status	statement	title
1	move	active	1c77a9411d162684	Increment
2	move	active	5ace86b109a74bf5	Reset
3	place	active	-	Counter window
6	invariant	retired	5de1d6cd863262b8	Moves never leave a press in progress
7	assume	active	-	The host reports clicks where they happen
```

- Lines that are empty or start with `#` are comments.
- Every other line is one id, five fields, separated by single tabs:
  1. `id`: decimal, no leading zeros.
  2. `kind`: `move`, `flow`, `invariant`, `assume` or `place`. A retired claim keeps its kind.
  3. `status`: `active`, `retired`, or `moved:<scope>`.
  4. `statement`: the statement fingerprint, as 16 lowercase hex digits, or `-` for a claim
     with no formal statement (a place, or an assumption without one).
  5. `title`: the rest of the line.
- Lines are in ascending id order, with one line per id.
- A tool MAY add one comment line of the form `# claims-lock-version: 1` and
  `# fingerprint: <algorithm>`. A ledger without these lines is version 1, and its fingerprint
  algorithm is the implementation's default.

Writers MUST produce exactly this format. Readers MUST accept it, and MUST reject a line with the
wrong number of fields, an unknown kind or an unknown status.

### 6.2 The statement fingerprint

The fingerprint is a 64-bit hash of the claim's **meaning-bearing definitions**:
- its kind;
- for a move: whether it is keyed, its triggers, `enabled`, and every handler's `post`;
- for a flow: `goal` and `witness`;
- for an invariant: `holds`;
- for an assumption: `statement`.

It follows the package's own helper definitions that these use, so changing a helper that a
`post` calls changes the fingerprint. It MUST NOT depend on:
- `apply`, so a move can be reimplemented without changing what it promises;
- proofs;
- binder or variable names;
- whitespace or comments;
- other claims or the machine.

Fingerprints are compared only with fingerprints from the same algorithm. An implementation MUST
document its canonical form. It SHOULD use FNV-1a 64 over that form:
- offset basis 14695981039346656037;
- prime 1099511628211;
- each byte XORed in, then multiplied by the prime.

A Lean implementation might walk elaborated expressions; a TypeScript one might walk a normalised
AST.

When an implementation changes its canonical form, every fingerprint changes. Its tool MUST then
offer a one-step re-baseline that rewrites every fingerprint and changes nothing else, and SHOULD
record the new algorithm in the `# fingerprint:` line.

### 6.3 Checks

`ledger check` compares the code's claims with the ledger. It MUST **fail** when:

1. an id in the ledger is missing from the code, and the ledger doesn't record it as retired or
   moved;
2. an id appears with a different kind than the ledger records;
3. a retired or moved id is live in the code;
4. the code issues an id twice;
5. a claim has no `does`, or a move has no `sees`;
6. the code has an id the ledger doesn't have, or retires an id the ledger has as active. Both
   are fixed by `ledger update` and committing the result;
7. the release rule of §5.3 is broken.

It MUST **warn** when a live claim's fingerprint or title differs from the ledger's.

`ledger update`:
- appends new ids;
- marks newly retired ones;
- with `--accept`, records changed fingerprints and titles.

It MUST NOT remove a line. It MUST refuse to write while `check` reports failures other than new
ids or new retirements.

`ledger move <scope> <id>…` marks ids as `moved:<scope>`. It requires that each id is in the
ledger and gone from this scope's code.

Implementations SHOULD run `ledger check` in the package's required CI check.

## 7. Runtime semantics

These are the guarantees that make "every behaviour is a claim" true of the running application,
not just of its source. They are normative at the Checked and Proved levels.

### 7.1 Dispatch

The application's update function for actions MUST be **dispatch** over its moves:

1. The action names a ref `^N` (and key K), a trigger, and an argument if the trigger has one.
2. Find the live move with id N. If there is none, the action is **unclaimed**: the state is
   unchanged and no effects are emitted.
3. If the move is keyed and the action has no key, or `enabled(s, K)` is false, or the move has
   no handler for the trigger, the state is unchanged and no effects are emitted.
4. Otherwise the result is the handler's `apply(s, K, arg)`.

From which:

- **No behaviour without a claim.** An action whose ref names no move changes nothing and emits
  nothing.
- **Every effect comes from a move.** Every effect the application emits in response to an
  action was emitted by an enabled handler of the move that action named.
- **Disabled means inert.** A move that isn't enabled does nothing, even if its control is
  somehow pressed.

At the Proved level these are theorems (§10.3). At the Checked level the runtime MUST enforce
them by construction or check them on every dispatch in development and test builds.

### 7.2 Messages from the world

Replies to effects, timers, clock ticks and subscription events reach the application as
**messages**, through a separate `receive(state, message) → (state′, effects)`. Messages are not
claims; they have no ref. An invariant (§4.3) MUST hold across `receive` as well as across moves,
so it holds whatever order the world answers in.

### 7.3 Refs in the UI

Every control a move puts on screen MUST carry its ref (`^N`, or `^N#K` for a keyed move) where
tools and assistive technology can read it. The binding names where that is:

| Platform | Where the ref goes |
|---|---|
| macOS / iOS | the accessibility identifier |
| Web | the `data-claim` attribute (its value `^N` or `^N#K`) |
| Android | the view's `testTag` / resource id |
| Windows (UIA) | `AutomationId` |

A control's ref MUST be the ref of the move whose handlers it triggers. Two different moves MUST
NOT share a control. Places SHOULD carry their ref on the region's container.

This is what lets a test, a screen reader, an agent or a bug report say "`^2`" and mean the same
control. In Playwright it is `page.locator('[data-claim="^2"]')`, in place of a private test id.

## 8. Evidence

### 8.1 Kinds

Strongest first:

| Kind | Rules out | Still trusts |
|---|---|---|
| **proved** | a counterexample anywhere: every state, every key, every argument | that the statement says what's meant; the proof checker and compiler; the assumptions |
| **fuzzed** | a counterexample in the random sequences run | that random sequences reach the case that matters |
| **tested** | the running application disagreeing with the claims, on the sequences run | the test harness; nothing about sequences not run |
| **seen** | a control that is never drawn, or a flow that can't be clicked through | whatever produced the capture |
| **assumed** | nothing | whoever wrote it, and its check |

Proved and fuzzed are about the model: the moves, machine and predicates. Tested and seen are
about the running application. A claim keeps every kind of evidence it has, not just its
strongest, because the kinds cover different things.

What each kind means for each claim:

| Claim | proved | fuzzed | tested | seen | assumed |
|---|---|---|---|---|---|
| move | each handler's `sound` | random action sequences; after each step, `post` held and the control was present iff enabled | the running app matched the model at every step that exercised the move | its control appears in a captured frame | |
| flow | `live` and `reaches` | (not applicable: a flow is a fixed path) | the witness replayed on the running app reached the goal | the witness replayed, each step on the control it names | |
| invariant | `init`, `step`, `receive` | `holds` after every step of random sequences | `holds` on the running app's model state at every step | (not applicable) | |
| assume | (never) | its `check` held in every case run | | | always |
| place | | | | its region appears in a captured frame | |

**Fuzzed** covers property-based testing in general. fast-check (TypeScript), proptest or
quickcheck (Rust) and Hypothesis (Python) running random action sequences against the machine
all produce fuzzed evidence.

**Tested** covers end-to-end and conformance testing. A Playwright, XCUITest or WebDriver run
that drives the real application by refs and compares with the model produces tested evidence.

### 8.2 Records

An evidence record is:

```json
{ "claim": "counter^2", "kind": "fuzzed", "result": "pass",
  "role": "post press", "runs": 200, "seed": "8f3a…",
  "replay": "claims fuzz counter --seed 8f3a…",
  "tool": "claims-fuzz 1.2.0", "at": "2026-10-26T16:30:00Z" }
```

- `result` is `pass`, `fail` or `gap`. A gap is evidence the claim could have and doesn't: the
  check wasn't run, or never reached the claim.
- A **fail** MUST carry what reproduces it. For fuzzing, that's the seed or the shrunk sequence.
  For a test, it's the step and the expected and actual values. For a proof, it's the missing
  theorem or the axiom it used.
- `role` names what was checked, as in the table above: `sound <trigger>`, `live`, `reaches`,
  `init`, `step`, `receive`, `reachable`, `post <trigger>`, `presence`, `check <name>`.

Evidence is computed by tools, never typed by hand.

### 8.3 How a claim reads

To make a list scannable, each claim also reads as one word. The first matching rule wins:

1. `retired`: it is retired. It shows its reason and nothing else.
2. `moved`: the ledger says another scope owns it.
3. `broken`: any of its evidence failed.
4. `assumed`: it is an assumption, even when fuzzed.
5. `proved`, `fuzzed`, `tested` or `seen`: the strongest kind it passes.
6. `untested`: it is live and has no passing evidence.

A reader SHOULD show the full list of evidence beside the word.

## 9. The export

The export is a package's claims as JSON, produced from the build, never written by hand. Tools
(claim pages, test generators, coverage reports, agents) read it instead of parsing source.

```json
{ "standard": "claims/0.1",
  "scope": "counter", "version": "0.3.0",
  "claims": [
    { "id": 2, "ref": "^2", "qualified": "counter^2", "kind": "move",
      "title": "Reset",
      "does": "Press Reset.", "sees": "The count is zero, and Reset disappears.",
      "place": 3, "retired": null,
      "statement": "5ace86b109a74bf5",
      "triggers": ["press"], "keyed": false,
      "witness": null, "passes": [], "machine": null, "check": null,
      "pos": { "file": "src/counter.rs", "line": 59 },
      "decl": "counter::reset",
      "evidence": [
        { "role": "sound press", "theorem": "counter::reset::press::sound",
          "proved": true, "extraAxioms": [] } ] } ] }
```

| Field | Type | Notes |
|---|---|---|
| `standard` | string | `claims/0.1`. Readers SHOULD accept a missing field as 0.1. |
| `scope` | string | the package's scope |
| `version` | string | the package version the export was built from, if known |
| `claims[].id` | integer | |
| `claims[].ref`, `qualified` | string | `^N` and `scope^N` |
| `claims[].kind` | string | one of §4 |
| `claims[].title`, `does` | string | |
| `claims[].sees` | string or null | moves and flows |
| `claims[].place` | integer or null | the place's id, in the same scope |
| `claims[].retired` | string or null | the reason, for a tombstone |
| `claims[].statement` | string or null | the fingerprint, as in the ledger |
| `claims[].triggers` | string[] | moves |
| `claims[].keyed` | boolean | moves |
| `claims[].witness` | step[] or null | flows: `{ "trigger", "ref", "claim", "key", "op"? }` |
| `claims[].passes` | integer[] | flows: the move ids the witness passes through, in order |
| `claims[].machine` | string or null | flows and invariants: the machine's declaration |
| `claims[].check` | string or null | assumptions: the check's name |
| `claims[].pos` | `{file, line}` | where it's declared, relative to the package root |
| `claims[].decl` | string | the declaration's name in the implementation's language |
| `claims[].evidence` | object[] | proof evidence the build itself knows (§10.3); other evidence is published separately (§8.2) |

Rules:
- The export MUST contain every claim in the ledger's scope, tombstones included, and nothing
  from other scopes.
- Each claim's id, kind and statement MUST agree with the ledger.
- Implementations MAY add fields. An added field's name SHOULD contain a `.` (such as
  `acme.statementDecls`) unless this standard later defines it.
- Readers MUST ignore fields they don't know.

## 10. Conformance levels

An implementation states the level it conforms to, and a package states the level it was
checked at. Each level includes the ones before it.

### 10.1 Claims-Ledger

For packages that want durable ids and a public record of what they promise, before any runtime
support.

- Claims are declared with every part of §4 except handlers' `apply`, which MAY live elsewhere.
- The build-time checks of §4.7, items 1–6.
- The ledger (§6), including the fingerprint, and its checks in CI.
- The release rule (§5.3).
- The export (§9).

A Ledger-level package can be published and read on claim pages. Its claims read `untested`
until some evidence attaches.

### 10.2 Claims-Checked

The level most implementations should target. No proofs are needed.

Everything in Claims-Ledger, plus:

- **Dispatch** (§7.1). The update function is built from the moves, and its guarantees are
  enforced by construction or by a runtime check that fails development and test builds.
- **`post` checked at runtime.** In development and test builds, after every handler runs, its
  `post` is evaluated against the result. A false `post` is a failure that names the claim, the
  state and the action.
- **Invariants checked at runtime.** In development and test builds, `holds` is evaluated after
  every action and message.
- **Flows replayed on every build.** Each flow's witness is run against the machine. `live` and
  `reaches` must both hold, or the build fails. This costs one run per flow and is exactly as
  strong as a proof for that one path.
- **Refs in the UI** (§7.3).
- **A fuzzer**: random sequences of enabled actions (and messages, if the machine receives any)
  from the initial state, checking every `post`, every invariant, and that each move's control is
  present exactly when it is enabled. It produces fuzzed evidence (§8). The defaults SHOULD be at
  least 200 sequences of 50 steps, and a failure MUST print a replayable seed.

An implementation at this level SHOULD also provide a conformance driver. It drives the running
application by refs and compares its state or rendered output with the model step by step, and
produces tested evidence.

### 10.3 Claims-Proved (optional)

For implementations in a language with a proof checker. Lean, Coq/Rocq, Agda, Isabelle, F\*,
Dafny and Idris all qualify, and so do verifiers for mainstream languages such as Verus, Creusot,
Prusti or Kani for Rust, and Liquid Haskell.

Everything in Claims-Checked, plus:

- **Each handler's `sound`**, each flow's `live` and `reaches`, and each invariant's `init`,
  `step` and (if the machine receives messages) `receive` is a theorem the checker accepts.
- **The dispatch guarantees of §7.1 are theorems** about the implementation's dispatch, stated
  once in the framework for every application:
  - unclaimed actions are no-ops;
  - every effect comes from an enabled move;
  - dispatch through an enabled handler meets its `post`;
  - an invariant holds in every reachable state.
- **The proved code is the code that runs.** Either the proofs are about the same definitions
  the application executes, or there is a checked refinement from the executed code to the
  proved model. An implementation MUST NOT prove things about a separate model and ship unrelated
  code.
- **An axiom audit.** Every evidence theorem's dependencies are listed. The implementation
  documents the axioms its logic accepts as standard (for Lean: `propext`, `Classical.choice`,
  `Quot.sound`). A theorem that depends on anything else, such as an admitted lemma, `sorry`,
  a user axiom or a trusted external solver, MUST NOT count as proved. It is recorded in
  `extraAxioms` and reads `broken` if it was meant to be a proof.
- **Escape hatches are banned or reported.** Mechanisms that skip the checker MUST either fail
  the build or appear in the audit. In Lean these are `native_decide`, `implemented_by` and
  skipping the kernel; elsewhere they are trusted externs, `assume`-style statements and
  unchecked casts.
- **Assumptions are hypotheses, not axioms** (§4.4).
- **A trusted base is written down.** Everything the proofs stop at, such as the host, the OS,
  the renderer or the network, is listed as `assume` claims with checks where possible.

**Bounded verification is not proof.** A model checker or SMT run with a bound (Kani's unwind
limit, a bounded TLA+ check) MUST NOT produce proved evidence unless the bound is itself proved
complete. Report it as fuzzed with the bound in `role`, or as an extension evidence kind with a
`.` in its name.

## 11. Registries (optional)

A registry that stores packages with claims MUST, when a version is published:

1. Refuse a ledger that drops any id an earlier version's ledger had, or changes its kind.
2. Refuse a version that breaks the release rule (§5.3) against the version before it.
3. Refuse an export that disagrees with the ledger: a different scope, a missing id, a different
   kind or fingerprint, a live moved id.
4. Store and serve the ledger and export as data (JSON or text), and keep only the fields it
   knows.

A registry MUST NOT accept or serve HTML or script from a publisher. Claim pages are rendered by
the registry or its site, with every field escaped.

A registry SHOULD say plainly that it shows what the publisher sent. Unless the registry rebuilt
the package and re-ran the checks itself, proved and fuzzed evidence is the publisher's report.

A registry's read API SHOULD include:

| Request | Returns |
|---|---|
| `GET /api/v1/packages/<owner>/<name>/versions/<v>/claims` | the ledger as rows `{id, kind, status, statement, title}` |
| `GET /api/v1/packages/<owner>/<name>/versions/<v>/claims/export` | the export (§9) |

## 12. Security considerations

The checks in this standard catch an honest author's mistakes. They are not a defence against an
author who means to mislead. A hostile author can:
- state a false assumption;
- write a weak `post` (`post := true` proves easily and promises nothing);
- write prose that promises more than the formal statement;
- put behaviour outside the claims in code the implementation doesn't cover, such as a host, a
  shader or a native module.

So, for a package you don't trust:
- Read its statements, not its green ticks.
- List its `assume` claims and decide whether you believe each one.
- Read each claim's formal statement beside its prose.
- Build it yourself and run the checks, so the evidence is your checker's and not the publisher's
  report.

Implementations SHOULD print each claim's formal statement next to its prose in generated
documentation, and SHOULD make a changed fingerprint visible in review.

## 13. Not covered in 0.1

- **Cross-scope refs in formal parts.** A claim can mention another package's claim
  (`platform^1005`) only in prose. A flow can't step through another package's move.
- **Effects in `post`.** `post` constrains state. A later version may add an `effects` clause that
  constrains what a handler emits.
- **Presence and pressability as claims.** "^2's control is on screen iff `enabled`" is checked
  by fuzzers and may be proved per application, but isn't a standard part of a move yet.
- **Prose against formal statements.** Nothing checks that `sees` says what `post` says. That is
  review's job.
- **Server-side and non-UI claims.** The kinds here are about a person using an interface. An API
  endpoint could be a move with an HTTP trigger, but this version doesn't define one.

## Appendix A. A Lean binding sketch (informative)

A Claims-Proved binding in Lean 4 can make each kind a command, with the proof inline:

```lean
move ^2 reset (s : State) "Reset" in ^3 where
  does "Press Reset."
  sees "The count is zero, and Reset disappears."
  enabled := resetEnabled s
  on press
    apply := { s with count := 0 }
    post s' := s'.count = 0 ∧ resetEnabled s' = false
    sound := by intro s _; exact ⟨rfl, rfl⟩

flow ^5 startOver "Start over" for machine where
  does "Count up, then press Reset."
  sees "The count is back to zero and Reset is gone."
  goal s := s.count = 0 ∧ resetEnabled s = false
  witness press ^1, press ^1, press ^2
```

- The command generates `reset.press.sound : ∀ s, reset.enabled s = true → reset.press.post s
  (reset.press.apply s)`, and the build fails if the proof doesn't check. A flow's `live` and
  `reaches` default to `by decide`, which runs the witness in the kernel.
- The framework proves the dispatch guarantees once, for every application:
  - an unclaimed action changes nothing;
  - every effect comes from an enabled move;
  - an invariant holds in every reachable state.
- The axiom audit allows only `propext`, `Classical.choice` and `Quot.sound`, and a lint bans
  `native_decide`, `implemented_by` and skipping the kernel. Because Lean compiles the same
  definitions it proves things about, the proved code is the code that runs.

## Appendix B. A Rust binding sketch (informative)

A Checked-level binding for Rust might use a proc macro and a build step:

```rust
#[derive(Clone, Default, PartialEq, Debug)]
struct State { count: u32 }

fn reset_enabled(s: &State) -> bool { s.count > 0 }

claims::place!(^3, window, "Counter window", does = "The counter's window.");

#[claims::r#move(^2, "Reset", in = ^3,
    does = "Press Reset.",
    sees = "The count is zero, and Reset disappears.",
    enabled = reset_enabled)]
mod reset {
    use super::*;
    #[on(press)]
    fn apply(s: &State) -> State { State { count: 0, ..s.clone() } }
    #[post(press)]
    fn post(_s: &State, s2: &State) -> bool { s2.count == 0 && !reset_enabled(s2) }
}

claims::flow!(^5, start_over, "Start over", machine = MACHINE,
    does = "Count up, then press Reset.",
    sees = "The count is back to zero and Reset is gone.",
    goal = |s: &State| s.count == 0,
    witness = [press(^1), press(^1), press(^2)]);

static MACHINE: claims::Machine<State> = claims::machine![increment, reset];
```

- `cargo claims check` / `update` maintain `claims.lock`, fingerprinting `post`, `enabled` and
  `goal` through a normalised `syn` tree. The `#[test]` the flow macro generates replays the
  witness.
- `claims::fuzz` runs proptest over action sequences.
- The UI layer (egui, Dioxus, a web front end) sets `data-claim` or the accessibility id from the
  move's ref.
- For Claims-Proved, the same macros could emit Verus `ensures` clauses: `sound` becomes
  `apply`'s postcondition, checked by Verus with no extra axioms.

## Appendix C. A TypeScript binding sketch (informative)

```ts
import { move, place, flow, machine } from "@claims/core";

type State = { count: number };
const resetEnabled = (s: State) => s.count > 0;

export const window = place(3, "Counter window", { does: "The counter's window." });

export const reset = move(2, "Reset", {
  in: window,
  does: "Press Reset.",
  sees: "The count is zero, and Reset disappears.",
  enabled: resetEnabled,
  on: {
    press: {
      apply: (s: State): State => ({ ...s, count: 0 }),
      post: (_s, s2) => s2.count === 0 && !resetEnabled(s2),
    },
  },
});

export const counter = machine<State>({ init: { count: 0 }, moves: [increment, reset] });

export const startOver = flow(5, "Start over", {
  machine: counter,
  does: "Count up, then press Reset.",
  sees: "The count is back to zero and Reset is gone.",
  goal: (s) => s.count === 0,
  witness: [["press", 1], ["press", 1], ["press", 2]],
});
```

- A `claims` CLI imports the package's entry points, collects every declaration, and maintains
  `claims.lock`, fingerprinting the normalised source of `post`, `enabled` and `goal` with the
  TypeScript compiler API.
- `dispatch` in `@claims/core` is the only way to get an update function, so unclaimed actions
  can't exist.
- In development, every `post` and invariant runs after every dispatch.
- `@claims/fuzz` drives fast-check over action sequences.
- `@claims/react` puts `data-claim="^2"` on the element a move renders.
- `@claims/playwright` drives the running app by refs, compares with the machine, and writes
  tested evidence.

A TypeScript binding is unlikely to reach Claims-Proved directly. A package can get there by
writing its machine in a proof language and compiling it to JavaScript.
