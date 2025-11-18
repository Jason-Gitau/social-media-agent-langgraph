# Social Media Agent - Claude Context

## Project Overview

An AI-powered agent that automates social media content creation and posting using LangGraph. Takes URLs (blog posts, GitHub repos, YouTube videos, etc.), extracts content, generates optimized posts for Twitter and LinkedIn, and includes human-in-the-loop approval before publishing.

**Real-world use case:** Team shares URLs in Slack → Agent generates posts overnight → Human reviews in Agent Inbox UI → Posts are scheduled and published

## Quick Reference

### Essential Commands

```bash
# Development
yarn install                  # Install dependencies
yarn langgraph:in_mem:up     # Start dev server (port 54367)
yarn generate_post           # Run demo post generation

# Testing
yarn test                    # Unit tests only
yarn test:int                # Integration tests
yarn lint                    # Lint code
yarn format                  # Format code

# Authentication
yarn start:auth              # OAuth server for Twitter/LinkedIn (localhost:3000)

# Cron Jobs
yarn cron:create             # Create scheduled job
yarn cron:list               # List all crons
yarn cron:delete             # Delete a cron

# Utilities
yarn get:used_links          # View all processed URLs
yarn graph:backfill          # Batch process multiple URLs
```

### Key File Locations

```
src/agents/generate-post/        # Main post generation workflow
├── generate-post-graph.ts       # Graph definition
├── prompts/index.ts             # CUSTOMIZE: Business context & style
├── prompts/examples.ts          # CUSTOMIZE: Few-shot post examples
└── nodes/                       # Individual processing steps

src/agents/verify-links/         # URL content extraction
├── verify-links-graph.ts        # Routes URLs to specialized handlers
└── nodes/                       # GitHub, YouTube, Twitter, Reddit handlers

src/agents/shared/nodes/         # Reusable nodes across graphs
├── generate-post/               # Post generation & scheduling
├── find-images/                 # Image discovery & ranking
└── human-node/                  # Human-in-the-loop interrupt

src/clients/                     # External service integrations
├── twitter/                     # Twitter API v2 client
├── linkedin.ts                  # LinkedIn OAuth & posting
├── slack/                       # Slack messaging
└── auth-server.ts              # OAuth flow for manual setup

src/utils/                       # Helper utilities
├── firecrawl.ts                # Web scraping
├── supabase.ts                 # Image storage
├── github-repo-contents.ts     # GitHub API
└── date.ts                     # Date/timezone handling

scripts/                         # CLI utilities
├── generate-post.ts            # Demo script
├── crons/                      # Cron job management
└── backfill.ts                 # Bulk processing

langgraph.json                  # Graph deployment config
.env                            # Environment variables (NEVER commit)
```

## Architecture

### Core Technology Stack

- **Framework:** LangGraph 1.0.2 (multi-agent orchestration)
- **Runtime:** Node.js 20+, TypeScript 5.3.3
- **LLM:** Anthropic Claude (via @langchain/anthropic 1.0.0)
- **Tracing:** LangSmith (debugging & monitoring)
- **Package Manager:** Yarn 1.22.22

### 14 Independent Graphs

| Graph | Purpose | File Path |
|-------|---------|-----------|
| `generate_post` | Main: URL → Post | `src/agents/generate-post/` |
| `verify_links` | Content extraction router | `src/agents/verify-links/` |
| `ingest_data` | Slack → generate_post | `src/agents/ingest-data/` |
| `curate_data` | Multi-source aggregation | `src/agents/curate-data/` |
| `generate_thread` | Twitter thread creation | `src/agents/generate-thread/` |
| `supervisor` | Multi-agent orchestrator | `src/agents/supervisor/` |
| `repurposer` | 1 content → N posts | `src/agents/repurposer/` |
| Others | Supporting workflows | See `langgraph.json` |

### Main Workflow: `generate_post`

**File:** `src/agents/generate-post/generate-post-graph.ts`

