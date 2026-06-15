# Developer Personality — Full Implementation Spec

> A personality content website for software engineers. Quizzes drive sharing; content drives SEO.

---

## 1. Core Concept

**Developer Personality** is a content site whose acquisition mechanism happens to be quizzes. Unlike generic MBTI sites, every piece of content is focused on software engineering identity: how developers think, work, collaborate, and build.

- **Niche:** Software engineers (all levels, all stacks)
- **Differentiator:** Custom 12-archetype framework (not MBTI), developer-native language, generated content at scale
- **Business model:** Google AdSense → affiliate partnerships → sponsorships
- **Traffic flywheel:** Quiz → Result → Type page → Comparison → Career page → Blog → Another quiz

---

## 2. Personality System — The 12 Archetypes

### Dimensions (3 axes)

| Axis | Pole A | Pole B |
|------|--------|--------|
| Thinking Style | Abstract | Practical |
| Working Preference | Individual | Collaborative |
| Approach | Exploration | Optimization |

### Archetypes

| # | Archetype | Description | Axis Profile |
|---|-----------|-------------|-------------|
| 1 | **Systems Thinker** | Deep architecture and abstraction | Abstract · Individual · Exploration |
| 2 | **Infrastructure Purist** | Reliability, scale, operational excellence | Practical · Individual · Optimization |
| 3 | **Startup Hacker** | Speed, experimentation, shipping | Practical · Individual · Exploration |
| 4 | **Product Builder** | Customer-focused engineering | Practical · Collaborative · Exploration |
| 5 | **Performance Optimizer** | Efficiency and refinement | Practical · Individual · Optimization |
| 6 | **Research Engineer** | Novel solutions and exploration | Abstract · Individual · Exploration |
| 7 | **AI Amplifier** | Automation and leverage | Practical · Individual · Exploration |
| 8 | **Debugging Detective** | Root-cause analysis specialist | Abstract · Individual · Optimization |
| 9 | **Technical Strategist** | Long-term planning and architecture | Abstract · Collaborative · Optimization |
| 10 | **Team Catalyst** | Collaboration and mentorship | Abstract · Collaborative · Exploration |
| 11 | **Craftsman Engineer** | Code quality and maintainability | Practical · Individual · Optimization |
| 12 | **Platform Architect** | Developer experience and systems design | Abstract · Collaborative · Optimization |

**Differentiation note:** With 3 binary axes, there are 8 unique axis-combination slots (2³). The 12 archetypes exceed this — 4 archetypes share the same axis profile (Infrastructure Purist, Performance Optimizer, and Craftsman Engineer all map to Practical·Individual·Optimization). Differentiation between shared-profile archetypes comes from fine-grained scoring position on each axis (continuous rather than categorical) and distinctive narrative content. The quiz engine emits continuous scores on each axis, not just pole assignments.

### Why 12 archetypes?

12 archetypes create the following SEO content surface immediately:

| Content type | Pages |
|-------------|-------|
| Profile pages | 12 |
| Pairwise comparisons (nC2) | 66 |
| Career pages | 12 |
| Learning-style pages | 12 |
| AI-tool preference pages | 12 |
| Language preference pages | 12 |
| Leadership pages | 12 |
| Team-role pages | 12 |
| **Total discoverable surface** | **150+** |

---

## 3. Site Architecture & Sitemap

### URL Structure

```
/                          ← Landing page (quiz directory + traffic funnel)
/quizzes/                  ← Quiz index

/quizzes/[slug]            ← Quiz page (the interactive quiz)
/quizzes/[slug]/[result]   ← Result page (unique canonical URL)

/types/                    ← Personality library index
/types/systems-thinker     ← Individual type profile (SSG)
/types/infrastructure-purist
/...

/compare/                  ← Comparison index
/compare/systems-thinker-vs-startup-hacker  ← Comparison page (ISR)

/careers/                  ← Career advice index
/careers/systems-thinker   ← Career page per type

/blog/                     ← Blog index
/blog/[slug]               ← Blog article

/learn/                    ← Learning styles index
/learn/systems-thinker

/tools/                    ← Tool & language preferences index
/tools/systems-thinker
```

### Rendering Strategy

| Section | Strategy | Rationale |
|---------|----------|-----------|
| Landing, quiz index | SSG | Stable, infrequent changes |
| Quiz pages | SSG | Content defined in YAML, built at deploy |
| Result pages | SSG | One per quiz-result combo, stable |
| Type profile pages | SSG | Core content, never changes between builds |
| Comparison pages | ISR | Generated from type definitions, cache until next deploy |
| Blog | SSG (launch), ISR (post-launch) | SSG for launch set; ISR with on-demand revalidation for future articles. Full SSR omitted at MVP to reduce operational surface — blog freshness is not critical at 10-20 launch articles. |
| Career/learn/tool pages | SSG | Generated from type data |

---

## 4. Information Architecture

### Navigation (Primary)

