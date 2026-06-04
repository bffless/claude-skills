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
| **Signed URL** | `signed_url` | Generate time-limited presigned URLs for storage files |
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

## HTTP Request Handler

Make outbound API calls to external services from within a pipeline.

Key config:
- `url`: target URL (supports expressions like `${steps.prev.apiUrl}`)
- `method`: GET, POST, PATCH, DELETE
- `body`: request body (expressions supported)
- `headers`: custom headers to add
- `forwardAuth`: forward the original request's auth header

## File Handlers

**Upload** (`file_upload_handler`): Handles multipart file uploads or downloads from URLs. Supports allowed MIME type filtering, max file size, optional image conversion, and date-bucketed storage.

**Serve** (`file_serve_handler`): Streams files from storage with HTTP Range support for video/audio playback. Config: `subDir`, `cacheMaxAge`.

**Image Convert** (`image_convert_handler`): Converts between PNG, JPEG, WebP. Config: `inputPath`, `outputFormat`, `quality`.

**Signed URL** (`signed_url`): Generates time-limited presigned URLs for private storage files. Config: `path`, `expiresIn` (seconds, default 3600).

## Stripe Handlers

**Checkout** (`stripe_checkout`): Creates Stripe Checkout sessions. Config: `priceId`, `mode` (payment/subscription), `successUrl`, `cancelUrl`, `customerEmail`, `environment` (live/test).

**Webhook** (`stripe_webhook`): Validates webhook signatures and parses events. Config: `allowedEventTypes` (optional filter), `environment`.

## Vector/Embedding Handlers

**Embed Store** (`embed_store`): Stores embedding vectors in the database. Supports single embeddings or chunked documents. Config: `schemaId`, `recordId`, `embedding` or `chunks`.

**Vector Search** (`vector_search`): Queries embeddings by cosine similarity. Config: `schemaId`, `queryVector`, `limit`, `threshold`.

**Replicate** (`replicate`): Calls Replicate ML models for image generation, embeddings, etc. Auto-uploads large files to Replicate's Files API. Config: `model`, `version`, `input`.

## Expression Syntax

Access data throughout the pipeline using expressions:

- `input.*` - Parsed request body
- `query.*` - URL query parameters
- `params.*` - URL path parameters
- `headers.*` - Request headers
- `steps.<name>.*` - Output from previous handler
- `user.*` - Authenticated user info (if applicable)

Example: `${input.email}` or `${steps.createUser.id}`

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

**File upload with processing:**
1. File Upload → store file
2. Image Convert → resize/convert
3. Data Create → store metadata
4. Response → return file URL

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

## Configuration Tips

1. Name handlers descriptively for readable expressions
2. Use Response handler last to control what client sees
3. Test with simple inputs before adding validation
4. Check pipeline logs for debugging failed executions
5. Use `postSteps` for async work after the response is sent (e.g., sending emails)
6. Format `function_handler` code and non-trivial `response_handler` bodies as multi-line, indented source — see "Authoring Handler Code via MCP" above

## Troubleshooting

**Pipeline not triggering?**
- Verify endpoint path matches request URL
- Check HTTP method (GET, POST, etc.) is correct
- Ensure pipeline is enabled and rule set is assigned to alias

**Expression returning undefined?**
- Check handler name matches exactly (case-sensitive)
- Verify previous handler completed successfully
- Use pipeline logs to see actual values at each step
