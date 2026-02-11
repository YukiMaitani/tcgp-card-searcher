# Pokemon Card Search App - Implementation Plan

## Context

ポケモンカード検索アプリを Next.js で新規作成する。カードデータ(~2500枚、月100枚追加)はアプリ内にバンドルし、検索・フィルタリングをクライアントサイドで行う。自然言語によるAI検索(LLM構造化フィルタ + Embeddingランキング + RRF統合)をサポートし、APIキーはユーザーが自分で入力する。英語/日本語の切り替えに対応。

GitHub Pagesで静的サイトとしてホスティングする（サーバーサイド処理なし）。

agent-skills のスキル(`react-best-practices`, `composition-patterns`, `web-design-guidelines`)を活用する。

---

## Architecture

### 静的ホスティング

Next.js の `output: 'export'` で静的HTMLを生成し、GitHub Pages でホスティングする。サーバーサイド処理は一切行わない。

```typescript
// next.config.ts
const nextConfig = {
  output: 'export',
  images: { unoptimized: true },
};
```

### API呼び出し方式

サーバーがないため、ブラウザからLLM/Embedding APIを直接呼び出す。ユーザーが自分のAPIキーを入力する方式のため、キーの露出はユーザー自身の管理下にある。

- **Anthropic API (Claude)**: `anthropic-dangerous-direct-browser-access` ヘッダーを付与してブラウザから直接呼び出し
- **OpenAI API (Embedding)**: `Authorization` ヘッダーでブラウザから直接呼び出し

APIキーは localStorage に保存し、サーバーには一切送信しない。

### Embeddingの事前計算

カードのテキスト情報（カード名、ワザ名、ワザ効果、フレーバーテキスト、画像説明）を結合してEmbeddingを事前計算する。言語別に `embeddings-ja.json` / `embeddings-en.json` として `public/data/` に配置し、AI検索時に遅延読み込みする。

HP、タイプ、にげるコスト等の数値・カテゴリデータはEmbedding対象外とする（構造化フィルタが担当）。

---

## Project Structure

```
pokemon-card-search/
├── scripts/                            # ビルド時のみ使用
│   ├── generate-card-data.ts           # カードデータ生成（日本語訳含む）
│   ├── generate-visual-tags.ts         # マルチモーダルLLMで画像タグ付け
│   └── generate-embeddings.ts          # Embedding事前計算（言語別）
├── public/
│   └── data/
│       ├── embeddings-ja.json          # 日本語Embedding（遅延読み込み）
│       └── embeddings-en.json          # 英語Embedding（遅延読み込み）
├── data/
│   ├── cards.ts                        # PokemonCard[] (~2500件)
│   ├── cards-index.ts                  # Map<cardId, PokemonCard>
│   ├── types.ts                        # Set<string> (タイプ一覧)
│   ├── rarities.ts                     # Set<string> (レアリティ一覧)
│   └── sets.ts                         # Set<string> (シリーズ一覧)
├── lib/
│   ├── types/
│   │   ├── card.ts                     # PokemonCard, LocalizedText, Attack
│   │   ├── search.ts                   # CardSearchState/Actions, ParsedQuery
│   │   └── settings.ts                 # SettingsState/Actions/Meta
│   ├── i18n/
│   │   ├── translations.ts             # { en: {...}, ja: {...} }
│   │   ├── t.ts                        # t(key, locale, vars?)
│   │   └── use-locale.ts              # useLocale() hook
│   ├── search/
│   │   ├── filter-cards.ts             # 構造化フィルタ（単一ループ）
│   │   ├── cosine-similarity.ts        # コサイン類似度
│   │   ├── rank-by-embedding.ts        # Embedding類似度ランキング
│   │   ├── keyword-search.ts           # キーワード部分一致検索
│   │   └── rrf-fusion.ts              # RRFランキング統合
│   ├── ai/
│   │   ├── parse-query.ts              # LLMフィルタ抽出（ブラウザ→Anthropic API直接）
│   │   ├── parse-query-prompt.ts       # フィルタ抽出プロンプト定義
│   │   ├── embed-query.ts              # Embedding取得（ブラウザ→OpenAI API直接）
│   │   ├── generate-answer.ts          # LLM回答生成（ブラウザ→Anthropic API直接）
│   │   └── load-embeddings.ts          # Embeddingファイル遅延読み込み
│   └── storage/
│       └── local-storage.ts            # バージョン付きlocalStorage
├── app/
│   ├── layout.tsx                      # SettingsProvider でラップ
│   ├── page.tsx                        # CardSearch.Provider + ページ構成
│   └── globals.css                     # Tailwind + content-visibility
│   (api/ ディレクトリなし — サーバーサイド処理なし)
├── components/
│   ├── card-search/                    # 検索Compound Component
│   │   ├── card-search-context.ts
│   │   ├── card-search-provider.tsx
│   │   ├── card-search-input.tsx
│   │   ├── card-search-filters.tsx
│   │   ├── card-search-results.tsx
│   │   ├── card-search-ai-panel.tsx    # lazy loaded
│   │   ├── card-search-sort.tsx
│   │   └── card-search-status.tsx
│   ├── card-display/                   # カード表示Compound Component
│   │   ├── card-grid-view.tsx          # 明示的バリアント: グリッド
│   │   ├── card-list-view.tsx          # 明示的バリアント: リスト
│   │   ├── card-item.tsx              # memo化
│   │   └── card-detail-modal.tsx       # lazy loaded
│   ├── settings/                       # 設定Compound Component
│   │   ├── settings-context.ts
│   │   ├── settings-provider.tsx
│   │   ├── settings-language-toggle.tsx
│   │   ├── settings-api-keys.tsx       # Anthropic/OpenAI キー入力
│   │   └── settings-modal.tsx          # lazy loaded
│   └── ui/                             # 共通UIプリミティブ
│       ├── button.tsx, input.tsx, modal.tsx, etc.
└── next.config.ts                      # output: 'export'
```

