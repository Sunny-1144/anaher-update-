# ANAHER

ANAHER is a Kotlin/Jetpack Compose Android app with a companion React web workspace, an Electron desktop shell, and a TypeScript API for account-scoped data and AI conversations.

## What is included

- `app/` — existing Android app and local Room database.
- `apps/web/` — responsive Tropical Teal and spatial-glass UI, OIDC sign-in integration, LLM-style chat, missions, projects, knowledge, memory, analytics, model selection, and account settings.
- `apps/api/` — authenticated Express API, PostgreSQL schema/migrations, account-scoped resource endpoints, sync change cursor, and a server-side multi-provider streaming layer.
- `apps/desktop/` — hardened Electron shell that packages the built web client behind the `anaher://app` protocol. It uses context isolation, a sandboxed renderer, a narrow preload bridge, and disables Node integration.

The backend uses verified OIDC bearer tokens and fails closed if issuer, audience, or JWKS configuration is missing. **A real sign-in provider, PostgreSQL database, and at least one model-provider key are external setup requirements; no credentials or provider account are included.**

## Runtime components and dependencies

| Component | Main dependencies | Workflow boundary |
|---|---|---|
| Android | Kotlin, Jetpack Compose, Room, AppAuth, OkHttp | Local/offline workspace and OIDC account link; snapshot upload is available. Current chat still uses the legacy Android `MatrixEngine`/`GeminiService` path. |
| Web | React, TypeScript, Vite, `oidc-client-ts`, Lucide | Shared browser UI; account data and AI responses come from the API. |
| Desktop | Electron, the same built web client | Native window and secure deep-link shell; no separate UI/API implementation. |
| API | Node.js 20+, Express, PostgreSQL (`pg`), `jose`, `zod`, rate limiting, Helmet | OIDC verification, account-scoped data, migrations, chat orchestration, SSE, and model adapters using Node's built-in `fetch`. |
| External services | OIDC provider, PostgreSQL, one or more AI provider endpoints | Operator configured; not provisioned by this source tree. Blob storage and background job queue are future integrations, not current requirements. |

## Local prerequisites

- Node.js 20 or newer and npm.
- Docker Desktop for local PostgreSQL (or another PostgreSQL 15+ instance).
- OIDC identity-provider registration supporting Authorization Code + PKCE, with a web redirect URL of `http://localhost:5173/auth/callback`.
- At least one server-side AI provider key for live assistant responses. Gemini is supported directly; OpenAI and Anthropic adapters are enabled only when configured.

## Configure and run

1. Copy `apps/api/.env.example` to `apps/api/.env` and set `DATABASE_URL`, `OIDC_ISSUER`, `OIDC_AUDIENCE`, and `OIDC_JWKS_URL`. Configure at least one provider key (`GEMINI_API_KEY`, `OPENAI_API_KEY`, or `ANTHROPIC_API_KEY`) and keep the actual `.env` file private. For OpenAI or Anthropic, set the corresponding comma-separated `*_MODELS` allowlist as well.
2. Copy `apps/web/.env.example` to `apps/web/.env.local`; set the OIDC authority and public client ID. Set `VITE_OIDC_SCOPE` to include the API scope configured by the identity provider.
3. Start local PostgreSQL: `docker compose up -d postgres`. The compose password is for local development only.
4. Install workspace packages: `npm install`.
5. Apply the schema: `npm run db:migrate`.
6. Start the API and web app: `npm run dev`.
7. Open `http://localhost:5173`, sign in through the configured provider, and create conversations. The Vite development server proxies API requests to port 3001.

For Electron development, start the Vite app first, then run `npm run dev:desktop` with `ANAHER_OIDC_AUTHORITY` set to the configured authority. For distribution, build the shared web client with `VITE_API_BASE_URL` set to the HTTPS API origin, then package via `npm --workspace @anaher/desktop run package`. Register `anaher://app/auth/callback` and `anaher://app/` as the OIDC redirect and post-logout URIs, and provide `ANAHER_OIDC_AUTHORITY` to the desktop process. The provider and client registrations are operator choices; none are provisioned by this project.

## API security and data behavior

- Protected `/v1` routes derive the account only from a verified token; client-supplied user IDs are ignored.
- Queries are parameterized and resource operations are scoped by the authenticated database user.
- Provider keys remain in the API environment. The API exposes only explicitly configured model IDs, bounds prompts and outputs, and streams provider responses through one shared SSE contract.
- Chat responses stream as server-sent events and are persisted in PostgreSQL.
- Gemini is the default when configured. OpenAI and Anthropic can be enabled with server-side keys and explicit model allowlists; OpenAI-compatible endpoints can be selected with `OPENAI_BASE_URL`. Provider failures are reported rather than silently switching providers, so user choice, data handling, and cost are predictable. Provider-specific SDK dependencies are not required; the API uses Node's built-in `fetch`.
- Android includes OIDC sign-in with encrypted token storage and can upload Room conversations/messages, missions/tasks/logs/artifacts, projects, knowledge snippets, and memories to the signed-in account. `/v1/sync/import` is account-scoped and idempotent; `/v1/sync` exposes account-scoped changes for clients that implement pull synchronization. Android currently uploads local snapshots; applying remote web/desktop changes back into Room and propagating Android deletions are not yet implemented.
- Android chat still uses the existing `MatrixEngine`/`GeminiService` path; the new provider registry currently serves web and desktop. Do not put production model keys in Android build configuration. Routing mobile chat through the authenticated API is a remaining integration step before claiming one shared AI workflow across all three clients.
- To prevent data from one Android profile being uploaded to another account, a device's local database is bound to the first account used for sync. Use a separate Android app installation/profile for another account; clearing app data also clears that device's local workspace.
- Knowledge Vault currently stores text and metadata. No document file upload is exposed until private object storage is configured.

