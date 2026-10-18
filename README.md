# PrepWise — AI Mock Interviews

A voice-based mock interview platform. Users sign up, generate a tailored interview by talking to a voice agent, take the interview with an AI interviewer, and get structured feedback scored by Gemini.

> Built following the [JS Mastery tutorial](https://github.com/adrianhajdin/ai_mock_interviews), extended with setup documentation: a `.env.example` covering every required variable, and a guide to the Firebase, Firestore index and Vapi workflow configuration the app depends on.

## Tech stack

- **Next.js 15** (App Router, Server Actions) + **React 19**, TypeScript
- **Tailwind CSS v4** + shadcn/ui components
- **Firebase** — Authentication (email/password) and Firestore; Admin SDK on the server
- **Vapi** — real-time voice agent (workflow for generating interviews, assistant for conducting them)
- **Google Gemini** via the Vercel **AI SDK** — question generation and feedback scoring

## How it works

1. **Sign up / sign in** — Firebase Auth on the client; the server exchanges the ID token for a session cookie ([lib/actions/auth.action.ts](lib/actions/auth.action.ts)).
2. **Generate an interview** — on `/interview`, the user talks to a Vapi *workflow* that collects role, level, tech stack, type and number of questions, then calls `POST /api/vapi/generate` ([app/api/vapi/generate/route.ts](app/api/vapi/generate/route.ts)). That route asks Gemini for questions and stores the interview in Firestore.
3. **Take the interview** — on `/interview/[id]`, a Vapi *assistant* ([constants/index.ts](constants/index.ts)) asks the stored questions by voice.
4. **Feedback** — when the call ends, the transcript is sent to Gemini, which returns structured scores and strengths/improvements ([lib/actions/general.action.ts](lib/actions/general.action.ts)). Results are shown on `/interview/[id]/feedback`.

## Project structure

```
app/
  (auth)/sign-in, sign-up     auth pages
  (root)/                     dashboard, interview, feedback pages
  api/vapi/generate/route.ts  endpoint called by the Vapi workflow
components/                   Agent (voice call UI), AuthForm, InterviewCard, ui/
constants/                    Vapi interviewer config, feedback schema, tech mappings
firebase/                     client.ts (browser SDK), admin.ts (Admin SDK)
lib/actions/                  server actions for auth and interviews/feedback
types/                        shared TypeScript types
```

## Getting started

### Prerequisites

- Node.js 18.18+ (20 LTS recommended)
- A Firebase project, a Vapi account and a Google AI Studio API key

### 1. Install

```bash
npm install
```

### 2. Configure environment

```bash
cp .env.example .env.local
```

Fill in `.env.local`:

| Variable | Where to get it |
|---|---|
| `GOOGLE_GENERATIVE_AI_API_KEY` | [Google AI Studio](https://aistudio.google.com/app/apikey) |
| `NEXT_PUBLIC_VAPI_WEB_TOKEN` | Vapi dashboard → API Keys → **public** key |
| `NEXT_PUBLIC_VAPI_WORKFLOW_ID` | ID of your interview-generation workflow in Vapi |
| `NEXT_PUBLIC_FIREBASE_*` (7 vars) | Firebase console → Project settings → Your apps → Web app config |
| `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` | Firebase console → Project settings → Service accounts → Generate new private key |

`FIREBASE_PRIVATE_KEY` must stay on one line, wrapped in double quotes, with literal `\n` sequences — [firebase/admin.ts](firebase/admin.ts) converts them back to newlines.

### 3. Set up Firebase

- **Authentication** → enable the **Email/Password** provider.
- **Firestore** → create a database. The app uses the collections `users`, `interviews` and `feedback`.
- **Indexes** — the interview list queries combine `where` and `orderBy`, so Firestore needs composite indexes on `interviews`. The first time a query runs, the server log prints a link that creates the required index; open it and wait for the index to build.

### 4. Set up the Vapi workflow

Create a workflow in the Vapi dashboard that:

- receives the variables `{{username}}` and `{{userid}}` (passed by [components/Agent.tsx](components/Agent.tsx)),
- collects `role`, `level`, `techstack` (comma-separated), `type` (e.g. technical / behavioural / mixed) and `amount`,
- sends them, plus `userid`, as a JSON `POST` to `https://<your-public-url>/api/vapi/generate`.

Vapi calls this endpoint from its servers, so it must be publicly reachable. For local development, expose port 3000 with a tunnel such as `ngrok http 3000` and use that URL in the workflow.

### 5. Run

```bash
npm run dev      # http://localhost:3000
```

## Scripts

| Command | Description |
|---|---|
| `npm run dev` | Start the dev server (Turbopack) |
| `npm run build` | Production build |
| `npm start` | Serve the production build |
| `npm run lint` | Run ESLint |

## Deployment

Deploy to any Next.js host (e.g. Vercel). Add every variable from `.env.example` to the host's environment settings, then point the Vapi workflow's API request at the deployed `/api/vapi/generate` URL.