---

## Data Model

```typescript
interface LocalizedText {
  en: string
  ja: string
}

interface Attack {
  name: LocalizedText
  cost: string[]
  damage: string
  description: LocalizedText
}

interface PokemonCard {
  id: string
  name: LocalizedText
  types: string[]
  hp: number
  stage: string
  attacks: Attack[]
  ability: { name: LocalizedText; text: LocalizedText } | null
  weakness: { type: string; value: string } | null
  resistance: { type: string; value: string } | null
  retreatCost: number
  rarity: string
  set: string
  imageUrl: string
  flavorText: LocalizedText
  // 画像タグ（事前にマルチモーダルLLMで生成）
  visualTags: string[]              // ["可愛い", "かっこいい", "幻想的", ...]
  visualDescription: LocalizedText  // イラストの説明
  artStyle: string                  // "ポップ", "リアル", "水彩", ...
  mood: string                      // "明るい", "暗い", "神秘的", ...
}

// AI検索のフィルタ抽出結果
interface ParsedQuery {
  filters: {
    name?: string
    category?: 'Pokemon' | 'Trainer' | 'Energy'
    stage?: 'Basic' | 'Stage1' | 'Stage2'
    types?: string[]
    hp_gte?: number
    hp_lte?: number
    attack_damage_gte?: number
    attack_damage_lte?: number
    retreat_lte?: number
    weakness_type?: string
    has_ability?: boolean
    rarity?: string
    set?: string
  }
  semantic_query: string | null
}

// 検索状態
interface CardSearchState {
  query: string
  filters: ParsedQuery['filters']
  semanticQuery: string | null
  filteredCards: PokemonCard[]
  rankedCards: PokemonCard[]
  aiAnswer: string | null
  isLoading: boolean
  searchMode: 'manual' | 'ai'
}
```

---

## AI検索フロー

### 概要

ユーザーの自然言語クエリを「構造化フィルタ」と「セマンティッククエリ」に分解し、それぞれ最適な検索手法で処理する。構造化フィルタは数値・カテゴリの正確な絞り込みを、セマンティッククエリはベクトル検索とキーワード検索の組み合わせ(RRF)で意味的なあいまい検索を担当する。

### フロー図

