# cpdatasource
A data source repo for a demo

# Ingestion speed and how to tune it

Written 2026-11-13 with the work for #45 (ingestion embedded one chunk per minute) and #46 (make the rate adjustable). The new code was deployed and measured on 2026-11-13 (section 8); the times in the tables of sections 3 and 7 are calculated from the pacing.

Related: `docs/PROJECT-STATE.md` section 3 (configuration) and section 6 (how to read the function log).

---

## 1. Where the time goes

Ingesting one document through the function has four stages. Figures in the "before" column are from the test ingestion on 2026-11-12 (`2017_atlassian_devops_survey.pdf`, 17 pages, 23 chunks).

| Stage | Before | Now | Tunable |
|---|---|---|---|
| Blob trigger notices the upload | about 10 minutes (container scan) | unchanged | No |
| Document Intelligence extracts the text | 7 seconds | unchanged | No |
| Embedding, one call per chunk | 22 minutes (one chunk per minute) | 12 seconds (measured, 23 chunks at 60 a minute) | **Yes, see below** |
| Write to the search index | once, at the end | every 50 chunks | No (constant in code) |

Embedding was the whole problem and is the only stage with a knob. The 10-minute wait before the trigger fires is the blob trigger's own behaviour and is not affected by any of this.

---

## 2. The two knobs

| Knob | What it is | Default | Where it applies |
|---|---|---|---|
| `Ingestion__EmbeddingsPerMinute` | How many embedding calls per minute ingestion makes. One call per chunk. This is the speed control. | 60 | App setting on the function; environment variable or `--embeddingsperminute` for the CLI uploader |
| `AZURE_OPENAI_EMBEDDING_DEPLOYMENT_CAPACITY` | Capacity of the Azure OpenAI `embedding` deployment, in thousands of tokens per minute. This is the hard limit on the Azure side. | 30 | The model deployment (bicep `embeddingDeploymentCapacity`) |

The first is a ceiling the app puts on itself; the second is what Azure will actually accept. Keep the first comfortably under what the second allows.

`EmbeddingRateLimit` in `src/cpshared/Services/` is the one place that reads the setting and builds the limiter, for both the function and the CLI.

---

## 3. Choosing a rate

A chunk is at most about 1,100 characters, roughly 275 tokens. So:

- tokens per minute used by ingestion is about `rate x 275`
- the most a deployment can take is about `capacity x 1000 / 275`, which is about `3.6 x capacity` calls per minute
- the chat API embeds **every user question on the same deployment**, so do not give ingestion all of it. The default uses a little over half: about `2 x capacity`

| Rate | Tokens per minute | Fits a 30K deployment | 23 chunks (the test file) | 1,750 chunks (the whole corpus) |
|---|---|---|---|---|
| 1 (the old behaviour) | 275 | Yes | 22 minutes | 29 hours |
| 30 | about 8K | Yes, lots of headroom | about 36 seconds | about 58 minutes |
| **60 (default)** | about 17K | Yes | about 13 seconds | about 29 minutes |
| 90 | about 25K | Barely; chat questions may see 429s | about 5 seconds | about 19 minutes |
| 120 | about 33K | **No**: raise capacity to 60 first | about 2 seconds | about 14 minutes |

The times are the pacing alone; add the time the calls themselves take (a fraction of a second each) and, for the function, the stages in section 1.

Chunks with HTML tables use more tokens for the same length, so treat 275 as an average, not a ceiling.

Azure also limits requests per minute, set from the capacity (typically 6 requests per minute for each 1,000 tokens per minute, so 180 at capacity 30). At this chunk size the token limit is reached first, so the request limit only matters for how calls are spaced, which is the next section.

---

## 4. How the pacing works

Calls are spread evenly, not released a minute's worth at a time. At 60 a minute that is one call a second.

The reason: Azure OpenAI checks the request limit over windows of 1 or 10 seconds, not over the whole minute. A burst of 60 calls in a few seconds gets 429 responses even though the minute as a whole is under the limit.

A short document should not wait at all, so the limiter starts with ten seconds' worth of calls available (10 at the default rate, `rate / 6` in general). After that burst is used, calls are paced. The burst refills while ingestion is idle.

In the function there is **one limiter for the whole process**. Several documents being ingested at the same time share the rate; they do not each get their own. (Before this change each document got its own limiter.)

The rate is per process. These add up, and the setting does not know about them:

- the function app scaled out to more than one instance (it runs on one today)
- the CLI uploader running while the function is also ingesting
- two people running the CLI at once

