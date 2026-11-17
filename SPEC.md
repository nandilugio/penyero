# penyero — product specification (draft)

Status: draft for discussion. Nothing here is implemented yet. Items marked **(verify)** are
assumptions that must be checked on a real machine before they are relied upon.

This document specifies **what** penyero is and does: its purpose, security model, and the
interfaces the agent and the human use. Two other documents cover the rest:

- [DESIGN.md](DESIGN.md): **how** it is built. Processes, protocol, technology, first iterations.
  It can change without changing this document.
- [SETUP.md](SETUP.md): one reference deployment around it: the isolated workspaces of
  [cayo](https://github.com/nandilugio/cayo), and penyero's own pieces (the approval pane, review
  tools, resources). penyero depends only on the deployment contract in §4.

## 1. Problem

AI coding agents work best with broad autonomy, but some operations need privileges that should
not be handed to them wholesale: querying production databases, calling cloud APIs, running
consoles on remote hosts, pushing code. The choice is roughly all-or-nothing: either the agent
holds the credentials (and can do anything with them, unobserved), or the human runs every
privileged command by hand and pastes the result back, which is slow and error-prone.

penyero mediates privileged operations: the agent requests, the human approves (or a rule
does), penyero executes with credentials the agent never sees, the human reviews the output,
and the result goes back to the agent.

## 2. Goals and non-goals

Goals:

- A **real boundary**: an agent that is careless, confused, or actively hostile cannot use
  privileged credentials except through approved requests.
- **Dependable approvals**: the human can tell what they are approving. Whatever makes a risky
  request easier to spot, or blocks it outright, serves this goal before convenience does.
- Low-friction approvals: most requests are reads, arrive in bursts, and should cost one key.
- Output review before results reach the agent.
- Small, POSIX-style, composable; macOS and Linux; minimal and stable dependencies (principles
  below).
- Independent of how the AI workspace is hosted (container, VM, separate OS user).
- Single user, forever.

Non-goals:

- **Protecting production from malicious code that passes code review.** penyero protects the
  developer's privileged context and mediates direct privileged actions. Code integrity is the job
  of review, CI, branch protection and deploy gates.
- Automating browser-only surfaces (cloud consoles, dashboards, admin UIs). penyero can
  route such a task to the human instead (§8, human resources).
- Network egress control. It is part of a guest's risk profile (§3) and the deployment provides
  it (§4); penyero may later act as its approval backend (§10).
- A team or multi-user product.

Principles, which every feature is weighed against:

- **One job.** penyero mediates privileged requests: approval, execution with credentials the
  agent never sees, output review, audit. Any other need is met by a separate tool that composes
  with it (the guest runtime, the egress proxy, the human's pager and editor, resource
  executables), not by growing penyero. A feature belongs inside only when it can't be done
  well from outside.
- **POSIX-style.** Processes, argv, stdin, stdout, stderr, exit statuses and files; text
  interfaces that compose with pipes.
- **Few, stable dependencies.** The standard library first. Any other dependency must be small,
  mature and likely to stay maintained, and is weighed against its upgrade and attack-surface
  cost.
- **Less code, less to trust.** The broker is the boundary's code; every feature in it is
  surface to review.

## 3. Threat model

### Parties

- **AI workspace** (untrusted): the agent **and all development tooling**, including the human's
  editor, LSP servers, tests, dev servers, package installs, git working copies and toolchains.
  The human does their coding *inside* it.
- **Guest**: the isolated environment the AI workspace runs in: a container, a VM, or a separate
  OS user. A guest is an instance of a **profile**, the named trust level that fixes what it may
  reach and how much confirmation its requests need (below, and §8).
- **Privileged side** (trusted): the human's own account on the host. It holds the secrets (cloud
  credentials, ssh keys, database credentials, OS keychain), runs the broker and its approval
  prompt, holds the credentials for pushing and deploying, and controls the guest runtime.
- **Broker**: the part of penyero on the privileged side. It receives requests, asks the human,
  runs resources, and keeps the audit log. One broker serves one guest.
- **Client**: the part of penyero inside the guest, the commands the agent runs (§6).

Why the human's tooling lives in the AI workspace: any tool that interprets project files can run
project code. LSP servers load project configuration, `node_modules` plugins, and virtualenv
interpreters.
Dev servers, test runners, installs, build and deploy scripts run it by design. Separating only
the agent from the human would still leave the human constantly running agent-written code with
full privileges. The split must follow *who executes project code*, not *who types*.

### Core rule

> The privileged side never executes, sources, or interprets anything the AI workspace can write:
> not code, not config, not build files, not paths. It handles workspace content only as inert
> data: bytes to display, diff, or transfer.

This rule is generic: it holds for any project, regardless of how vulnerable that project's dev
server, app or build scripts are. Anything that goes wrong inside the AI workspace stays inside it.

The one deliberate exception is **reviewed code at a pinned commit**: deploying it, running a
project's compose file from it, or using a script from it in a resource. That code was written in
the AI workspace, so running it with privileges relies on code review, which is the first
non-goal (§2), not on penyero.

### The guest is the security principal

Processes inside one guest are not isolated from each other: in a container or VM they share a
kernel and usually a uid; as a separate OS user they share a uid, and same-uid processes can read
each other's memory and files. So **isolation is per guest, never per session**. A rogue agent
session cannot be prevented from using another session's grant in the same guest, reading its
traffic, or acting as it. Consequences:

- **One guest per trust domain.** If two projects (or a trusted model and an experimental one)
  must not impersonate each other, they run in separate guests.
- **The broker must know which guest a request comes from without relying on anything the guest
  says.** No token, password or claim sent by the guest may serve as its identity; the guest must
  have nothing to forge. (DESIGN.md §3 achieves this by having the broker open the connection into
  the guest itself.) A token scheme would not help anyway: whatever an honest client could present,
  another process in the same guest can obtain.
- **The client sends nothing about itself.** No working directory, pid or agent name: such labels
  could only be trusted if the guest were, and then they would not be needed. A request carries
  the resource, its inputs, and the agent's stated reason, which is judged, not trusted.