```
ユーザー: 「草タイプのポケモンで可愛くて強いやつ」
    │
    ▼
━━━ Step 1: フィルタ抽出 (Anthropic API) ━━━━━━━━━━━━━━
    │
    │  ブラウザ → Anthropic API (Claude Haiku) 直接呼び出し
    │  システムプロンプト + few-shot で構造化フィルタを抽出
    │
    │  出力:
    │  {
    │    "filters": {
    │      "types": ["Grass"],
    │      "category": "Pokemon",
    │      "hp_gte": 200
    │    },
    │    "semantic_query": "可愛い 強い 高火力"
    │  }
    │
    ▼
━━━ Step 2: 構造化フィルタリング (クライアント JS) ━━━━━━
    │
    │  cards データをメモリ上でフィルタ（LLM不要、高速）
    │  2500枚 → 草タイプ & Pokemon & HP200以上 → 約50枚に絞る
    │
    ▼
━━━ Step 3: セマンティック検索 ━━━━━━━━━━━━━━━━━━━━━
    │  （semantic_query が null の場合はスキップ → Step 5へ）
    │
    │  3a. semantic_query をEmbedding化
    │      ブラウザ → OpenAI API 直接呼び出し
    │
    │  3b. Embeddingファイルを遅延読み込み
    │      fetch('/data/embeddings-{locale}.json')
    │      ※初回のみ読み込み、以降はキャッシュ
    │
    │  3c. Step 2 の候補に対して複数手法でランキング
    │      手法A: コサイン類似度（ベクトル検索）→ ランキングA
    │      手法B: キーワード部分一致検索 → ランキングB
    │
    │  3d. RRF（Reciprocal Rank Fusion）でランキング統合
    │      score = Σ 1/(k + rank_i)  (k=60)
    │      → 最終ランキング上位N件
    │
    ▼
━━━ Step 4: 回答生成 (Anthropic API) ━━━━━━━━━━━━━━━━
    │
    │  ブラウザ → Anthropic API (Claude Haiku) 直接呼び出し
    │  ストリーミングレスポンスで逐次表示
    │  上位N件のカード情報 + 元の質問 → 自然言語で回答
    │
    ▼
━━━ Step 5: 結果表示 ━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  AI回答テキスト + カード一覧（画像付き）を表示
    │  semantic_query が null の場合: フィルタ結果をそのまま一覧表示
```

### 各手法の役割分担

| 担当 | 得意なこと | 例 |
|------|-----------|-----|
| 構造化フィルタ | 数値比較、完全一致 | HP≥200, type=Grass, retreat≤1 |
| ベクトル検索 | 意味的なあいまい検索 | 「守る」→「ダメージを防ぐ」にヒット |
| キーワード検索 | 特定の単語を含むか | 「ベンチ」を含むテキストにヒット |
| RRF | 上記の統合 | 複数手法で上位のカードを信頼 |

### レスポンス時間見積もり

```
Step 1: フィルタ抽出 (Haiku)      0.5〜1.0秒
Step 2: フィルタリング (JS)        ~5ms
Step 3a: Embedding (OpenAI)        0.3〜0.5秒
Step 3b: Embedding読み込み          初回のみ ~1秒、以降キャッシュ
Step 3c: 類似度計算 (JS)           ~5ms
Step 3d: RRF統合 (JS)              ~1ms
Step 4: 回答生成 (Haiku streaming) 最初の文字まで ~1.0秒
────────────────────────────────────
合計: 約 2.0〜3.5秒（ストリーミングで体感は ~2秒）
```

---

## フィルタ抽出プロンプト

```typescript
// lib/ai/parse-query-prompt.ts

export const FILTER_EXTRACTION_PROMPT = `
あなたはポケモンカードの検索アシスタントです。
ユーザーの質問から、検索フィルタとセマンティック検索クエリを抽出してください。

## 出力形式
必ず以下のJSON形式のみを出力してください。
該当しないフィールドは含めないでください。
JSONのみ出力し、それ以外のテキストは一切含めないでください。

{
  "filters": {
    "name": "カード名の部分一致",
    "category": "Pokemon | Trainer | Energy",
    "stage": "Basic | Stage1 | Stage2",
    "types": ["Grass","Fire","Water","Lightning","Psychic","Fighting",
              "Darkness","Metal","Dragon","Colorless"],
    "hp_gte": 数値,
    "hp_lte": 数値,
    "attack_damage_gte": 数値,
    "attack_damage_lte": 数値,
    "retreat_lte": 数値,
    "weakness_type": "弱点タイプ",
    "has_ability": true/false,
    "rarity": "レアリティ",
    "set": "シリーズ名"
  },
  "semantic_query": "フィルタで表現できない意味的な検索キーワード。不要ならnull"
}

## タイプの対応表
草=Grass, 炎/火=Fire, 水=Water, 雷=Lightning, 超=Psychic,
闘=Fighting, 悪=Darkness, 鋼=Metal, 竜/ドラゴン=Dragon, 無=Colorless

