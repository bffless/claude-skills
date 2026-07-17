---
name: pipelines
description: Backend automation without code using handler chains
---

# Pipelines

**Docs**: https://docs.bffless.app/features/pipelines/

Pipelines provide backend functionality for static sites without writing server code. Chain handlers together to process forms, store data, send emails, call AI models, accept payments, and more.

## Handler Types

| Handler | Type | Purpose |
|---------|------|---------|
| **Form** | `form_handler` | Parse form submissions (multipart, JSON, URL-encoded) |
| **Data Create** | `data_create` | Create DB records in a pipeline schema |
| **Data Query** | `data_query` | Read/list DB records with filters, sorting, pagination |
| **Data Update** | `data_update` | Update existing DB records |
| **Data Delete** | `data_delete` | Delete DB records |
| **Aggregate** | `db_aggregate` | Count/Sum/Avg/Min/Max on data, with optional groupBy for grouped results |
| **Email** | `email_handler` | Send emails via configured provider |
| **Response** | `response_handler` | Return custom JSON, status codes, or redirect |
| **Function** | `function_handler` | Custom JavaScript for transformation/logic |
| **AI** | `ai_handler` | Call OpenAI/Anthropic/Google AI models (chat or completion) |
| **HTTP Request** | `http_request` | Make outbound HTTP requests to external APIs |
| **File Upload** | `file_upload_handler` | Upload files from forms or URLs to storage |
| **File Serve** | `file_serve_handler` | Serve files from storage with Range request support |
| **Image Convert** | `image_convert_handler` | Convert images between PNG/JPEG/WebP using sharp |
| **Signed URL** | `signed_url` | Generate time-limited presigned URLs for downloading storage files |
| **Presigned Upload** | `presigned_upload` | Issue a presigned URL so clients upload large files directly to the bucket (prepare) |
| **Register Upload** | `register_upload` | Record a file that was uploaded directly to the bucket (finalize) |
| **Replicate** | `replicate` | Call Replicate ML models (image gen, embeddings, etc.) |
| **Embed Store** | `embed_store` | Store embedding vectors for semantic search |
| **Vector Search** | `vector_search` | Query embeddings by cosine similarity |
| **Stripe Checkout** | `stripe_checkout` | Create Stripe Checkout sessions for payments/subscriptions |
| **Stripe Webhook** | `stripe_webhook` | Validate Stripe webhook signatures and parse events |

## DB Records

Schema-based data storage built into BFFless:

1. Define schema in project settings (fields, types, validation)
2. Use Data CRUD handlers to interact with records
3. Query with filters, sorting, pagination

## AI Handler

Call AI models directly from pipelines. Supports chat (multi-turn) and completion (single-turn) modes.

Key config:
- `provider`: `openai`, `anthropic`, or `google`
- `mode`: `chat` or `completion`
- `messageField`: field name containing the user message (from input)
- `systemPrompt`: system instructions for the AI
- `persistMessages`: store conversation history in a pipeline schema
- `persistMessagesSchemaId`: which schema to store messages in

### `systemPrompt` vs `messageField` — expression syntax (important, easy to get wrong)

These two fields resolve dynamic values with **different rules** — a bare dotted path works for one but not the other:

| Value you write | `messageField` resolves to | `systemPrompt` resolves to |
| --- | --- | --- |
| `steps.prep.system` (bare path) | the step's value (treated as an expression) | **the literal string `"steps.prep.system"`** ⚠️ |
| `{{steps.prep.system}}` (template) | the value | the value |
| `Write about {{request.body.topic}}` | interpolated | interpolated |
| `Plain literal text` | sent as-is | sent as-is |

- **`messageField`** accepts a bare dotted path (`steps.x.y` → expression), a bare word (`message` → `request.body.message`), a `{{…}}` template, or literal text with spaces.
- **`systemPrompt`** only interpolates via `{{…}}` templates (or a whole-string `$`-prefixed expression); **a bare `steps.x.y` path is sent to the model verbatim**, silently dropping your intended system prompt. To reference another step's output, always use **`{{steps.prep.system}}`**.
- `{{…}}` substitution here is a plain string replace (custom, not real Handlebars): no HTML/JSON escaping of string values, and it does not re-process the inserted text, so multi-line prompts with quotes/braces come through intact. (`{{{triple}}}` exists too and behaves the same for strings; it only differs for objects, which it `JSON.stringify`s.)
- `model` is **always literal** — it never evaluates expressions.