```
[Home] [Quizzes] [Types] [Compare] [Blog]
```

### Navigation (Secondary — Footer)

```
Quizzes  |  Types  |  Comparisons  |  Careers  |  Learning Styles  |  Tools  |  About  |  Privacy
```

### Internal Linking Flywheel

Every page type links to specific other page types:

```
Result Page:
  → Links to primary type page (the result they got)
  → Links to 3-5 related quizzes
  → Links to 2-3 comparison pages with their type
  → Links to career page for their type

Type Page:
  → Links to related quizzes that produce this type
  → Links to comparison pages (this type vs others)
  → Links to career, learning, and tool pages for this type
  → Links to relevant blog articles

Comparison Page:
  → Links to both type pages being compared
  → Links to quizzes that produce either type
  → Links to blog articles about team dynamics

Blog Article:
  → Links to relevant type pages
  → Links to relevant quizzes
  → Links to comparison pages
  → Links to other blog articles (related posts)
```

**Target: 5-10 internal links per page, minimum.**

---

## 5. Database Schema

### Tables

#### `quiz_completions`

Stores anonymous quiz completions. Minimal schema — no auth, no users.

```sql
CREATE TABLE quiz_completions (
  id         SERIAL PRIMARY KEY,
  quiz_id    TEXT NOT NULL,          -- slug of the quiz
  result_id  TEXT NOT NULL,          -- slug of the result archetype
  session_id TEXT NOT NULL,          -- anonymous session identifier
  referrer   TEXT,                    -- optional referrer URL
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_completions_quiz ON quiz_completions(quiz_id);
CREATE INDEX idx_completions_result ON quiz_completions(result_id);
CREATE INDEX idx_completions_created ON quiz_completions(created_at DESC);
```

**Why this minimal schema:**
- Enables social proof ("27% of developers got Systems Thinker")
- Enables content generation ("Rust developers are 2.3x more likely to be Infrastructure Purists")
- Enables viral statistics for blog content
- No auth, no user management, no overhead

### Drizzle Schema

```typescript
// src/db/schema.ts
import { serial, text, timestamp, pgTable, index } from "drizzle-orm/pg-core";

export const quizCompletions = pgTable("quiz_completions", {
  id: serial("id").primaryKey(),
  quizId: text("quiz_id").notNull(),
  resultId: text("result_id").notNull(),
  sessionId: text("session_id").notNull(),
  referrer: text("referrer"),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
}, (table) => [
  index("idx_completions_quiz").on(table.quizId),
  index("idx_completions_result").on(table.resultId),
  index("idx_completions_created").on(table.createdAt),
]);
```

---

## 6. Data Architecture

### All content authored in YAML

No CMS. No admin panel. All content lives in the repo as YAML files.

#### Quiz Definition (`content/quizzes/language-matchmaker.yaml`)

```yaml
id: language-matchmaker
title: Which Programming Language Matches Your Personality?
description: Discover which programming language aligns with your engineering personality type.
slug: language-matchmaker

questions:
  - id: q1
    question: "When starting a new project, you:"
    answers:
      - text: "Design the architecture first, then code"
        weights:
          systems-thinker: 3
          research-engineer: 2
      - text: "Ship something minimal, then iterate"
        weights:
          startup-hacker: 3
          product-builder: 2
      - text: "Research existing solutions thoroughly"
        weights:
          debugging-detective: 2
          infrastructure-purist: 2
      - text: "Set up the dev environment and toolchain"
        weights:
          platform-architect: 3
          craftsman-engineer: 2

  - id: q2
    question: "You prefer code that is:"
    answers:
      - text: "Elegant and abstract"
        weights:
          systems-thinker: 3
          research-engineer: 2
      - text: "Fast and efficient"
        weights:
          performance-optimizer: 3
          infrastructure-purist: 2
      - text: "Readable and maintainable"
        weights:
          craftsman-engineer: 3
          team-catalyst: 2
      - text: "Pragmatic and working"
        weights:
          startup-hacker: 3
          product-builder: 2

# ... more questions

dimensions:
  - name: thinking-style
    description: Abstract ↔ Practical
    weight_map:
      systems-thinker: abstract
      research-engineer: abstract
      debugging-detective: abstract
      technical-strategist: abstract
      team-catalyst: abstract
      platform-architect: abstract
      infrastructure-purist: practical
      startup-hacker: practical
      product-builder: practical
      performance-optimizer: practical
      ai-amplifier: practical
      craftsman-engineer: practical

  - name: working-preference
    description: Individual ↔ Collaborative
  - name: approach
    description: Exploration ↔ Optimization

results:
  - id: systems-thinker
    title: Systems Thinker
    tagline: "You see the whole system before anyone else does."
    description: "..."
    traits:
      - Abstract reasoning
      - Big-picture thinking
      - Deep architecture
    strengths:
      - System design
      - Debugging complex interactions
      - Long-term planning
    weaknesses:
      - May over-engineer
      - Can be slow to ship
    recommended_roles:
      - Staff Engineer
      - Solutions Architect
      - Principal Engineer
    recommended_languages:
      - Rust
      - Go
      - TypeScript
    team_compatibility:
      - Startup Hacker (builds what you design)
      - Craftsman Engineer (implements your architecture)
    learning_style:
      - Deep dives into documentation
      - Whiteboard-first thinking
```

