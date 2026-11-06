# AGENTS.md

Cadence is a course platform (a mini Udemy) and the exercise repo for the AI Coding Crash Course. See `README.md` for setup and `package.json` for scripts.

## Commands

- `npm run typecheck` runs `react-router typegen` before `tsc`. Bare `tsc` reports missing `./+types/*` imports.
- Single test: `npx vitest run <file> -t "<name>"`.
- No linter; formatting is Prettier.

## Architecture

### Layering

Requests flow **route → service → Drizzle**. Routes own HTTP concerns (loaders, actions, redirects, status codes) and rendering. Services export plain synchronous functions that use the shared `db` from `~/db` directly — they never take a db handle — and use positional parameters, not options objects.

### Routing

Routing is config-based: a new page needs a module in `app/routes/` **and** an entry in `app/routes.ts`. Files use dots for segments and `$` for params (`courses.$slug.lessons.$lessonId.tsx`) and import types from `./+types/<route-file-name>`.

### Schema changes

Follow the `database-migrations` skill. `data.db` is disposable; `scripts/seed.ts` is the source of truth for starting data.

### Auth and access control

There are no passwords. The session (`app/lib/session.ts`) stores a `userId` and optional `devCountry`, set from the floating DevUI panel. A 401 "Select a user from the DevUI panel" means no user is selected.

`app/lib/access.server.ts` centralises authorisation. `getAccess(request, courseId)` never throws (lesson pages render a paywall preview); the `require*` guards (`requireUserId`, `requireContentAccess`, `requireStaff`, `requireInstructorOrAdmin`) throw 401/403.

Loaders and actions are authorised independently: every mutating action calls its own `require*` guard.

### Purchasing power parity

`app/lib/ppp.ts` defines discount tiers; `app/lib/country.server.ts` resolves the country (dev override → `CF-IPCountry` → ip-api.com → `null`, treated as Tier 1). A student who bought at a regional discount and browses from a full-price country is `pppBlocked`, losing `canViewContent`. Staff and full-price buyers are exempt.

### Validation

Use the Zod helpers in `app/lib/validation.ts`: `parseFormData` and `parseJsonBody` return per-field errors for inline display; `parseParams` throws a 400.

### Markdown

Two renderers, not interchangeable — both feed `dangerouslySetInnerHTML`:

- `renderMarkdown` (`app/lib/markdown.server.ts`) — trusted instructor content; raw HTML passes through.
- `renderComment` (`app/lib/comment-markdown.server.ts`) — untrusted student comments; escapes HTML, allows only http/https/mailto links, drops images. Treat changes here as security-sensitive.

### Testing

`createTestDb` in `app/test/setup.ts` builds an in-memory database from the real migrations; `seedBaseData` adds a user, instructor, category, and course. Mock `~/db` with a getter and import the module under test **after** the mock — importing it first silently binds the real database:

```ts
let testDb: ReturnType<typeof createTestDb>;

vi.mock("~/db", () => ({
  get db() {
    return testDb;
  },
}));

import { getCourseById } from "~/services/courseService";

beforeEach(() => {
  testDb = createTestDb();
});
```

## Course repo workflow

Lesson checkpoints are slug-prefixed commits on `live-run-through`, driven by `npm run reset`, `cherry-pick`, and `pull`. Slugs are a public interface (CLI and recorded video) — keep them stable.

## Conventions

- Service and lib modules open with a `// ─── Name ───` header recording their responsibility and non-obvious decisions; read it before changing the module.
- Enums are TypeScript `enum`s from `app/db/schema.ts`, used at every insert site including `scripts/seed.ts`.
- Server-only modules use the `.server.ts` suffix.