```
INPUT: { links: ["https://example.com/article"] }
  ↓
1. authSocialsPassthrough        # Authenticate with social platforms
  ↓
2. verifyLinksSubGraph           # Extract content (routes to specialized handlers)
  ↓
3. generateReportOrEnd           # Check duplicates, create marketing report
  ↓
4. generatePost                  # LLM generates post (uses prompts + examples)
  ↓
5. [condensePost loop]           # Shorten if > 280 chars (max 3 attempts)
  ↓
6. findImagesSubGraph            # Discover, rank, select best image
  ↓
7. humanNode [INTERRUPT]         # PAUSE for human review in Agent Inbox
  ↓
8. routeResponse                 # Parse user action (schedule/rewrite/reject)
  ↓
9. schedulePost / rewritePost    # Publish or regenerate
  ↓
OUTPUT: Posted to Twitter & LinkedIn
```

**Key Nodes:**
- `verifyLinksSubGraph`: Parallel content extraction (GitHub, YouTube, Twitter, Reddit, general web)
- `generateContentReport`: LLM analyzes content for marketing angles
- `generatePost`: LLM generates post using few-shot examples
- `findImagesSubGraph`: Image discovery, validation, LLM ranking
- `humanNode`: Interrupts execution for approval (view in Agent Inbox)
- `schedulePost`: Publishes to platforms via Arcade or direct APIs

### State Management

Uses **LangGraph Annotation** system:

```typescript
// Example state structure
{
  links: string[]                     // Input URLs
  pageContents: {url, content}[]      // Extracted content
  report: string                      // Marketing analysis
  post: string                        // Generated post text
  image: {imageUrl, mimeType}         // Selected image
  scheduleDate: Date                  // Publishing time
  userResponse: string                // Human feedback
  next: string                        // Routing decision
  configurable: {...}                 // Runtime config overrides
}
```

**Persistent Storage:** LangGraph Store
- Namespace: `/post_urls/used` - Tracks processed URLs to prevent duplicates
- Used in: `generateReportOrEndConditionalEdge`, saved in `schedulePost`

### Content Extraction: `verify_links` Subgraph

**Routing Logic:**

```typescript
// Routes URLs to specialized extractors
"twitter.com/*"       → verifyTweetContent (Twitter API)
"youtube.com/*"       → verifyYouTubeContent (Google Vertex AI transcript)
"github.com/*"        → verifyGitHubContent (Octokit + README)
"reddit.com/*"        → verifyRedditContent (snoowrap)
"lu.ma/*"            → verifyLumaContent (event scraping)
default              → verifyGeneralContent (FireCrawl)
```

All URLs processed **in parallel** using LangGraph's `Send` API for fan-out/fan-in.

## External Integrations

### Social Media Authentication

**Two modes:**

1. **Arcade AI (Recommended)**
   - Single API key for Twitter + LinkedIn
   - No OAuth setup required
   - Set: `USE_ARCADE_AUTH=true`, `ARCADE_API_KEY=xxx`
   - Note: Still need Twitter API for media uploads

2. **Direct APIs**
   - Manual OAuth via `yarn start:auth`
   - Requires developer accounts on each platform
   - More complex but full control

### Key Services

| Service | Purpose | Config |
|---------|---------|--------|
| **Anthropic Claude** | Content generation | `ANTHROPIC_API_KEY` |
| **FireCrawl** | Web scraping | `FIRECRAWL_API_KEY` |
| **Supabase** | Image storage (public URLs) | `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` |
| **Slack** | URL ingestion + notifications | `SLACK_BOT_TOKEN`, `SLACK_CHANNEL_ID` |
| **GitHub** | Repo content extraction | `GITHUB_TOKEN` |
| **Google Vertex AI** | YouTube transcripts | `GOOGLE_VERTEX_AI_WEB_CREDENTIALS` |
| **LangSmith** | Tracing (optional but recommended) | `LANGCHAIN_API_KEY` |

