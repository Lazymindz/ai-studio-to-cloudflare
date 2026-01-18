# AI Studio to Cloudflare Workers

A comprehensive migration guide for converting Google AI Studio-built Vite applications from insecure client-side API calls to a secure server-side architecture using Hono and Cloudflare Workers.

## The Problem You Face

You built an amazing AI app in Google AI Studio with React and Vite. It works great, but there's a critical security issue: **your API keys are exposed client-side**. Anyone can view them in browser dev tools, making your application vulnerable to abuse and unexpected costs.

This repo provides everything you need to migrate from an insecure client-side app to a secure, production-ready architecture.

## What This Guide Provides

- **Complete Migration Steps**: Step-by-step instructions to move from client to server architecture
- **Production-Ready Code**: Modular Hono API patterns (chat, analysis, streaming)
- **Security Best Practices**: API key protection, environment management, deployment security
- **TypeScript Configuration**: Proper TS setup for client and server code
- **CI/CD Deployment**: GitHub Actions workflow for automated deployments
- **Troubleshooting Guide**: Solutions to common issues you'll encounter

## Architecture Overview

### Before (Insecure)
```
Browser → Vite App → Google AI API (with exposed key)
```

### After (Secure)
```
Browser → Cloudflare Worker → Hono API → Google AI API (key secured)
            ↓
        Vite Static Assets
```

## Quick Start

1. Install dependencies:
   ```bash
   npm install hono @google/genai
   npm install -D wrangler @cloudflare/workers-types
   ```

2. Create `wrangler.toml` and `src/worker.ts` (see [specs/hono-cloudflare-migration.md](specs/hono-cloudflare-migration.md) for full code)

3. Migrate your client-side service to API routes:
   - Move logic from `services/` to `api/`
   - Update frontend to fetch from `/api/*` endpoints

4. Set environment variables:
   - Local: Create `.dev.vars` with your API key
   - Production: `wrangler secret put GEMINI_API_KEY`

5. Deploy:
   ```bash
   npm run deploy
   ```

## Key Benefits

- **Security**: API keys never leave the server
- **Performance**: Edge computing with Cloudflare's global network
- **Scalability**: Workers automatically scale to any load
- **Cost**: Generous free tier (100,000 requests/day)
- **Maintainability**: Clear separation of client/server logic
- **Type Safety**: Full TypeScript support throughout

## Repository Structure

```
├── specs/                      # Detailed migration guide
│   └── hono-cloudflare-migration.md
├── README.md                   # This file
```

## Who This Is For

Developers who built React apps with Google AI Studio and need to secure their API keys before production deployment.

## Next Steps

Read the complete migration guide: [specs/hono-cloudflare-migration.md](specs/hono-cloudflare-migration.md)

This guide includes:
- Detailed architecture explanations
- Full code examples for all migration steps
- API patterns (chat, analysis, streaming)
- TypeScript configuration
- GitHub Actions CI/CD setup
- Troubleshooting for common issues

## AI Agent Instructions

If you're an AI assistant helping a developer, see [AGENTS.md](AGENTS.md) for detailed workflow instructions on how to use the specs folder effectively.
