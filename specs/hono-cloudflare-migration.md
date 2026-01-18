# Generic Guide: Migrating AI Studio Apps to Hono + Cloudflare Workers

This guide helps you migrate any Google AI Studio-built Vite application from client-side API calls to a secure server-side architecture using Hono and Cloudflare Workers.

---

## Prerequisites

- Vite-based React application built with Google AI Studio
- Uses `@google/genai` or similar AI SDK
- Current API keys exposed client-side via Vite `define` config
- Git repository initialized

---

## Security Issue

Your current setup exposes API keys client-side:

```typescript
// vite.config.ts - INSECURE
export default defineConfig({
  define: {
    'process.env.API_KEY': JSON.stringify(env.GEMINI_API_KEY),
    // This bundles the key into client-side JavaScript!
  }
})
```

**Risk**: API keys are visible in browser dev tools and bundled JS files.

**Solution**: Move API calls to server-side Hono backend running on Cloudflare Workers.

---

## Architecture Overview

**Before (Insecure)**:
```
Browser → Vite App → Google AI API (with exposed key)
```

**After (Secure)**:
```
Browser → Cloudflare Worker → Hono API → Google AI API (key secured)
            ↓
        Vite Static Assets
```

---

## Migration Steps

### 1. Install Dependencies

```bash
npm install hono @google/genai
npm install -D wrangler @cloudflare/workers-types
```

**Recommended versions:**
- `hono`: ^4.8.4
- `@google/genai`: ^1.8.0
- `wrangler`: ^4.24.1

### 2. Create `wrangler.toml`

```toml
name = "[YOUR_PROJECT_NAME]"
main = "src/worker.ts"
compatibility_date = "2024-05-01"

[assets]
directory = "./dist"
binding = "ASSETS"

[observability.logs]
enabled = true
```

### 3. Create `src/worker.ts` (Hono Entry Point)

```typescript
import { Hono } from 'hono';
import apiRoutes from '../api/index';

export type Env = {
  GEMINI_API_KEY: string;
  ASSETS: {
    fetch: (req: Request) => Promise<Response>;
  };
};

const app = new Hono<{ Bindings: Env }>();

// Mount all API routes
app.route('/api', apiRoutes);

// SPA fallback for client-side routing
app.get('*', async (c) => {
  if (c.req.path.startsWith('/api')) {
    return new Response('Not Found', { status: 404 });
  }

  let res = await c.env.ASSETS.fetch(c.req.raw);

  if (res.status === 404) {
    const url = new URL(c.req.url);
    url.pathname = '/index.html';
    const indexRequest = new Request(url.toString(), {
      headers: c.req.raw.headers,
      method: 'GET',
    });
    res = await c.env.ASSETS.fetch(indexRequest);
  }

  return res;
});

export default app;
```

### 4. Create API Directory Structure

```bash
mkdir api
touch api/index.ts
```

### 5. Migrate Service to API Route

**Before**: `services/[YOUR_SERVICE].ts` (client-side)

**After**: `api/[your-endpoint].ts` (server-side)

Create `api/index.ts` to export all routes:

```typescript
import { Hono } from 'hono';
import { aiRouter } from './ai';

const app = new Hono();

app.route('/ai', aiRouter);
// Add more routes as needed:
// app.route('/chat', chatRouter);
// app.route('/analyze', analyzeRouter);

export default app;
```

Create `api/ai.ts` (migrate your service logic):

```typescript
import { Hono } from 'hono';
import { GoogleGenAI } from '@google/genai';

export const aiRouter = new Hono<{ Bindings: Env }>();

// Example: Chat endpoint
aiRouter.post('/chat', async (c) => {
  const apiKey = c.env.GEMINI_API_KEY;
  if (!apiKey) {
    return c.json({ error: 'API key not configured' }, 500);
  }

  const { history, message } = await c.req.json<{
    history: Array<{ role: string; content: string }>;
    message: string;
  }>();

  const ai = new GoogleGenAI({ apiKey });
  
  // Your existing logic from services/[YOUR_SERVICE].ts
  const response = await ai.models.generateContent({
    model: 'gemini-2.5-flash',
    contents: [...history, { role: 'user', parts: [{ text: message }] }],
  });

  return c.json({ response: response.text });
});

// Example: Analysis endpoint
aiRouter.post('/analyze', async (c) => {
  const apiKey = c.env.GEMINI_API_KEY;
  if (!apiKey) {
    return c.json({ error: 'API key not configured' }, 500);
  }

  const { videoUrl, prompt } = await c.req.json<{
    videoUrl: string;
    prompt: string;
  }>();

  const ai = new GoogleGenAI({ apiKey });
  
  // Your existing streaming logic
  const stream = await ai.models.generateContentStream({
    model: 'gemini-3-pro-preview',
    contents: {
      parts: [
        { fileData: { fileUri: videoUrl, mimeType: 'video/mp4' } },
        { text: prompt }
      ]
    },
  });

  // Handle streaming response...
  return c.json({ result: 'analysis complete' });
});
```

### 6. Update `package.json` Scripts

```json
{
  "scripts": {
    "dev": "wrangler dev src/worker.ts --local",
    "build": "vite build",
    "preview": "vite preview",
    "deploy": "npm run build && wrangler deploy src/worker.ts",
    "lint": "eslint ."
  }
}
```