#### Type Definition (`content/types/systems-thinker.yaml`)

```yaml
id: systems-thinker
name: Systems Thinker
slug: systems-thinker
emoji: 🏗️

dimensions:
  thinking-style: abstract
  working-preference: individual
  approach: exploration

description: >
  Systems Thinkers see the whole architecture before anyone else does.
  They excel at understanding complex interactions and designing systems
  that scale. They prefer abstract reasoning over concrete implementation.

traits:
  - Abstract reasoning
  - Big-picture orientation
  - Deep curiosity
  - Pattern recognition

strengths:
  - System design and architecture
  - Debugging complex interactions
  - Long-term technical planning
  - Cross-system integration

weaknesses:
  - May over-engineer simple solutions
  - Can be slow to ship
  - Might miss pragmatic shortcuts
  - Prefers theory over execution

recommended_roles:
  - Staff Engineer
  - Solutions Architect
  - Principal Engineer
  - CTO

recommended_languages:
  - Rust
  - Go
  - TypeScript
  - Haskell

ai_tool_preferences:
  - Uses AI for architecture brainstorming
  - Less interested in AI code completion
  - Values AI for documentation generation

learning_style:
  - Deep documentation dives
  - Whiteboard-first thinking
  - Academic papers and RFCs
  - System design interviews

compatible_with:
  - startup-hacker: "They build what you design"
  - craftsman-engineer: "They implement your architecture with care"
  - product-builder: "They ground your abstractions in user needs"

conflicts_with:
  - performance-optimizer: "You prioritize structure over micro-optimization"

famous_examples:
  - "Martin Kleppmann"
  - "Rich Hickey"

seo:
  meta_title: "Systems Thinker — Developer Personality Archetype"
  meta_description: "Learn about the Systems Thinker developer archetype: traits, strengths, recommended languages, careers, and compatible team roles."
```

---

## 7. Tech Stack & Dependencies

### Core

| Package | Version | Purpose |
|---------|---------|---------|
| `next` | ^16.2.6 | Framework (App Router) |
| `react` / `react-dom` | ^19.2.4 | UI |
| `tailwindcss` | ^4 | Styling |
| `@tailwindcss/postcss` | ^4 | PostCSS plugin |
| `tw-animate-css` | latest | Animation utilities |
| `typescript` | ^5 | Type safety |

### shadcn/ui Components (MVP)

```bash
npx shadcn@latest add button card dialog alert badge navigation-menu sheet separator skeleton tabs tooltip
```

### Database

| Package | Version | Purpose |
|---------|---------|---------|
| `drizzle-orm` | latest | ORM |
| `@neondatabase/serverless` | latest | Serverless PostgreSQL driver |
| `drizzle-kit` | latest (dev) | Migrations |

### OG Image Generation

| Package | Version | Purpose |
|---------|---------|---------|
| `@vercel/og` | latest | (bundled with `next/og`) |
| System fonts | — | Inter font for OG images |

### Analytics (MVP)

| Tool | Purpose |
|------|---------|
| Google Analytics 4 | Core analytics |
| Google Search Console | SEO monitoring |
| (Future: PostHog or Plausible) | Product analytics v2 |

---

## 8. SEO Strategy

### On-Page SEO

Every page type has a metadata template:

| Page Type | Title Pattern | Description Pattern |
|-----------|---------------|---------------------|
| Quiz | `[Quiz Title] — Developer Personality Test` | `Take the [Quiz Title] quiz. Discover your developer personality type and find out which [topic] matches your engineering style.` |
| Result | `[Archetype Name] — [Quiz Title] Result` | `You got [Archetype Name]! Learn what this means for your career, preferred languages, team fit, and more.` |
| Type | `[Archetype Name] — Developer Personality Archetype` | `Learn about the [Archetype Name] developer archetype: traits, strengths, recommended languages, careers, and compatible team roles.` |
| Comparison | `[Archetype A] vs [Archetype B] — Developer Personality Comparison` | `Compare [Archetype A] and [Archetype B] developer archetypes. See their traits, strengths, team dynamics, and career paths side by side.` |
| Career | `Best Engineering Roles for [Archetype Name] Developers` | `Discover the best engineering career paths for [Archetype Name] developers. From Staff Engineer to CTO, find your fit.` |
| Blog | Standard blog title | Standard blog description |

### Open Graph

Every page gets:

```typescript
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  return {
    openGraph: {
      title: "...",
      description: "...",
      type: "website",
      images: [{ url: `/og/...`, width: 1200, height: 630 }],
    },
    twitter: {
      card: "summary_large_image",
      title: "...",
      description: "...",
    },
  };
}
```

