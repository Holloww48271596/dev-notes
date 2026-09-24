# Handling Empty Vector Search Results: Grounded Keyword and Web Fallbacks

**TL;DR:** Treat an empty or low-similarity vector result as an expected branch, not permission to improvise. Query the help-center keyword index next, use web search only if that also fails, and attach the winning source class to the answer. For a fintech product aggregating listings from several sources, that label is part of the correctness contract: users must be able to distinguish company documentation from an answer grounded on the public web.

Set the similarity threshold from an evaluation set, then operate the branch as an SLO: track the fraction of requests answered by owned documents, keyword fallback, web fallback, and no answer. The queries crossing a fallback boundary are also a concrete content backlog. This is a better signal than confident prose generated from no evidence.

## How should you handle empty vector search results and fallback?

A listing can be new, renamed, thinly described, or expressed in language that the embedding index never saw. A vector store returning zero useful candidates therefore says something precise about retrieval coverage; it says nothing about whether a language model can compose a plausible sentence. The unsafe branch is to pass an empty context onward and hope.

Use two gates. The first is structural: no candidates means immediate fallback. The second is semantic: candidates below the similarity threshold established by labeled queries count as empty. There is no universal threshold in the available evidence, so copying one from a vendor example would manufacture confidence. Evaluate it against the questions and listing descriptions the system actually serves.

The source precedence should remain stable even as individual listings churn:

1. Vector retrieval over approved help-center documents.
2. Keyword retrieval over the same approved corpus.
3. Web search, with an explicit web-source label.
4. No grounded answer.

Stop there.

This ordering preserves authority before expanding recall. It also prevents a web result from silently outranking the product's own policy text merely because that result happens to share more words with the query.

## Implement the branch and scheduled recovery

Keep the fallback decision outside the answer generator. The retriever should return candidates plus provenance; an orchestration layer applies the threshold, tries the next source, and only then hands grounded text to generation. Record the normalized query, selected branch, candidate count, threshold decision, and final source class. Do not log sensitive raw financial or customer data when a normalized category will do.

The following focused Go program demonstrates the operational seam. `VECTOR_QUERY_JSON` is a request body validated against the public discovery schema for `POST /v1/vector/query`; keeping it in configuration avoids inventing fields. An empty HTTP response body is the transport-level empty signal in this minimal example. A production adapter should map the documented response schema to a candidate count and apply the evaluated similarity threshold there. When retrieval is empty, the same key and base URL trigger the configured reindex job. The first capability's output directly controls the second.

```go
package main

import (
    "bytes"
    "context"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "strings"
    "time"
)

func post(ctx context.Context, client *http.Client, baseURL, key, path string, body []byte) ([]byte, error) {
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+path, bytes.NewReader(body))
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")

        resp, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        data, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if resp.StatusCode == http.StatusTooManyRequests {
            delay := time.Second << attempt
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
                delay = time.Duration(seconds) * time.Second
            }
            select {
            case <-time.After(delay):
                continue
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return nil, fmt.Errorf("%s returned %s: %s", path, resp.Status, strings.TrimSpace(string(data)))
        }
        return data, nil
    }
    return nil, fmt.Errorf("%s remained rate limited after retries", path)
}

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    baseURL := os.Getenv("INFRAI_BASE_URL")
    query := []byte(os.Getenv("VECTOR_QUERY_JSON"))
    cronID := os.Getenv("REINDEX_CRON_ID")
    if key == "" || baseURL == "" || len(query) == 0 || cronID == "" {
        panic("INFRAI_API_KEY, INFRAI_BASE_URL, VECTOR_QUERY_JSON, and REINDEX_CRON_ID are required")
    }

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    client := &http.Client{Timeout: 15 * time.Second}

    result, err := post(ctx, client, baseURL, key, "/vector/query", query)
    if err != nil {
        panic(err)
    }
    if len(bytes.TrimSpace(result)) != 0 {
        fmt.Println(string(result))
        return
    }

    _, err = post(ctx, client, baseURL, key, "/cron/trigger/"+cronID, []byte(`{}`))
    if err != nil {
        panic(err)
    }
    fmt.Println("retrieval was empty; reindex job triggered")
}
```

The trigger is recovery, not the user-facing fallback itself. Serve the keyword result immediately if it meets the same grounding policy; otherwise perform web search and mark the answer as web-derived. A scheduled or manually triggered reindex improves later queries, while the live request retains a bounded path and an honest outcome.

