# Instructions for AI Agents

This repository provides a comprehensive migration guide for helping developers move from Google AI Studio/Vite applications to a secure production architecture using **Hono + Cloudflare Workers**.

## How to Use This Guide

When a developer asks for help migrating their AI Studio app, follow this workflow:

1. **Understand their current setup**
   - Vite-based React app built with Google AI Studio
   - API keys exposed via Vite `define` config in `vite.config.ts`
   - Using `@google/genai` or similar AI SDKs
   - Likely has a `services/` directory with client-side API calls

2. **Copy migration specs**
   - Copy the entire migration guide from `specs/hono-cloudflare-migration.md`
   - Do NOT commit this file to their repository
   - Use it as a reference while refactoring their codebase

3. **Execute migration step-by-step**
   - **Create infrastructure files**: `wrangler.toml`, `src/worker.ts`, `api/` directory
   - **Migrate service logic**: Move client-side services from `services/` to `api/` routes
   - **Update frontend**: Change imports from local services to `fetch('/api/...')`
   - **Configure TypeScript**: Update `tsconfig.json` and related configs
   - **Setup secrets**: Create `.dev.vars` for local development
   - **Update package.json**: Add deployment scripts (`npm run deploy`)

4. **Verify and test**
   - Ensure API calls work locally with `npm run dev`
   - Build succeeds with `npm run build`
   - Guide them on setting production secrets: `wrangler secret put GEMINI_API_KEY`

5. **Optional enhancements**
   - Add GitHub Actions workflow for CI/CD
   - Implement rate limiting
   - Add caching strategies
   - Set up monitoring

## Repository Structure

- **`specs/hono-cloudflare-migration.md`** - Complete migration guide (450+ lines)
  - Contains all code examples, configuration files, and troubleshooting
  - Use it as your instruction manual while refactoring
- **`README.md`** - Project overview for human developers
- **`examples/`** - Example implementations (if available)
- **`components/`** - Reusable UI components (if available)

## Key Migration Points

### Infrastructure to Create
- `wrangler.toml` - Cloudflare Workers configuration
- `src/worker.ts` - Hono entry point with SPA fallback
- `api/index.ts` - API route aggregator  
- `api/ai.ts` - Migrated AI service endpoints
- `.dev.vars` - Local development secrets

### Files to Modify
- `vite.config.ts` - Remove `define` block exposing API keys
- `package.json` - Add deployment scripts
- `tsconfig.json`, `tsconfig.app.json` - Include API directory
- Frontend components - Update to fetch from `/api/*` endpoints

### Files to Delete
- `services/[SERVICE].ts` - Old client-side service files

## Agent Workflow Summary

1. Read the specs file thoroughly
2. Copy its contents for personal reference (don't commit it)
3. Create new infrastructure files in the developer's repo
4. Migrate service logic from client to server
5. Update frontend code to use new API endpoints
6. Test locally using `npm run dev`
7. Guide deployment using `npm run deploy`

The specs file is your playbook - the developer's repo should contain only the actual migrated code, not the guide itself.