### Schema.org Structured Data

```json
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "Developer Personality",
  "description": "Discover your developer personality type. Quizzes and content for software engineers.",
  "url": "https://developerpersonality.com"
}
```

Quiz pages get `@type: Quiz` structured data. Blog articles get `@type: Article`.

### Technical SEO

- **SSG for core pages** → fast load, fully indexable
- **ISR for generated pages** → always fresh, never stale
- **Canonical URLs** on every page
- **Sitemap.xml** (auto-generated via `next-sitemap` or manual)
- **Robots.txt** — allow all, sitemap reference
- **No JavaScript required** for content rendering (progressive enhancement)
- **Core Web Vitals** target: all green

---

## 9. Programmatic SEO Strategy

### Content Generation Pipeline

```
Type Definitions (YAML)
        │
        ▼
┌─────────────────────────────┐
│   Build-time Generation     │
│                              │
│  Type → Profile page (SSG)   │
│  Types → Comparison (ISR)   │
│  Type → Career page (SSG)   │
│  Type → Learning page (SSG) │
│  Type → Tools page (SSG)    │
│  Quiz → Quiz page (SSG)     │
│  Quiz → Result page (SSG)   │
└─────────────────────────────┘
```

### Content Templates

#### Type Profile Page (`/types/[slug]`)

```
Layout:
  1. Hero with type name, emoji, tagline
  2. Dimension badges (Abstract · Individual · Exploration)
  3. "Are you a [Type]? Take the quiz →" CTA
  4. Traits section
  5. Strengths & Weaknesses
  6. Recommended Roles
  7. Recommended Languages
  8. AI Tool Preferences
  9. Compatible / Conflicting types (with comparison links)
  10. Learning Style
  11. Famous Examples
  12. "Find your type →" CTA
  13. Related quizzes
  14. Related blog articles
  <!-- AD SLOTS added post-launch after AdSense approval: one between sections 5-6, one between 11-12 -->
```

#### Comparison Page (`/compare/[type-a]-vs-[type-b]`)

```
Layout:
  1. Title: "[Type A] vs [Type B] — Developer Personality Comparison"
  2. Side-by-side trait comparison
  3. When they work well together
  4. When they conflict
  5. Team composition advice
  6. Which roles suit each type
  7. Communication tips between types
  8. Links to both type profiles
  9. "Which one are you? Take the quiz →"
  <!-- AD SLOT added post-launch after AdSense approval: one between sections 5-6 -->
```

### Long-tail Keyword Targeting

| Keyword Type | Example | Target Page |
|-------------|---------|-------------|
| Personality quiz | "what kind of developer am i" | Landing |
| Archetype | "systems thinker developer" | Type page |
| Comparison | "systems thinker vs startup hacker" | Comparison page |
| Career | "best engineering role for systems thinker" | Career page |
| Learning | "how do systems thinkers learn programming" | Learning page |
| Team | "building a team with systems thinker and startup hacker" | Blog / Comparison |
| Language | "best programming language for systems thinker" | Tools page |

**Keyword research note:** Before scaling content production, validate monthly search volume for archetype-specific terms via Google Keyword Planner, Ahrefs, or SEMrush. Focus content investment on term classes with proven volume. If custom archetype terms show low volume, weight the content mix toward natural-language blog queries and known developer personality search terms.

---

## 10. Ad Placement Strategy

### Layout Rules

| Device | Ad Slots | Format |
|--------|----------|--------|
| Desktop | Sidebar (1) + In-content (2) | Display ads |
| Mobile | In-content (2) only | Responsive display |

### Page Layouts

#### Result Page

```
Hero Section
Result Summary
Share CTA
Recommended Careers
[AD SLOT 1 — in-content]
Recommended Languages
Related Personality Types
Comparison Links
[AD SLOT 2 — in-content]
Related Quizzes
```

#### Type Page

```
Overview
Traits
Strengths & Weaknesses
[AD SLOT 1 — in-content]
Career Recommendations
Tool Recommendations
Compatibility
[AD SLOT 2 — in-content]
Related Content
```

#### Desktop Sidebar

- Sticky sidebar ad (300x250 or 300x600)
- Present on type pages, comparison pages, and blog articles

### Ad Placement Notes

- Ad placement infrastructure (AdSlot.tsx, Sidebar.tsx) is built as post-launch work after AdSense approval. At MVP, sidebar serves navigation/related content.
- Page templates reserve ad slot positions (documented with HTML comments) for post-launch activation.
- AdSense RPM for the developer niche is structurally lower than general content due to ad-block rates (40%+ among developers). Factor this into revenue projections — adjust RPM estimates accordingly.

### Anti-Patterns (Avoid at Launch)

- ❌ Sticky footer ads
- ❌ Interstitials
- ❌ Pop-ups
- ❌ Auto-play video
- ❌ Ad density > 25% of viewport