## 解釈ルール
- 「強い」→ hp_gte: 200 + semantic_query に「強い 高火力」を追加
- 「軽い」「速い」→ retreat_lte: 1 + semantic_query に「低コスト」を追加
- 「重い」「耐久」→ hp_gte: 200
- 「可愛い」「かっこいい」等の見た目 → semantic_query に含める

## 例

ユーザー: 草タイプでHP200以上のポケモン
出力: {"filters":{"types":["Grass"],"hp_gte":200,"category":"Pokemon"},"semantic_query":null}

ユーザー: ベンチを守れるカード
出力: {"filters":{"category":"Pokemon"},"semantic_query":"ベンチポケモンを守る ダメージを防ぐ"}

ユーザー: 逃げエネ1以下の炎ポケモン
出力: {"filters":{"types":["Fire"],"category":"Pokemon","retreat_lte":1},"semantic_query":null}

ユーザー: 草タイプのポケモンで可愛くて強いやつ
出力: {"filters":{"types":["Grass"],"category":"Pokemon","hp_gte":200},"semantic_query":"可愛い 強い 高火力"}

ユーザー: エネ加速できるサポート
出力: {"filters":{"category":"Trainer"},"semantic_query":"エネルギー加速 エネルギーをつける"}

ユーザー: トロピウス
出力: {"filters":{"name":"トロピウス"},"semantic_query":null}

ユーザー: ダメージ100以上の水タイプで可愛いやつ
出力: {"filters":{"types":["Water"],"attack_damage_gte":100},"semantic_query":"可愛い"}

ユーザー: ドラゴンタイプの2進化ポケモン
出力: {"filters":{"types":["Dragon"],"category":"Pokemon","stage":"Stage2"},"semantic_query":null}

ユーザー: 相手のワザを封じるポケモン
出力: {"filters":{"category":"Pokemon"},"semantic_query":"ワザを封じる ワザを使えなくする ロック"}
`;
```

---

## Embedding事前計算

### ベクトル化対象テキスト

カード名（日英）、ワザ名、ワザ効果テキスト、特性テキスト、フレーバーテキスト、画像説明（visualDescription）を結合する。HP、タイプ等の数値・カテゴリデータは含めない。

```typescript
// 日本語用テキスト組み立て
function buildEmbeddingText_ja(card: PokemonCard): string {
  return [
    card.name.ja,
    card.name.en,
    card.ability ? `特性:${card.ability.name.ja} ${card.ability.text.ja}` : '',
    ...card.attacks.map(a => `ワザ:${a.name.ja} ${a.description.ja}`),
    card.flavorText.ja,
    card.visualDescription?.ja ?? '',
  ].filter(Boolean).join(' ');
}

// 英語用テキスト組み立て
function buildEmbeddingText_en(card: PokemonCard): string {
  return [
    card.name.en,
    card.ability ? `Ability:${card.ability.name.en} ${card.ability.text.en}` : '',
    ...card.attacks.map(a => `Attack:${a.name.en} ${a.description.en}`),
    card.flavorText.en,
    card.visualDescription?.en ?? '',
  ].filter(Boolean).join(' ');
}
```

### Embeddingファイル形式

```json
{
  "card-001": [0.55, 0.23, -0.41, ...],
  "card-002": [0.12, -0.67, 0.33, ...],
  ...
}
```

### サイズ見積もり

- 1ベクトル: 1536次元 × 4バイト(float32)
- 2500枚: 約15MB/言語
- 2言語合計: 約30MB
- GitHub Pages 制限: ファイル100MB以下、リポジトリ1GB以下 → 問題なし

### 遅延読み込み

AI検索を使わないユーザーにはダウンロード不要。

```typescript
// lib/ai/load-embeddings.ts
const cache: Record<string, Record<string, number[]>> = {};

export async function loadEmbeddings(
  locale: 'en' | 'ja'
): Promise<Record<string, number[]>> {
  if (cache[locale]) return cache[locale];

  const res = await fetch(`/data/embeddings-${locale}.json`);
  cache[locale] = await res.json();
  return cache[locale];
}
```

---

## 画像タグ付け（事前バッチ処理）

マルチモーダルLLM（Claude Haiku）で全カード画像を分析し、視覚的な特徴をテキスト化する。「可愛いカード」「かっこいいイラスト」等の検索を可能にする。

### 出力フィールド

