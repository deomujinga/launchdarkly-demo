# LaunchDarkly SE Technical Exercise — Cursor Build Prompt

## 🎯 What You're Building

A polished **Next.js 14 (App Router)** web app for "ABC Company" — a fictional SaaS landing page — that demonstrates LaunchDarkly feature flags across **three scenarios**:

| Scenario | What It Proves |
|---|---|
| **Part 1: Release & Remediate** | Feature flag around an AI chatbot widget, real-time listener (no page reload), and a trigger URL to kill the feature via curl |
| **Extra Credit: Experimentation** | Custom metric tracking chatbot interactions, wired to a LaunchDarkly experiment |
| **Extra Credit: AI Configs** | Server-side Node.js AI SDK managing chatbot prompts/models from the LaunchDarkly dashboard — swap models/prompts without redeploying |

> **NOT doing Part 2 (Target).** Skip individual/rule-based targeting entirely.

---

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────────┐
│                   BROWSER                         │
│                                                   │
│  Next.js App (React)                              │
│  ┌─────────────────────────────────────────────┐  │
│  │  LaunchDarkly React Web SDK                 │  │
│  │  (launchdarkly-react-client-sdk v3.x)       │  │
│  │                                             │  │
│  │  • Uses Client-side ID (not SDK key)        │  │
│  │  • Streaming enabled → instant updates      │  │
│  │  • useFlags() hook for flag evaluation      │  │
│  │  • useLDClient().track() for experiments    │  │
│  │                                             │  │
│  │  Components:                                │  │
│  │    LandingPage → always visible             │  │
│  │    AIChatbotWidget → gated by flag          │  │
│  │      "ai-chatbot" (boolean)                 │  │
│  └──────────────┬──────────────────────────────┘  │
│                 │ POST /api/chat                   │
└─────────────────┼─────────────────────────────────┘
                  │
┌─────────────────▼─────────────────────────────────┐
│              NEXT.JS SERVER (API Route)            │
│                                                    │
│  /api/chat/route.ts                                │
│  ┌──────────────────────────────────────────────┐  │
│  │  LaunchDarkly Node.js Server SDK             │  │
│  │  (@launchdarkly/node-server-sdk v9.x)        │  │
│  │                                              │  │
│  │  LaunchDarkly AI SDK                         │  │
│  │  (@launchdarkly/server-sdk-ai v0.5.x)        │  │
│  │                                              │  │
│  │  • Uses SDK Key (server-side, secret)        │  │
│  │  • initAi(ldClient) → LDAIClient            │  │
│  │  • aiClient.completionConfig() evaluates     │  │
│  │    AI Config → returns messages + model      │  │
│  │  • tracker.trackOpenAIMetrics() records      │  │
│  │    token usage, latency, generation count    │  │
│  │                                              │  │
│  │  OpenAI SDK (openai v4.x)                    │  │
│  │  • Receives messages/model from AI Config    │  │
│  │  • Returns completion to client              │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────────────────────┐
│            LAUNCHDARKLY DASHBOARD                   │
│                                                     │
│  Feature Flag: "ai-chatbot" (boolean)               │
│  • Toggle ON = show chatbot widget                  │
│  • Toggle OFF = hide chatbot widget                 │
│  • Streaming delivers instant updates to React SDK  │
│  • Generic Trigger attached → curl URL kills flag   │
│                                                     │
│  AI Config: "chatbot-config" (completion mode)      │
│  • Variation 1: gpt-4o-mini + helpful prompt        │
│  • Variation 2: gpt-4o + detailed prompt            │
│  • Swap model/prompt from dashboard, no redeploy    │
│                                                     │
│  Metric: "chatbot-message-sent" (custom conversion) │
│  • Fires when user sends a message in chatbot       │
│  • Event key: "chatbot-message-sent"                │
│                                                     │
│  Experiment: uses "ai-chatbot" flag + metric        │
│  • Measures chatbot engagement per variation         │
└─────────────────────────────────────────────────────┘
```

---

## 📦 Tech Stack & Dependencies

```json
{
  "dependencies": {
    "next": "^14.2.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "launchdarkly-react-client-sdk": "^3.9.0",
    "@launchdarkly/node-server-sdk": "^9.7.0",
    "@launchdarkly/server-sdk-ai": "^0.5.0",
    "openai": "^4.70.0"
  }
}
```

---

## 📁 File Structure

```
abc-company-launchdarkly-demo/
├── .env.example              # Template with all required env vars
├── .env.local                # Actual keys (gitignored)
├── .gitignore
├── package.json
├── next.config.js
├── tsconfig.json
├── README.md                 # Detailed setup + run instructions (CRITICAL for submission)
│
├── src/
│   ├── app/
│   │   ├── layout.tsx        # Root layout, loads fonts, wraps with LD provider
│   │   ├── page.tsx          # Landing page with hero, features, chatbot widget
│   │   ├── globals.css       # Global styles
│   │   │
│   │   └── api/
│   │       └── chat/
│   │           └── route.ts  # POST handler: AI Config eval → OpenAI call → response
│   │
│   ├── components/
│   │   ├── LandingPage.tsx       # The full ABC Company landing page
│   │   ├── AIChatbotWidget.tsx   # The chatbot component (gated by feature flag)
│   │   ├── ChatMessage.tsx       # Individual chat message bubble
│   │   └── FlagStatusBanner.tsx  # Dev-facing banner showing flag state (optional, nice for demo)
│   │
│   └── lib/
│       ├── launchdarkly-provider.tsx  # Client-side: withLDProvider wrapper
│       └── launchdarkly-server.ts     # Server-side: singleton LDClient + AI client init
│
└── docs/
    └── LAUNCHDARKLY_SETUP.md  # Step-by-step LD dashboard configuration guide