- What a guest may request, and with how much confirmation, is fixed by its profile (§8).
- **The client cannot authenticate the relay.** Another process in the guest can replace the
  relay's socket and answer clients itself: read their requests, feed them fabricated results.
  That is the guest lying to itself, and it is accepted like every other within-guest attack.
- **The broker treats every byte from the guest as hostile input**, and stays bounded: it accepts
  only messages that match its schema, limits the size of a request's payload and the number of
  requests a guest may have outstanding, and rejects the rest. Nothing a guest sends can name a
  path, choose a file, or select anything on the privileged side.
- Anything configured inside the guest (the agent's permission settings, its instructions) is
  convenience, not a security control. The boundary is the guest itself.

### Task risk and profiles

Prompt injection is unsolved: an agent that reads text an attacker controls may follow
instructions in it, and no filter reliably prevents that. The more capable the agent, the more it
can do with one injected instruction. So the design assumes the agent *will* eventually act on
hostile input, and limits what that can reach.

The framing is Meta's *Agents Rule of Two*, itself inspired by Willison's *lethal trifecta*. Meta
names three properties: [A] processing untrustworthy inputs, [B] access to sensitive systems or
private data, [C] changing state or communicating externally; an agent may hold at most two
without a human supervising. This document uses them with two refinements from the discussion
that followed: Meta's authors clarified that B covers access to *any* sensitive system, so a
state change that matters already needs B; and Willison pointed out that untrusted input with the
ability to change state is harmful on its own. So write access belongs with B, C keeps only the
communication half, and the rule splits by the harm it prevents.

- **A. Untrustworthy input**: text or code an attacker may have written. Public issue trackers,
  pull requests, arbitrary web pages, dependency sources, hostile samples.
- **B. Sensitive reach**, through a credential in the guest or a penyero resource it may
  request, in two halves:
  - **B-read**: reading private data or sensitive systems, private source included.
  - **B-write**: changing their state: pushes, PR comments, ticket edits, prod writes.
- **C. Egress**: sending data out of the guest at all. With unrestricted egress, any leak needs
  nothing but `curl`.

Rules:

- **Integrity: A with B-write needs a human.** An injected agent with write access does damage
  without leaking anything. So every write is approved by the human: a resource with a write
  credential is never auto-approved (§8), and the guest holds no write credential to a sensitive
  system.
- **Confidentiality: A, B-read and C together need a human.** That is the lethal trifecta:
  injected instructions read private data and send it out. One leg is broken: outputs are
  reviewed before they reach the guest (`confirm_output`), egress goes, or the read access does.

The guest's **profile** is the concrete choice of B and C for a given kind of A. Profiles are
named in the configuration (§8); a deployment implements each one as a guest shape (image,
mounted credentials, egress policy). The reference set:

| Profile | A: input | B-read | B-write | C: egress |
|---|---|---|---|---|
| `trusted` | The human's own projects and vetted sources | Resources; read-only ones may auto-approve and release unreviewed | Through penyero, asked | Allow-list |
| `public` | Open-source work: public issues, PRs, general web | Resources only, no credentials in the guest; every request and output asked | Through penyero, asked | Allow-list |
| `hostile-web` | Content that may target automated readers | **Nothing**: no credentials, no resources | Nothing | Broad to the internet, needed by the task; no host or local network |
| `hostile-sample` | Samples, exploits, CTF material | Nothing | Nothing | **None** |

Three consequences worth stating:

- **Open-source work is not the safe middle.** Reading strangers' text (A) while holding write
  credentials to one's own repos is the integrity case outright, and with egress any private
  source in reach is the confidentiality case too. `public` keeps every write behind approval,
  no credential in the guest, and every output reviewed.
- **In `trusted`, the allow-list limits A as well as C.** What the agent reads from the network
  (package pages, raw files on a code host) is strangers' text too, and any allowed host that
  accepts uploads from any account is a way out. Unattended prod reads (`confirm_output = false`)
  are only as safe as that list: keep hosts that store data off it, or review the output.
- **Hostile content is handled by removing B, not by trusting filters.** For `hostile-web` the
  task needs broad egress, so the guest holds nothing worth taking; the host and the local network
  stay out of reach, since services there are something worth taking too. For samples, egress
  goes too. What A with C can still do is harm others (abuse sent from the guest), which, as in
  Meta's rule, is out of penyero's scope.

The same project may need guests of different profiles: developing it in `trusted`, triaging its
public tracker in `public`.

### penyero's own code

- The broker is installed from a privileged-owned source (for example a clean clone at a reviewed
  tag). It must never run from a copy the AI workspace can write. This matters because penyero
  itself will be developed in an AI workspace.
- The client is untrusted like everything else in the guest. Its version and integrity don't
  matter: whatever runs there speaks for that guest by definition.

### What the boundary protects

- Secrets on the privileged side (keychain, `~/.aws`, `~/.ssh`, database credentials, browser
  sessions, other projects, personal files).
- Unmediated privileged actions: an agent running cloud CLIs with the human's credentials, or
  pushing against instructions, becomes structurally impossible rather than a matter of
  obedience.

### Residual risks (accepted or mitigated elsewhere)

- Malicious code that passes review (non-goal).
- A resource executable that runs its input on the host, or lets it pick or reveal a credential.
  The executables are the human's; §8 states what they must guarantee.
- Anything the AI workspace can reach over the network with credentials it legitimately holds or
  finds in its own files. Secrets must not be placed in the guest.
- **Known issue: the agent's own credential.** Every guest that runs an agent holds the agent's
  login or API key, including profiles that otherwise hold nothing. Use a dedicated key with a
  spending limit for untrusted profiles. In every profile, a login to an account carries whatever
  else that account has: hosted connectors (§5) reach the agent with it, unmediated.
- An egress allow-list narrows C but doesn't close it. Any allowed host that stores data for
  whoever authenticates (a model provider's API with an attacker's key, a package registry, a code
  host) can carry data out. That is why `public` reviews every output rather than relying on its
  allow-list.
- Network services the guest can reach: dev services (which hold dev data only, and must have no
  route out of their own, since the guest can often run code in them) and, in deployments that
  don't block it, services listening on the host. The reference deployment blocks it (SETUP.md
  §1); a guest that is a separate OS user reaches the host's loopback like any local process.
