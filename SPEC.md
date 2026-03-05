# Card Ticker - MVP Specification

## Overview

Card Ticker is a React Native mobile application for tracking trading card game prices using the [JustTCG API](https://justtcg.com/docs). Users can search for cards, favorite them to a personal collection, and monitor price movements across supported games.

### Supported Games (MVP)

| Game | JustTCG Game ID | Cards | Variants |
|---|---|---|---|
| Pokemon | `pokemon` | 30K+ | 150K+ |
| Star Wars: Unlimited | TBD (confirm via `/games`) | 6K+ | 10K+ |

The app is designed to easily add more games in the future — games are driven by configuration, not hardcoded logic.

---

## Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Framework | React Native + Expo (managed) | Fastest path to iOS + Android, no native config |
| Routing | Expo Router (file-based) | Standard for new Expo projects, tab + stack nav |
| State Management | Zustand + AsyncStorage | Lightweight, persists favorites locally |
| Data Fetching | TanStack Query (React Query) | Caching, pagination, stale-while-revalidate, retries |
| UI Components | React Native Paper | Pre-built themed components (cards, chips, searchbar, lists) |
| Charts | react-native-gifted-charts or victory-native | Price history sparklines on Card Detail |
| HTTP Client | fetch (built-in) or axios | Simple REST calls with API key header |
| Secure Storage | expo-secure-store | Encrypted storage for API key |
| Query Persistence | @tanstack/query-async-storage-persister | Survives app restarts, saves API calls |

---

## Navigation Structure

```
Tab Navigator (3 tabs)
├── Home (Stack Navigator)
│   ├── HomeScreen (favorites + search + browse)
│   └── CardDetailScreen (push)
├── Movers (Stack Navigator)
│   ├── MoversScreen (gainers/losers)
│   └── CardDetailScreen (push)
└── Settings
    └── SettingsScreen
```

### Tab Bar

```
┌────────────────────────────────────┐
│    Home    │    Movers    │  Settings  │
└────────────────────────────────────┘
```

---

## Screens

### 1. Home Screen

The primary screen. Combines favorites display, card search, and set browsing into a single unified view.

#### Layout

```
┌──────────────────────────────────────────┐
│  🔍 Search cards...              [Filter] │
│  [Pokemon]  [Star Wars: Unlimited]        │
│  Set: [All Sets ▼]                        │
│  Sort: [Relevance ▼]                     │
├──────────────────────────────────────────┤
│                                           │
│  State A: FAVORITES (default)             │
│  ┌─────────────────────────────────────┐  │
│  │ 🖼 Charizard  Base Set   $245.00   │  │
│  │    Pokemon       NM      ▲ 3.2%    │  │
│  ├─────────────────────────────────────┤  │
│  │ 🖼 Darth Vader  SOR      $18.50    │  │
│  │    SW:U          NM      ▼ 1.1%    │  │
│  └─────────────────────────────────────┘  │
│                                           │
│  State B: SEARCH RESULTS                  │
│  (replaces favorites when query entered)  │
│                                           │
│  State C: SET BROWSE                      │
│  (shows cards from selected set)          │
│                                           │
│  Pull to refresh                          │
│  Infinite scroll pagination               │
└──────────────────────────────────────────┘
```

#### Behavior

| State | Trigger | Content Shown |
|---|---|---|
| Favorites | Search bar empty, no set filter | Favorited cards with prices and change indicators |
| Search | User types in search bar (debounced 500ms) | API search results filtered by selected game chip(s) |
| Browse | User selects a specific set from dropdown | Cards in that set |

- **Game chips** are toggleable filters. One, both, or neither can be active.
- **Set dropdown** populates based on selected game chip(s). Disabled when no game is selected.
- **Sort dropdown** allows ordering results. Available options depend on state:

| Sort Option | Available In | Implementation | Notes |
|---|---|---|---|
| Relevance | Search | Default API order | Default for search state |
| Card Number | Search, Browse | API param `orderBy=number` | Default for browse state |
| Price (High → Low) | Search, Browse, Favorites | API param `orderBy=price` | — |
| Price (Low → High) | Search, Browse, Favorites | API param `orderBy=price&order=asc` | — |
| Rarity | Search, Browse | Client-side sort | Sort order: Secret Rare > Ultra Rare > Holo Rare > Rare > Uncommon > Common (game-specific) |
| Price Movement (24h) | Search, Browse, Favorites | API param `orderBy=24h` | — |
| Price Movement (7d) | Search, Browse, Favorites | API param `orderBy=7d` | — |
| Price Movement (30d) | Search, Browse, Favorites | API param `orderBy=30d` | — |

- **Clearing** search text and set filter returns to favorites state.
- **Pull-to-refresh** on favorites triggers a batch refresh of all favorited cards.
- **Pagination**: infinite scroll, 20 cards per page.

#### Card List Item

Each card in the list displays:

```
┌──────────────────────────────────────┐
│ [IMG]  Card Name                     │
│        Set Name · Rarity             │
│        Game                          │
│                          $XX.XX      │
│                          ▲ X.X% 7d   │
│                              [♡/♥]   │
└──────────────────────────────────────┘
```

- Thumbnail image (left)
- Card name, set, rarity, game label
- Market price (Near Mint, default printing) displayed prominently
- 7-day price change with color (green up, red down, gray neutral)
- Favorite toggle (heart icon) — filled = favorited

---

### 2. Card Detail Screen

Pushed onto the stack when a card is tapped from Home or Movers.

#### Layout

```
┌──────────────────────────────────────┐
│  ← Back                     [♡/♥]    │
├──────────────────────────────────────┤
│                                      │
│         ┌──────────────┐             │
│         │              │             │
│         │  Card Image  │             │
│         │   (large)    │             │
│         │              │             │
│         └──────────────┘             │
│                                      │
│  Card Name                           │
│  Set Name · #Number · Rarity         │
│  Game                                │
│                                      │
├──── Price Overview ─────────────────┤
│                                      │
│  Market Price:  $XX.XX               │
│  24h: ▲ X.X%  7d: ▼ X.X%           │
│  30d: ▲ X.X%                        │
│                                      │
│  [  7d  |  30d  ] ← toggle          │
│  ┌──────────────────────────────┐    │
│  │  📈 Price History Sparkline  │    │
│  └──────────────────────────────┘    │
│                                      │
│  All-Time Low:  $X.XX  (date)       │
│  All-Time High: $X.XX  (date)       │
│                                      │
├──── Variants ───────────────────────┤
│                                      │
│  Condition    Printing       Price   │
│  ──────────   ──────────   ──────── │
│  Near Mint    Normal         $12.00  │
│  Near Mint    Holofoil       $45.00  │
│  Lightly Pl.  Normal          $9.50  │
│  Moderately   Normal          $7.00  │
│  ...                                 │
│                                      │
└──────────────────────────────────────┘
```

#### Data Displayed

- **Card image**: Full-size card image (source TBD — JustTCG may provide image URLs, otherwise use external sources)
- **Card metadata**: Name, set, number, rarity, game
- **Price overview**: Current market price (NM default), 24h/7d/30d percentage changes
- **Price history**: Sparkline chart with 7d/30d toggle, uses `priceHistory` array from API
- **All-time stats**: Min/max price with dates from `minPriceAllTime`, `maxPriceAllTime`
- **Variants table**: All condition/printing combinations with individual prices
- **Favorite button**: Heart icon in header, toggles favorite status

---

### 3. Movers Screen

Shows the biggest price gainers and losers.

#### Layout

```
┌──────────────────────────────────────┐
│  [My Cards | All Cards] ← toggle     │
│  [Pokemon]  [Star Wars: Unlimited]   │
│  Time: [24h]  [7d]  [30d]           │
├──────────────────────────────────────┤
│                                      │
│  ▲ TOP GAINERS                       │
│  ┌────────────────────────────────┐  │
│  │ 🖼 Card A    $45.00  ▲ 24.5%  │  │
│  │ 🖼 Card B    $12.30  ▲ 18.2%  │  │
│  │ 🖼 Card C     $8.00  ▲ 15.0%  │  │
│  │ 🖼 Card D     $3.50  ▲ 12.1%  │  │
│  │ 🖼 Card E    $67.00  ▲ 10.8%  │  │
│  └────────────────────────────────┘  │
│                                      │
│  ▼ TOP LOSERS                        │
│  ┌────────────────────────────────┐  │
│  │ 🖼 Card F    $22.00  ▼ 19.3%  │  │
│  │ 🖼 Card G     $5.60  ▼ 14.7%  │  │
│  │ 🖼 Card H    $90.00  ▼ 11.2%  │  │
│  │ 🖼 Card I     $1.25  ▼  8.9%  │  │
│  │ 🖼 Card J    $15.00  ▼  6.4%  │  │
│  └────────────────────────────────┘  │
│                                      │
│  Tap any card → Card Detail          │
└──────────────────────────────────────┘
```

#### Behavior

| Mode | Data Source | API Cost |
|---|---|---|
| My Cards | Sorts cached favorites data by price change | 0 requests (local) |
| All Cards | Fetches top movers per selected game from API | 2 per game (gainers + losers) |

- **My Cards mode**: Sorts the locally cached favorites by the selected time range's price change. Top 5 gainers (desc) and top 5 losers (asc). No API call needed.
- **All Cards mode**: Queries `/cards?game={id}&orderBy={timeRange}&order=desc&limit=5` for gainers and `order=asc&limit=5` for losers. One pair of requests per active game chip.
- **Game chips**: Filter which game(s) to show movers for. Can select one or both.
- **Time range**: Switches the `orderBy` parameter (`24h`, `7d`, `30d`).
- Tap any card to push Card Detail screen.

---

### 4. Settings Screen

```
┌──────────────────────────────────────┐
│  SETTINGS                            │
├──────────────────────────────────────┤
│                                      │
│  API Key                             │
│  ├── Status: ✓ Connected             │
│  ├── Key: ••••••••••••abc123         │
│  └── [Edit Key]  [Remove Key]        │
│                                      │
│  API Usage                           │
│  ├── Today: 34 / 100 requests        │
│  ├── This month: 412 / 1,000         │
│  └── [████████░░] 41.2%              │
│                                      │
│  Cache                               │
│  ├── Last refreshed: 5 min ago       │
│  ├── Cached cards: 142               │
│  └── [Clear Cache]                   │
│                                      │
│  About                               │
│  ├── Version: 1.0.0                  │
│  └── API: JustTCG Free Tier          │
│                                      │
└──────────────────────────────────────┘
```

---

## Data Models

### Unified Card Type

A single `Card` type is used across the entire application — for API responses, search results, favorites, and movers. There is no separate "favorites card" model. Favorite status is determined by checking the favorites store.

```typescript
// Card object — single source of truth for all card data
// Matches JustTCG API response shape directly
interface Card {
  id: string;
  name: string;
  game: string;
  set: string;
  set_name: string;
  number: string;
  rarity: string;
  tcgplayerId: string | null;
  imageUrl: string | null;
  variants: Variant[];
}

// Variant (specific condition + printing combination)
interface Variant {
  id: string;
  condition: string;
  printing: string;
  language: string;
  price: number | null;
  lastUpdated: number;
  priceChange24hr: number | null;
  priceChange7d: number | null;
  priceChange30d: number | null;
  priceChange90d: number | null;
  priceHistory: PricePoint[] | null;
  minPriceAllTime: number | null;
  maxPriceAllTime: number | null;
  minPriceAllTimeDate: string | null;
  maxPriceAllTimeDate: string | null;
  // Statistical fields
  avgPrice: number | null;
  minPrice7d: number | null;
  maxPrice7d: number | null;
  trendSlope7d: number | null;
}

interface PricePoint {
  timestamp: number;
  price: number;
}
```

### Favorites Store (Zustand + AsyncStorage)

The favorites store is lightweight — it tracks *which* cards are favorited and user preferences, not card data itself. Card data lives in TanStack Query's cache.

```typescript
// Favorites metadata — stored locally, keyed by card ID
interface FavoriteEntry {
  cardId: string;
  trackedVariantId: string;  // which variant to display in lists (default: NM)
  favoritedAt: string;       // ISO timestamp
}

// Favorites store shape
interface FavoritesState {
  favorites: Record<string, FavoriteEntry>;  // keyed by cardId for O(1) lookup

  // Actions
  addFavorite: (cardId: string, variantId: string) => void;
  removeFavorite: (cardId: string) => void;
  isFavorited: (cardId: string) => boolean;
  getFavoriteIds: () => string[];
}
```

### Other Stores

```typescript
// API usage tracking
interface ApiUsageState {
  dailyCount: number;
  monthlyCount: number;
  dailyResetDate: string;    // ISO date (YYYY-MM-DD)
  monthlyResetDate: string;  // ISO date (YYYY-MM)

  // Actions
  incrementCount: () => void;
  syncFromApiResponse: (metadata: ApiMetadata) => void;
  isNearDailyLimit: () => boolean;
  isNearMonthlyLimit: () => boolean;
}

// User preferences
interface PreferencesState {
  moversTimeRange: '24h' | '7d' | '30d';
  moversMode: 'my-cards' | 'all-cards';
}
```

### API Response Types

```typescript
// JustTCG API standard wrapper
interface ApiResponse<T> {
  data: T[];
  meta: {
    total: number;
    limit: number;
    offset: number;
    hasMore: boolean;
  };
  _metadata: {
    plan: string;
    requestsToday: number;
    requestsThisMonth: number;
    rateLimit: number;
    rateLimitRemaining: number;
  };
}

// Game object from /games
interface Game {
  id: string;
  name: string;
  cards_count: number;
  variants_count: number;
  sets_count: number;
  last_updated: number;
  game_value_index_cents: number;
  game_value_change_7d_pct: number;
  game_value_change_30d_pct: number;
  game_value_change_90d_pct: number;
}

// Set object from /sets
interface CardSet {
  id: string;
  name: string;
  game_id: string;
  game: string;
  cards_count: number;
  variants_count: number;
  release_date: string;
  set_value_usd: number;
  set_value_change_7d_pct: number;
  set_value_change_30d_pct: number;
  set_value_change_90d_pct: number;
}
```

### How Favorite Status Flows Through the App

```
Any card list (search results, browse, movers, favorites view):

1. Render CardListItem with Card data from TanStack Query
2. Check favoritesStore.favorites[card.id] to determine heart state
3. On heart tap → toggle favoritesStore.addFavorite / removeFavorite
4. Home screen idle state: filter TanStack Query cache to only
   cards where favoritesStore.favorites[card.id] exists

Card Detail screen:
1. Receives card.id via route params
2. Fetches full card data via TanStack Query (useCard hook)
3. Checks favoritesStore for heart state
4. Same toggle behavior
```

---

## Authentication

### MVP: User-Provided API Key

Users provide their own JustTCG API key. This means:
- Each user gets their own rate limits (no shared quota)
- No backend infrastructure required
- Zero cost to operate

### API Key Flow

```
First Launch:
1. App opens → no API key detected
2. Show onboarding/welcome screen explaining:
   - "Card Ticker uses JustTCG for price data"
   - "Sign up for a free API key at justtcg.com"
   - Link to JustTCG sign-up page
3. User pastes API key into text field
4. App validates key with a test request (GET /games)
5. On success → key saved to expo-secure-store → navigate to Home
6. On failure → show error, let user retry

Subsequent Launches:
1. App opens → key found in secure storage
2. Load key into API client → proceed to Home

Key Management (Settings):
- View masked key (last 6 chars visible)
- Edit key (re-enter flow)
- Remove key (clears secure storage, returns to onboarding)
```

### Storage

The API key is stored using `expo-secure-store`, which provides:
- **iOS**: Keychain Services (encrypted)
- **Android**: SharedPreferences encrypted with Android Keystore

The key is NEVER stored in:
- AsyncStorage (unencrypted)
- Source code or environment files bundled with the app
- TanStack Query cache

### Future: Backend Proxy (Post-MVP)

When user accounts are added, the architecture shifts to:
1. User authenticates with the app's own auth system
2. App sends requests to a backend proxy
3. Proxy injects the JustTCG API key server-side
4. User never sees or manages an API key

This is out of scope for MVP but the API client (`services/api.ts`) should be designed so swapping from direct API calls to a proxy is a single base URL change.

---

## Data Persistence Architecture

### Overview

| Data | Storage Layer | Persists Across Restarts | Encrypted |
|---|---|---|---|
| API key | expo-secure-store | Yes | Yes |
| Favorites metadata | Zustand + AsyncStorage | Yes | No |
| API usage counters | Zustand + AsyncStorage | Yes | No |
| User preferences | Zustand + AsyncStorage | Yes | No |
| Card/price data cache | TanStack Query + AsyncStorage persister | Yes | No |
| Games/sets metadata cache | TanStack Query + AsyncStorage persister | Yes | No |

### Layer 1: Secure Storage (expo-secure-store)

Used exclusively for the API key. Encrypted at rest by the OS.

```typescript
import * as SecureStore from 'expo-secure-store';

// Store
await SecureStore.setItemAsync('justtcg_api_key', apiKey);

// Retrieve
const apiKey = await SecureStore.getItemAsync('justtcg_api_key');

// Delete
await SecureStore.deleteItemAsync('justtcg_api_key');
```

### Layer 2: Zustand + AsyncStorage

Used for small, structured app state that must persist. Zustand's `persist` middleware handles serialization automatically.

**What's stored:**
- `favoritesStore`: Record of favorited card IDs, tracked variant, timestamp
- `apiUsageStore`: Daily/monthly request counts, reset dates
- `preferencesStore`: Movers time range, movers mode

**Estimated size:** < 50KB total (even with 100+ favorites)

### Layer 3: TanStack Query + AsyncStorage Persister

Used for API response caching. The `@tanstack/query-async-storage-persister` package serializes the entire query cache to AsyncStorage, so it survives app restarts.

```typescript
import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';
import AsyncStorage from '@react-native-async-storage/async-storage';

const persister = createAsyncStoragePersister({
  storage: AsyncStorage,
  key: 'card-ticker-query-cache',
});
```

**Benefits:**
- App opens with cached data instantly — no loading spinner, no wasted API call
- Stale-while-revalidate: shows cached data immediately, refreshes in background if stale
- Respects stale times (games/sets: 24h, cards: 1h) — won't re-fetch if cache is fresh

**Cache size considerations:**
- AsyncStorage has a ~6MB limit on Android
- Estimated cache size for 200 cached cards with variants: ~500KB
- Games + sets metadata: ~50KB
- Well within limits for MVP

### Future: SQLite Migration (Post-MVP)

When portfolio features are added (quantity owned, cost basis, transaction history), the app will need:
- Structured queries ("total portfolio value", "all cards in set X that I own")
- Larger dataset support
- Relational data (cards ↔ transactions)

At that point, migrate favorites and portfolio data to `expo-sqlite`. TanStack Query cache can remain in AsyncStorage since it's ephemeral by nature.

---

## API Integration Map

### Base Configuration

```
Base URL: https://api.justtcg.com/v1
Auth Header: x-api-key: <key from expo-secure-store>
```

The API key is loaded from `expo-secure-store` at app launch and injected into the API client. It is never hardcoded or bundled with the app.

### Endpoint Usage

| App Feature | Endpoint | Method | Key Params | Cache TTL |
|---|---|---|---|---|
| Game list | `/games` | GET | — | 24 hours |
| Set list (per game) | `/sets` | GET | `game` | 24 hours |
| Card search | `/cards` | GET | `q`, `game`, `limit`, `offset` | 1 hour |
| Browse set cards | `/cards` | GET | `set`, `game`, `limit`, `offset` | 1 hour |
| Card detail | `/cards` | GET | `cardId`, `include_price_history=true` | 1 hour |
| Favorites refresh | `/cards` | POST | Array of `cardId` objects (batch) | 1 hour |
| Movers (gainers) | `/cards` | GET | `game`, `orderBy`, `order=desc`, `limit=5` | 1 hour |
| Movers (losers) | `/cards` | GET | `game`, `orderBy`, `order=asc`, `limit=5` | 1 hour |

### Request Flow: Favorites Refresh

```
1. Get all cardIds from favoritesStore
2. Chunk into batches of 20 (free tier POST limit)
3. POST /cards with batch payload
4. TanStack Query caches the refreshed card data
5. Increment API usage counter by number of batches sent
```

### Request Flow: Search

```
1. User types query → debounce 500ms
2. GET /cards?q={query}&game={selectedGame}&limit=20&offset=0
3. Display results, replacing favorites view
4. On scroll to bottom → GET next page (offset += 20)
5. Increment API usage counter
```

---

## Component Hierarchy

```
App
├── ApiKeyGate (checks secure storage)
│   ├── No key → OnboardingScreen (welcome + API key entry)
│   └── Has key → TabNavigator
├── TabNavigator
│   ├── HomeStack
│   │   ├── HomeScreen
│   │   │   ├── SearchBar (React Native Paper Searchbar)
│   │   │   ├── GameChips (horizontal chip row)
│   │   │   ├── SetPicker (dropdown, conditional)
│   │   │   └── CardList (FlatList)
│   │   │       └── CardListItem (repeating)
│   │   │           ├── CardThumbnail
│   │   │           ├── CardInfo (name, set, rarity)
│   │   │           ├── PriceDisplay (price + change)
│   │   │           └── FavoriteButton (heart toggle)
│   │   └── CardDetailScreen
│   │       ├── CardImage
│   │       ├── CardMetadata
│   │       ├── PriceOverview
│   │       │   ├── CurrentPrice
│   │       │   └── PriceChangeBadges (24h, 7d, 30d)
│   │       ├── PriceChart (sparkline)
│   │       ├── AllTimeStats
│   │       ├── VariantsTable
│   │       └── FavoriteButton (header)
│   ├── MoversStack
│   │   ├── MoversScreen
│   │   │   ├── ModeToggle (My Cards / All Cards)
│   │   │   ├── GameChips
│   │   │   ├── TimeRangeSelector (24h, 7d, 30d)
│   │   │   ├── GainersSection
│   │   │   │   └── CardListItem (x5)
│   │   │   └── LosersSection
│   │   │       └── CardListItem (x5)
│   │   └── CardDetailScreen (shared)
│   └── SettingsScreen
│       ├── ApiKeyCard (masked key, edit/remove)
│       ├── ApiUsageCard
│       ├── CacheManagementCard
│       └── AboutCard
└── Providers
    ├── QueryClientProvider (TanStack Query)
    └── PaperProvider (React Native Paper theme)
```

### Shared Components

| Component | Used In | Purpose |
|---|---|---|
| `CardListItem` | Home, Movers | Consistent card display across screens |
| `CardDetailScreen` | HomeStack, MoversStack | Same detail view pushed from either tab |
| `GameChips` | Home, Movers | Reusable game filter chips |
| `FavoriteButton` | CardListItem, CardDetail | Heart toggle, reads/writes Zustand favorites store |
| `PriceChangeBadge` | CardListItem, CardDetail, Movers | Color-coded percentage change pill |

---

## Caching Strategy

### TanStack Query Cache

| Query Key Pattern | Stale Time | Cache Time | Notes |
|---|---|---|---|
| `['games']` | 24 hours | 48 hours | Rarely changes |
| `['sets', gameId]` | 24 hours | 48 hours | Rarely changes |
| `['cards', 'search', query, game]` | 1 hour | 2 hours | Search results |
| `['cards', 'set', setId]` | 1 hour | 2 hours | Browse results |
| `['card', cardId]` | 1 hour | 2 hours | Card detail |
| `['movers', game, timeRange, order]` | 1 hour | 2 hours | Movers lists |

### Zustand Persistence (AsyncStorage)

- Favorites metadata persisted on every change (lightweight — just IDs and timestamps)
- API usage counters persisted and reset on date boundaries
- User preferences persisted

### Offline Behavior

- Favorites view displays last-known card data from TanStack Query's cache (stale data is better than no data)
- Search and browse show error state with retry button
- Movers "My Cards" mode works fully offline (sorts cached data)

---

## Free Tier Budget Management

### Limits

| Metric | Free Tier | Starter (future) |
|---|---|---|
| Monthly | 1,000 | 10,000 |
| Daily | 100 | 1,000 |
| Per Minute | 10 | 50 |
| POST batch size | 20 | 100 |

### Daily Budget Estimate

| Action | Requests | Frequency |
|---|---|---|
| Favorites refresh (≤20 cards) | 1 | Per app open |
| Search queries | ~3 | Per session |
| Card detail views | ~5 | Per session |
| All Cards movers (2 games) | 4 | Per visit to Movers tab |
| Set/game metadata | 0 | Cached 24h, ~1/day |
| **Typical daily total** | **~13** | Well under 100 |

### Mitigation Strategies

1. **Debounced search**: 500ms delay prevents request-per-keystroke
2. **Batch refresh**: Single POST for up to 20 favorited cards
3. **Aggressive caching**: TanStack Query prevents redundant fetches within stale windows
4. **Local-first movers**: "My Cards" mode costs zero requests
5. **Usage tracking**: Counter in Zustand store, warning UI at 80% daily/monthly threshold
6. **Rate limit header tracking**: Read `_metadata.rateLimitRemaining` from every response, pause requests if near zero

---

## Project Structure (Expo Router)

```
card-ticker/
├── app/
│   ├── _layout.tsx              # Root layout (providers + API key gate)
│   ├── onboarding.tsx           # Welcome screen + API key entry
│   ├── (tabs)/
│   │   ├── _layout.tsx          # Tab navigator
│   │   ├── index.tsx            # Home screen
│   │   ├── movers.tsx           # Movers screen
│   │   └── settings.tsx         # Settings screen
│   └── card/
│       └── [id].tsx             # Card detail screen (shared)
├── components/
│   ├── ApiKeyCard.tsx           # Settings: masked key display + edit/remove
│   ├── CardListItem.tsx
│   ├── CardThumbnail.tsx
│   ├── FavoriteButton.tsx
│   ├── GameChips.tsx
│   ├── PriceChangeBadge.tsx
│   ├── PriceChart.tsx
│   ├── SearchBar.tsx
│   ├── SetPicker.tsx
│   ├── SortPicker.tsx           # Sort dropdown for search/browse results
│   └── VariantsTable.tsx
├── hooks/
│   ├── useApiKey.ts             # Secure store read/write/validate for API key
│   ├── useCards.ts              # TanStack Query hooks for /cards
│   ├── useGames.ts              # TanStack Query hooks for /games
│   ├── useSets.ts               # TanStack Query hooks for /sets
│   └── useFavorites.ts          # Zustand favorites actions + queries
├── services/
│   ├── api.ts                   # JustTCG API client (base config, auth from secure store)
│   ├── secureStore.ts           # expo-secure-store wrapper for API key
│   └── queryClient.ts           # TanStack Query client + AsyncStorage persister config
├── store/
│   ├── favoritesStore.ts        # Zustand favorites store
│   ├── apiUsageStore.ts         # Request counter tracking
│   └── preferencesStore.ts      # User preferences
├── types/
│   └── index.ts                 # TypeScript interfaces (from Data Models section)
├── constants/
│   └── games.ts                 # Supported game IDs and display names
├── app.json                     # Expo config
├── tsconfig.json
└── package.json
```

---

## Open Questions

1. **Card images**: Does JustTCG return image URLs in the card response? If not, we may need a secondary source (e.g., PokemonTCG API for Pokemon images, or SWU card image CDN). This needs to be verified with a live API call.
2. **Star Wars: Unlimited game ID**: The exact `game` parameter value needs to be confirmed by calling `/games`. It is likely `star-wars-unlimited` but must be verified.
3. **Favorites over 20 cards**: With free tier's 20-card POST batch limit, favorites lists over 20 cards require multiple batch requests to refresh. Should we cap favorites at 20 for free tier, or allow larger lists with multi-batch refresh?
4. **Price history chart library**: Need to evaluate `react-native-gifted-charts` vs `victory-native` for bundle size and ease of use with the sparkline format.

---

## MVP Milestone Checklist

- [ ] Project scaffolding (Expo + dependencies)
- [ ] API client + secure store for API key
- [ ] Onboarding screen (API key entry + validation)
- [ ] Type definitions
- [ ] Zustand stores (favorites, API usage, preferences)
- [ ] TanStack Query hooks + AsyncStorage persister
- [ ] Home screen (favorites state)
- [ ] Home screen (search state)
- [ ] Home screen (browse state)
- [ ] Sort functionality (price, rarity, card number, price movement)
- [ ] Card Detail screen
- [ ] Favorite/unfavorite flow
- [ ] Movers screen (My Cards mode)
- [ ] Movers screen (All Cards mode)
- [ ] Settings screen (API key management, usage, cache)
- [ ] Caching layer (TanStack Query persisted to AsyncStorage)
- [ ] API budget tracking + warnings
- [ ] Polish + testing