```

---

## 🔑 Environment Variables

```bash
# .env.local

# Client-side ID → LaunchDarkly > Project Settings > Environments > Client-side ID
# Safe to expose in browser (it's NOT a secret)
NEXT_PUBLIC_LD_CLIENT_SIDE_ID=your-client-side-id-here

# SDK Key → LaunchDarkly > Project Settings > Environments > SDK Key  
# ⚠️ Server-side only, NEVER expose in client code
LD_SDK_KEY=your-sdk-key-here

# OpenAI API Key → https://platform.openai.com/api-keys
OPENAI_API_KEY=sk-your-openai-key-here
```

---

## 🔨 Step-by-Step Build Instructions for Cursor

### Step 1: Scaffold the Next.js Project

```
Create a Next.js 14 App Router project with TypeScript.

- Use `src/` directory structure
- Install these exact packages:
  - launchdarkly-react-client-sdk@^3.9.0
  - @launchdarkly/node-server-sdk@^9.7.0
  - @launchdarkly/server-sdk-ai@^0.5.0
  - openai@^4.70.0
- Create .env.example with NEXT_PUBLIC_LD_CLIENT_SIDE_ID, LD_SDK_KEY, OPENAI_API_KEY
- Create .gitignore that excludes .env.local, node_modules, .next
```

### Step 2: Create the LaunchDarkly Client-Side Provider

```
Create src/lib/launchdarkly-provider.tsx

This is a "use client" component that:
1. Imports { withLDProvider } from "launchdarkly-react-client-sdk"
2. Uses process.env.NEXT_PUBLIC_LD_CLIENT_SIDE_ID as the clientSideID
3. Sets a default context:
   {
     kind: "user",
     key: "user-abc-demo",
     name: "Demo User",
     email: "demo@abccompany.com"
   }
4. Sets options: { streaming: true } — this is CRITICAL for Part 1 
   (instant flag updates without page reload)
5. Sets reactOptions: { useCamelCaseFlagKeys: true }
6. Exports a <LaunchDarklyProvider> component that wraps children

IMPORTANT COMMENTS TO INCLUDE:
- Comment explaining Client-side ID vs SDK Key
- Comment explaining streaming: true enables real-time flag listener (Part 1 requirement)
- Comment explaining the context object and what each field means
```

### Step 3: Create the Server-Side LaunchDarkly + AI Client

```
Create src/lib/launchdarkly-server.ts

This is a server-only module that:
1. Imports LaunchDarkly from "@launchdarkly/node-server-sdk"
2. Imports { initAi } from "@launchdarkly/server-sdk-ai"
3. Creates a singleton LDClient using process.env.LD_SDK_KEY
4. Waits for initialization with a 10-second timeout
5. Initializes the AI client: initAi(ldClient)
6. Exports both the ldClient and aiClient

The function signature should be:
  export async function getLDServerClients()

Return type: { ldClient: LaunchDarkly.LDClient, aiClient: LDAIClient }

