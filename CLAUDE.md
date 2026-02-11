# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pokemon card (TCGP) search app — a fully static Next.js site deployed to GitHub Pages. ~2500 cards bundled client-side with AI-powered natural language search (LLM filter extraction + embedding similarity + RRF ranking). Users provide their own Anthropic/OpenAI API keys stored in localStorage. Bilingual (EN/JA).

## Commands

```bash
npm run dev      # Start dev server
npm run build    # Production build (static export to out/)
npm run lint     # ESLint (flat config, ESLint 9)
npm start        # Serve production build locally
```

## Tech Stack

- **Next.js 16** with App Router, `output: 'export'` (static only, no API routes)
- **React 19** (use `use()` instead of `useContext()`, no `forwardRef`)
- **TypeScript 5** (strict mode), path alias `@/*` maps to repo root
- **Tailwind CSS 4** via PostCSS
- **ESLint 9** flat config with `eslint-config-next` (core-web-vitals + typescript)

## Architecture

### Static-first, no server

All API calls (Anthropic Claude, OpenAI Embeddings) go directly from the browser. The Anthropic API requires the `anthropic-dangerous-direct-browser-access` header. No server-side processing exists.

### Data flow

Card data (~2500 cards) is bundled in `data/`. Embedding vectors (~15MB per language) live in `public/data/` and are lazy-loaded only when AI search is used.

### AI search pipeline (5 steps)

1. **Filter extraction** — Anthropic API (Haiku) parses natural language into `ParsedQuery` (structured filters + semantic query)
2. **Structured filtering** — Client-side JS filters cards by type, HP, rarity, etc. (single `for...of` loop)
3. **Semantic search** — OpenAI embeddings → cosine similarity + keyword search → RRF fusion ranking
4. **Answer generation** — Anthropic API (Haiku) streaming response
5. **Display** — Card grid/list with AI answer panel

If any AI step fails, structured filter results still display (graceful degradation).

### Key data types

- `PokemonCard` — core card type with `LocalizedText` (`{en, ja}`) fields
- `ParsedQuery` — LLM-extracted filters + semantic query
- `CardSearchState` — search UI state (query, filters, results, loading)

### Component architecture

Uses **Compound Components** pattern: `CardSearch.*`, `CardDisplay.*`, `Settings.*` — each with Context/Provider. Contexts follow `{state, actions, meta}` interface pattern.

- **Explicit variants** over boolean props (e.g., `CardGridView` / `CardListView`, not `<CardView grid={true}>`)
- **Children composition** over render props
- **Lazy loading** via `next/dynamic` for AI panel, modals

### Performance patterns

- `content-visibility: auto` on card items for virtualized-like rendering
- `memo()` on CardItem components
- `startTransition()` for search/filter state updates
- `Map<cardId, PokemonCard>` and `Set<string>` for O(1) lookups
- `useMemo` for derived state (filteredCards), never store derived data in state
- `useState(() => loadFromStorage())` for lazy initialization
- `.toSorted()` for immutable sorting

### i18n

Lightweight custom implementation (no library). `t(key, locale)` function, `useLocale()` hook, `LocalizedText` type for card data. Embedding files and filter extraction prompts are language-specific.

### localStorage

Versioned keys (e.g., `pokemon-search-settings:v1`). API keys stored locally, never sent to any server.

## Pre-computation scripts

Three batch scripts in `scripts/` run before build using developer API keys:

1. `generate-card-data.ts` — Convert source JSON to `PokemonCard[]` with LLM translations
2. `generate-visual-tags.ts` — Multimodal LLM image analysis (visualTags, artStyle, mood)
3. `generate-embeddings.ts` — OpenAI embedding vectors per language → `public/data/`

## Design Document

`docs/DESIGN.md` contains the full implementation plan including data models, AI search flow diagrams, filter extraction prompts, RRF algorithm, error handling strategy, skill application mapping, and 8-phase implementation roadmap. **Read this before making architectural decisions.**