## Customization Guide

### 1. Change Post Style/Voice

**File:** `src/agents/generate-post/prompts/index.ts`

**Key prompts to edit:**

```typescript
// Define your business domain and target audience
export const BUSINESS_CONTEXT = `
  Your company/product description here.
  Target audience, key topics, brand voice.
`;

// Structure guidelines (header, body, CTA)
export const POST_STRUCTURE_INSTRUCTIONS = `
  <section key="1">Hook (5 words)</section>
  <section key="2">Body (2-3 sentences)</section>
  <section key="3">Call to action (3-6 words)</section>
`;

// Writing style rules
export const POST_CONTENT_RULES = `
  - Use active voice
  - No hashtags or emojis
  - Keep it conversational
  - Include URL in final section
`;
```

**File:** `src/agents/generate-post/prompts/examples.ts`

```typescript
// Add 5-10 examples of posts in your desired style
export const TWEET_EXAMPLES = [
  {
    content: "Example post in your brand voice...",
    url: "https://example.com"
  },
  // More examples...
];
```

### 2. Add Support for New URL Type

**Example: Product Hunt**

1. Create verifier: `src/agents/shared/nodes/verify-producthunt.ts`
   ```typescript
   export async function verifyProductHuntContent(state) {
     const { link } = state;
     const product = await fetchProductHuntData(link);
     return {
       pageContents: [{
         url: link,
         content: `${product.name}: ${product.description}`,
         images: [product.thumbnail]
       }],
       relevantLinks: [link]
     };
   }
   ```

2. Update router: `src/agents/verify-links/verify-links-graph.ts`
   ```typescript
   function routeLinkTypes(state) {
     if (state.link.includes("producthunt.com")) {
       return new Send("verifyProductHuntContent", { link: state.link });
     }
     // ... existing routes
   }
   ```

3. Add node to graph:
   ```typescript
   verifyLinksGraph.addNode("verifyProductHuntContent", verifyProductHuntContent);
   ```

### 3. Add Support for New Social Platform

**Example: Bluesky**

1. Create client: `src/clients/bluesky/client.ts`
2. Update `schedulePost` node: `src/agents/shared/nodes/generate-post/schedule-post.ts`
3. Add env vars: `POST_TO_BLUESKY=true`, credentials

## Configuration

### Environment Variables

**Quickstart mode (.env.quickstart.example):**
- Minimal setup: Anthropic, LangSmith, FireCrawl, Arcade
- Skips: GitHub, YouTube, image handling, Slack

**Full mode (.env.full.example):**
- All features enabled
- Requires all service API keys

### Runtime Overrides (Configurable Parameters)

Override env vars per-run:

```typescript
await client.runs.create(threadId, "generate_post", {
  input: { links: ["https://example.com"] },
  config: {
    configurable: {
      textOnlyMode: true,                    // Skip images
      skipUsedUrlsCheck: true,               // Allow duplicates
      skipContentRelevancyCheck: true,       // Bypass validation
      postToLinkedInOrganization: true,      // Post as company
      twitterUserId: "other@example.com",    // Override account
      scheduleDate: "12/25/2024 10:00 AM PST"
    }
  }
});
```

**Constants:** `src/agents/shared/constants.ts`

## Common Workflows

### 1. Generate a Single Post

```bash
# Edit scripts/generate-post.ts to change URL
yarn generate_post

# View in Agent Inbox: https://dev.agentinbox.ai/
# Or LangSmith: https://smith.langchain.com/
```

### 2. Automated Slack Ingestion

```bash
# Setup: Set SLACK_BOT_TOKEN and SLACK_CHANNEL_ID in .env
# Edit scripts/crons/create-cron.ts with your channel ID
yarn cron:create

# Agent fetches URLs from Slack daily
# View interrupted runs in Agent Inbox
```

### 3. Batch Process Multiple URLs