IMPORTANT COMMENTS TO INCLUDE:
- Comment: "Server-side SDK Key — NEVER expose in client code"
- Comment: "AI SDK is used for AI Config evaluation (Extra Credit)"
- Comment: "Singleton pattern prevents multiple SDK initializations"
```

### Step 4: Build the Landing Page

```
Create src/components/LandingPage.tsx

Build a polished, professional "ABC Company" SaaS landing page. This is being presented 
to LaunchDarkly as a prospective customer demo, so make it look GOOD.

The page should include:
- A hero section with company name, tagline, and CTA
- A features section highlighting ABC Company's SaaS product
- Clean, modern design with good typography
- The page itself is ALWAYS visible (it's not behind a flag)

Style notes:
- Use a distinctive, non-generic font pairing (e.g., Sora + Source Serif, or DM Sans + Playfair)
- Dark or sophisticated color scheme — NOT generic purple gradients
- Professional SaaS aesthetic
- Responsive design
```

### Step 5: Build the AI Chatbot Widget (Feature-Flagged Component)

```
Create src/components/AIChatbotWidget.tsx

This is the CORE component — it's gated behind the "ai-chatbot" feature flag.

"use client" component that:
1. Imports { useFlags, useLDClient } from "launchdarkly-react-client-sdk"
2. Reads the flag: const { aiChatbot } = useFlags()
   - Note: flag key in LD dashboard is "ai-chatbot", but useCamelCaseFlagKeys 
     converts it to "aiChatbot"
3. If aiChatbot is false → render nothing (or a subtle placeholder)
4. If aiChatbot is true → render a floating chat widget (bottom-right corner)

Chat widget functionality:
- Collapsible chat window (click to open/close)
- Message input field
- Display chat messages (user + assistant)
- On message send:
  a. POST to /api/chat with { message: userMessage }
  b. Display streaming/loading state
  c. Show AI response

EXPERIMENTATION (Extra Credit):
- When user sends a message, call: ldClient.track("chatbot-message-sent")
  This fires a custom event that LaunchDarkly uses for the experiment metric.
- Add comment: "// EXPERIMENT: Track chatbot engagement for A/B testing"

PART 1 — INSTANT UPDATES (CRITICAL):
- Because streaming: true is set in the provider, the useFlags() hook 
  AUTOMATICALLY receives real-time updates when the flag is toggled in the 
  LaunchDarkly dashboard.
- The React SDK's withLDProvider handles this via the Context API — when 
  flag values change, all components using useFlags() re-render instantly.
- NO page reload required. This is the "listener" requirement from Part 1.
- Add a comment block explaining this behavior:
  // PART 1 — REAL-TIME FLAG LISTENER
  // The LaunchDarkly React SDK with streaming: true automatically 
  // subscribes to flag change events via SSE (Server-Sent Events).
  // When "ai-chatbot" is toggled ON/OFF in the LaunchDarkly dashboard,
  // this component instantly re-renders — no page reload needed.
  // This satisfies the "listener" and "instant releases/rollbacks" requirement.

IMPORTANT COMMENTS:
- Comment where the flag key "ai-chatbot" needs to be created in LD dashboard
- Comment explaining the track() call for experimentation
- Comment explaining the real-time update mechanism
```

### Step 6: Build the Chat API Route (AI Config Integration)

```
Create src/app/api/chat/route.ts

This is a Next.js API route (POST handler) that:
1. Imports getLDServerClients from the server-side lib
2. Imports OpenAI from "openai"
3. Receives { message } from the request body

Flow:
a. Initialize LD server clients (singleton)
b. Create a context object for the AI Config evaluation:
   { kind: "user", key: "user-abc-demo", name: "Demo User" }
c. Call aiClient.completionConfig() to evaluate the AI Config:
   
   const { messages, model, tracker, enabled } = await aiClient.completionConfig(
     "chatbot-config",    // AI Config key in LaunchDarkly
     context,
     {                     // Fallback config (used if LD is unreachable)
       enabled: true,
       model: { name: "gpt-4o-mini" },
       messages: [{ role: "system", content: "You are a helpful assistant for ABC Company." }]
     }
   );

d. If not enabled, return a fallback response
e. Append the user's message to the messages array from the AI Config
f. Call OpenAI with the messages and model from the AI Config:
   
   const completion = await openai.chat.completions.create({
     model: model?.name || "gpt-4o-mini",
     messages: [...messages, { role: "user", content: message }],
   });

g. Track metrics with the LaunchDarkly AI SDK:
   
   tracker.trackOpenAIMetrics(completion);

h. Return the assistant's response as JSON

IMPORTANT COMMENTS:
- Comment: "AI CONFIG: The model and system prompt come from LaunchDarkly, not from code"
- Comment: "To change the prompt or model, update the AI Config in the LD dashboard — no redeploy needed"
- Comment: "trackOpenAIMetrics automatically records token usage, latency, and generation count"
- Comment: "FALLBACK: If LaunchDarkly is unreachable, this default config is used"
- Comment explaining that "chatbot-config" must be created as an AI Config in the LD dashboard
```

### Step 7: Wire Everything Together in the Root Layout and Page

```
src/app/layout.tsx:
- Import LaunchDarklyProvider from lib
- Wrap {children} in <LaunchDarklyProvider>
- Load Google Fonts in <head>
- Set metadata title/description

src/app/page.tsx:
- Import LandingPage component
- Import AIChatbotWidget component
- Render both:
  <main>
    <LandingPage />
    <AIChatbotWidget />
  </main>

The AIChatbotWidget internally checks the flag — if flag is OFF, it renders nothing.
If flag is ON, it renders the floating chat widget.
```

### Step 8: Create a Flag Status Banner (Optional, Great for Demo)

```
Create src/components/FlagStatusBanner.tsx (optional but impressive for demo)

A small dev-tools-like banner at the top of the page that shows:
- Current flag state: "ai-chatbot: ON ✅" or "ai-chatbot: OFF ❌"
- Current user context key
- A note: "Toggle this flag in LaunchDarkly to see instant updates"

This makes the demo very visual — the reviewer can see the flag state 
change in real-time as they toggle it in the dashboard.
```

### Step 9: Write the README.md (CRITICAL for Submission)

```
The README is EXPLICITLY required by the exercise. It must explain IN DETAIL 
how to set up and run the project. Write it as if delivering to a prospective customer.

Structure:

# ABC Company — LaunchDarkly Feature Flag Demo

## Overview
Brief description of what this demo shows (Part 1 + Extra Credits).

## Architecture
Brief description of the tech stack and how the pieces connect.

## Prerequisites
- Node.js 18+
- A LaunchDarkly account (free trial: https://launchdarkly.com/start-trial/)
- An OpenAI API key (for AI Config extra credit)

## LaunchDarkly Dashboard Setup

### 1. Create a Feature Flag
1. Go to Feature Flags > Create Flag
2. Name: "ai-chatbot"
3. Key: "ai-chatbot"  
4. Variations: Boolean (true/false)
5. ✅ Check "SDKs using Client-side ID" (IMPORTANT — React SDK requires this)
6. Default variation (when targeting is OFF): false
7. Save

### 2. Create a Flag Trigger (Part 1: Remediate)
1. Go to the "ai-chatbot" flag > Settings tab
2. Under "Triggers", click "Add trigger"
3. Type: Generic trigger
4. Action: "Turn flag targeting OFF" 
5. Save and COPY the trigger URL
6. Test it: curl -X POST "YOUR_TRIGGER_URL"
   This will turn off the flag, instantly hiding the chatbot from all users.

### 3. Create an AI Config (Extra Credit: AI Configs)
⚠️ AI Configs may require an add-on on your LaunchDarkly plan. If you don't see
the option, contact LaunchDarkly or use the trial account.

1. Click Create > AI Config
2. Name: "chatbot-config"
3. Key: "chatbot-config"
4. Mode: Completion
5. Create a variation:
   - Name: "helpful-mini"
   - Model: Select OpenAI > gpt-4o-mini
   - Add a System message:
     "You are a helpful AI assistant for ABC Company, a leading SaaS platform. 
      Be concise, friendly, and professional. Help users understand our product features."
6. (Optional) Create a second variation:
   - Name: "detailed-4o"  
   - Model: Select OpenAI > gpt-4o
   - Add a System message:
     "You are an expert AI assistant for ABC Company. Provide detailed, thorough 
      responses about our SaaS platform. Include specific examples and use cases."
7. Go to the Targeting tab:
   - Enable the AI Config
   - Set default rule to serve "helpful-mini"
   - Save

### 4. Create a Metric (Extra Credit: Experimentation)
1. Go to Experiments > Metrics tab > Create metric
2. Event key: "chatbot-message-sent"
3. Type: Custom conversion (occurrence)
4. Name: "Chatbot Message Sent"
5. Save

### 5. Create an Experiment (Extra Credit: Experimentation)
1. Go to Experiments > Create experiment
2. Name: "Chatbot Engagement Test"
3. Flag: "ai-chatbot"
4. Metric: "Chatbot Message Sent"
5. Randomization unit: user
6. Audience: 100% of users
7. Start the experiment
8. Use the app to send chatbot messages — events will appear in the experiment

## Environment Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/YOUR_USERNAME/abc-company-launchdarkly-demo.git
   cd abc-company-launchdarkly-demo
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Copy environment variables:
   ```bash
   cp .env.example .env.local
   ```

4. Fill in your keys in .env.local:
   - NEXT_PUBLIC_LD_CLIENT_SIDE_ID: Found in LD > Project Settings > Environments > Client-side ID
   - LD_SDK_KEY: Found in LD > Project Settings > Environments > SDK Key
   - OPENAI_API_KEY: From https://platform.openai.com/api-keys

5. Run the development server:
   ```bash
   npm run dev
   ```

6. Open http://localhost:3000

## Demo Walkthrough

### Part 1: Release and Remediate

1. **Release**: Go to LaunchDarkly > Feature Flags > "ai-chatbot" > toggle ON
   → The chatbot widget instantly appears on the landing page (no page reload!)

2. **Rollback**: Toggle the flag OFF
   → The chatbot instantly disappears

3. **Remediate with Trigger**: Run this curl command:
   ```bash
   curl -X POST "YOUR_TRIGGER_URL_HERE"
   ```
   → The flag turns OFF automatically, removing the chatbot

### Extra Credit: AI Configs

1. With the chatbot visible, send a message
2. The response comes from the model/prompt configured in the "chatbot-config" AI Config
3. Go to LaunchDarkly > AI Configs > "chatbot-config"
4. Change the default rule to serve "detailed-4o" instead of "helpful-mini"
5. Send another message — the response now comes from gpt-4o with the detailed prompt
6. **No code changes. No redeploy.** The model and prompt changed at runtime.

### Extra Credit: Experimentation

1. With the experiment running, send several chatbot messages
2. Go to LaunchDarkly > Experiments > "Chatbot Engagement Test"
3. View the results — you'll see conversion data for the chatbot-message-sent metric
4. This data helps ABC Company decide whether to ship the chatbot to all users

## Assumptions
- Node.js 18+ is installed
- You have a modern browser (Chrome, Firefox, Safari, Edge)
- LaunchDarkly trial account is active
- OpenAI API key has available credits
```

### Step 10: Create the LaunchDarkly Setup Guide

```
Create docs/LAUNCHDARKLY_SETUP.md

A more detailed guide with screenshots descriptions for setting up each 
LaunchDarkly resource. This supplements the README with extra detail.

Include:
- Where to find Client-side ID vs SDK Key (Project Settings > Environments)
- How to check "SDKs using Client-side ID" on the flag (Settings tab > Client-side SDK availability)
- How to create a generic trigger and copy the URL
- How to create an AI Config with completion mode
- How to create a custom conversion metric
- How to create and start an experiment
```

---

## 🚨 Critical Implementation Notes

### Flag Key Convention
- Dashboard flag key: `ai-chatbot` (kebab-case)
- In React code with `useCamelCaseFlagKeys: true`: `aiChatbot` (camelCase)
- The SDK auto-converts. Don't create flags with camelCase keys in the dashboard.

### Client-side ID vs SDK Key
- **Client-side ID**: Used by the React Web SDK in the browser. Safe to expose. Starts with a hex string.
- **SDK Key**: Used by the Node.js Server SDK on the server. **SECRET.** Starts with `sdk-`.
- **Never** put the SDK Key in `NEXT_PUBLIC_` variables or client-side code.

### "SDKs using Client-side ID" Checkbox
- When creating the `ai-chatbot` flag, you MUST check "SDKs using Client-side ID" on the flag's Settings page.
- Without this, the React SDK will receive the fallback value and the flag won't work.

### Streaming = Real-Time Listener (Part 1)
- Setting `streaming: true` in the React SDK options enables SSE-based flag updates.
- The `useFlags()` hook automatically re-renders when flag values change.
- This is what makes "instant releases/rollbacks" work without page reload.
- The React SDK handles this internally — you don't need to write any custom listener code.

### AI Config vs Feature Flag
- Feature flags: Boolean on/off, evaluated CLIENT-SIDE by the React SDK
- AI Configs: Evaluated SERVER-SIDE by the Node.js AI SDK, returns messages + model config
- They are DIFFERENT resources in LaunchDarkly, created from different menus

### Trigger for Remediation (Part 1)
- A trigger generates a unique webhook URL
- Sending a POST request to that URL toggles the flag
- For the exercise: create a "Generic" trigger with action "Turn flag targeting OFF"
- Test with: `curl -X POST "YOUR_TRIGGER_URL"`
- The trigger URL only displays ONCE when created — copy it immediately

### Experimentation Events
- `ldClient.track("chatbot-message-sent")` fires a custom event
- The metric in LaunchDarkly must have the SAME event key: "chatbot-message-sent"
- Events are batched and sent periodically; they may take a minute to appear in the LD dashboard
- The experiment must be STARTED (not just created) to collect data

### AI SDK Initialization Pattern
```typescript
import LaunchDarkly from "@launchdarkly/node-server-sdk";
import { initAi } from "@launchdarkly/server-sdk-ai";

const ldClient = LaunchDarkly.init(process.env.LD_SDK_KEY);
await ldClient.waitForInitialization({ timeout: 10 });
const aiClient = initAi(ldClient);
```

### AI Config completionConfig() Pattern
```typescript
const { messages, model, tracker, enabled } = await aiClient.completionConfig(
  "chatbot-config",      // AI Config key
  context,               // LaunchDarkly context
  {                      // Fallback value
    enabled: true,
    model: { name: "gpt-4o-mini" },
    messages: [{ role: "system", content: "You are helpful." }]
  }
);

// After OpenAI call:
tracker.trackOpenAIMetrics(completion);
```

---

## 🎨 Design Direction

This is being presented to LaunchDarkly's SE team as a "sample app you are delivering to a prospective customer." Make it look professional and polished.

- **Aesthetic**: Clean, modern SaaS — think Linear, Vercel, Stripe
- **Color scheme**: Dark background with accent colors (teal/cyan or warm amber)
- **Fonts**: Use Google Fonts — something distinctive like Sora, DM Sans, or Outfit for headings; Source Serif or IBM Plex for body
- **Chatbot widget**: Floating bottom-right, glass-morphism or sleek card style
- **Animations**: Smooth fade-in/out when flag toggles the chatbot (CSS transitions)
- **No AI slop**: Avoid purple gradients, generic Inter/Roboto, cookie-cutter layouts

---

## ✅ Submission Checklist

Before pushing to GitHub, verify:

- [ ] `README.md` has complete setup and run instructions
- [ ] `.env.example` has all required variables with descriptive comments
- [ ] Code has relevant comments (where SDK key goes, where flags are created, etc.)
- [ ] `ai-chatbot` flag works — toggling ON shows widget, OFF hides it
- [ ] Streaming works — flag changes reflect instantly without page reload
- [ ] Trigger URL can turn off the flag via curl
- [ ] AI Config integration works — chatbot uses prompt/model from LD dashboard
- [ ] Changing AI Config variation in LD changes chatbot behavior without redeploy
- [ ] `track("chatbot-message-sent")` fires when user sends a message
- [ ] Metric + experiment are documented in setup guide
- [ ] No secrets committed (SDK key not in client code, .env.local gitignored)

---

## 📚 Key Documentation References

- React Web SDK: https://launchdarkly.com/docs/sdk/client-side/react/react-web
- Node.js Server SDK: https://launchdarkly.com/docs/sdk/server-side/node-js
- Node.js AI SDK: https://launchdarkly.com/docs/sdk/ai/node-js
- AI Configs Quickstart: https://launchdarkly.com/docs/home/ai-configs/quickstart
- AI Config Guide (Node.js): https://launchdarkly.com/docs/guides/ai-configs/config-outside-code-nodejs
- Flag Triggers: https://launchdarkly.com/docs/home/releases/triggers
- Sending Custom Events (track): https://docs.launchdarkly.com/sdk/features/events
- Experimentation Quickstart: https://launchdarkly.com/docs/home/experimentation/quickstart
- Example Repo (AI Config + Node.js): https://github.com/launchdarkly-labs/aiconfig-nodejs-docs-example
