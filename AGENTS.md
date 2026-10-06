# AGENT.md — Sentinel AI Engineering & Design Guidelines

This file governs all AI coding agent operations for the Sentinel AI project. All generated code, database models, mathematical functions, and component styles MUST strictly align with the constraints outlined in this document.

---

## 1. System Architecture & Operating Principles

1. **Zero Downtime Interception:** The proxy layer must never crash client applications. If an upstream AI provider fails or rate-limits, the proxy must catch the error, log the incident, execute failover routing, and relay a clean stream back to the caller.
2. **Asynchronous Non-Blocking Logging:** Database writes for token consumption and metrics must run asynchronously (using `waitUntil` or detached background promises) so they add < 5ms of overhead to the AI response latency.
3. **Exact Mathematical Accounting:** Token counts, costs, and rate-limit forecasts must be calculated using exact, non-rounded internal precision before formatting for display.

---

## 2. Color System & UI Guidelines

The project uses a strict, consistent palette across all pages, charts, components, and status indicators.

### Color Tokens

| Token Name | Hex Code | Role | Usage |
| :--- | :--- | :--- | :--- |
| `brand-dark` | `#243B35` | Primary Dark Accent | Card headers, secondary button fills, hover states |
| `brand-muted` | `#6B8E7B` | Muted Green | Secondary text, inactive tab borders, grid lines |
| `brand-sage` | `#B7C9B1` | Light Sage | Subheadings, status badges, secondary chart series |
| `brand-cream` | `#F1E9D2` | Cream / Accent Highlight | Primary headings, KPI values, active state highlights |
| `brand-bg` | `#121E1B` | Base Background | Page background body |
| `brand-surface` | `#1B2B27` | Card Surface | Dashboard card containers, sidebar panel background |
| `brand-border` | `#2E453F` | UI Borders | Card borders, table dividers, input borders |
| `status-error` | `#E06C75` | Critical Alert | 429 Rate Limit events, quota exhaustion warnings |
| `status-success` | `#6B8E7B` | Normal Status | Healthy quota indicator, proxy online state |

### Tailwind Extension (`tailwind.config.ts`)

```typescript
import type { Config } from "tailwindcss";

const config: Config = {
  theme: {
    extend: {
      colors: {
        brand: {
          dark: "#243B35",
          muted: "#6B8E7B",
          sage: "#B7C9B1",
          cream: "#F1E9D2",
          bg: "#121E1B",
          surface: "#1B2B27",
          border: "#2E453F",
        },
      },
    },
  },
};
export default config;
```


# 2 UI Component Standards

- Backgrounds: Use bg-brand-bg for page bases and bg-brand-surface with border border-brand-border for cards.
- Typography: Primary metric numbers use text-brand-cream with font-mono. Section headings use text-brand-sage. Muted captions use text-brand-muted.
- Buttons: Primary CTA uses bg-brand-dark hover:bg-brand-muted text-brand-cream border border-brand-border.

### Typography Hierarchy

- **Font Families:**
  - `font-sans`: Primary UI typeface (Inter, Geist, or system sans-serif) for titles, body text, labels, and navigation.
  - `font-mono`: Data typeface (Geist Mono, JetBrains Mono, or Fira Code) for all numeric values, token counts, costs, timestamps, model names, and status codes.

| Context | Tailwind Utility Classes | Text Color | Notes |
| :--- | :--- | :--- | :--- |
| **Hero / Page Title** | `text-2xl font-bold tracking-tight` | `text-brand-cream` | Main page headers |
| **KPI Metric Value** | `text-3xl font-mono font-extrabold` | `text-brand-cream` | Token counts, $ amounts, hours left |
| **Section / Card Header**| `text-base font-semibold` | `text-brand-sage` | Card titles, drawer headers |
| **Body / Table Text** | `text-sm font-normal` | `text-brand-cream/90` | Standard table cell data |
| **Secondary / Meta Text**| `text-sm font-normal` | `text-brand-muted` | Subtext, descriptions |
| **Badges & Micro Labels**| `text-xs font-mono font-medium uppercase tracking-wider` | `text-brand-muted` | Status pills, provider tags, headers |

---

### Spacing & Grid System

All layout layouts follow a strict 4px/8px rhythm to maintain visual alignment across all screens.

- **Card Containers:**
  - Outer Padding: `p-5` (1.25rem / 20px) or `p-6` (1.5rem / 24px).
  - Border Radius: `rounded-xl` (`12px`).
  - Container Border: `border border-brand-border` (`#2E453F`).
  - Gap Between Cards: `gap-4` (16px) on mobile, `gap-6` (24px) on desktop.

- **Tables & Logs Stream:**
  - Header Row Height / Padding: `px-4 py-3 text-xs uppercase tracking-wider bg-brand-dark/40`.
  - Data Row Padding: `px-4 py-3.5 border-b border-brand-border/50 text-sm`.
  - Row Hover State: `hover:bg-brand-dark/30 transition-colors`.

