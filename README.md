# The write succeeded. Your recovery code still has a problem.

*By José Siroco*

A forecasting bot submitted a prediction, then asked the server to read it back. That second request hit HTTP 429: too many requests.

The forecast was already saved. The command still exited with an error.

This is where a helpful retry loop can become an unhelpful second write. The operator sees one failed command; the server has already completed part of its work. Restarting the command treats those two views as though they agree.

The integration was small: a PowerShell client, a frozen JSON forecast, and a private comment explaining the prediction. No queue, no worker fleet, no distributed transaction coordinator waiting for its moment. Yet it already needed a recovery protocol.

I think this is an underappreciated threshold in automation. Once a program performs a remote write and is expected to recover without supervision, “try again” becomes an engineering decision about what the server might already have done.

Let's follow that decision through the actual client, then push beyond what its implementation can safely support.

## One command, two side effects, several opportunities to get confused

A successful submission has four network steps. The sequence below is schematic, not a captured server log:

```text
POST forecast       → response received
GET forecast        → values match the intended prediction
POST explanation    → comment ID received
GET explanation     → text, privacy flag, and author match
```

The final two reads establish that the requested state is observable. They are useful even when the write returned successfully: a request that reached the correct endpoint can still contain the wrong distribution, target the wrong question, or create a public explanation where a private one was intended.

For this client, delivery begins with a fixed input file. The prediction and explanation are decided before sending. Validation checks the distribution, question membership, and server-provided numeric grid. Recovery is supposed to deliver that decision, not generate a new prediction because the previous attempt was inconvenient.

After the forecast POST returns, the helper records `forecast_sent` in a local receipt. If the next GET is rate-limited, the receipt tells recovery which step remains. The recorded 429 was handled this way, without another forecast POST.

This is a more useful interface than a single Boolean named `success`. The program can say that the forecast was acknowledged while its confirmation remains incomplete. That distinction is small enough to fit in a JSON file and important enough to determine which requests may run next.

## The two histories your client cannot distinguish

A successful response is the easy case. The harder case is a connection that breaks after a write might have reached the server.

Imagine these two histories for creating the explanation:

```text
History A                         History B
save local intent                 save local intent
connection fails before delivery  server creates comment
                                  connection fails before response

local state: attempting           local state: attempting
server: no comment                server: one comment
```

The durable local evidence is identical. The correct next action differs.

Create another comment in both histories, and History B can produce a duplicate. Skip creation in both, and History A leaves the explanation missing. A decision made only from that local state cannot satisfy both requirements.

This is the reason to represent an unknown outcome explicitly. It is information the workflow genuinely lacks, not an exception that a more confident catch block can resolve.

The actual helper saves `comment_creating` before the comment request. Receiving an ID advances it to `comment_sent`. If the response is lost, automatic recovery stops at the uncertain state. The forecast has an analogous `starting` state around its write.

I'd accept that interruption for one locally operated bot. It gives the operator a specific reconciliation task instead of quietly taking a second guess at publication. The tradeoff is availability: even History A, where nothing happened remotely, needs investigation.

The protocol has chosen to leave some work unfinished rather than risk repeating an uncertain write. That is a defensible choice for this workload, and an expensive one for a service processing thousands of operations a minute.

## A small model makes the tradeoff visible

I built a companion model of a **toy creation API** to check that argument. It does not call Metaculus or execute its submission helper. It enumerates five possible interruption boundaries around one logical creation, then applies one recovery policy.

The boundaries are: before intent is saved; after intent is saved but before the server acts; after the server commits but before acknowledgement is recorded; after acknowledgement is recorded; and after verification.

| Recovery policy | Boundaries producing a duplicate | Boundaries requiring manual work | Boundaries completing with one effect |
| --- | ---: | ---: | ---: |
| Replay the creation on every restart | 3 of 5 | 0 of 5 | 2 of 5 |
| Keep a journal; stop on an ambiguous attempt | 0 of 5 | 2 of 5 | 3 of 5 |
| Replay a stable intent key to an ideal deduplicating server | 0 of 5 | 0 of 5 | 5 of 5 |

These fractions are counts of deliberately chosen boundaries, not incident rates. A result of 3 of 5 does not mean 60% of real retries duplicate something. The first policy is deliberately naive: an unconditional restart, including after verification. The last row assumes an atomic server ledger, retained keys, one worker, and immediate authoritative reads. Local steps are assumed durable and atomic too. Those assumptions are stronger than the file-based client provides.

The model's job is narrower: exhibit the counterexample and show what additional capability removes it. The conservative journal avoids duplicate creation in this model by surrendering automatic completion at two boundaries. Server-side deduplication supplies information that the journal alone does not have.

The companion `retry-boundary-model.py` runs with Python's standard library and includes assertions for these outcomes. Its results are saved separately. This is a reproducible reasoning aid, not a claim of production reliability or an exhaustive model of network failures.

## Give the operation an identity that survives the process

The local helper records a SHA-256 hash of the request file and checks it before resuming. That prevents a changed file from quietly being treated as the previous request. It also gives troubleshooting a precise input to refer to.

But a payload hash and an operation ID answer different questions.

Suppose two deliberate comments have identical text. They might belong to two different forecasts. Even within one target, a user might intentionally perform the same action twice. Deduplicating only by content would collapse distinct intentions. Conversely, changing JSON whitespace changes a byte hash without necessarily changing the API operation.

For a more general client, I would assign a durable ID when the operation is created, before the first network attempt. That ID survives restarts. A separate fingerprint binds it to the target and payload. Reusing the ID with different input should produce a conflict that someone investigates, not silently redefine the old request.

The model checks those three identity rules: the same intent replays the same result; a new intent can have identical content; and changed input under an existing intent is rejected.