| フィールド | 型 | 例 |
|-----------|-----|-----|
| `visualTags` | `string[]` | `["可愛い", "パステルカラー", "笑顔"]` |
| `visualDescription` | `LocalizedText` | イラストの2-3文の描写 |
| `artStyle` | `string` | `"ポップ"`, `"リアル"`, `"水彩"` |
| `mood` | `string` | `"明るい"`, `"神秘的"`, `"激しい"` |

### コスト見積もり

- 2500枚 × ~1200トークン: 約600円（初回のみ）
- 毎月100枚追加: 約25円/月

---

## RRF（Reciprocal Rank Fusion）

ベクトル検索とキーワード検索のランキングを統合する。

```typescript
// lib/search/rrf-fusion.ts
export function rrfFusion(
  rankings: { cardId: string }[][],
  k = 60
): { cardId: string; score: number }[] {
  const scores = new Map<string, number>();

  for (const ranking of rankings) {
    ranking.forEach((item, index) => {
      const rank = index + 1;
      const current = scores.get(item.cardId) ?? 0;
      scores.set(item.cardId, current + 1 / (k + rank));
    });
  }

  return [...scores.entries()]
    .map(([cardId, score]) => ({ cardId, score }))
    .sort((a, b) => b.score - a.score);
}
```

---

## エラーハンドリング

ユーザーがAPIキーを入力する方式のため、以下のエラーを適切にハンドリングする。

| エラー | 対応 |
|--------|------|
| 無効なAPIキー (401) | 設定画面でキーの再入力を促すメッセージ |
| レート制限 (429) | リトライ表示 + 時間をおいて再試行を案内 |
| ネットワークエラー | 「接続を確認してください」メッセージ |
| LLMレスポンスのJSONパース失敗 | フォールバック: 全文をsemantic_queryとして扱う |
| API未設定でAI検索実行 | 設定画面への誘導メッセージ |
| Embeddingファイル読み込み失敗 | フィルタ結果のみ表示（ベクトル検索スキップ） |

### フォールバック方針

AI検索のいずれかのステップが失敗した場合でも、構造化フィルタの結果だけは表示する。ベクトル検索やLLM回答生成が失敗しても、検索自体が完全に失敗することはない。

---

## Skill適用マッピング

### composition-patterns

| ルール | 適用箇所 |
|--------|----------|
| `architecture-compound-components` | CardSearch, CardDisplay, Settings を Compound Component 構造に |
| `state-context-interface` | 各Context を `state/actions/meta` パターンで定義 |
| `state-lift-state` | CardSearchProvider でサイドバーと結果一覧が状態共有 |
| `state-decouple-implementation` | Provider が実装を隠蔽、UIは interface のみ消費 |
| `patterns-explicit-variants` | CardGridView / CardListView を明示的バリアントに |
| `patterns-children-over-render-props` | ページ構成は children で合成 |
| `react19-no-forwardref` | `use(Context)` を使用、forwardRef 不使用 |

### react-best-practices

| ルール | 適用箇所 |
|--------|----------|
| `rendering-content-visibility` | `.card-item { content-visibility: auto; contain-intrinsic-size: 0 320px }` |
| `rerender-derived-state-no-effect` | filteredCards は useMemo で導出、state に保存しない |
| `rerender-memo` | CardItem を `memo()` でラップ |
| `rerender-transitions` | setQuery, toggleFilter を `startTransition` で実行 |
| `bundle-dynamic-imports` | AI検索パネル、設定モーダル、詳細モーダルを `next/dynamic` |
| `js-set-map-lookups` | フィルタ値を `Set<string>` で O(1) 判定 |
| `js-index-maps` | `Map<cardId, PokemonCard>` で O(1) カード取得 |
| `js-combine-iterations` | フィルタを単一 `for...of` ループに統合 |
| `js-early-exit` | フィルタ不一致で即 `continue` |
| `js-tosorted-immutable` | `.toSorted()` でイミュータブルソート |
| `client-localstorage-schema` | `pokemon-search-settings:v1` でバージョン管理 |
| `rerender-lazy-state-init` | SettingsProvider で `useState(() => loadFromStorage())` |

---

## i18n

軽量カスタム実装（2言語のみのためライブラリ不要）:
- `translations.ts`: 静的翻訳マップ `{ en: {...}, ja: {...} }`
- `t(key, locale)` 関数で翻訳取得
- `useLocale()` で SettingsContext から現在の言語を取得
- カードデータは `LocalizedText` 型（`name.en` / `name.ja`）
- フィルタ抽出プロンプトは言語に応じて切り替え
- Embeddingファイルは言語別に読み込み

---

## 事前バッチ処理

