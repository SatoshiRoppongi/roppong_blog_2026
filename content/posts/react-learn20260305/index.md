---
title: "react関連勉強20260305"
date: 2026-03-05T22:59:28+09:00
lastmod: 2026-03-05T22:59:28+09:00
draft: false
description: "2026/3/5に学んだことです"
summary: "2026/3/5に学んだことをまとめます"
tags: ["Next", "react", "cleanArchiecture"]
categories: ["技術"]
cover:
  image: ""
  alt: ""
  relative: true
showToc: true
---

# Next.js × React 実践パターン集 — App Router 時代のアーキテクチャと設計テクニック

Next.js 15 (App Router) + React + Hono を使ったプロダクト開発で実際に使われているパターンやテクニックを、具体的なコード例とともにまとめました。

## 1. Next.js App Router のルーティングパターン

### Dynamic Segments の種類

App Router では、ディレクトリ名の `[]` 記法でルーティングを制御します。

| パターン      | 例                 | マッチするパス     | `params` の値             |
| ------------- | ------------------ | ------------------ | ------------------------- |
| `[id]`        | `/users/[id]`      | `/users/123`       | `{ id: "123" }`           |
| `[...slug]`   | `/docs/[...slug]`  | `/docs/a/b/c`      | `{ slug: ["a","b","c"] }` |
| `[[...slug]]` | `/api/[[...slug]]` | `/api`, `/api/a/b` | `undefined`, `["a","b"]`  |

### `[...slug]` vs `[[...slug]]` — Catch-all と Optional Catch-all

名前の通り、`[[...slug]]`（二重括弧）は**ルート自体にもマッチする**点が異なります。

```
# [...]  → Catch-all（1セグメント以上が必須）
/api/chat/messages   ✅ → slug = ["messages"]
/api/chat/a/b/c      ✅ → slug = ["a", "b", "c"]
/api/chat             ❌ → マッチしない

# [[...]] → Optional Catch-all（0セグメントでもOK）
/api/chat/messages   ✅ → slug = ["messages"]
/api/chat/a/b/c      ✅ → slug = ["a", "b", "c"]
/api/chat             ✅ → slug = undefined
```

### `...`（スプレッド構文）の意味

`...` は JavaScript のスプレッド/レスト構文 (`...args`) からの借用です。単なる慣習ではなく、**Next.js の構文として「複数のパスセグメントを配列で受け取る」**という意味を持ちます。`...` がなければ1セグメントしかマッチしません。

### Route Groups: `(authenticated)` パターン

括弧 `()` で囲んだディレクトリは **Route Group** と呼ばれ、URL パスには影響しません。

```
app/api/
├── (authenticated)/    # URL に反映されない。認証 middleware を適用するグループ
│   ├── users/
│   └── orders/
└── health/             # 認証不要
```

`/api/users` と `/api/orders` には認証 middleware が適用され、`/api/health` には適用されない、という整理が可能です。

---

## 2. Hono を Next.js に統合するブリッジパターン

### なぜ Hono を使うのか

Next.js の Route Handler だけでもAPIは作れますが、Hono を導入すると以下の利点があります。

- ミドルウェアの柔軟な組み合わせ
- バリデーション（Zod 連携）の統一的な記法
- Next.js に依存しないルーティングロジック（テスタビリティ向上）

### 実装パターン

```ts
// app/api/(authenticated)/chat/[[...route]]/route.ts
import app from "@/features/chat/api/hono/chat.router";
import { handle } from "hono/vercel";

export const GET = handle(app);
export const POST = handle(app);
```

**`[[...route]]`（Optional Catch-all）が必須**な理由は、Hono 側で定義した全てのサブルートをキャッチするためです。

```
/api/chat              → Hono が "/" として処理
/api/chat/messages     → Hono が "/messages" として処理
/api/chat/history/123  → Hono が "/history/123" として処理
```

Next.js 側の `route.ts` は「薄いアダプター」に徹し、ビジネスロジックは一切持ちません。実質的な責務は以下の2つだけです。