**Not yet verified: a CLI run may be embedded twice.** The CLI uploads its page blobs to `content`, which is the container the function's blob trigger watches. Read from the code, the function then embeds each of those pages as well, about ten minutes behind the CLI. Both write the same index ids, so the index ends up the same, but the embedding calls double and the two limiters do not know about each other: up to twice the configured rate while both are busy. If that is what happens, it also means a PDF page the CLI failed to embed (section 6) is normally picked up by the function anyway. Check it during the next CLI run (the reload in #38): look for `Executing 'Functions.embed-blob'` lines for the page blobs in the function log, and at the request count in the metrics of section 8. Then either halve the rate for CLI runs or stop the function for the duration.

---

## 5. Changing the settings

Both changes below alter live Azure.

**The normal way, through azd** (survives every later provision):

```bash
azd env set Ingestion__EmbeddingsPerMinute 90
azd env set AZURE_OPENAI_EMBEDDING_DEPLOYMENT_CAPACITY 60
azd provision
```

Check the regional quota before raising capacity, as with the chat deployment. `azd provision --preview` does not show app-setting changes.

**The rate only, without a provision:**

```bash
az functionapp config appsettings set -g rg-xa-devopscopilot -n func-function-mhkb4lb44qdeg --settings Ingestion__EmbeddingsPerMinute=90
```

Two things to know about this route:

- Changing an app setting restarts the function app. A document that is being ingested at that moment is started again from the beginning.
- The next `azd provision` puts the value back to whatever the azd environment holds (60 if never set). Use `azd env set` as well if the change is meant to stay.

**The CLI uploader** (`src/CpDocUploaderCli`, used by `scripts/prepdocs.ps1`) takes the rate from, in order: the `--embeddingsperminute` option, the `Ingestion__EmbeddingsPerMinute` environment variable, the default of 60. When it runs from the azd postprovision hook it sees the azd environment value, so the function and the CLI agree without doing anything. For a one-off faster run:

```bash
dotnet run --project src/CpDocUploaderCli -- './data/**/*' --embeddingsperminute 90 ...
```

