# Compliance Evidence for OCR Jobs — Validation, Retries, Privacy, and 30-Day Retention

If a scanned invoice moves through your pipeline and no durable record survives to prove what happened to it, the OCR was decorative. So: use one evidence row per document as the unit of work, write the validation result and the retention deadline into that row inside the same transaction that marks the job finished, keep extracted text out of your logs, and stage every temporary file in a private directory that dies with the process. Retries get cheap after that, because the evidence row — not the queue message, which any broker is free to redeliver — becomes the source of truth about what a Node.js service actually did with someone's paperwork.

The rest of this is the trace backwards from the page that eventually fires.

## The page that fires, and what it refuses to tell you

Here is the alert an on-call engineer gets at 03:40, halfway through the nightly scan run for a mid-sized e-commerce returns desk:

```text
[FIRING] ocr_batch_backlog > 5000 for 15m
  queue: ocr-ingest   depth: 7412   oldest_message_age: 41m
  runbook: https://runbook.internal/ocr-ingest
```

Queue depth. That's all of it.

The responder can see that documents are piling up, and from that single number they cannot tell whether the workers are dead, whether one 900-page supplier catalogue is jamming a single consumer, or — the case that actually costs money — whether the workers are happily churning through pages while the evidence write silently fails on every one of them. Backlog is a symptom that a dozen unrelated conditions share. It fires late by construction, because a queue has to visibly accumulate before it crosses any threshold worth paging on, and by then the batch window is half gone and the makeup run collides with the next night's ingest.

The failure that got expensive is the quiet one. OCR succeeds, the searchable text lands in the index, the returns team is happy, and the attempt record that was supposed to prove which engine version read which page hash never commits because the evidence table is in a different transaction than the text upsert. Nobody notices for months. Then an auditor asks for the chain of custody on 200 specific documents and the honest answer is that you have the output but not the proof.

## How should asynchronous jobs handle validation, retries, and evidence retention?

Make the evidence row the thing you commit, and make the job idempotent on content rather than on message identity.

A queue message ID is worthless as an idempotency key, since redelivery after a broker failover produces a new one for the same document, and a naive worker will then write a second attempt row that looks like a genuine retry. Hash the bytes instead. The SHA-256 of the scan is stable across redeliveries, across broker migrations, and across the day somebody replays a dead-letter queue by hand, and it doubles as the integrity anchor an auditor wants anyway.

Retry classification matters more than retry count. Transient failures — a timeout talking to the OCR engine, a connection reset, a 5xx from an object store — deserve backoff and another attempt. Deterministic failures do not: a corrupt TIFF, a password-protected PDF, or a page whose confidence score sits below your gate will fail identically on attempt seven, and the only thing the extra attempts buy is six more rows of noise and a longer backlog. Record the rejection, route it to a human review lane, move on.

Validation is a gate, not a log line. The worker decides `validated` or `rejected` before anything reaches the search index, so the index only ever contains text that cleared the confidence threshold and the schema check, and the evidence table contains every attempt including the ones that failed. Both halves commit together.

```go
// One scanned document, one attempt row. The evidence write and the
// searchable text land in the same transaction, or neither lands.
type Attempt struct {
	DocumentID  string
	Attempt     int
	ContentSHA  string
	Engine      string // e.g. "tesseract-5.5.0"
	StartedAt   time.Time
	FinishedAt  time.Time
	Outcome     string // "validated" | "rejected" | "retryable"
	Reason      string
	DeleteAfter time.Time
}

var errRetryLater = errors.New("transient failure, redeliver")

func processScan(ctx context.Context, db *sql.DB, doc Document) error {
	// Private per-job directory on a tmpfs mount; MkdirTemp creates it 0700.
	dir, err := os.MkdirTemp("/run/ocr", "doc-")
	if err != nil {
		return fmt.Errorf("temp dir: %w", err)
	}
	defer os.RemoveAll(dir)

	page := filepath.Join(dir, "page.tiff")
	if err := os.WriteFile(page, doc.Bytes, 0o600); err != nil {
		return fmt.Errorf("stage page: %w", err)
	}

	sum := sha256.Sum256(doc.Bytes)
	now := time.Now().UTC()
	att := Attempt{
		DocumentID:  doc.ID,
		Attempt:     doc.Attempt,
		ContentSHA:  hex.EncodeToString(sum[:]),
		Engine:      doc.Engine,
		StartedAt:   now,
		DeleteAfter: now.AddDate(0, 0, 30),
	}

	text, confidence, err := runOCR(ctx, page)
	switch {
	case errors.Is(err, context.DeadlineExceeded), errors.Is(err, syscall.ECONNRESET):
		att.Outcome, att.Reason = "retryable", err.Error()
	case err != nil:
		att.Outcome, att.Reason = "rejected", err.Error()
	case confidence < 0.82:
		att.Outcome, att.Reason = "rejected", "confidence below gate"
	default:
		att.Outcome = "validated"
	}
	att.FinishedAt = time.Now().UTC()

	if err := withTx(ctx, db, func(tx *sql.Tx) error {
		if err := insertAttempt(ctx, tx, att); err != nil {
			return err
		}
		if att.Outcome != "validated" {
			return nil
		}
		return upsertSearchable(ctx, tx, doc.ID, redact(text), att.ContentSHA)
	}); err != nil {
		return err
	}

	if att.Outcome == "retryable" {
		return errRetryLater
	}
	return nil
}
```

