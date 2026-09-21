# AMSIO Website — Standalone

## Stack
- Next.js 15 (server deployment, NOT static export)
- Supabase Auth + RLS
- Tailwind CSS 4
- i18n: 5 locales (en/vi/zh/fr/ar)

## Supabase
- Project: `nnqwjreinqiohdxtqzkk.supabase.co`
- Schemas: `cms` (articles, collections, translations, media), `core` (users, institutions), `exam` (competitions, results, certificates)

## Environment Variables (Vercel)
| Variable | Required |
|----------|----------|
| `NEXT_PUBLIC_SUPABASE_URL` | Yes |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Yes |
| `SUPABASE_SERVICE_ROLE_KEY` | Yes (CMS writes) |
| `AMSIO_TENANT_ID` | Yes (`00000000-0000-0000-0000-000000000002`) |
| `ANTHROPIC_API_KEY` | Yes (chatbot) |
| `GEMINI_API_KEY` | Yes (chatbot fallback) |
| `NEXT_PUBLIC_GA_ID` | Optional (`G-95XZMVC7DH`) |
| `UPSTASH_REDIS_REST_URL` | Yes (rate limiting) |
| `UPSTASH_REDIS_REST_TOKEN` | Yes (rate limiting) |

## CMS Auth
- Login at `/cms/login`
- Role: `marketing_staff` in Supabase JWT claims
- Auth uses `@supabase/ssr` directly (no external auth packages)

## Key Directories
- `src/app/` — Next.js pages (public site + CMS)
- `src/lib/cms/` — CMS queries, auth, i18n
- `src/lib/gpu/` — WebGPU/WebGL renderer (3D chatbox fox)
- `src/components/` — Shared React components
- `supabase/migrations/` — Database migrations

## Deploy & Workflow
- GitHub `amsio-international/amsio` → Vercel team `amsio`, project `amsio` → domain `amsio.org`.
- Push to `main` = production deploy (Vercel Git integration, ~2-5 min). There is no CI workflow; do NOT add a second deploy path (it caused duplicate deployments before).
- Always `git pull --ff-only origin main` before starting work — other people push to this repo (PRs are merged on GitHub).
- Before pushing: `npx tsc --noEmit -p tsconfig.json` must exit 0.
- Check the deploy: Vercel → Deployments (should be Ready), then open the live page. Local `npm run dev` needs `.env.local` (copy `.env.example`; real values live only in Vercel env vars).
- This repo is a separate project from the itran-core monorepo. Editing `itran-core/apps/amsio-website` does NOT change amsio.org.

## Hosting Gotchas (learned the hard way)
- Repo is **PUBLIC on purpose**: Vercel Hobby blocks deployments of private repos when the commit author is not the Vercel account owner ("commit author does not have contributing access"). Making it private again requires Vercel Pro. So: NEVER commit secrets, passwords, tokens or account lists to this repo (incl. this file).
- Supabase Storage bucket `cms-assets` MUST stay `public = true`. Article covers are served from `/storage/v1/object/public/cms-assets/...`; if the bucket is private the images break (404 "Bucket not found"). Uploads go through `/api/v1/cms/media` using the service-role key.
- Chatbot (`src/app/api/v1/chatbox/route.ts`): `max_tokens` / `maxOutputTokens` = 1024. It was 400 and answers were cut off mid-sentence — do not lower it. Tries Claude first, falls back to Gemini. As of 2026-08-28 Vercel only had `GEMINI_API_KEY` (no `ANTHROPIC_API_KEY` / `UPSTASH_*`), so it runs on Gemini with in-memory rate limiting.
- `/cms/login` just redirects to `/portal/login` (single shared login form); `marketing_staff` users are sent to `/cms` after sign-in.
- `NEXT_PUBLIC_FB_PIXEL_ID` (optional) is read by the Facebook Pixel script in `src/app/layout.tsx`.

## CMS Articles (how it behaves)
- Editor: `src/components/cms/ArticleEditor.tsx`. Slug auto-generates from the EN title until edited by hand. Buttons: Save Draft and Publish (Publish = save + publish in one click; shows a success message).
- Body is Markdown, rendered with `react-markdown` (`.article-body` styles in `src/app/typography.css`). Excerpt is plain text — do not paste Markdown there.
- Cover image: "Upload image" button (from computer) or paste a URL.
- Delete is a soft delete (`DELETE /api/v1/cms/articles/[id]`).

## Known Debt
- No article approval workflow: `articles_status_check` is draft|published only
- `/api/v1/cms/health` is a diagnostic route — remove before production
- The CMS test account (`cms.test@amsio.org`) still has a weak password — change it in Supabase Auth before real use. Credentials are handed over outside this repo.