### KPI Philosophy

**Optimize for revenue per visitor, not revenue per pageview.**

The growth loop (Quiz → Result → Type → Comparison → Career → Blog → Another Quiz) creates 5-10 pageviews per session. More pages = more impressions. Protect shareability above all else — traffic is the bottleneck, not ad density.

---

## 11. Monetization Roadmap

### Phase 1 — AdSense (MVP → Month 6)

- Google AdSense auto-ads (carefully configured)
- 2-3 ad slots per page
- Focus on traffic volume and session depth
- **AdSense contingency:** Apply for AdSense pre-launch to establish timeline. If delayed or rejected, fall back to alternative ad networks (Carbon, BuySellAds) and accelerate affiliate link integration (tool recommendations, book recommendations, course referrals) to MVP phase. Validate site operates within hosting budget even without AdSense revenue.

### Phase 2 — Affiliate Partnerships (Month 6-12)

- Dev tool affiliates: JetBrains, Datadog, Sentry, GitHub Copilot
- Book affiliates: O'Reilly, Manning, No Starch Press
- Course affiliates: Frontend Masters, Pluralsight, Egghead
- Content recommendation: "Recommended Languages" → affiliate links
- Career pages: "Recommended Roles" → affiliate job boards

### Phase 3 — Sponsorships (Month 12+)

- Sponsored quiz partnerships (company-specific personality quizzes)
- Newsletter sponsorship (post-MVP)
- Content sponsorship (sponsored comparison pages)
- Direct ad sales (higher CPM than AdSense)

---

## 12. Content Cluster Strategy

### Pillar Content

| Pillar | Contains | Target Keywords |
|--------|----------|-----------------|
| Developer Personality Types | 12 type pages + comparisons | developer personality type, what kind of developer am I |
| Engineering Careers | 12 career pages + guides | software engineer career path, staff engineer, engineering manager |
| Programming Languages | 12 language pages + comparisons | best programming language for [trait], language personality |
| Engineering Teams | Comparison + blog + compatibility | engineering team composition, developer team dynamics |

### Blog Content Clusters

#### Cluster 1: Personality & Productivity
- "Which Programming Language Fits Your Personality?"
- "The Best Engineering Roles for Analytical Thinkers"
- "Why Some Developers Love AI Pair Programming"
- "Introverts vs Extroverts in Software Engineering"
- "Staff Engineer vs Engineering Manager Personality Traits"

#### Cluster 2: Team Dynamics
- "How to Build a Balanced Engineering Team"
- "When Systems Thinkers and Startup Hackers Collide"
- "The Best Team Composition for Startup vs Enterprise"
- "Code Review Styles by Personality Type"

#### Cluster 3: Career Growth
- "Which Engineering Role Matches Your Personality?"
- "From IC to Manager: Personality Fit for Each Path"
- "Developer Personality and Career Satisfaction"

#### Cluster 4: Tools & Tech
- "Best IDE for Your Developer Personality Type"
- "AI Coding Tools: Who Loves Them and Who Avoids Them"
- "Version Control Habits by Developer Personality"

---

## 13. Internal Linking Strategy

### Automated Link Targets

Every page template includes computed links:

```typescript
// For a type page (e.g., /types/systems-thinker)
const links = [
  // Quiz that produces this type
  { to: "/quizzes/language-matchmaker", text: "Take the Language Matchmaker quiz" },
  // Career page
  { to: "/careers/systems-thinker", text: "Best careers for Systems Thinkers" },
  // Compatible types
  { to: "/types/startup-hacker", text: "Startup Hacker (compatible)" },
  // Comparisons
  { to: "/compare/systems-thinker-vs-startup-hacker", text: "Systems Thinker vs Startup Hacker" },
  { to: "/compare/systems-thinker-vs-craftsman-engineer", text: "Systems Thinker vs Craftsman Engineer" },
  // Learning & tools
  { to: "/learn/systems-thinker", text: "How Systems Thinkers Learn" },
  { to: "/tools/systems-thinker", text: "Best Tools for Systems Thinkers" },
  // Related blog
  { to: "/blog/engineering-roles-for-analytical-thinkers", text: "Engineering Roles for Analytical Thinkers" },
];
```

### Minimum Internal Links per Page

| Page Type | Minimum Internal Links |
|-----------|----------------------|
| Landing | 8+ |
| Quiz page | 3+ |
| Result page | 5+ |
| Type page | 8+ |
| Comparison page | 6+ |
| Career page | 5+ |
| Blog article | 3+ |

---

## 14. OG Image Strategy

### Dynamic OG Image Template (Result Pages)

Each result page generates a unique OG image showing:

```
┌─────────────────────────────────────┐
│                                     │
│       🏗️ THE SYSTEMS THINKER       │
│                                     │
│    INTP Developer Archetype         │
│                                     │
│    Strengths:                       │
│    ✓ Architecture                   │
│    ✓ Debugging                      │
│    ✓ Research                       │
│                                     │
│    Top Languages:                   │
│    Rust  •  Go  •  TypeScript       │
│                                     │
│    developerpersonality.com         │
└─────────────────────────────────────┘
```