1. **Next.js ↔ Hono のブリッジ**: `handle(app)` でリクエストを委譲
2. **HTTP メソッドの宣言**: `export const GET/POST/PUT/DELETE`

---

## 3. API Route からの型 re-export — フロントとバックの契約

### パターン

```ts
// app/api/(authenticated)/chat/[[...route]]/route.ts
import app from "@/features/chat/api/hono/chat.router";
import { handle } from "hono/vercel";

// 型の re-export
export type {
  ConversationResponse,
  MessageResponse,
} from "@/features/chat/common/types";

export const GET = handle(app);
export const POST = handle(app);
```

```ts
// フロントエンド側（hooks）
import type { MessageResponse } from "@/app/api/(authenticated)/chat/[[...route]]/route";
```

### なぜ直接 features から import しないのか

型の定義元は `features/chat/common/types.ts` ですが、API route ファイルを経由してインポートすることで、**「この型は API のレスポンス仕様（契約）である」**ということを明示できます。

ただし、`[[...route]]` を含むパスをインポートに書くと可読性が下がるという面もあり、チームのルール次第です。

---

## 4. バレルファイル — 公開APIの明示

### バレルファイルとは

ディレクトリ内のモジュールをまとめて re-export する `index.ts` のことです。

```ts
// components/Chat/index.ts
export { ChatProvider } from "./ChatProvider";
export { FloatingButton } from "./FloatingButton";
export { ChatPanel } from "./ChatPanel";
```

### 効果

**1. インポートの簡潔化**

```ts
// Before
import { ChatProvider } from "@/components/Chat/ChatProvider";
import { FloatingButton } from "@/components/Chat/FloatingButton";
import { ChatPanel } from "@/components/Chat/ChatPanel";

// After
import { ChatProvider, FloatingButton, ChatPanel } from "@/components/Chat";
```

**2. 公開 / 非公開の境界を明示**

バレルファイルに含まれないコンポーネント（例: `MessageBubble`, `InputArea`）は「内部実装」であることが分かります。外部からは `index.ts` に export されたものだけを使うべき、という設計意図が伝わります。

---

## 5. React の Context + Provider パターン

### 問題: Props のバケツリレー

```
App
└── Layout
    └── Panel
        └── Header   ← isOpen, onClose が必要
        └── Body
            └── List ← conversationId が必要
```

すべての中間コンポーネントに props を渡し続けるのは煩雑です。

### 解決: Context API

```tsx
// ChatProvider.tsx
"use client";

import { createContext, useContext, useState, useCallback, ReactNode } from "react";

type ChatContextValue = {
  isOpen: boolean;
  conversationId: string | null;
  open: () => void;
  close: () => void;
  setConversationId: (id: string | null) => void;
};

const ChatContext = createContext<ChatContextValue | null>(null);

// カスタムフック（ガード付き）
export function useChat() {
  const context = useContext(ChatContext);
  if (!context) {
    throw new Error("useChat must be used within ChatProvider");
  }
  return context;
}

export function ChatProvider({ children }: { children: ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);
  const [conversationId, setConversationId] = useState<string | null>(null);

  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);

  return (
    <ChatContext.Provider value={{ isOpen, conversationId, open, close, setConversationId }}>
      {children}
    </ChatContext.Provider>
  );
}
```

### ポイント解説

**`"use client"` ディレクティブ**

Next.js App Router ではコンポーネントはデフォルトで Server Component です。`useState` や `useContext` などの React hooks を使うコンポーネントは Client Component として `"use client"` を宣言する必要があります。

**ガード付きカスタムフック**

```ts
if (!context) throw new Error("useChat must be used within ChatProvider");
```

Provider の外で誤ってフックを使った場合に、原因が分かりやすいエラーメッセージを出す定番パターンです。`null` チェックにより、戻り値の型から `null` が除外され、利用側で非 null アサーションが不要になる利点もあります。

**`useCallback` によるメモ化**

```ts
const open = useCallback(() => setIsOpen(true), []);
```

Provider が再レンダリングしても関数の参照が変わらないようにします。これにより、`open` を受け取る子コンポーネントが不要に再レンダリングされることを防ぎます。依存配列 `[]` は「この関数は初回のみ生成し、以後は同じ参照を返す」ことを意味します。