Scope matters too. Two accounts or endpoints should not accidentally share a deduplication namespace. For a forecast followed by an explanation, use a workflow identity plus separate identities for each write. One key for the entire workflow does not tell the server whether a retry refers to the forecast or the comment.

And actually reuse the ID. Generating a fresh UUID inside a retry loop announces a new intention every time. The server will take the hint.

[Amazon's discussion of idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) is valuable here because it treats caller intent and parameter mismatch as part of the contract. Those are API semantics to design and verify, not properties obtained merely by naming a header `Idempotency-Key`.

## Read the server's contract, including its expiry date

A server that recognizes an operation ID can make retries much easier, but the precise contract still matters.

For example, [Stripe's API v1 documentation](https://docs.stripe.com/api/idempotent_requests) says it saves the initial response once endpoint execution starts, including a `500` response. It also documents parameter comparison and permits removal of keys after they are at least 24 hours old. Reusing a pruned key can create a new request. The same page describes different recovery behavior for API v2.

That makes “retry with the same key” incomplete advice. Which API version? How long is the key retained? Did the original request reach the stage where a result is saved? Can the key still distinguish a late retry from new work?

A workflow whose recovery backlog can outlive the server's retention window needs a different plan at expiry. It may need to locate the resulting resource through a durable reference, or stop for reconciliation. Reusing an expired key with unlimited optimism is effectively another unprotected write.

The [HTTP specification's discussion of idempotency](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.2) gives the general rule for retrying non-idempotent requests. The service's documented behavior tells you whether a particular retry meets it.

## A database fixes one boundary, not every boundary

The existing helper writes its receipt directly to a JSON file. A partially written file, two simultaneous processes, or a machine failure at an awkward moment can undermine the recovery evidence itself.

For a single-machine service with several workers, I would consider a transactional local store before adding more retry logic. Persist the immutable intent and establish exclusive ownership of each operation. Keep transitions and the evidence supporting them together.

[SQLite's description of atomic commit](https://www.sqlite.org/atomiccommit.html) explains the local durability problem, including journaling, flushing, and recovery. The protection depends on the storage environment and configuration. A database is a substantial improvement over treating ordinary file writes as transactional.

It still cannot make an unrelated HTTP service participate in that local commit. There remains a point where the remote service may have acted and the local database has not yet recorded the result.

Consider two workers that both see no receipt and both send. A uniqueness constraint can prevent both from claiming the operation locally. But if ownership is a lease, a worker can pause, lose its lease, and resume after a replacement starts. Remote duplication still needs protection. Where supported, a downstream idempotency mechanism or fencing scheme must enforce the relevant boundary; a lease timestamp in our database does not enforce it at the API.

The design choice is therefore about how much concurrency and recovery the workload needs. For this bot, I'd first keep execution serialized and preserve uncertain operations for inspection. If it becomes a multi-worker service, the persistence and remote contract need to support that change before throughput increases.

## The forecast and its explanation can drift apart

Even perfect per-request deduplication would leave the two-write workflow unfinished.

Suppose the forecast succeeds and the explanation does not. A second authorized writer updates the forecast. Recovery then creates the original explanation. Both resources may exist exactly once while the explanation describes the wrong prediction.

The unit of business correctness is the relationship between those resources. Counting successful requests will miss it.

The current helper compares the account's latest forecast with the frozen input before creating the comment. That is useful evidence under its single-writer assumption. It is weaker under concurrency: another writer can change the forecast between the read and the comment POST. Matching values also cannot distinguish two versions that happen to contain identical numbers.

If an API exposes immutable forecast versions and lets the comment reference one, bind the explanation to that version. If it offers conditional writes, use the documented precondition. If it exposes neither, serialize writers where possible and be explicit about the remaining race. Do not describe a sequence of independently accepted writes as an atomic operation.

For most discussion-style workflows, I would not automatically delete the forecast to compensate for a failed explanation. Deletion is another remote effect with its own ambiguity and consequences. Keeping the partial state visible and completing or reconciling it is often the more manageable choice.

## A deadline check can block the wrong kind of recovery

Static inspection of this helper revealed a more immediate limitation: it checks that the question is still open before entering its read-back recovery path.

That is appropriate before publishing a new prediction. But a forecast can be acknowledged just before the deadline, with confirmation delayed until after the question closes. The helper then refuses to resume, even though reading the existing result is a different operation from submitting a late forecast.

This is not a reproduced incident. It follows from the order of the current checks, and it gives the article's recovery argument a concrete implementation consequence.

I would separate reconciliation from permission to create new work. First establish the outcome of acknowledged writes, including after a deadline where read access remains permitted. Then check the current rules before any still-missing write, such as an explanation. Closure must not be interpreted as permission to submit late, but it also should not automatically erase the ability to verify what already happened.

That separation also gives the operator a better answer: “forecast confirmed; explanation requires review after closure” is more useful than restarting the workflow's original eligibility checks and returning one undifferentiated error.

## Retry less work, with more context

The observed 429 is a useful starting point because it happened after an acknowledged write. The client knew enough to retry only confirmation. Lost responses, simultaneous workers, expired keys, and mismatched forecast versions require additional evidence or stronger contracts; more attempts cannot supply those by themselves.

For an integration review, I'd begin with three requests from the same workflow: one rejected before execution, one committed but unacknowledged, and one acknowledged whose confirmation failed. Ask the recovery code to distinguish them. If all three arrive at the same unconditional POST, the next retry deserves a closer look.

The engineering value of a receipt is that it preserves what the client actually knows. When that knowledge runs out, a safe workflow has a way to discover more or a visible place to stop. That is what makes a small bot recoverable before anyone starts calling it autonomous.