```bash
# Edit scripts/backfill.ts with URLs list
yarn graph:backfill

# Processes all URLs, interrupts for each
```

### 4. Manual OAuth Setup (if not using Arcade)

```bash
# Start auth server
yarn start:auth

# Visit http://localhost:3000
# Click "Login with Twitter" or "Login with LinkedIn"
# Copy tokens from terminal to .env
```

## Development Patterns

### Parallel Processing

```typescript
// Fan-out: Process multiple items concurrently
const sends = urls.map(url => new Send("processNode", { url }));
return sends;

// Results automatically aggregated (fan-in)
```

### Conditional Routing

```typescript
function conditionalEdge(state: State): string {
  if (state.post.length > 280) return "condensePost";
  if (state.configurable?.textOnlyMode) return "humanNode";
  return "findImagesSubGraph";
}
```

### Human Interrupts

```typescript
// In graph definition
.addNode("humanNode", humanNode)
.addEdge("previousNode", "humanNode")
.addConditionalEdges("humanNode", routeResponse, {
  "schedule": "schedulePost",
  "rewrite": "rewritePost",
  "reject": END
});

// humanNode implementation
export function humanNode(state: State) {
  // Execution PAUSES here
  // Resume via Agent Inbox or SDK
  return state;
}
```

### LangGraph Store Usage

```typescript
import { Store } from "@langchain/langgraph";

const store = new Store();

// Save
await store.put(["namespace", "key"], value, { metadata });

// Retrieve
const item = await store.get(["namespace", "key"], value);
if (item?.value) {
  // Value exists
}

// List
const items = await store.search(["namespace", "key"]);
```

## Testing

### Test Organization

- `*.test.ts` - Unit tests (fast, no external APIs)
- `*.int.test.ts` - Integration tests (require API keys)
- `src/evals/` - LLM output evaluation (content quality, etc.)

### Writing Tests

```typescript
import { describe, it, expect } from "@jest/globals";
import { generatePostGraph } from "../agents/generate-post/generate-post-graph";

describe("Generate Post", () => {
  it("generates post from URL", async () => {
    const result = await generatePostGraph.invoke({
      links: ["https://example.com"],
      configurable: {
        textOnlyMode: true,  // Skip slow image operations
        skipUsedUrlsCheck: true
      }
    });

    expect(result.post).toBeDefined();
    expect(result.post.length).toBeLessThanOrEqual(280);
  });
});
```

### Run Tests

```bash
yarn test           # Unit tests (fast)
yarn test:int       # Integration tests (requires API keys)
yarn test:single path/to/test.ts
```

## Debugging

### LangSmith Tracing

Every run is automatically traced if `LANGCHAIN_TRACING_V2=true`:
- View at https://smith.langchain.com/
- See: Node execution times, LLM prompts/responses, state at each step, errors

### Agent Inbox (HITL UI)

- Deployed: https://dev.agentinbox.ai/
- Add graph: URL `http://localhost:54367`, ID `generate_post`
- View interrupted runs, approve/edit/reject posts

### Common Issues

**"Post too long":**
- `condensePost` node loops max 3 times
- If still too long, proceeds anyway
- Check: `POST_CONTENT_RULES` prompt, character counting logic

**"Image MIME type error":**
- Some images fail Twitter/LinkedIn upload
- TODO: Re-route to humanNode with error
- Workaround: Set `textOnlyMode: true`

**"URL already used":**
- Stored in LangGraph Store (`/post_urls/used`)
- Override: `skipUsedUrlsCheck: true`
- Clear: Delete from store (no UI yet)

**"Content not relevant":**
- `generateReportOrEndConditionalEdge` checks against `BUSINESS_CONTEXT`
- Override: `skipContentRelevancyCheck: true`
- Fix: Update `BUSINESS_CONTEXT` in prompts

## Known TODOs & Limitations

Based on codebase comments:

1. **Schedule dates for repurposed posts** - R1/R2/R3 priority undefined
   - Location: `src/agents/repurposer-post-interrupt/nodes/human-node/utils.ts:143-145`