- **What a request tells the agent before its output is reviewed.** `confirm_output` reviews
  content. Whether a request succeeded, failed without output, or is still running, and how long
  it took, reach the agent unreviewed. With `confirm_request = false` an agent can turn that into
  a slow channel (a query that fails or not depending on a value) with no approval in its way,
  though each probe shows in the pane.
- Escape from the guest through a runtime or kernel vulnerability. The strength of this boundary
  is a deployment choice. One class deserves naming, because penyero triggers it: running a
  program inside a hostile guest (contract item 2) is what container-escape flaws such as
  CVE-2019-5736 in runc exploited. Keep the runtime patched, and prefer runtimes that put a VM
  around each guest (SETUP.md §1 and §8).
- The human being misled by what the agent shows them. Mitigated by rendering request and output
  content safely (§7), by keeping approvals specific, and later by checkers (§10). Look-alike
  letters from another script (a Cyrillic `а` in a table name) can't be escaped away; a request
  that holds any non-ASCII character is flagged (§7), and the rest is the reviewer's eye.

## 4. Deployment contract

penyero assumes the deployment guarantees the following. Everything else about the setup is
free to change.

1. **Separation.** The AI workspace cannot read or write the privileged side's secrets, penyero's
   configuration, or its state.
2. **A way to run a program inside a guest.** The privileged side has a command that starts a
   process inside a given guest with its stdin and stdout connected to the caller, for example
   `docker exec -i <name>`, `container exec -i <name>`, `ssh <host>`, or `sudo -u <user>`.
   penyero uses it to reach the guest (`guest_exec`, §8; DESIGN.md §3).
3. **No path back.** The guest has no way to act as the privileged side: no sudo to it, no
   credentials for it, no access to the guest runtime's control socket, no shared terminal
   device.
4. **One guest per trust domain** (§3).
5. **Profiles are real.** A guest of a given profile holds only the credentials its profile
   allows (§3), and its egress is restricted to what the profile allows, denied by default where
   the profile says so. penyero cannot check this; the deployment guarantees it.

## 5. Capability placement

Every privileged capability is placed by one rule:

> **Credentials define capability; penyero rules define convenience.**

There are two ways to give the AI workspace a capability:

- **A penyero resource** (§8). The credential stays on the privileged side; the agent invokes it
  through the client. Each profile decides whether it asks or auto-approves, and whether the
  output is reviewed. Every use is audited.
- **A native grant**: a token in the agent's own environment. Nothing stands between the agent
  and the token, so this is only acceptable when both hold:
  1. The service enforces the scope server-side (a read-only token, a read-only DB role).
  2. The confidentiality impact of that scope is acceptable if fully exercised, in every profile
     the token is present in.

**Prefer a resource for anything a command-line tool can do**, even read-only and even
auto-approved: the credential never enters the guest, so it cannot leak; the resource is
configured once and each profile sets its own confirmation; and its use is logged. A resource
with auto-approval still exposes the *data* it returns to the guest, so condition 2 applies to
it as well. Native grants remain for tools that cannot go through the client, such as git's
credential for `fetch` and `pull`.

Auto-approval is a per-resource setting of each profile, never a judgement of the request's
content ("this SQL is a read"). To auto-approve less, narrow the credential.

Initial placement (to be confirmed per service):

| Capability | Placement | Notes |
|---|---|---|
| Git upstream read | Native, read-only token | git needs it non-interactively. A credential helper that requests it through penyero makes it short-lived and audited per use (SETUP.md); it still enters the guest |
| Git push | Privileged side only | Feeds the deploy pipeline; never native, never a resource |
| PR/CI read (code host) | Resource, read-only token, auto-approve eligible | |
| Error tracker read | Resource, read-only token, auto-approve eligible | Errors may contain PII: condition 2 per profile |
| Issue tracker read | Resource, read-only token if one exists **(verify)** | |
| Issue tracker write | Resource, always ask | Low blast radius |
| Team chat (any) | Resource, always ask, or not at all | Even reads are high-impact |
| Prod DB, read-only role | Resource, auto-approve eligible | Role enforces read-only; bound query cost (§8) |
| Prod DB, read-write | Resource, always ask | |
| Cloud CLI | Resources per credential | e.g. a read-only policy vs an admin one |
| Consoles and shells on remote hosts | Session resources (§8), always ask | Full power once inside |
| Browser-only surfaces | Human resource (§8) | The human performs the step and returns the result |

