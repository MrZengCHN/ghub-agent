# Repository Guidelines

## Project Structure & Module Organization
This Vite + Vue 3 workspace keeps executable sources in src/. main.js wires Vue, Pinia, and Vue Router, while App.vue hosts shared layout. Route-level code lives in src/views and is lazy-loaded through src/router/index.js. Reusable UI belongs in src/components (icons sit under components/icons/) and shared state stores in src/stores (counter.js shows the pattern). Static assets are in src/assets, and files that should bypass bundling go to public/. Use the @/ alias from jsconfig.json for imports (import Card from '@/components/Card.vue').

## Build, Test, and Development Commands
Run 
pm install once to pull Node 20+ dependencies. 
pm run dev starts Vite with hot module replacement. 
pm run build emits the production bundle to dist/, and 
pm run preview serves that bundle locally for smoke tests.

## Coding Style & Naming Conventions
Stick to ESM, single quotes, and 2-space indentation as seen in outer/index.js. Name Vue files in PascalCase (AgentsPanel.vue) and Pinia stores in camelCase modules (stores/userSession.js). Prefer <script setup> blocks for new SFCs, keep template roots lean, and favor Tailwind utility classes with any global overrides in src/assets/main.css. Place shared helpers near their usage inside src/ so they can be imported via @/.

## Testing Guidelines
Automated tests are not bundled yet. When adding business logic or complex UI, introduce itest + @vue/test-utils, store specs under src/__tests__/ or alongside components as ComponentName.test.js, and cover Pinia stores, route guards, and critical rendering branches. Use descriptive test names (should navigate to /agents when…) and ensure suites pass via 
px vitest run before opening a PR.

## Commit & Pull Request Guidelines
History is sparse, so follow Conventional Commits to keep logs searchable. Example: eat(agents-view): render assignment list. Keep summaries imperative and under 72 characters, and add body details for context. Each PR should describe intent, list functional changes, record manual test commands, and include screenshots or GIFs for UI updates. Link related issues (Closes #12) and highlight follow-up work.

## Security & Configuration Tips
Load environment-specific values through .env files read by Vite (prefix keys with VITE_). Never commit API keys or tokens; instead, document them in .env.example. Because Vue Devtools plugins are enabled in development, guard any debug-only hooks with if (!import.meta.env.PROD) before merging.