### Implementation

```typescript
// app/quiz/[slug]/[result]/opengraph-image.tsx
import { ImageResponse } from 'next/og';

export const size = { width: 1200, height: 630 };
export const contentType = 'image/png';

export default async function Image({
  params,
}: {
  params: Promise<{ slug: string; result: string }>;
}) {
  const { slug, result } = await params;
  const archetype = archetypes[result];
  // ...render branded image
}
```

---

## 15. MVP Roadmap

### Phase 0 — Foundation (Week 1-2)

- [x] Next.js 16 project initialized
- [ ] shadcn/ui setup with theme configuration
- [ ] Tailwind CSS v4 globals with custom color tokens
- [ ] Project folder structure (see below)
- [ ] Drizzle ORM + PostgreSQL (Neon) setup
- [ ] Database migration: `quiz_completions` table
- [ ] YAML content loader utility
- [ ] OG image generation utility with Inter font
- [ ] Google Analytics 4 integration

### Accessibility Requirements

All pages and components target WCAG 2.1 AA compliance:

- **Quiz engine:** ARIA live regions announce question text and answer feedback. Focus management moves to first answer option on each new question. Keyboard-only flow supported through all quiz states. Result announcement via `aria-live` region.
- **Content pages:** Skip-link to bypass navigation and jump to main content. Proper heading hierarchy (h1→h2→h3) on every page template. Alt text on all OG images and illustrations. Color contrast minimums for archetype badges and dimension labels.
- **Page templates:** NotFound pattern for 404s (Next.js not-found.tsx). Empty state content for blog, quiz, and type indexes with onboarding CTAs. Loading states for ISR and SSR pages during initial hydration.

### Quiz Interaction State Machine

The quiz engine supports these interaction states:

| State | Description |
|-------|-------------|
| `idle` | Pre-quiz intro screen with title, description, start CTA |
| `answering` | Question visible with answer options + progress indicator (N of M) |
| `transitioning` | Brief animation between questions (slide/fade, 300ms) |
| `calculating` | Short overlay after last answer while scores compute |
| `result` | Result reveal with archetype name, emoji, tagline, strengths |
| `error` | Data-load failure or YAML parse error with retry CTA |

Default interaction: one question per page with back navigation. Multi-select answers supported via YAML schema. On mobile, use `navigator.share()` for share CTAs with a custom share-sheet fallback on desktop.

### Phase 1 — Core Quizzes (Week 3-4)

- [ ] Define 1 flagship quiz in YAML (e.g., "What's Your Developer Archetype?" — the general typing quiz)
- [ ] Quiz engine component (render questions, calculate results, animate transitions)
- [ ] Quiz page template (`/quizzes/[slug]`)
- [ ] Result page template (`/quizzes/[slug]/[result]`) with unique canonical URL
- [ ] Dynamic OG image for each result
- [ ] Completion tracking (POST to API route → insert into PostgreSQL)
- [ ] Share CTAs (copy link, social pre-filled text, native share sheet on mobile)
- [ ] 2 more themed quizzes in YAML (e.g., "Language Matchmaker", "Engineering Role Finder")
- [ ] **Validation gate:** Measure quiz completion-to-share rate. If > 5%, proceed to Phase 2. If below, investigate and iterate before expanding content investment.

**Validation gate:** Before starting Phase 2, confirm the Phase 1 flagship quiz achieves a completion-to-share rate > 5% and generates measurable organic impressions. If below threshold, iterate on quiz quality, sharing mechanics, or acquisition channels before expanding content investment.

### Phase 2 — Personality Library (Week 5-6)

- [ ] All 12 type profile definitions in YAML (`content/types/*.yaml`)
- [ ] Type profile page template (`/types/[slug]`) — SSG
- [ ] `/types/` index page
- [ ] 10-15 top comparison pages (`/compare/[a]-vs-[b]`) — ISR
- [ ] `/compare/` index page
- [ ] Career pages (`/careers/[slug]`) per type
- [ ] Internal linking system (computed links)

### Phase 3 — Blog & Launch (Week 7-8)

- [ ] Blog section with 10-20 launch articles
- [ ] Blog index page
- [ ] Related posts component
- [ ] Full sitemap generation
- [ ] robots.txt
- [ ] JSON-LD structured data
- [ ] Google Search Console setup
- [ ] Google AdSense application & setup
- [ ] Performance audit (Lighthouse, Core Web Vitals)
- [ ] SEO audit (meta tags, OG tags, structured data, internal links)
- [ ] **Public launch**

### Folder Structure (MVP)