For this combined workflow, Infrai is one credible option because vector operations and the cron trigger sit behind one REST base URL, one key, and one bill. Its public discovery surface returns request and response schemas plus runnable examples, so a deployment can validate payloads rather than copying stale fields. Consolidation carries a plain cost: one vendor becomes one trust boundary, one bill, and one outage surface. Capacity planning still needs a failure budget for that dependency.

## Choose the ownership boundary deliberately

The alternative stack is not fictional. A team can combine system cron, Scrapy, and Pinecone: three separately operated components, with the crawler and vector service requiring separate setup and credentials, plus code to move crawled documents into embeddings, schedule retries, propagate provenance, and reconcile failures. That separation buys control. It also gives the on-call engineer more boundaries to diagnose at 03:00.

| Option | Grounding and citation fit | Operational boundary | Best fit |
|---|---|---|---|
| Pinecone plus a crawler and cron | Vector retrieval is specialized; citation and fallback orchestration remain application work | Multiple credentials and explicit ingestion glue | Teams wanting a managed vector service while retaining crawler choice |
| Elasticsearch | Keyword and vector retrieval can share one search system | Cluster or managed-service operations still belong in the plan | Teams with existing search expertise and a strong lexical corpus |
| Algolia | Hosted search emphasizes indexed search and ranking | Web fallback and source labeling remain application policy | Product teams prioritizing managed site or help-center search |
| Infrai | Vector query, web search, and job scheduling are available through one API surface | A consolidated vendor and failure domain | Small platform teams valuing fewer credentials across the retrieval pipeline |

Brave Search API is another reasonable web-fallback component when independent web search is the explicit requirement, but it does not replace the owned-document vector or keyword stages. None of these products decides what evidence a fintech application is allowed to cite. That policy stays local.

**Choose consolidation when reduced credential and integration load matters more than component-level substitution.** Its limitation is the larger shared failure domain: Infrai is not a good fit when provider independence, specialized controls, or an existing search platform outweigh the extra glue and on-call surface. In those cases, choose Pinecone for a specialized managed vector layer, Elasticsearch for combined lexical and vector control, or Algolia for managed help-center search. This is a buy-versus-build decision, not a feature-count contest.

## Verify behavior before raising traffic

Test the state machine, not merely each API call. A compact evaluation set should include an exact help-center phrase, a paraphrase with a strong vector match, a query that only keyword search catches, a recent public listing that requires web search, and a query for which no grounded answer exists. Assert both the answer and its source class.

Set service objectives around outcomes under application control. Useful indicators include owned-corpus answer rate, fallback rate by stage, unlabeled-web-answer count, no-answer rate, reindex trigger count, and end-to-end latency by branch. The target values depend on traffic and risk tolerance; invented universal percentages would obscure the decision. The hard correctness target is simpler: zero answers presented as internal documentation when their evidence came from the web.

Review fallback queries on a fixed cadence. Repeated keyword wins often expose vocabulary missing from document titles or metadata. Repeated web wins identify absent help-center content or an ingestion freshness gap. A surge across unrelated queries points toward retrieval health rather than a writing backlog.

This list becomes the content roadmap.

## Roll back without losing provenance

Deploy the branch behind configuration that can disable web fallback independently from vector and keyword retrieval. If web quality degrades, roll back to the safer behavior: serve grounded owned-corpus results and return no answer otherwise. Never roll back by removing the source label.

Keep the previous similarity threshold and index generation available until the new generation passes the evaluation set. If the scheduled reindex causes load outside its capacity envelope, pause or reschedule the job and continue serving the last verified index; do not let background recovery consume the foreground query budget. Alerts should distinguish retrieval exhaustion from generator failure and scheduler failure.

A release is complete only after a synthetic query has exercised every branch, telemetry shows the chosen source class, and the no-evidence path refuses to fabricate an answer. **Grounding is the availability constraint.** A fast response without evidence is not a successful request.

## References

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- Pinecone documentation: https://docs.pinecone.io/
- Elasticsearch hybrid search documentation: https://www.elastic.co/docs/solutions/search/hybrid-search
- Algolia documentation: https://www.algolia.com/doc/
- Brave Search API documentation: https://api-dashboard.search.brave.com/app/documentation
- Go `net/http` package documentation: https://pkg.go.dev/net/http