## Shared AI and application workflow

```text
Android (Room/offline) ─┐
Browser (React/Vite) ───┼─ OIDC access token ─> Express API ─> PostgreSQL
Desktop (Electron shell)┘                              └─> configured model adapter
```

1. Android, browser, and desktop authenticate through OIDC. The API validates the bearer token and derives the account identity; clients never choose a database user.
2. The browser and desktop use the same React application and `/v1` API. Android keeps its existing Room database and can upload account-scoped snapshots after sign-in or via Settings; its current chat inference still uses the pre-existing mobile path.
3. Web/desktop chat history is bounded by the API, the selected model is checked against configured provider/model allowlists, and provider credentials remain on the API. Provider streams are normalized to the same SSE `delta`, `completed`, and `error` events; completed conversations and messages persist in PostgreSQL.
4. Model options appear only when their provider key and model allowlist are configured. Gemini stays the default when configured. OpenAI and Anthropic are optional; the OpenAI base URL can target an OpenAI-compatible service. The API does not silently switch providers because each provider has different privacy and billing behavior.

The API currently handles chat as a bounded streaming request. It does not yet include a durable worker queue for long-running mission jobs, nor an Android pull/apply loop for changes made in web or desktop. Those should be added before claiming complete offline-first bidirectional synchronization or asynchronous agent workflows. A production deployment also needs TLS, managed secret storage, PostgreSQL backups/migrations, provider-specific quotas and monitoring, and an OIDC registration. Azure resources are not provisioned by this repository.

## Verification

Run `npm run build` and `npm test` from the repository root. The build does not require OIDC or Gemini credentials. To run migrations and end-to-end account/AI flows, the local database and external provider credentials must be configured.

## Room sync API

Every `/v1` request uses `Authorization: Bearer <OIDC access token>`. Do not send an account/user ID; the server obtains the account only from verified `(iss, sub)` claims.

`POST /v1/sync/import` accepts a JSON batch containing 1–100 mutations (maximum HTTP JSON body is 1 MB). Keep `clientMutationId` (a UUID) stable and retry the **identical batch** after an uncertain response. A repeated ID returns the stored result with `replayed: true`. Each entity has a stable `clientId`, an increasing integer `version` starting at 1, an `operation` (`upsert` or `delete`), and a type-specific `data` object for upserts. Use install-scoped client IDs such as `android:<installation-uuid>:missions:<Room-id>`; stable IDs are mapping keys and are not user identifiers.

Supported `entityType`: `conversation`, `message`, `mission`, `mission_task`, `mission_log`, `mission_artifact`, `project`, `knowledge`, `memory`. Import order is handled server-side so missions and conversations precede their child rows. Mission children identify their parent using `missionClientId`. A Room message can use `sessionId` or `conversationClientId`; absent conversations are created using that client key. Room message roles `USER`, `ANAHER`, `AGENT`, and `SYSTEM` map to API roles `user`, `assistant`, `assistant`, and `system`. Timestamps accept Unix epoch milliseconds or ISO 8601. Knowledge accepts text and metadata only; binary uploads are not implemented.

Example:

```json
{
  "clientMutationId": "4c4a0f7e-93df-4d82-9cc5-a489a09c0f1b",
  "changes": [
    {
      "clientId": "android:install-uuid:missions:42",
      "entityType": "mission",
      "version": 1,
      "operation": "upsert",
      "data": {
        "title": "Build an ANAHER companion",
        "description": "Cross-client project",
        "stage": "PLANNING",
        "progressPercent": 20,
        "modelVersion": "Matrix 4.1 Forge",
        "createdAt": 1793484000000,
        "updatedAt": 1793484000000
      }
    },
    {
      "clientId": "android:install-uuid:mission_tasks:7",
      "entityType": "mission_task",
      "version": 1,
      "operation": "upsert",
      "data": {
        "missionClientId": "android:install-uuid:missions:42",
        "title": "Review API contracts",
        "assignedAgent": "A4 Planner",
        "status": "PENDING",
        "orderIndex": 0,
        "durationMs": 0,
        "outputSummary": ""
      }
    }
  ]
}
```

The response contains per-item `clientId`, `entityType`, server `entityId` (UUID), applied `version`, and `outcome` (`applied`, `stale`, or `deleted`). Versions at or below the last applied version do not overwrite server state. Persist the returned ID mapping if useful; stable client IDs also resolve it on later import.

Poll `GET /v1/sync?cursor=0&limit=200`; advance only after applying returned changes successfully in the local database. Follow the returned cursor while `hasMore` is true. Each change includes `cursor`, `entityType`, server `entityId`, `operation`, `entityVersion`, UTC `createdAt`, and camelCase `payload`. Upsert payloads are complete snapshots; imported payloads contain `clientId` and child payloads also contain `missionClientId` or `conversationClientId`. Delete payloads are tombstones with `id` and `clientId` when known. All queries and mutation results are scoped to the verified user.