```
content/
  quizzes/
    developer-archetype.yaml
    language-matchmaker.yaml
    engineering-role-finder.yaml
  types/
    systems-thinker.yaml
    infrastructure-purist.yaml
    startup-hacker.yaml
    ... (all 12 types)
  comparisons.yaml             # Comparison definitions

src/
  app/
    layout.tsx                 # Root layout (fonts, theme, analytics)
    page.tsx                   # Landing page
    globals.css                # Tailwind v4 + theme tokens
    quiz/
      page.tsx                 # Quiz index
      [slug]/
        page.tsx               # Quiz page (SSG)
        [result]/
          page.tsx             # Result page (SSG)
          opengraph-image.tsx  # Dynamic OG image
    types/
      page.tsx                 # Type index (SSG)
      [slug]/
        page.tsx               # Type page (SSG)
    compare/
      page.tsx                 # Comparison index
      [slugs]/
        page.tsx               # Comparison page (ISR)
  careers/
    page.tsx                 # Career index
    [slug]/
      page.tsx               # Career page (SSG)
    learn/
    page.tsx                 # Learning styles index
      [slug]/
        page.tsx             # Learning style page (SSG)
    tools/
      page.tsx               # Tools & languages index
      [slug]/
        page.tsx             # Tool/language preference page (SSG)
    blog/
      page.tsx                 # Blog index
      [slug]/
        page.tsx               # Blog article
    api/
      completions/
        route.ts               # POST endpoint for quiz completions
    sitemap.ts                 # Dynamic sitemap

  components/
    ui/                        # shadcn/ui components
    quiz/
      QuizEngine.tsx           # Interactive quiz component
      QuizQuestion.tsx         # Single question card
      QuizResult.tsx           # Result display card
    layout/
      Header.tsx               # Site header
      Footer.tsx               # Site footer
      Sidebar.tsx              # Desktop navigation sidebar (ad slot added post-launch)
    ads/
      AdSlot.tsx               # Ad placement component (post-launch, after AdSense approval)
    seo/
      JsonLd.tsx               # Structured data component

  lib/
    content/
      loadQuiz.ts              # YAML quiz loader
      loadType.ts              # YAML type loader
      loadAllQuizzes.ts        # Load all quizzes
      loadAllTypes.ts          # Load all types
    loadComparison.ts        # Comparison YAML loader
    loadAllComparisons.ts    # Load all comparisons
    utils.ts                   # cn() utility
    seo.ts                     # Metadata helpers

  db/
    schema.ts                  # Drizzle schema
    drizzle.ts                 # DB client

next.config.ts
drizzle.config.ts
```

---

## 15b. Content Completeness Note

The Section 2 table claims 150+ discoverable pages including leadership pages (12), team-role pages (12), learning-style pages (12), AI-tool preference pages (12), and language preference pages (12). At MVP launch, the following pages are included:

| Page type | MVP included? | Notes |
|-----------|--------------|-------|
| Profile pages (12) | Yes | All 12 type profiles |
| Pairwise comparisons (66) | Yes | 10-15 launch comparisons, rest programmatic post-launch |
| Career pages (12) | Yes | All 12 career pages |
| Learning-style pages (12) | Follow-up | Add post-launch from type data |
| AI-tool preference pages (12) | Follow-up | Add post-launch from type data |
| Language preference pages (12) | Follow-up | Add post-launch from type data |
| Leadership pages (12) | Follow-up | Add post-launch from type data |
| Team-role pages (12) | Follow-up | Add post-launch from type data |

**Competitive analysis:** Before scaling content investment, research existing developer personality quiz sites, their traffic sources, and keyword positioning. Key competitors to evaluate include existing MBTI-for-engineers content, CodinGame's developer survey data, and any established personality quiz sites targeting the developer audience. Understanding the competitive landscape will inform keyword strategy and content differentiation.

## 16. Growth Roadmap

### Pre-Launch (Now)
- Define 12 archetypes and write all YAML definitions
- Create 1 flagship quiz (the general archetype matcher)
- Build all type pages from YAML data
- Generate 10-15 comparison pages
- Write 10-20 blog articles

### Launch (Day 1)
- Announce on:
  - Hacker News (Show HN)
  - Reddit (r/webdev, r/programming, r/javascript)
  - Dev.to
  - Twitter/X with OG image showcases
- Organic search begins indexing

### Month 1-3
- Add 2-3 new quizzes per month
- Add 4-8 comparison pages per week (programmatic generation)
- Add 2 blog articles per week
- Monitor Search Console for indexing and keyword performance
- Identify top-performing queries → create more content for them

### Month 3-6
- Build programmatic quiz engine (generate quiz variants from templates)
- Add GitHub README badge (viral distribution channel)
- Experiment with:
  - Quiz result sharing to X/Twitter with auto-image
  - "Most developers got X" social proof widgets
  - Embeddable result cards
- Launch newsletter (optional, if traffic warrants)

### Month 6-12
- Affiliate partnerships
- Sponsored quiz content
- Guest post outreach for backlinks
- Community building (Discord or similar)
- Advanced features: user accounts (optional), saved results