---

## 6. React コンポーネント設計の基本パターン

### プレゼンテーショナルコンポーネント

```tsx
"use client";

import { useChat } from "./ChatProvider";

export function ChatHeader() {
  const { close, startNewConversation, toggleHistory, showHistory } = useChat();

  return (
    <div className="flex items-center justify-between px-4 py-3">
      <span className="font-semibold">AI Chat</span>
      <div className="flex items-center gap-1">
        <button onClick={startNewConversation}>New</button>
        <button
          className={showHistory ? "bg-gray-600" : ""}
          onClick={toggleHistory}
        >
          History
        </button>
        <button onClick={close}>×</button>
      </div>
    </div>
  );
}
```

このコンポーネントの特徴：

- **状態を持たない (stateless)**: `useState` がない
- **Context から受け取った関数を紐づけるだけ**: ロジックは Provider に集約
- **条件付きクラス**: テンプレートリテラルで表示状態を切り替える

```ts
className={showHistory ? "bg-gray-600" : ""}
```

### `useRef` + `useEffect` で前回値を検知

```tsx
const prevIdRef = useRef<string | undefined>(contextId);

useEffect(() => {
  if (prevIdRef.current !== contextId) {
    prevIdRef.current = contextId;
    setConversationId(null); // コンテキストが変わったらリセット
  }
}, [contextId]);
```

React には「前回のレンダリング時の値」を自動で保持する仕組みがないため、`useRef` で手動管理します。`useRef` は再レンダリングを引き起こさないため、値の保持に適しています。

---

## 7. Feature ベースの Clean Architecture

### ディレクトリ構成

```
src/features/{feature-name}/
├── api/hono/                    # インフラ層: Hono ルーター
│   └── {feature}.router.ts
├── application/                 # ユースケース層 (CQRS)
│   ├── commands/                # 更新系 (Create/Update/Delete)
│   │   ├── createSomething.ts
│   │   └── updateSomething.ts
│   └── queries/                 # 参照系 (Read)
│       ├── getSomething.ts
│       └── listSomethings.ts
├── common/                      # 共有定義
│   ├── schemas.ts               # Zod スキーマ + 型定義
│   ├── types.ts                 # レスポンス型など
│   ├── errors.ts                # カスタムエラー
│   └── constants.ts             # 定数
├── domain/                      # ドメインロジック
│   └── business-rules.ts
└── infrastructure/              # 外部サービス連携
    ├── external-api/
    │   └── api-client.ts
    └── repositories/
        └── something.repository.ts
```

### CQRS (Command Query Responsibility Segregation)

更新系と参照系を明確に分離するパターンです。

```
commands/ (更新系)          queries/ (参照系)
├── createOrder.ts         ├── getOrder.ts
├── updateOrder.ts         └── listOrders.ts
└── cancelOrder.ts
```

- **Command**: 副作用を持つ（DB 書き込み、外部API呼び出し等）
- **Query**: 副作用なし（データの取得のみ）

この分離により、「この関数はデータを変更するか？」が一目で分かり、テストやレビューの観点も明確になります。

### 層の依存ルール

```
api/hono (インフラ層)
    ↓ 呼び出す
application (ユースケース層)
    ↓ 呼び出す
domain (ドメイン層)        ← 最も内側。外部に依存しない
    ↑ 参照のみ
infrastructure (インフラ層) ← DB, 外部API等の具体的な実装
```

内側の層は外側の層を知りません。これにより、ドメインロジックが特定のフレームワークやライブラリに依存しない設計になります。

---

## 8. Fire-and-Forget パターン — 非同期処理の切り離し

### ユースケース

メインの処理（例: メッセージ送信）の成否に影響しないが、裏でやっておきたい処理（例: 検索用インデックスの更新）がある場合に使います。

### 実装