Hosted connectors (an assistant's cloud integrations) run outside the machine and request
whatever scopes the connector defines, often including write. They cannot be mediated from the
host at all; the only control is whether the account has them.

## 6. Agent interface

The client is one program, `penyero`, used inside the guest. Every operation is asynchronous:
a request is submitted, waited for, and retrieved, in three separate commands. Each command does
one thing, so any of them can be composed with other tools without ambiguity.

- `penyero list`: the resources this guest may request.
- `penyero describe <resource>`: its description, what input it takes, and whether it is
  auto-approved for this guest. (An agent would learn that from its first request anyway; told,
  it can say which requests need the human.)
- `penyero request <resource> -r <reason> [-- args...] [< payload]`: submit a request. Prints
  the request id on stdout and exits at once.
- `penyero wait <id>... [--timeout <seconds>]`: block until every listed request is settled (its
  result is ready, it failed, was denied, or was dropped). Without `--timeout` it waits
  indefinitely. Once all are settled it exits 0 and prints one line per request, in the order
  given: `<id> ready`, `<id> failed`, `<id> denied` or `<id> dropped`, which is what `retrieve`
  will answer (0, 1, 251, 249). So the agent knows each outcome before it pipes a result anywhere.
- `penyero retrieve <id>`: write the result to stdout. Never blocks: if the request is not
  settled, it exits with a distinct status.
- `penyero pending`: this guest's requests not yet retrieved, with resource and state.
- `penyero cancel <id>`: withdraw a request that has not started (waiting for approval or for
  a free slot of its resource, §8). It is discarded.

```sh
id=$(penyero request prod-db-ro -r "count overdue orders by region" <<'SQL'
SELECT region, count(*) FROM orders WHERE status = 'open' GROUP BY region
SQL
)
penyero wait "$id" --timeout 100
penyero retrieve "$id" > counts.csv

a=$(penyero request aws-readonly -r "find restarts" -- logs filter-log-events ...)
b=$(penyero request prod-db-ro -r "recent refunds" < refunds.sql)
penyero wait "$a" "$b" && penyero retrieve "$a" | jq ... && penyero retrieve "$b"
```

- `-r <reason>` is required. It is shown to the human and written to the audit log.
- Inputs travel **by value**: the payload on stdin and extra arguments after `--`, each only if
  the resource declares it (§8). A file the agent wants to use (a
  `.sql` file or a Python script) is read by the *client* and sent as content. The broker never
  opens workspace paths, and never writes into them: the agent redirects `retrieve`'s stdout where
  it wants it.
- `wait` with a timeout exists because agent shell tools have hard timeouts (two minutes by
  default in some, ten at most). A timed-out `wait` changes nothing: the agent waits again.
- A client that exits does not cancel its request. `pending` recovers ids the agent lost.
- **No order between outstanding requests.** Requests outstanding at the same time may run
  concurrently and finish in any order, whatever their resource: the same contract as concurrent
  HTTP calls. An agent that needs one request's effect before another's waits for and retrieves
  the first, which also lets it act on the outcome (the first may be denied or fail).
- **Request ids are opaque and random** (for example `k7f2qa-3mxp9dq2vt`). They are long enough
  that in practice they don't repeat for a guest, not even across broker restarts, and they say
  nothing about how many other requests exist. An id from before a restart is reported as lost
  (248), not confused with a newer request.
- **A result is kept until it has been retrieved completely, then discarded.** Completely means
  the client wrote the last byte to its stdout without error and reported that to the broker; a
  broken pipe leaves the result in place for another `retrieve`. An agent that needs a result
  again keeps its own copy.
- A request runs exactly as submitted, or not at all: the human approves or denies it, and a
  denial's message says what to change. If the human changed the output (redacted it, or wrote an
  answer into it), `retrieve` says so on stderr, always, so the agent does not take withheld data
  for absent data. stdout carries only the result. The original output is never sent, and is
  kept nowhere (§9).

**Exit status.** Resource executables never talk to the agent through exit codes (§8), so the
client's own codes cannot collide with theirs:

| Code | Meaning | Commands | The agent should |
|---|---|---|---|
| 0 | Success. For `retrieve`: the resource succeeded and its output is on stdout | all | |
| 1 | The resource failed. stdout carries whatever the resource chose to tell the agent; if the broker stopped it at its timeout or the output cap (§8), stdout holds what it wrote until then and stderr says which, and that the output may be partial | `retrieve` | Read stdout |
| 248 | Lost: the broker restarted before the request was retrieved. stderr says whether it had started | `wait`, `retrieve` | Check its effects before repeating it, if it may have run |
| 249 | Dropped: the broker connection was lost before the request ran; nothing happened | `retrieve` | Request again |
| 250 | Not settled (`retrieve`), or timed out (`wait`) | `wait`, `retrieve` | Wait again |
| 251 | Denied by the human: the request, the release of its output, or the request while it ran (killed, §7), with their message on stderr | `retrieve` | Not repeat it as is; if it was killed while running, check its effects |
| 252 | Unknown id: never existed, already retrieved, or cancelled by the agent | `wait`, `retrieve`, `cancel` | Nothing to fetch |
| 253 | Invalid request: a malformed command line, unknown resource, undeclared input, missing reason, payload too large, too many outstanding requests, or `cancel` on a request that has started | all | Fix the call, or retrieve or cancel outstanding requests |
| 254 | Broker not connected | all | Retry once it is back |

Codes 248–254 stay clear of what a shell reports around a command: 2 for misuse, 126 and 127 for
a command that can't run or isn't found, 128 plus the signal number for one that was killed, and
255, the highest status there is, which ssh among others uses for its own failures. So a status
in this range always comes from penyero. Every non-zero exit comes with a one-line
`penyero: ...` message on stderr.

**When the broker connection is lost** (guest restarted, broker stopped):

- Requests that have **not started** are dropped: nothing ran, and `retrieve` reports 249 once the
  connection is back, so the agent can request again.
- Requests that are **running, awaiting release, or settled but not retrieved** may already have
  had effects (a write on a write resource). They are kept, flagged to the human as "not
  retrieved", and the agent can still `retrieve` them once the connection is back.
- If the broker itself restarts, outstanding requests are lost: `wait` and `retrieve` answer 248
  for their ids. The audit log (§9) records how far each one got. Requests are not persisted: a
  crash is rare, and a deliberate restart (a configuration change, an upgrade) asks the human to
  confirm when requests are outstanding, and stops the running ones.

## 7. Approval

Each guest has its own approval prompt, on the terminal where the human runs
`penyero serve <profile> <name>`. What is specified is the interaction: which requests wait for
the human, in what order, what is shown and which decisions exist. It doesn't depend on how the
pane is drawn. The first front end is line-oriented, like `git add -p`: the cheapest to build, and
one that tests can drive through a pipe. Review has to be comfortable, because a cumbersome one
gets approved unread, so a full-screen front end may replace it once the core is in use (§10).
Layouts and wording in this section are examples of the line-oriented one.

```
[3mxp9dq2vt] trusted/shop · prod-db-ro               running: aws-readonly 1 · waiting: 1 run
     reason: count overdue orders by region
     SELECT region, count(*) FROM orders WHERE status = 'open' GROUP BY region
run? [y]es [Y]es+release [n]o(+msg) [s]kip [v]iew [r]unning [?]
  · [k7f2qa01ab] aws-readonly · exit 0 · 12 lines · released (auto)
[3mxp9dq2vt] run? [y]es [Y]es+release [n]o(+msg) [s]kip [v]iew [r]unning [?]
...
[3mxp9dq2vt] trusted/shop · prod-db-ro · exit 0 · 214 lines · 6.1 KB · 1.4 s   waiting: 1 run
     reason: count overdue orders by region
    request: SELECT region, count(*) FROM orders WHERE status = 'open' GROUP BY region
     region,count
     eu-west,412
     ... (first/last lines)
release? [y]es [n]o(+msg) [v]iew [e]dit/redact [r]unning [?]
```