**The CLI is slower by default than it used to be.** It was hardcoded at 256 calls per 1.1 minutes, which is far more than a 30K deployment accepts and only worked because the client retried the 429s. It now uses the shared default of 60. For a full corpus reload (#38, #13), decide the rate deliberately; see section 7.

---

## 6. What happens at the limits

**The rate is set too high for the capacity.** Azure answers 429, and each one is written to the log as a warning that names the setting (section 8). The OpenAI client retries on its own, honouring the wait the service asks for (three retries by default in the function, five in the CLI). Ingestion slows down and usually still finishes. If the retries run out, the document fails:

- In the function the blob trigger runs the document again, up to five times, then gives up on it. Chunks already written stay in the index, and a repeat run overwrites them under the same ids, so nothing is duplicated.
- In the CLI the file is reported as failed (`Failure processing <file>`) and the run continues with the other files. For an HTML or Markdown file, run it again. For a PDF, a rerun is not enough: the CLI uploads each page before embedding it and skips pages whose blob already exists, so the page that failed is never retried. Delete that page's blob (`<name>-<page>.pdf`) from the `content` container first, then run the file again. The CLI's `--remove <file>` does that and more: it removes all of the document's page blobs, its corpus text and its index sections, after which a normal run ingests the whole file again. (Before #54 was fixed, `--remove` with a file name deleted every blob in both containers.)

Meanwhile chat questions are competing for the same deployment and may fail or be slow. That is the real cost of setting the rate too high.

**The setting is not a valid number.** Anything other than a whole number of 1 or more (`0`, `-5`, `1.5`, `fast`) stops the function host at startup, with an error that names `Ingestion__EmbeddingsPerMinute` and the value. Nothing is ingested until it is corrected. This is deliberate: a silent fall-back to the default would hide the mistake. A missing or blank setting is fine and means the default. Bicep passes a number, so this can only happen when the setting is typed by hand.

**One very large document.** The function timeout is the Functions default of 30 minutes; `host.json` does not set it. At 60 a minute, 30 minutes is about 1,800 chunks in a single document, which is more than the whole current corpus. If the rate is turned down a long way, that margin shrinks in proportion (at 10 a minute it is about 300 chunks). A document that does hit the timeout now leaves its finished batches in the index and a progress line in the log for each, where before it left nothing. On this App Service plan the timeout can be raised or made unbounded in `host.json` if that is ever needed.

---

## 7. Suggested settings

| Situation | Rate | Capacity | Why |
|---|---|---|---|
| Normal operation, occasional uploads | 60 | 30 | The defaults. Any ordinary document embeds in seconds and chat keeps its headroom. |
| Full corpus reload or re-embedding (#38, #13), site in use | 60 | 30 | About half an hour for 1,750 chunks, with no effect on chat. |
| Full reload, want it faster | 120 to 200 | 60 to 100 | Raise the capacity first, then the rate to about twice the capacity. Put both back afterwards if the extra quota is wanted elsewhere. |
| Quota is short, or chat is seeing 429s during ingestion | 20 to 30 | 30 | Slower ingestion, more room for chat. |

Standard deployments are billed by tokens used, not by the capacity assigned, so raising capacity does not by itself add cost; it takes from the subscription's regional quota for the model.

---

## 8. Checking that it works

After a deploy, upload a test document to `content` and read the function log (the query is in `docs/PROJECT-STATE.md` section 6). Look for:

- `Embedding rate limit: 60 per minute (Ingestion__EmbeddingsPerMinute)`: written once, when the first document after a start is processed. It confirms the value the function is really using.
- `Indexed 50 sections, 50 succeeded, 50 so far for '<blob>'`: one line per batch of 50 chunks, and one for the remainder at the end. The time between these lines is the real rate.
- `Embedded 23 sections of '<blob>' in 12.4 s (111 a minute), 0 throttled attempts meanwhile`: one line per document, added with #52. The rate is sections over elapsed time, so a short document that fits in the starting burst shows more than the setting; a long one settles at the setting. The limiter and the count are shared, so with several documents in progress the rate is this document's share and the count is everything throttled while it ran.
- `Azure OpenAI throttled an embedding call (429, asked to wait 1.5 s; 3 so far)…`: a warning for every attempt Azure OpenAI throttled, including the ones the client then retried successfully. None of these is the normal case. A few during a reload are harmless; a steady stream means the rate is too high for the capacity.
- `Executed 'Functions.embed-blob' (Succeeded, Duration=…ms)`: the total for the document.

The CLI writes the same lines to the console.

Then check `DocumentProcessingStatus=Succeeded` on the source blob, as in #33.

**Was anything throttled?** Since #52 the log says so (the warning above). Azure OpenAI's own metrics give the same answer from the other side, and they also cover the chat API's use of the deployment. This lists the requests to the `embedding` deployment per minute, split by status code. A `429` row means the rate is too high for the capacity:

```bash
az monitor metrics list \
  --resource /subscriptions/bbb09af1-95c3-4cbf-94f4-3811a1965f54/resourceGroups/rg-xa-devopscopilot/providers/Microsoft.CognitiveServices/accounts/cog-mhkb4lb44qdeg \
  --metric AzureOpenAIRequests --interval PT1M --aggregation Total \
  --start-time <utc> --end-time <utc> \
  --filter "StatusCode eq '*' and ModelDeploymentName eq 'embedding'"
```

Three things about reading it: the metric counts every call twice (23 chunks show as 46; the one-a-minute run of 2026-11-12 shows as 2 a minute), its minute can be one later than the function log's, and chat questions embed on the same deployment so they appear here too. `--metric ProcessedPromptTokens` gives the tokens used. In Git Bash prefix the command with `MSYS_NO_PATHCONV=1`.

To re-run the #45 test file, upload `2017_atlassian_devops_survey.pdf` again (a re-upload clears the status metadata, which is what lets it be processed again). At the default rate the 23 chunks take about 12 seconds after Document Intelligence finishes.

### Measured results

| Date | Document | Chunks | Rate | Capacity | Embedding time | Total function duration |
|---|---|---|---|---|---|---|
| 2026-11-12 (before) | `2017_atlassian_devops_survey.pdf` | 23 | 1 | 30 | 22 minutes | 1,321 seconds |
| 2026-11-13 (after) | same file, re-uploaded | 23 | 60 | 30 | 12 seconds | 20 seconds |

Detail of the second run (function invocation `f2907826-f7a5-4b31-a66e-6bd18c30abed`, times UTC on 2026-11-14): uploaded 03:32:48, trigger fired 03:43:06, Document Intelligence finished 03:43:14, `Indexed 23 sections, 23 succeeded` at 03:43:26, status `Succeeded` on the blob. The 12 seconds match the pacing: ten calls from the starting burst, then thirteen at one a second. At the time of this run the function log could not show whether Azure OpenAI had throttled anything (#52 added that afterwards). The Azure OpenAI metrics could: for this run they show only status 200 on the `embedding` deployment, no 429, and 6,376 tokens for the 23 chunks, which is 277 a chunk.