The shape is the point, not the language. A Node.js service does the same thing with `fs.promises.mkdtemp` and a transaction helper; I write my workers in Go because the batch tier is CPU-bound and I want the memory ceiling to be boring, which is a preference and not an argument.

## Temporary files are where the privacy problem actually lives

Scanned commercial paperwork is full of personal data — names, home addresses on return labels, sometimes a signature or a partial card number — and it spends its life in three places that are easy to forget: the staging file on disk, the process memory holding the decoded page, and the log line somebody added while debugging.

Never `/tmp`. It is shared, it is world-traversable on most images, and predictable names in a shared directory are the classic symlink race. `os.MkdirTemp` (and `fs.promises.mkdtemp` on the Node side) gives you an unguessable directory at 0700, and mounting that path as tmpfs means the bytes never touch a persistent volume or a snapshot of one. `defer os.RemoveAll(dir)` covers the normal exit; the tmpfs mount covers the abnormal one, which is the case that actually matters when a worker is OOM-killed mid-page.

In-memory handling deserves the same suspicion. A browser-side upload widget that wraps a scan in a `Blob` before posting it is fine for a 200 KB receipt and wrong for a 300-page catalogue, and the server side should stream to the staging file rather than materialise the whole document — a 2 GB batch of buffered pages is how a worker discovers its memory limit at the worst moment.

Retention is arithmetic, not policy prose. Stamp `DeleteAfter` at write time from a class the record carries with it, since the deadline for a returns photo and the deadline for a tax-relevant invoice are not the same number, and let a sweeper delete on that column rather than asking anyone to remember. Storage limitation is an actual legal principle, not a hygiene suggestion. What survives the sweep is the tombstone: document ID, content hash, engine version, outcome, timestamps. That's the compliance evidence. The pixels aren't.

## Buy, build, or split the difference

The decision axis here is batch throughput, so run the arithmetic before the vendor conversation. A returns desk scanning 40,000 pages into a six-hour overnight window needs roughly 1.9 pages per second sustained; a worker that clears 1.2 pages per second means two at saturation, three if you want one to be able to restart without the backlog growing, four if the same tier also absorbs the Monday morning re-scan spike.

| Approach | Throughput ceiling | Evidence you control | On-call load |
|---|---|---|---|
| Self-hosted OCR workers | Bounded by CPU you provision; nightly peaks need standing headroom | All of it: attempt rows, engine version, temp-file policy | Highest — crash loops and disk pressure are yours |
| Managed OCR API | Elastic, but the rate limit is the real ceiling | Only what the response carries; you still write your own attempt row | Lower, until an upstream incident becomes your incident |
| Managed for spikes, self-hosted baseline | Baseline plus burst | Same schema both paths, `Engine` field records which one ran | Middle, in exchange for two failure modes to learn |

The catch with the split is that it doubles your evidence surface, and if the two paths write different fields you have effectively lost the audit trail for half the corpus. Stick with a single path until the burst is a real, recurring capacity problem rather than an imagined one. Self-hosting also stops being sensible when your volume is genuinely spiky and small — nobody should run a GPU-backed OCR tier for 300 documents a week.

## What the wrong threshold costs you

Go back to that 03:40 page. The signal that should have fired hours earlier is not queue depth; it's the ratio of documents with a committed evidence row to documents that entered the pipeline, measured over a rolling window, plus the age of the oldest attempt still stuck in `retryable`. Both go bad while the queue still looks healthy. An SLO gives them a threshold you can defend: 99.5% of documents accepted in a calendar month get a `validated` or `rejected` evidence row within 30 minutes of upload, and the alert fires on burn rate against that budget rather than on any raw instantaneous number.

Get the threshold too tight and you pay in a currency that compounds. A page every night at 03:40 because the nightly batch legitimately builds a backlog trains the responder to acknowledge without reading, and six weeks later the one page that meant something gets the same reflex. Multi-window burn-rate alerting exists specifically to make a fast burn wake somebody and a slow burn open a ticket.

Too loose costs more, just later. I'm not certain there's a clean rule for where to start, and the honest version is that you tune it once you know your own batch shape; a defensible default is to alert when the evidence-write ratio drops below the SLO for a full batch window, page only when the burn rate says the monthly budget dies within a day, and treat every silent month as evidence the threshold is doing nothing.

## References

- MDN, Blob — https://developer.mozilla.org/en-US/docs/Web/API/Blob
- Node.js, `fs.promises.mkdtemp` — https://nodejs.org/api/fs.html#fspromisesmkdtempprefix-options
- Go standard library, `os.MkdirTemp` — https://pkg.go.dev/os#MkdirTemp
- Linux kernel documentation, tmpfs — https://docs.kernel.org/filesystems/tmpfs.html
- GDPR Article 5, principles relating to processing of personal data — https://gdpr-info.eu/art-5-gdpr/
- NIST SP 800-92, Guide to Computer Security Log Management — https://csrc.nist.gov/pubs/sp/800/92/final
- Google SRE Workbook, Alerting on SLOs — https://sre.google/workbook/alerting-on-slos/
- OWASP Cheat Sheet Series, Logging — https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