Source of truth: `apps/backend/src/pipelines/handlers/ai.handler.ts` (systemPrompt handling) and `execution/expression-evaluator.ts` (`evaluateTemplate`/`evaluateExpression`).

### `ai_handler` output

Access the model's reply at **`steps.<stepName>.content`** (a string) — not via the `messageField` name. In completion mode you can attach images/files with `attachments: [{ type: "image", source: "<expr resolving to a URL or array of URLs>" }]` (an array fans out to one image part per URL; empty values are skipped).

## HTTP Request Handler

Make outbound API calls to external services from within a pipeline.

Key config:
- `url`: target URL (supports expressions like `steps.prev.apiUrl`)
- `method`: GET, POST, PATCH, DELETE
- `body`: request body (expressions supported)
- `headers`: custom headers to add
- `forwardAuth`: forward the original request's auth header

## File Handlers

**Upload** (`file_upload_handler`): Handles multipart file uploads or downloads from URLs. Bytes are **proxied through the backend**, so this path is capped (10MB default `maxFileSize`, plus the server's `client_max_body_size`). Best for small files, server-side image conversion, and local storage. Supports allowed MIME type filtering, max file size, optional image conversion, and date-bucketed storage.

**Serve** (`file_serve_handler`): Streams files from storage with HTTP Range support for video/audio playback. Config: `subDir`, `cacheMaxAge`.

**Image Convert** (`image_convert_handler`): Converts between PNG, JPEG, WebP. Config: `inputPath`, `outputFormat`, `quality`.

**Signed URL** (`signed_url`): Generates time-limited presigned URLs for **downloading** private storage files. Config: `path`, `expiresIn` (seconds, default 3600).

### Direct-to-bucket uploads (large files)

For files larger than the proxied limit, upload **directly to the storage bucket** with a presigned URL — the bytes never pass through nginx or the backend. This is a **two-step, two-pipeline** flow (the client uploads to the bucket between the steps):

**Presigned Upload** (`presigned_upload`) — *prepare*: mints a presigned PUT URL. Config: `subDir`, `filename` (expression, default `request.body.filename`), `dateBucket`, `expiresIn`, `maxFileSize`, `allowedMimeTypes`. Output: `uploadUrl` (client PUTs the file here), `storageKey`, `publicPath`, `originalName`, `expiresAt`.

**Register Upload** (`register_upload`) — *finalize*: verifies the uploaded object, reads its real size/MIME from storage, enforces limits, and writes the **same `pipeline_data` + `asset` record** a normal file upload would. Config: `schemaId`, `subDir`, `storageKey` (expression, default `request.body.storageKey`), `originalName`, `maxFileSize` (default 500MB), `allowedMimeTypes`, `deleteOnViolation`, `extraFields`.

Client flow:
1. `POST` your prepare pipeline with `{ filename }` → returns `{ uploadUrl, storageKey, originalName }`
2. `PUT` the file bytes to `uploadUrl` (straight to the bucket)
3. `POST` your register pipeline with `{ storageKey, originalName }` → writes the record, returns `{ id, url, ... }`

**Requirements & caveats:**
- **Bucket storage only** (S3, GCS, MinIO, Azure). On **local** storage `presigned_upload` errors with `PRESIGNED_NOT_SUPPORTED` — the admin UI disables the handler in the picker when storage can't presign. Use `file_upload_handler` instead.
- The bucket needs **CORS** allowing `PUT` from the site's origin, or the browser blocks the direct upload.
- Use `generate_upload_schema` (or a matching manual schema) for the record fields, and keep `subDir` identical between the prepare and register steps.

## Stripe Handlers

**Checkout** (`stripe_checkout`): Creates Stripe Checkout sessions. Config: `priceId`, `mode` (payment/subscription), `successUrl`, `cancelUrl`, `customerEmail`, `environment` (live/test).

**Webhook** (`stripe_webhook`): Validates webhook signatures and parses events. Config: `allowedEventTypes` (optional filter), `environment`.

## Vector/Embedding Handlers

**Embed Store** (`embed_store`): Stores embedding vectors in the database. Supports single embeddings or chunked documents. Config: `schemaId`, `recordId`, `embedding` or `chunks`.

**Vector Search** (`vector_search`): Queries embeddings by cosine similarity. Config: `schemaId`, `queryVector`, `limit`, `threshold`.

**Replicate** (`replicate`): Calls Replicate ML models for image generation, embeddings, etc. Auto-uploads large files to Replicate's Files API. Config: `model`, `version`, `input`.

## Expression Syntax

Expressions are **bare dot-notation paths** — there is no `${...}` wrapper. Write the path
directly as the config value:

```yaml
fields:
  email: request.body.email        # correct
  name: ${request.body.name}       # WRONG — no such syntax
```

There are exactly **six roots**. Anything else is not an expression:

| Root | Contains |
| --- | --- |
| `request.*` | `body`, `query`, `method`, `path`, `headers`, `ip`, `userAgent` — any other property throws |
| `steps.<name>.*` | Output of a previous step (`steps.<name>` alone gives the whole object) |
| `user.*` | `id`, `email`, `role` — `null` when unauthenticated |
| `metadata.*` | The raw request metadata `request.*` is a friendly alias onto |
| `deployment.*` | `owner`, `repo`, `commitSha`, `alias` |
| `secrets.<NAME>` | Project secrets; a missing one resolves to `null`, it does not throw |

Built-ins: `now()` (ISO 8601 timestamp), `uuid()`. Literals pass through unchanged:
`true`, `false`, `null`, integers, floats, and `"quoted strings"`.

> ⚠️ **`input.*` was removed in CE v0.2.0** (released 2026-07-12) — use `request.body.*`.
> This matters more than it looks: an expression whose root isn't one of the six above is
> **returned as a literal string**, not an error. A pipeline with `name: input.name` writes
> the eight characters `input.name` into your database and mails it out, with a 200 and no
> warning. The same applies to any typo'd root.

### Templates

Inside string values (email bodies, subjects, response bodies), interpolate with braces:

- `{{expr}}` — string interpolation. `null`/`undefined` become empty string; objects become `[object Object]`.
- `{{{expr}}}` — raw output, no HTML escaping. Objects are JSON-stringified — this is how you
  emit a JSON array or object into a response body.

```yaml
subject: "New submission from {{request.body.name}}"
body: '{"episodes": {{{steps.shape.episodes}}}}'
```

## Validators

Pipelines support validators that run before any steps execute:

- `auth_required` - Require authenticated user
- `rate_limit` - Rate limit requests (by IP or user)

Both support conditions to selectively apply (e.g., only rate limit POST requests).

## Common Workflows

**Contact form:**
1. Form handler → parse submission
2. Data CRUD → store in "submissions" schema
3. Email → notify admin
4. Response → thank you message

**AI chat:**
1. Form handler → parse user message
2. AI handler → call model with conversation context
3. Response → return AI response

**File upload with processing (small files, proxied):**
1. File Upload → store file
2. Image Convert → resize/convert
3. Data Create → store metadata
4. Response → return file URL

**Large file upload (direct-to-bucket, two pipelines):**
- Prepare pipeline: Presigned Upload → Response (return `uploadUrl` + `storageKey`)
- Client PUTs the file to `uploadUrl` (bucket), then calls:
- Register pipeline: Register Upload → Response (return the record `url`)

**Stripe payment:**
1. Form handler → parse product selection
2. Stripe Checkout → create session
3. Response → redirect to Stripe

**Semantic search:**
1. Form handler → parse query
2. AI/Replicate → generate query embedding
3. Vector Search → find similar records
4. Response → return results

## Authoring Handler Code via MCP

When you create or update a pipeline through the MCP, the `code` string inside `function_handler.config` and the `body` template inside `response_handler.config` are stored verbatim and displayed verbatim in the admin UI (e.g. `https://admin.j5s.dev/repo/<owner>/<project>/proxy-rules/<setId>/<ruleId>`). The UI does not reformat them.

**Always emit multi-line, indented source — never a minified one-liner.**

- Use real newlines (`\n` in the JSON payload) between statements.
- Indent with 2 spaces.
- Put one statement per line; don't chain multiple statements onto one line with semicolons.
- This applies to any non-trivial `function_handler` body and any `response_handler` body template longer than one expression.
- One-line `function handler() { return {}; }` is fine for true one-liners; everything else should be expanded.

`function_handler` runs in a sandboxed VM: **no** `crypto`, `Buffer`, `require`, `process`, or `fetch`. Use `Math.random()` for randomness, and prefer `var` over `const`/`let` (data_query results may be frozen, and `var` avoids TDZ surprises in the sandbox).

Bad (do not submit code like this):

```json
{ "code": "function handler({ request }) { var h = request.headers || {}; var ip = (h['x-forwarded-for']||'').split(',')[0].trim() || 'unknown'; return { ip: ip }; }" }
```

Good:

```json
{
  "code": "function handler({ request }) {\n  var headers = (request && request.headers) || {};\n  var xff = headers['x-forwarded-for'] || '';\n\n  var firstIp = '';\n  if (typeof xff === 'string' && xff.length > 0) {\n    var parts = xff.split(',');\n    firstIp = parts[0] ? parts[0].trim() : '';\n  }\n\n  return {\n    ip: firstIp || 'unknown',\n  };\n}\n"
}
```

The user opens these rules in the admin UI to review and edit them later. A wall-of-text `code` field forces them to manually reformat before they can read it — treat unformatted handler code the same as committing a minified file to the repo.

### Don't route large result sets through `function_handler` (VM copy cost)

Because `function_handler` runs in a sandboxed VM, **everything it reads from `steps.*` is structure-cloned *into* the VM and everything it returns is cloned back *out***. For a step that does real computation this is fine. But a step whose only job is to **select, wrap, rename, or copy a large result set** pays two-to-three full copies of that data for no real work.

With content-heavy rows (full article HTML, large text/JSON blobs) this materially inflates peak heap for the endpoint and, on a small-heap self-hosted backend, can tip an already-tight process into a V8 "heap out of memory" crash — taking the whole worker down and 502-ing concurrent requests. (Observed live: a reader's `GET /api/items` crashed the backend on ~40 full-content rows until a redundant `merge` step was removed — bffless/apps#148.)

**Respond directly from the data step instead.** `response_handler` bodies interpolate `steps.*`, and `{{{triple}}}` emits the raw JSON of an object/array — so a query result goes straight to the response with no intermediate function:

```jsonc
// ✅ lean: respond straight from the query — no VM round-trip
{ "handlerType": "response_handler",
  "config": { "body": "{\"items\":{{{steps.query}}}}", "contentType": "application/json" } }
```

**To choose between conditional branches, use conditional `response_handler`s — not a `merge`/select function.** Have an early step compute mutually-exclusive flags (e.g. `hasFeed`/`noFeed`), gate each branch's query on one flag, then emit each branch's result from its own `response_handler` carrying the same `condition`. Exactly one query and one responder fire, and the large result never enters a VM:

```
prep (sets hasFeed/noFeed) → queryFeed (cond: hasFeed) → queryAll (cond: noFeed)
                           → respondFeed (cond: hasFeed) → respondAll (cond: noFeed)
```

Rule of thumb: reach for `function_handler` only when you need computation the templating and typed handlers can't express — **never just to rename, wrap, merge, or copy a list.** If the function body is a loop that only pushes rows into a new array, delete it and respond from the source step.

## Configuration Tips

1. Name handlers descriptively for readable expressions
2. Use Response handler last to control what client sees
3. Test with simple inputs before adding validation
4. Check pipeline logs for debugging failed executions
5. Use `postSteps` for async work after the response is sent (e.g., sending emails)
6. Format `function_handler` code and non-trivial `response_handler` bodies as multi-line, indented source — see "Authoring Handler Code via MCP" above
7. Don't add a `function_handler` just to select, wrap, or copy a large result set — respond directly from the data step; see "Don't route large result sets through `function_handler`" above

## Troubleshooting

**Pipeline not triggering?**
- Verify endpoint path matches request URL
- Check HTTP method (GET, POST, etc.) is correct
- Ensure pipeline is enabled and rule set is assigned to alias

**Expression returning undefined?**
- Check handler name matches exactly (case-sensitive)
- Verify previous handler completed successfully
- Use pipeline logs to see actual values at each step