### 7. Create `.dev.vars` (Local Development Secrets)

```ini
GEMINI_API_KEY=your-api-key-here
```

**Add to `.gitignore`:**
```
.dev.vars
.env.local
.env
```

### 8. Update `vite.config.ts`

**Remove** the `define` block that exposes API keys:

```typescript
import path from 'path';
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### 9. Update TypeScript Configuration

**`tsconfig.json`:**
```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

**`tsconfig.app.json`:**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["src", "api", "src/worker.ts"]
}
```

**`tsconfig.node.json`:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["vite.config.ts"]
}
```

### 10. Update Frontend API Calls

**Before** (client-side service import):
```typescript
import { getAIResponse } from '../services/aiService';
const response = await getAIResponse(data);
```

**After** (fetch to backend):
```typescript
const response = await fetch('/api/ai/chat', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ history, message }),
});

if (!response.ok) {
  const err = await response.json();
  throw new Error(err.error || 'API Error');
}

const data = await response.json();
return data.response;
```

### 11. Delete Client-Side Service File

```bash
rm services/[YOUR_SERVICE].ts
```

### 12. GitHub Actions Deployment (Optional)

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Cloudflare

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install
      - run: npm run build
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

---

## New Project Structure

```
[project-name]/
├── src/
│   ├── worker.ts          # Hono entry point
│   ├── App.tsx
│   ├── main.tsx
│   └── components/
├── api/
│   ├── index.ts           # API route aggregator
│   └── ai.ts              # Migrated AI service logic
├── services/              # DELETE: Old client-side services
├── dist/                  # Vite build output
├── wrangler.toml          # Cloudflare Workers config
├── .dev.vars              # Local dev secrets
├── vite.config.ts         # Modified (no define block)
├── tsconfig.json          # Modified (project references)
├── tsconfig.app.json      # App-specific TS config
├── tsconfig.node.json     # Node-specific TS config
└── package.json           # Modified scripts + deps
```

---

## Common API Patterns

### Pattern 1: Chat/Conversation
```typescript
// Frontend
const response = await fetch('/api/ai/chat', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ history, message }),
});

// Backend (api/ai.ts)
aiRouter.post('/chat', async (c) => {
  const { history, message } = await c.req.json();
  // ... process with AI SDK
  return c.json({ response: aiResponse });
});
```

### Pattern 2: File/Video Analysis
```typescript
// Frontend
const response = await fetch('/api/ai/analyze', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ videoUrl, prompt }),
});

// Backend (api/ai.ts)
aiRouter.post('/analyze', async (c) => {
  const { videoUrl, prompt } = await c.req.json();
  // ... process with AI SDK
  return c.json({ result: analysis });
});
```

### Pattern 3: Streaming Response
```typescript
// Frontend
const response = await fetch('/api/ai/stream', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ prompt }),
});

const reader = response.body?.getReader();
// ... handle streaming

// Backend (api/ai.ts)
aiRouter.post('/stream', async (c) => {
  const { prompt } = await c.req.json();
  const stream = await ai.models.generateContentStream({...});
  
  return new Response(stream, {
    headers: { 'Content-Type': 'text/plain' },
  });
});
```

---

## Cloudflare Secrets (Production)

Set your API key via Cloudflare dashboard or Wrangler CLI:

```bash
# Using Wrangler CLI
wrangler secret put GEMINI_API_KEY

# Or via Cloudflare Dashboard
# Workers & Pages → Your Worker → Settings → Variables
```

For CI/CD (GitHub Actions), add these secrets:
- `CLOUDFLARE_API_TOKEN`
- `CLOUDFLARE_ACCOUNT_ID`
- `GEMINI_API_KEY`

---

## Testing

### Local Development
```bash
npm run dev
```

### Preview Production Build
```bash
npm run build
npm run preview
```

### Deploy
```bash
npm run deploy
```

---

## Troubleshooting

### Issue: API returns 404
- Ensure your route is mounted in `src/worker.ts`
- Check the path matches exactly (`/api/ai/chat` vs `/api/chat`)

### Issue: Environment variable not found
- Local: Check `.dev.vars` exists and is formatted correctly
- Production: Run `wrangler secret put GEMINI_API_KEY`

### Issue: TypeScript errors
- Ensure `tsconfig.app.json` includes `"api"` directory
- Check `@cloudflare/workers-types` is installed

### Issue: Build fails
- Run `npm run build` separately to see Vite errors
- Ensure `vite.config.ts` has no `define` block

### Issue: Streaming not working
- Ensure you're returning a `Response` object with proper headers
- Check browser console for CORS issues (shouldn't occur with same-origin)

---

## Benefits of This Architecture

1. **Security**: API keys never leave the server
2. **Performance**: Edge computing with Cloudflare's global network
3. **Scalability**: Workers automatically scale
4. **Cost**: Generous free tier (100,000 requests/day)
5. **Maintainability**: Clear separation of client/server logic
6. **Type Safety**: Full TypeScript support throughout

---

## Next Steps

- Add rate limiting to prevent abuse
- Implement caching for repeated requests
- Add authentication if needed
- Monitor usage with Cloudflare analytics
- Set up custom domain

---

## Support

For issues specific to:
- **Hono**: https://hono.dev/docs
- **Cloudflare Workers**: https://developers.cloudflare.com/workers
- **Google GenAI SDK**: https://github.com/googleapis/js-genai