- The guest (`profile/name`) is established by the broker and is authoritative. The reason is the
  agent's text and is shown as such.
- **One prompt at a time, many requests running.** Approved requests run concurrently, up to
  each resource's `concurrency` (§8); the human answers one prompt at a time. `Y` approves and
  releases the output without a second prompt, for requests whose result the human does not
  need to see.
- **Prompt order.** Approval prompts come before release prompts: approving costs one key and
  starts work, so a burst is approved first and runs while results are reviewed. Among release
  prompts, those that complete a blocked `wait` (every other id it waits for is settled) come
  first, since that result is what unblocks an agent; the rest follow in arrival order. `s` moves
  a prompt to the back of its kind. This order needs no knowledge of which agent sent what, and
  it can only reorder the guest's own prompts, never change what is shown or decided.
- **A request is seen in full before it is approved.** Its payload and arguments are shown whole
  when they fit the preview. When they don't, `y` and `Y` are refused until the request has been
  opened with `v`: a line buried in the middle of a long request is exactly what a hostile agent
  would hide. Outputs need no full view before release; `v` shows them whole. Whatever the front
  end, content is wrapped, never clipped at the pane's edge, and anything left out is marked as
  left out.
- **A request is approved or denied, not edited.** The agent's request runs as written; `n` with
  a message ("add a LIMIT") costs the agent seconds and leaves it knowing what ran. Only outputs
  are edited (below).
- **A prompt never changes under the human.** The active prompt ends with their input. When its
  request is cancelled by the agent or dropped with the channel, the prompt stays on screen,
  marked withdrawn with the reason, still viewable with `v`, and the next one is drawn only after
  the human continues with a key that decides nothing (Enter). Otherwise the agent, which chooses
  when to cancel, would choose the moment another request appears under a key meant for this
  one. A request withdrawn while it waits behind the active prompt just gets an entry. Keys typed
  before a prompt is drawn are discarded, so each key answers a prompt that was on screen. In a
  front end that lists requests, a decision applies to the request in focus, and the focus never
  moves by itself.
- **Everything that runs is shown.** Requests that need no confirmation, results released
  without review, failures and channel events each get a one-line entry. One that arrives while a
  prompt waits is printed, and the prompt line repeated with its request id; while the human
  types a message or works in the pager or editor, entries are held until they finish.
- **Each prompt header carries the state**: requests running per resource, and prompts waiting
  of each kind. A resource with `write_credential` (§8) is marked as a write, so a write never
  looks like a read, and one approved while another on the same resource still runs is visible as
  such.
- **Release prompts repeat their context**: reason and request above the output, since a result
  can arrive long after its approval.
- The resource's stderr (§8) is never sent to the guest. Its tail is shown with the request's
  release prompt or completion entry, and all of it through `view` and the running view (below)
  while the request exists; the audit log keeps its tail (§9).
- Notification is configurable (`notify`, §8): a terminal bell, a command, or nothing. It fires
  when the pane goes from idle to having a prompt, not for each queued one.
- **Everything that didn't come from the configuration is rendered escaped**: requests, reasons,
  outputs, a resource's stderr, the channel's stderr. Escaped means every character that is not
  printable is shown as its code (`\x1b`, `‮`), and so is every byte that isn't valid UTF-8.
  That covers more than the control characters that move the cursor, hide lines, restyle the
  prompt or query the terminal: Unicode's format characters are invisible and reorder or join
  what is displayed (a right-to-left override makes a query read differently from what runs), so
  they are escaped too. The test is the standard library's (`str.isprintable`), not a list kept
  here. It holds for outputs as for requests: a stored value an attacker wrote must not disguise
  what is being released.
- **A request with non-ASCII characters is flagged**, with their count, in its prompt. Printable
  letters of other scripts can pass for Latin ones, and no escaping tells them apart; in SQL or a
  command line they are rare enough for a flag to mean something. Outputs are not flagged:
  names and text in them are ordinary.
- The inline prompt shows the content as it is, escaped, never reformatted: a view rebuilt from
  parsed content can differ from what is approved (a JSON object with a duplicate key shows only
  one of them). The prompt shows sizes, and content longer than its preview as head and tail with
  what is omitted. Output is limited only by a per-request cap (a resource that exceeds it fails,
  telling the agent to narrow the request); request payloads are capped too (§3).
- `view` opens the content in the review pager, from a privileged-owned temp file whose
  extension comes from the resource's `input_ext` / `output_ext` (§8). `edit`, for outputs, does
  the same with the review editor. Both are argv lists in the configuration (`[review]`, §8), run
  without a shell, with the file's path appended as the last argument, and with the fixed
  environment (§8) plus `TERM` and `COLORTERM`: variables exported where `serve` started (`LESS`,
  `LESSOPEN`, `VIMINIT`) can't change how they treat the content. The defaults work out of the
  box and are safe as they are:
  - pager: `less -+r -+R --no-lessopen`: raw control characters off, input preprocessor disabled.
  - editor: `vi -u NONE -i NONE -n -c "set nocompatible nomodeline"`: no vimrc or plugins, no
    viminfo, no swap file, modelines off, normal vim editing. `vi` because that is the only name
    vim has on minimal installs (Debian and Ubuntu's vim-tiny, Fedora's vim-minimal).

  SETUP.md describes richer choices.
- The two settings may name the same program. They are separate because viewing is the common
  case and deserves the smallest surface (a pager interprets nothing and streams any size), and
  because a viewer can't change what is being approved. An **edit is a change in content**: a
  file saved unchanged counts as a view, and the agent gets no edit notice for it.

### Running requests

`r` at any prompt lists the running requests, with what helps decide whether one is stuck: time
running, output size so far, time since it last wrote anything, and the last line of its stderr.
(Many tools buffer their output when it isn't a terminal, so a long silence is a hint, not
proof.) Selecting one offers:

```
running:
  1 [k7f2qa01ab] prod-db-ro · 4m12s · 0 B out · silent 4m12s · stderr: (none)
  2 [9dq2vtx0mm] aws-readonly · 38s · 1.2 MB out · silent 2s · stderr: Retrying (3/5)...
which? 1
[k7f2qa01ab] [o]utput [e]rrors [k]ill(+msg) [b]ack
```

- `o` and `e` open the review pager on a snapshot of the stdout or stderr written so far; again
  for a newer one.
- `k` kills the request, with an optional message as for `n`. It is a denial of a running
  request: its partial output is discarded, and the agent gets 251 with the message and a note
  that the request was stopped while running, so it may have had effects.
- `b` returns to the prompt queue.

### Review surfaces run as privileged over hostile content

Whatever displays a request or output runs on the **privileged side** over **agent-authored**
text, exactly the content the core rule says privileged tools must not interpret. A painful
review is itself a risk, because that is where a sneaked-in change gets past a tired reviewer.
Comfortable review and small attack surface pull in opposite directions, and the rule that
settles it:

> penyero itself renders untrusted content only as escaped plain text, with the standard
> library and no parser. Any richer view (syntax colour, diffing, folding) is delegated to an
> external tool the human chooses, which works on a privileged-owned *copy* of the content and
> must never execute anything from it. penyero's own surface stays small and auditable; the
> richer tool's surface is the human's explicit choice.

For those external tools: no LSP, ever (LSP servers execute project code); syntax highlighting is
a small, accepted risk (in-process parsers of untrusted input, with a history of memory-safety
bugs, but no code execution) when the tool's configuration is privileged-owned and plugin-free.
The same rule applies to checkers (§10): external executables, not in-process parsers.

## 8. Configuration, profiles and resources

### Configuration file

`~/.penyero/config.toml` (privileged-owned, mode 0600). Everything that defines the human's
privileged capabilities lives under `~/.penyero` (mode 0700): the configuration, the resource
executables by convention, and the audit logs (§9). One tree to protect, inspect and back up, as
with `~/.ssh`. The broker refuses to start if the config file or any resource's executable (the
first element of its `run`), or any directory above either of them up to the home directory, is
group- or world-writable or not owned by the privileged user: a writable directory would let
someone swap the file. Links are resolved first, so the check is on the real files. A leading `~/` (or a bare `~`) is replaced with the home directory, by the broker when
it loads the file, in every path and every element of an argv list; nothing else is expanded, since
no shell is involved.

The file declares **resources** (what exists) and **profiles** (who may use what, with how much
confirmation). Guests are not in the file: a guest is an instance of a profile, created by the
deployment and named when the broker starts (`penyero serve <profile> <name>`).

```toml
notify = "bell"            # when the pane gets a prompt (§7)

[review]                   # the pager and editor for `view` and `edit` (§7)
pager = ["less", "-+r", "-+R", "--no-lessopen"]
editor = ["nvim", "--clean", "-u", "~/.penyero/review.lua"]

[resources.prod-db-ro]
description = "Production Postgres (read-only role). Bound queries on large tables by time."
run = ["~/.penyero/resources/pg", "prod-ro"]   # one executable, the credential named here
input = "stdin"            # takes a payload: the SQL
input_ext = "sql"
output_ext = "csv"

[resources.prod-db-rw]
description = "Production Postgres (read-write role)."
run = ["~/.penyero/resources/pg", "prod-rw"]
input = "stdin"
input_ext = "sql"
output_ext = "csv"
write_credential = true

[resources.aws-readonly]
description = "AWS CLI with the read-only role. Pass the aws arguments after --."
run = ["~/.penyero/resources/aws-readonly"]
args = true
output_ext = "json"

[resources.ask-human]
description = "A task for the human: a dashboard query, a value off a console, a question. Payload: the task."
run = ["~/.penyero/resources/ask-human"]   # echoes the task back (§8, human resources)
input = "stdin"

[profiles.trusted]
guest_exec = ["docker", "--context", "colima-cayo", "exec", "-i", "cayo-{profile}-{name}"]
[profiles.trusted.resources]
prod-db-ro = { confirm_request = false, confirm_output = false }
prod-db-rw = {}
aws-readonly = { confirm_request = false }
ask-human = {}

[profiles.public]
guest_exec = ["docker", "--context", "colima-cayo", "exec", "-i", "cayo-{profile}-{name}"]
[profiles.public.resources]
prod-db-ro = {}            # every request and every output is confirmed
aws-readonly = {}

[profiles.hostile-web]     # no resources: no broker runs for such guests
[profiles.hostile-sample]
```

### Resolution

A resource is available to a guest if it appears in its profile's `resources` table; the value
holds that profile's settings for it. `confirm_request` and `confirm_output` default to `true`.
The broker validates at start:

- `confirm_request = false` is only allowed for resources without `write_credential`. Otherwise
  writes would run unapproved. This is the integrity rule (§3) held by the broker, not by the
  memory of whoever edits a profile months later. Releasing output unreviewed is a
  confidentiality choice and is allowed for any resource.
- A profile entry for a resource that does not exist is an error.
- `guest_exec` must contain both `{profile}` and `{name}`. With the name, two guests of one
  profile cannot resolve to the same command. With the profile, a broker started for the wrong
  profile finds no guest: penyero can't check what a guest really is (§4, item 5), but the
  deployment names guests by profile, so `serve trusted shop` can't reach the `public` guest
  `shop` and hand it `trusted`'s resources. The template should keep the two apart, so that no
  two pairs give the same command (the reference deployment allows no `-` in names).

### General settings

| Field | Meaning |
|---|---|
| `notify` | `"bell"`, `"none"`, or an argv list, run when the pane goes from idle to having a prompt (§7). Default `"bell"`. It describes the human's terminal, so it is one setting, not one per profile |
| `[review]` | `pager` and `editor`, argv lists (§7) |

### Profile fields

| Field | Meaning |
|---|---|
| `guest_exec` | The command that runs a program inside a guest (§4, item 2), as an argv list with `{profile}` and `{name}` substituted; penyero appends its own program name and arguments. Required for a profile with resources |
| `resources.<resource>` | Makes the resource available to guests of this profile. Keys: `confirm_request` (approve before running), `confirm_output` (approve before releasing the output); both default `true` |

