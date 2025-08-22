# Instructions for Codex Contributors

Welcome to the Glass project! This file provides guidance for Codex agents working in this repository. Please follow these conventions when modifying the codebase.

## Development environment
- Use **Node.js v20.x.x** and have **Python** installed. Windows users also need Visual Studio Build Tools.
- Clone submodules with `git clone --recurse-submodules` or run `git submodule update --init --recursive` after cloning.
- Run `npm run setup` the first time you clone the repo to install dependencies and build the web dashboard.
- Start the app in development with `npm start` once setup completes.
- Install the Firebase CLI (`npm install -g firebase-tools`) if you need to work on Cloud Functions.

## Repository structure
- `src/` – Electron app source code
  - `features/` – feature modules under `src/features/<feature-name>`
  - `bridge/` – IPC bridges between renderer and main processes
  - `ui/` – renderer UI components and styles
  - `window/` – window configuration and creation
- `pickleglass_web/` – web dashboard built during setup
- `functions/` – Firebase Cloud Functions (Node 18)
- `docs/` – architecture and design documents
- `public/` – static assets used by the app
- `aec/` – audio echo cancellation native module (git submodule)

## Code style and formatting
- The project uses **Prettier** and **ESLint**. Run formatting tools before committing.
- Execute `npm run lint` in the repository root to check for lint errors.

## Architecture guidelines
The codebase follows a strict architectural structure:
- **Service–Repository pattern**: business logic lives in Services, data access in Repositories.
- **Centralized data logic**: Only the Electron Main Process reads or writes persistent data.
- **Feature-based modularity**: New features live under `src/features/<feature-name>`.
- **Dual repositories**: For user data, implement both a local SQLite repository and a Firebase repository exposing the same interface.
- **AI providers**: Add new provider modules in `src/common/ai/providers/` and register them in the provider factory.
- **Schema source of truth**: Update database schema definitions in `src/common/config/schema.js` when making schema changes.
- **Encryption by default**: Sensitive user data must be encrypted before storage.

For deeper architectural details, see `docs/architecture.md` and `docs/DESIGN_PATTERNS.md`.

## Testing & Verification
- After code changes, run `npm run lint` and ensure it passes before committing.
- Make a best effort to run any other relevant build or test scripts provided in `package.json`.

## Pull Requests
- Link each pull request to a corresponding issue using `Closes #<issue-number>` in the description.
- Provide clear summaries of changes and follow the repository's contributing guidelines in `CONTRIBUTING.md`.

Thank you for contributing!