アプリのビルド前に実行する3つのスクリプト。開発者のAPIキーで実行する。

### 1. カードデータ生成 (`scripts/generate-card-data.ts`)

- ソースJSONから PokemonCard 型に変換
- 英語のみのフィールドは LLM (Haiku) で日本語訳を生成
- 公式日本語名がある場合はそちらを優先

コスト: 約400円（2500枚、初回のみ）

### 2. 画像タグ付け (`scripts/generate-visual-tags.ts`)

- マルチモーダル LLM (Haiku) で各カード画像を分析
- visualTags, visualDescription, artStyle, mood を生成

コスト: 約600円（2500枚、初回のみ）

### 3. Embedding生成 (`scripts/generate-embeddings.ts`)

- 各カードのテキストフィールドを結合（言語別）
- OpenAI Embedding API でベクトル化
- `embeddings-ja.json`, `embeddings-en.json` を出力

コスト: 約5円（2500枚、初回のみ）

### 毎月の新カード追加

同じスクリプトを新カード分だけ実行し、既存データにマージ。月100枚で約40円。

---

## 実装順序

### Phase 1: 基盤
1. Next.js プロジェクト初期化（TypeScript, Tailwind, App Router, `output: 'export'`）
2. `lib/types/` にデータ型定義（PokemonCard, ParsedQuery, CardSearchState）
3. `data/` にサンプルカードデータ（開発用50-100枚）
4. `lib/storage/local-storage.ts` 作成

### Phase 2: Settings
5. Settings Compound Component（Context, Provider, LanguageToggle, ApiKeys）
6. `lib/i18n/` 翻訳基盤

### Phase 3: 検索コア
7. CardSearch Compound Component（Context, Provider, Input, Filters, Sort, Status）
8. `lib/search/filter-cards.ts` 構造化フィルタロジック

### Phase 4: カード表示
9. CardItem（memo化）、CardGridView、CardListView
10. CardSearchResults
11. `content-visibility` CSS適用

### Phase 5: ページ組み立て
12. `app/layout.tsx` と `app/page.tsx` で全コンポーネント合成
13. 共通UIコンポーネント（`components/ui/`）

### Phase 6: AI検索
14. `lib/ai/parse-query.ts` + `parse-query-prompt.ts`（ブラウザ→Anthropic API直接）
15. `lib/ai/embed-query.ts`（ブラウザ→OpenAI API直接）
16. `lib/ai/generate-answer.ts`（ストリーミング回答生成）
17. `lib/ai/load-embeddings.ts`（遅延読み込み）
18. `lib/search/cosine-similarity.ts`, `keyword-search.ts`, `rrf-fusion.ts`
19. CardSearchAiPanel（lazy loaded）
20. エラーハンドリング・フォールバック実装

### Phase 7: データ＆ビルド
21. カード詳細モーダル（lazy loaded）
22. 全カードデータ投入（~2500枚）
23. `scripts/generate-card-data.ts`（日本語訳バッチ）
24. `scripts/generate-visual-tags.ts`（画像タグ付けバッチ）
25. `scripts/generate-embeddings.ts`（Embedding事前計算）

### Phase 8: レビュー＆デプロイ
26. `web-design-guidelines` スキルでUIレビュー
27. `next build` で静的エクスポート
28. GitHub Pages にデプロイ

---

## Verification

1. **ローカル動作確認**: `npm run dev` で起動し、検索・フィルタ・言語切替が動作すること
2. **パフォーマンス**: Chrome DevTools で 2500枚表示時に `content-visibility` が効いていること
3. **AI検索（フィルタのみ）**: 「草タイプでHP200以上」→ 構造化フィルタのみで正しく絞り込まれること
4. **AI検索（セマンティック）**: 「ベンチを守れるポケモン」→ ベクトル検索+RRFで関連カードが上位に来ること
5. **AI検索（回答生成）**: 検索結果に対してLLMが自然言語で回答を生成すること
6. **AI検索（フォールバック）**: API未設定/エラー時にフィルタ結果のみ表示されること
7. **Embedding遅延読み込み**: AI検索未使用時に embeddings ファイルがダウンロードされないこと
8. **i18n**: EN/JA切替でUI文言、カード名、AI回答言語が切り替わること
9. **静的ビルド**: `npm run build` が成功し、`out/` ディレクトリに静的ファイルが生成されること
10. **デプロイ**: GitHub Pages でアクセスし、全機能が動作すること