Whether to trust a resource unattended is a property of the profile, not of the resource, which
is why the confirmation settings live here.

### Resource fields

| Field | Meaning |
|---|---|
| `description` | Shown to the agent by `list`/`describe` and to the human in the prompt |
| `run` | An argv list: the privileged-owned executable and any fixed arguments, such as the name of the credential to use, so one executable can serve several resources. The agent can't change them; its own arguments follow them |
| `input` | `"stdin"` if the resource takes a payload. Omitted: no payload accepted |
| `args` | `true` if the agent's extra arguments after `--` are passed as argv, after `run`'s own. Default `false` |
| `input_ext`, `output_ext` | The file extension of the temp file `view` and `edit` open, which is how the pager and editor pick a syntax |
| `write_credential` | `true` if the resource's credential can change state. Default `false`. The human's declaration of what the credential enforces: such a resource is never auto-approved (above) and is marked as a write in its prompts (§7) |
| `concurrency` | How many of its requests may run at once, per guest: a broker knows only its own guest. Default `1`. Approved requests beyond it wait for a free slot. It bounds what one guest can ask of the service; a limit across guests belongs to the service (a connection limit) or to the executable (a lock file). It is not an ordering guarantee (§6) |
| `timeout` | Seconds a request may run before the broker stops it. Default `300`: long enough for a slow query, short enough to bound a broken setup. Only execution counts, not waiting for approval or a slot. A stopped request fails with the output it wrote so far (§6, exit 1) |

### The resource contract

A resource has one audience on each side, and the broker keeps them apart:

- **stdout is for the agent.** Whatever the resource writes there is the result, released to the
  guest after review (or at once, if the profile says so).
- **stderr is for the human.** Diagnostics, the wrapped tool's own errors, anything that mentions
  hosts, users or paths. Its tail is shown in the approval pane and all of it on request (§7), its
  tail is written to the audit log, and it is never sent to the guest.