- **Buttons & Interactive Inputs:**
  - Primary Height / Padding: `px-4 py-2 text-sm font-medium`.
  - Corner Radius: `rounded-lg` (`8px`).
  - Focus Ring: `focus:outline-none focus:ring-2 focus:ring-brand-sage/50`.

- **Dashboard Layout Offsets:**
  - Main Content Container: `max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8`.
  - Sidebar Width: `w-64 flex-shrink-0 bg-brand-surface border-r border-brand-border`.


# 3 Mathematical Formulas & Precision Rules

### Token Cost Calculation Formula

Cost is calculated dynamically per request using the prompt (input) and completion (output) rates per 1,000,000 tokens for the specific model:

$$\text{Total Cost (USD)} = \left( \frac{T_{\text{prompt}}}{1,000,000} \times R_{\text{input}} \right) + \left( \frac{T_{\text{completion}}}{1,000,000} \times R_{\text{output}} \right)$$

Where:

- $T_{\text{prompt}}$ = Number of prompt tokens parsed from request header/body.
- $T_{\text{completion}}$ = Number of completion tokens parsed from stream.
- $R_{\text{input}}$ = Price per 1,000,000 input tokens for model $M$.
- $R_{\text{output}}$ = Price per 1,000,000 output tokens for model $M$.

### Rolling Burn Rate Formula

$$\text{Hourly Token Burn Rate } (B_h) = \frac{\sum_{t=0}^{H} \text{Total Tokens}_t}{H}$$$$\text{Estimated Hours to Quota Exhaustion } (E_h) = \frac{Q_{\text{remaining}}}{B_h}$$

Rules:

- $H$ defaults to a rolling 24-hour window ($H = 24$).
- If $B_h = 0$, display $E_h = \infty$ (formatted in UI as Quota Stable).
- If $E_h \le 2.0$, switch card border to status-error (#E06C75) and display a visual alert banner.


# . Proxy Interception & Auto-Failover Protocol

[ Client Request ]
       │
       ▼
┌─────────────────────────┐
│ Next.js Proxy Gateway   │ ──(Reads auth headers & target model)
└─────────────────────────┘
       │
       ├──────> Attempt 1: Send to Primary Model (e.g., gpt-4o / claude-3-5-sonnet)
       │           │
       │           ├── [200 OK] ──> Log metrics async -> Return SSE stream
       │           │
       │           └── [429 / 503 Error Captured]
       │                   │
       │                   ▼
       ├──────> Attempt 2: Rewrite payload model to Fallback (e.g., gemini-1.5-flash)
       │           │
       │           ├── Inject SSE header note: "⚠️ Primary limit reached. Retried with Fallback."
       │           └── Log incident in database -> Return SSE stream

# Rate-Limit Header Mapping

Always parse provider-specific headers into standardized metrics:

| Provider | TPM Header | RPM Header | Reset Time Header |
|----------|------------|------------|-------------------|
| OpenAI / Grok | x-ratelimit-remaining-tokens | x-ratelimit-remaining-requests | x-ratelimit-reset-tokens |
| Anthropic | anthropic-ratelimit-input-tokens-remaining | anthropic-ratelimit-requests-remaining | anthropic-ratelimit-input-tokens-reset |
| Gemini | Quota payload / HTTP 429 status | Quota payload / HTTP 429 status | Retry-After header |


# 5. Directory Structure & Conventions
├── app/
│   ├── (dashboard)/
│   │   ├── layout.tsx         # Dashboard layout shell with branding colors
│   │   ├── page.tsx           # Main analytics summary dashboard
│   │   ├── logs/page.tsx      # Filterable request log stream
│   │   └── settings/page.tsx  # Fallback routing rules & API keys
│   └── api/
│       └── v1/
│           └── [...path]/     # Catch-all proxy route handler
│               └── route.ts
├── components/
│   ├── ui/                    # Reusable Tailwind atomic UI components
│   ├── BurnRateChart.tsx      # Recharts burn rate visualization
│   ├── MetricCard.tsx         # Standardized KPI card component
│   └── StatusIndicator.tsx   # Live connection badge
├── lib/
│   ├── db.ts                  # Database client setup
│   ├── encryption.ts          # AES-256-GCM key encryption utilities
│   ├── metrics.ts             # Mathematical formulas for burn rate & costs
│   └── rates.ts               # Dynamic model pricing reference table
└── tailwind.config.ts         # Palette configuration


# 6. Code Quality Rules
- Strict TypeScript: No use of any. Define explicit types or interfaces for all proxy requests, database payloads, and API parameters.

- No Unhandled Promises: Wrap all fetch requests inside the proxy in try/catch blocks.

- No Visual Inconsistencies: Use bg-brand-surface, border-brand-border, text-brand-cream, and text-brand-sage exclusively across all pages.