### Year 2+
- Direct sponsorship model
- Premium content / reports
- API for quiz embedding on other sites
- Job board integration (personality-matched job listings)

---

## 17. Key Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| Low quiz sharing rates | Invest in OG image quality; test social copy variations |
| Slow SEO ramp | Generate 150+ content pages at launch; target long-tail keywords |
| AdSense rejection | Ensure content is substantial (500+ words per page); avoid thin content |
| Quiz engine complexity | Start with simple scored quiz; add multi-dimensional later |
| Build time too long with SSG | Use ISR for comparison pages; lazy-load less critical pages |
| Content feels generic | Custom 12-archetype system + developer-native language + real examples |
| Solo dev burnout | YAML-driven architecture minimizes manual work; programmatic generation scales without effort |

---

## 18. Technical Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Rendering | SSG + ISR hybrid | Fastest UX, SEO-friendly, cost-effective |
| Content storage | YAML in repo | No CMS overhead, version-controlled, easy to generate |
| Database | PostgreSQL (Neon) + Drizzle | Minimal schema, anonymous tracking, serverless-friendly |
| Quiz engine | Client-side React | Dynamic interactions, server delivers data |
| OG images | `next/og` (`ImageResponse`) | Dynamic per result, no external service needed |
| Styling | Tailwind CSS v4 + shadcn/ui | Rapid UI development, accessible defaults |
| Analytics | Google Analytics 4 | Free, universal, essential for AdSense |
| Deployment | Vercel | Native Next.js support, ISR, edge functions |
| Ad placement | 2 in-content + desktop sidebar | Protects shareability, optimizes revenue per visitor |
| Personality system | 12 custom archetypes, 3 axes | Differentiated from MBTI, rich content surface |

---

## 19. Quick Start (Implementation Sequence)

```bash
# 1. Install shadcn/ui
npx shadcn@latest init -d

# 2. Install remaining dependencies
npm install drizzle-orm @neondatabase/serverless
npm install -D drizzle-kit

# 3. Add shadcn components
npx shadcn@latest add button card dialog alert badge navigation-menu sheet separator skeleton tabs tooltip

# 4. Create PostgreSQL database on Neon
#    (or use local PostgreSQL for development)

# 5. Set DATABASE_URL in .env.local

# 6. Generate and push migration
npx drizzle-kit generate
npx drizzle-kit push

# 7. Create content directory structure
mkdir -p content/quizzes content/types

# 8. Create first quiz and type definitions
#    (copy templates from this document)

# 9. Build the quiz engine, type pages, comparison pages
#    (follow the folder structure above)

# 10. Add OG image generation for result pages

# 11. Add analytics, sitemap, robots.txt, structured data

# 12. Deploy to Vercel
git push
```

---

---

## 20. Deferred / Open Questions

These items were identified during document review and deferred for later resolution:

- **Archetype framework validation:** The 12-archetype taxonomy has not been validated with real developers before content commitment. Consider a lightweight validation step (survey, pilot quiz) before full 150+ page build.
- **Quiz-sharing behavior:** The traffic flywheel assumes developers share personality quiz results on social media at meaningful velocity. This has not been validated. Consider monitoring sharing rates from Week 1 and having a fallback acquisition strategy.
- **Solo developer capacity:** The 8-week MVP timeline (12 archetypes + 66 comparisons + blog + quizzes + full page templates) may exceed solo developer capacity. Monitor progress weekly and adjust scope if needed.
- **Database timing:** Database infrastructure (Neon, Drizzle) is committed in Phase 0 but its consumers (social proof widgets, statistical content) don't appear until Month 3-6. Consider whether DB setup can be deferred or simplified in Phase 0.
- **SSG build-time scaling:** All non-comparison pages use SSG (type pages, career, learn, tools, quiz pages). Build times grow linearly with page count. Monitor and plan migration to ISR for slow-building page types as content scales.
- **Client-side quiz vs. progressive enhancement:** The quiz engine requires JavaScript but the SEO section claims "No JavaScript required for content rendering." Consider whether the quiz can work as a server-rendered form (progressive enhancement) or whether the SEO claim should qualify the quiz engine exception.
- **Responsive layout:** Core page templates (14-section type profile, side-by-side comparison) have no documented responsive behavior. Define mobile reflow, stacking order, and content prioritization before implementing.
- **Leadership and team-role pages:** These 24 pages are counted in the 150+ content surface claim but have no implementation tasks or URL structure. Either add them to the roadmap or adjust the count.
- **AdSense economics:** Developer-niche CPMs and ad-block rates (40%+) may make AdSense-only monetization challenging. Validate projected RPM with realistic ad-block-adjusted estimates.
- **YAML iteration friction:** Every SEO metadata change requires a git commit, CI run, and Vercel deploy. Plan for a lightweight editing interface or CMS migration path when content operations scale beyond one developer.

---

*This spec represents the complete implementation plan for Developer Personality. All strategic decisions have been validated through the interview process and are ready for development.*