```ts
/**
 * 重い処理を非同期で実行する（fire-and-forget）
 * エラーはログのみ出力し、呼び出し元に伝播しない
 */
async function processInBackground(params: {
  id: string;
  content: string;
}): Promise<void> {
  try {
    const result = await callExternalAPI(params.content);
    await saveToDatabase(params.id, result);
  } catch (error) {
    console.error(`[Background] Failed for ${params.id}:`, error);
    // エラーを throw しない → 呼び出し元に伝播しない
  }
}

/**
 * 公開関数 — 戻り値が void（Promise ではない）
 */
export function triggerBackgroundProcess(params: {
  id: string;
  content: string;
}): void {
  processInBackground(params).catch(() => {
    // 内部で既にログ済み
  });
}
```

### 2重のエラーガード構造

| ガード                    | 役割                                        |
| ------------------------- | ------------------------------------------- |
| 内部の `try-catch`        | 通常のエラーをキャッチしてログ出力          |
| 外部の `.catch(() => {})` | 万が一の Unhandled Promise Rejection を防止 |

`.catch(() => {})` がないと、`try` ブロック外で例外が発生した場合に Node.js の **Unhandled Promise Rejection** となり、プロセスがクラッシュする可能性があります。

### 処理の流れ

```
API ハンドラ
  ├── メインの処理（DB保存など） ← await する
  ├── triggerBackgroundProcess() ← await しない（裏で勝手に走る）
  └── レスポンスを返す           ← バックグラウンド処理の完了を待たない
```

### SQS 等のキューとの違い

|              | Fire-and-Forget            | メッセージキュー (SQS 等) |
| ------------ | -------------------------- | ------------------------- |
| リトライ     | なし                       | あり                      |
| 永続性       | サーバー再起動で消える     | キューに残る              |
| 複雑さ       | ゼロ（コードだけ）         | インフラ構築が必要        |
| 適するケース | 失敗しても再生成可能な処理 | 確実に実行が必要な処理    |

---

## 9. フィーチャーフラグによる段階的リリース

### レイアウトへの機能埋め込み

```tsx
export function AppLayout({ children }: { children: ReactNode }) {
  const featureEnabled = isFeatureEnabled("AI_CHAT");
  const { contextType, contextId } = usePageContext();

  return (
    <div className="flex flex-col h-screen">
      <Header />
      <main>{children}</main>
      {featureEnabled && (
        <ChatProvider contextType={contextType} contextId={contextId}>
          <FloatingButton />
          <ChatPanel />
        </ChatProvider>
      )}
    </div>
  );
}
```

### URL パスからコンテキストを判定するフック

```ts
function usePageContext() {
  const pathname = usePathname();
  return useMemo(() => {
    const match = pathname.match(
      /^\/orders\/([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})/i
    );
    if (match) {
      return { contextType: "ORDER" as const, contextId: match[1] };
    }
    return { contextType: undefined, contextId: undefined };
  }, [pathname]);
}
```

`useMemo` で URL パスが変わらない限り再計算しないようにしています。UUID の正規表現でパスパラメータを抽出し、アプリのどの画面にいるかを判定します。

### ポイント

- フラグ OFF なら**コンポーネントツリー自体が生成されない**（パフォーマンスに影響なし）
- 全ページ共通のレイアウトに配置することで、**どの画面からでも機能にアクセス可能**
- `contextType` / `contextId` により、ページごとに異なるコンテキストを機能に渡せる

---

## まとめ

| パターン                                 | 解決する課題                                |
| ---------------------------------------- | ------------------------------------------- |
| Optional Catch-all `[[...route]]`        | Hono 等のフレームワークへのルーティング委譲 |
| Hono ブリッジ                            | Next.js に依存しない API ロジックの実装     |
| 型の re-export                           | フロントとバックの型安全な契約              |
| バレルファイル                           | 公開 / 非公開の境界を明示                   |
| Context + Provider                       | コンポーネント間の状態共有                  |
| Feature ベース Clean Architecture + CQRS | 機能単位の凝集度と責務分離                  |
| Fire-and-Forget                          | メイン処理をブロックしない非同期実行        |
| フィーチャーフラグ                       | 安全な段階的リリース                        |

これらのパターンは単独でも有用ですが、組み合わせることで「型安全で、テスタブルで、段階的にリリースできる」アプリケーションを構築できます。