2. **Incomplete eval logic** - Placeholder implementations
   - Files: `src/evals/youtube/`, `src/evals/general/`, `src/evals/twitter/`

3. **YouTube error handling** - Needs improvement
   - Location: `src/agents/shared/nodes/youtube.utils.ts:87`

4. **Deprecated reflection utils** - Should migrate to memory graph
   - Location: `src/utils/reflections.ts:50,69`

5. **Image MIME errors** - Should re-route to humanNode
   - Multiple files, search for "TODO" in find-images/

6. **No input sanitization** - XSS risk on user-provided URLs
   - Security recommendation

## Extension Points

### Adding a New LLM Call

```typescript
import { ChatAnthropic } from "@langchain/anthropic";

const model = new ChatAnthropic({
  modelName: "claude-3-5-sonnet-20241022",
  temperature: 0.7,
  maxTokens: 4096
});

const response = await model.invoke([
  { role: "system", content: "System prompt..." },
  { role: "user", content: userInput }
]);
```

### Creating a New Graph

1. Define state: `src/agents/my-graph/state.ts`
2. Create nodes: `src/agents/my-graph/nodes/*.ts`
3. Build graph: `src/agents/my-graph/index.ts`
4. Register in `langgraph.json`:
   ```json
   {
     "graphs": {
       "my_graph": "./src/agents/my-graph/index.ts:myGraph"
     }
   }
   ```

### Adding a Cron Job

```typescript
// scripts/crons/create-cron.ts
import { Client } from "@langchain/langgraph-sdk";

const client = new Client({ apiUrl: process.env.LANGGRAPH_API_URL });

await client.crons.create({
  assistantId: "generate_post",
  schedule: "0 9 * * *",  // Daily at 9 AM
  payload: {
    links: [],
    configurable: { ... }
  }
});
```

## Project-Specific Context

### Target Use Case
- **Built for:** LangChain team to automate social media
- **Content focus:** AI/ML, developer tools, LangChain products
- **Prompts reflect:** LangChain brand voice and business context

### To Adapt for Your Use Case
1. Update `BUSINESS_CONTEXT` in `src/agents/generate-post/prompts/index.ts`
2. Replace `TWEET_EXAMPLES` with your brand's post examples
3. Adjust `POST_STRUCTURE_INSTRUCTIONS` and `POST_CONTENT_RULES`
4. Test with your URLs and iterate on prompts

### Timezone Handling
- Default: America/Los_Angeles (PST/PDT)
- Set in: `jest.config.js` (tests), `src/utils/date.ts` (runtime)
- Schedule parsing: `date-fns-tz` library

### Character Limits
- Twitter: 280 characters (excluding URLs)
- LinkedIn: No strict limit, but concise is better
- Logic: `src/agents/shared/nodes/generate-post/condense-post.ts`

## Quick Troubleshooting

| Issue | Solution |
|-------|----------|
| Server won't start | Check `LANGSMITH_API_KEY` is set (required even if tracing disabled) |
| OAuth fails | Redirect URLs must be `http://localhost:3000/auth/{platform}/callback` |
| Image upload fails | Check Supabase bucket is public, max size ≥5MB |
| Post generation hangs | Check LangSmith for errors, verify API keys |
| "No thread found" | Thread IDs are ephemeral in in-memory mode, use deployed graph for persistence |

## Resources

- **LangGraph Docs:** https://langchain-ai.github.io/langgraph/
- **LangSmith:** https://smith.langchain.com/
- **Agent Inbox:** https://github.com/langchain-ai/agent-inbox
- **Arcade AI Docs:** https://docs.arcade.dev/
- **Project Docs:** `START_HERE.md`, `SYSTEM_OVERVIEW.md`, `CONTRIBUTING.md`

---

**Last Updated:** 2025-11-18
**LangGraph Version:** 1.0.2
**Node Version:** 20+
