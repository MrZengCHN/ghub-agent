# Repository Guidelines

## Project Structure & Module Organization
- Root contains `ghub-agent-frontend` (Vue 3 + Vite). App source in `ghub-agent-frontend/src`: `components/`, `views/`, `router/`, `stores/`, `assets/`. Public files live in `ghub-agent-frontend/public`. Builds output to `ghub-agent-frontend/dist`. Path alias `@` -> `src` (see `ghub-agent-frontend/vite.config.js`).

## Build, Test, and Development Commands
- Prereq: Node 20+. From `ghub-agent-frontend`:
  - `npm install` - install dependencies
  - `npm run dev` - start dev server with HMR
  - `npm run build` - production build to `dist/`
  - `npm run preview` - serve the built app locally
- Automated tests are not configured yet; see Testing Guidelines.

## Coding Style & Naming Conventions
- 2-space indentation; Prettier defaults if configured.
- Vue SFCs use `<script setup>`; components in PascalCase (e.g., `TaskList.vue`).
- Pinia stores named `useXStore` under `src/stores/` (e.g., `useCounterStore`).
- Tailwind CSS utility-first; keep global styles minimal in `src/assets/main.css`.
- Prefer alias imports: `import Foo from '@/components/Foo.vue'`.

## Testing Guidelines
- Until tooling lands, include a manual test plan in PRs (pages/routes and edge cases touched).
- Recommended next: Vitest + Vue Test Utils for unit; Playwright for e2e.
- Place tests under `tests/unit` and `tests/e2e`.

## Commit & Pull Request Guidelines
- Use Conventional Commits, e.g.: `feat(frontend): add AgentsView`, `fix(router): handle 404`, `chore: deps`.
- PRs small and focused; include description, linked issues (e.g., `#123`), and UI before/after screenshots.
- Ensure `npm run build` passes and the dev server runs cleanly.

## Security & Configuration Tips
- Do not commit secrets; use Vite env vars with `VITE_` via `.env.local`.
- Match Node to `engines` in `package.json`; use `nvm`/`fnm` to switch.