- **Exit 0 is success; any other status is failure.** The status itself is not forwarded. A
  resource that wants the agent to know *why* it failed writes that to stdout before exiting
  ("syntax error at line 3"), and leaves out what the agent has no business knowing ("connection
  refused to prod-db-3.internal").

Resource executables are therefore wrappers written for this contract, not stock tools exposed
directly. They may be shared, and penyero may ship some, but each one is the human's choice.

### Resource executables

A resource is an executable owned by the privileged user. It is executed directly (never
through `sh -c`), as its `run` list. It receives the payload on stdin and, if `args = true`, the
agent's extra arguments appended to that list. The agent sets nothing else: no option of the
executable's own unless it passes them on, and no part of its environment. It fetches its own
secrets with whatever the OS provides (macOS `security`, Linux `secret-tool`, `pass`, `op read`,
...). One way in for the agent's words keeps the executable simple to review: one that wants
named inputs parses them from its arguments.

**The environment is fixed, not inherited.** Everything the broker starts (resources, the review
pager and editor, `guest_exec`, a `notify` command) gets `PATH`, `HOME`, `USER` and the locale
variables, and nothing else from the shell that started `serve`. Many tools let the environment
choose their credential and configuration (for the AWS CLI, variables like `AWS_ACCESS_KEY_ID`
override its credentials file; libpq reads `PG*` variables), so an admin key exported there for
some unrelated task would otherwise reach every resource, and a read-only one that auto-approves
would silently run with it. A resource that needs more (an agent socket, a tool's own variables)
sets it itself.

Resource executables must reference only privileged-owned files. A script taken from a project
(for example a repo's `bin/console`) is used from a privileged clean clone at a reviewed
commit (§3, the exception to the core rule), never from the AI workspace.

### Resources run agent input on the privileged side

A resource runs on the privileged side, and its payload and arguments are written by the agent.
So the executable is exactly the kind of program the core rule (§3) is about, and it must hold
three properties. These protect the host and the credential itself; they don't classify
requests as reads or writes, so they don't conflict with §5.

1. **Nothing in the input runs on the host or touches its files.** Many clients interpret part of
   their input locally, and handing them agent input directly gives the agent a shell as the
   privileged user, or the privileged user's files:
   - psql's meta-commands: `\!` runs a shell command, `\o |cmd` and `\copy ... TO PROGRAM` pipe to
     one, `\i`, `\o file` and `\w` read and write host files. `-c` doesn't help: a single
     meta-command is accepted there too.
   - The AWS CLI reads any parameter written as `file://<path>` from the host, and `aws s3 cp`
     writes to host paths.
   - Database shells that embed a scripting runtime run any input as code in that runtime.
   - Local interpreters generally: `python`, `node`, `sh`, a local `manage.py shell`.

   Either use a client that only forwards the input to the service (for SQL, a short script built
   on a database driver: it reads the query, sends it, writes CSV), or confine the tool: run it in
   a disposable container that holds only this resource's credential and mounts nothing from the
   host. A console that runs remotely (a Django shell on a cloud task, reached through
   `bin/console`) executes its input on the remote side, which is the resource's purpose.
2. **The input can't select another credential or configuration.** Pin them in the executable,
   so that none of the human's other credentials are reachable: for example
   `AWS_CONFIG_FILE` and `AWS_SHARED_CREDENTIALS_FILE` pointing at files holding only this
   resource's profile, since otherwise `--profile admin` in the arguments would pick the admin
   credential.
3. **The input can't make the tool reveal its credential.** A leaked credential becomes a native
   grant (§5) that bypasses approval and the audit log. For example, `aws configure
   export-credentials` and `aws sts get-session-token` print one, and a confined psql told to
   `\connect` to another host sends it the password. With `args = true`, allow-list the operations
   in the executable rather than deny-listing the dangerous ones. Output review is a backstop only
   when `confirm_output = true`.

SETUP.md shows two example wrappers written to these rules.

**Read-only is not harmless on a primary database.** A read-only role can't write, but an
auto-approved query can still load production. Bound the cost with a `statement_timeout` on the
role or the connection, and have the client send exactly one statement per request: a `SET`
earlier in the same request would lift the timeout. The load is that timeout times the queries
running at once: bound those with the resource's `concurrency`, which counts per guest, and
across guests with a connection limit on the role.

**Read-only is not harmless to auto-approve either.** With `args = true` the agent chooses the
operation, and with auto-approval nobody looks at the choice. So the agent can read anything the
credential can (§5, condition 2), and a read-only cloud role can often read decrypted
parameters, secrets stores and the contents of object storage. Narrow the credential, or the
operations the executable allows, before auto-approving it.

Write these executables in a language with real argument parsing and no shell between the input
and the tool. Their risk is not memory safety but what the wrapped tool makes of its input, and
shell patterns are easy to get subtly wrong (a pattern that blocks `--endpoint-url` misses
`--endpoint`, which the AWS CLI accepts as an abbreviation).

### Human resources

Some tasks only the human can do, and the agent needs them done without ending its turn:
running a query in a browser console, reading a value off a dashboard, answering a question.
The agent's alternative is to stop and ask, which tends to make it summarise, draw conclusions
and act as if the turn were over, when the missing information may change those conclusions.

No special kind of resource is needed: an executable that echoes its payload back (`exec cat`),
with `confirm_output` left `true`. The human approves the request, the task comes back as the
output, and at the release prompt the human edits it into the answer (`e`) and releases it. The
agent gets the answer with §6's notice that the human changed the output; `n` denies, with a
message, as for any request.

### Session resources

Long-lived interactive processes: psql, a Django shell reached through a cloud exec shell, a
remote ssh shell. Essential in practice (slow start-up, loaded state, interactive auth at start).
Each approved snippet runs in the same live session and returns its output.

penyero doesn't hold sessions. A separate **session holder** keeps the process alive, in its
own terminal where the human completes any interactive authentication, and a resource executable
sends it each approved snippet and returns the output. The holder runs one snippet at a time and
queues the rest itself: the resource's `concurrency` counts per guest, so it can't keep a session
that several guests use serial. A holder may ship with penyero as a separate tool (§10). The
same three properties apply: a local psql session would have to be confined, while a remote
console runs its input remotely. Until a holder exists, the same work is done with repeated
requests (each query is one request), which is slower but simple and safe.

## 9. Audit log

Every request is recorded on the privileged side, one log per guest (`profile/name`), with each
broker run marked so that a guest recreated under the same name stays distinguishable. Entries
carry: request id, reason, the request in full, decisions, timestamps, exit status, the tail of
the resource's stderr, and the size and a hash of the output: of what the resource wrote and,
when the human changed it, of what was released too. Each step of a request (received, decided,
started, finished, stopped or killed, released, retrieved, cancelled, dropped) is recorded as it
happens, so a crash leaves a record of how far every request got.

- **Requests are logged whole, outputs never.** The request is the audit: what was done to a
  privileged system, on whose approval. An output is production data, and a log that kept it
  would be a second, long-lived copy of whatever was read, in a file that gets backed up. That
  holds most of all for an output the human redacted: the original is exactly what was judged
  too sensitive to pass on. The hashes tell whether an output was changed and let a kept copy be
  matched; they don't bring it back.
- **The log is hostile content at rest.** Requests and stderr in it were written or influenced by
  the agent. Its encoding keeps every non-ASCII and control character escaped, so printing the
  file is safe; a tool that decodes a field (`jq -r`) puts the raw text on the terminal. Read
  fields through the review pager.

## 10. Backlog

Development is iterative: after each item ships, the next one is chosen. The order below is the
current intent, not a plan. Releases use semantic versioning, `0.x` while the interfaces in this
document may still change, `1.0` when they stop.

1. **The core**: resources, profiles, the client (`list`, `describe`, `request`, `wait`,
   `retrieve`, `pending`, `cancel`), concurrent execution with per-resource `concurrency` and
   `timeout`, the approval prompt with both confirmations, output review and the running view,
   the audit log.
2. **Checkers**: privileged-owned executables that receive a request before the prompt and return
   *pass*, *flag* (with a note the prompt shows) or *block*. Pattern rules first; other kinds
   possible. They follow the review-surface rule (§7) and the resource input rules (§8).
3. **A session holder** (§8): a separate tool that keeps an interactive process alive and runs
   the snippets a resource executable sends it, outside the broker.
4. **Review formatting**: formatters for the inline prompt that only insert whitespace and
   colour (pattern-based, per `input_ext`/`output_ext`) and never rebuild the content, so
   what is shown is still exactly what is approved (§7). To be designed when picked; one option
   is an external formatter whose output the broker checks equals the input up to whitespace.
5. **An MCP client as a resource executable.** Some integrations exist only as MCP servers. A
   wrapper runs the server with its credential, confined like any resource that processes agent
   input with one, and makes one tool call per request, with the arguments as JSON on stdin.
6. **HTTP resources**: `kind = "http"` with a fixed base URL, allowed methods and paths, a
   credential injected on the privileged side, and JSON review hints. Most SaaS reads,
   declaratively; the fixed base URL satisfies the credential rules by construction. To be
   weighed against the principles (§2) when picked: a wrapper executable can do the same.
7. **An MCP adapter for the client's own commands** (`penyero mcp`), only if a client needs
   it and it shows value over its cost. The CLI is POSIX-composable, works in every agent, and
   keeps the surface small and free of MCP spec churn.
8. **Egress approval**: the deployment's egress proxy asks penyero before allowing a new host,
   so the human approves domains the way they approve requests. It needs a way for the proxy to
   reach the broker, which listens on nothing today.
9. **A full-screen front end** for the approval pane (§7): the queue, the running requests and
   the content in view at once, with the standard library's `curses`. Same interaction, same
   rendering rule (escaped plain text, wrapped, nothing clipped unmarked); the line-oriented one
   stays, for tests and for scripting. To be decided after the core has been used for real.
10. **Editing a request before approving it.** Left out of the core: deny with a message does the
    same at the cost of one round trip. It would bring back a notice to the agent that what ran
    is not what it sent, both versions in the log, and a format for editing arguments.

## 11. Open questions

- Which SaaS tokens can actually be scoped read-only (trackers, code hosts, chat) **(verify)**.
