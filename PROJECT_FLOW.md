# PROJECT_FLOW — FastMap 実装プロジェクトフロー

---

## 1. リポジトリ構成

```
fastmap/
├── PRD.md                      # 製品要求仕様（メイン）
├── PROJECT_FLOW.md             # このファイル
├── README.md                   # プロジェクト概要
├── docs/
│   ├── architecture.md         # 技術アーキテクチャ詳細
│   ├── api-research.md         # API調査結果
│   └── ui-wireframes.md        # UIワイヤーフレーム
├── extension/                  # Chrome拡張機能
│   ├── manifest.json
│   ├── popup/
│   │   ├── popup.html
│   │   ├── popup.tsx
│   │   └── popup.css
│   ├── content/
│   │   └── content.ts          # fast.com スクプト
│   ├── background/
│   │   └── service-worker.ts   # バックグラウンド処理
│   ├── utils/
│   │   ├── screenshot.ts       # スクリーンショット
│   │   ├── geolocation.ts      # 位置情報
│   │   ├── ai.ts               # AI API クライアント
│   │   └── maps.ts             # Google Maps API
│   ├── storage/
│   │   └── db.ts               # chrome.storage ラッパー
│   ├── settings/
│   │   └── settings.html       # 設定画面
│   └── icons/
│       ├── icon16.png
│       ├── icon48.png
│       └── icon128.png
├── server/                     # オプション: サーバーサイド
│   ├── package.json
│   └── src/
│       └── index.ts            # API プロキシ（API key保護用）
├── tests/
│   ├── unit/
│   └── e2e/
├── package.json
├── tsconfig.json
├── vite.config.ts
└── .gitignore
```

---

## 2. 開発フロー

### Step 1: 調査（1-2日）

#### 1.1 fast.com 自動テスト調査
- [ ] fast.com の DOM構造を調査
- [ ] テスト自動開始が可能か確認（`window` オブジェクト / URL パラメータ）
- [ ] 結果表示のセレクタを特定
- [ ] スクリーンショットの画質要件確認

#### 1.2 Google Maps API 調査
- [ ] Places API で口コミ投稿が可能か確認
- [ ] Maps JS API の料金体系確認
- [ ] API Key の制限設定方法確認
- [ ] 代替案：Google Maps の URL スキームで投稿

#### 1.3 AI API 調査
- [ ] Gemini 2.0 Flash API の無料枠確認
- [ ] プロンプト設計（速度結果→自然な投稿文）
- [ ] レスポンスフォーマット定義

### Step 2: 基盤構築（2-3日）

- [ ] Chrome拡張機能の骨格作成
  - `manifest.json` (Manifest V3)
  - TypeScript + Vite 環境構築
  - React + TailwindCSS 導入
- [ ] ビルド・開発ワークフロー確立
  - `npm run dev` → ホットリロード
  - `npm run build` → 本番ビルド
  - Chrome への手動ロード確認

### Step 3: コア機能実装（3-5日）

#### 3.1 スクリーンショット機能
- [ ] `chrome.tabs.captureVisibleTab()` 実装
- [ ] fast.com 結果画面の自動検出
- [ ] 画像のクリップボードコピー / ストレージ保存

#### 3.2 位置情報取得
- [ ] `navigator.geolocation` 実装
- [ ] IPベース位置情報フォールバック（ip-api.com等）
- [ ] 精度表示

#### 3.3 AIテキスト生成
- [ ] Gemini API クライアント実装
- [ ] プロンプトテンプレート設計
- [ ] 結果表示・編集UI

#### 3.4 Google Maps 投稿
- [ ] Maps JS API 組み込み
- [ ] 場所検索・選択UI
- [ ] 投稿実行

### Step 4: 統合・テスト（2-3日）

- [ ] 全フローの統合テスト
- [ ] エッジケーステスト
  - GPS取得失敗
  - AI API エラー
  - Google Maps API 制限
- [ ] パフォーマンス最適化

### Step 5: 仕上げ（1-2日）

- [ ] アイコン作成
- [ ] 設定画面
- [ ] エラーハンドリング強化
- [ ] ドキュメント整備

---

## 3. 依存関係グラフ

```
[fast.com DOM調査] ──→ [スクショ機能] ──→ [AIテキスト生成]
                                              ↓
[Google Maps API調査] ──→ [Maps投稿機能] ←──┘
        ↑                                    ↓
[位置情報取得] ────────────────────────→ [統合フロー]
```

---

## 4. マイルストーン

| マイルストーン | 成果物 | 予定 |
|---|---|---|
| M1: 調査完了 | api-research.md | Day 1-2 |
| M2: 拡張機能骨格 | manifest.json + ビルド環境 | Day 3 |
| M3: スクショ動作 | fast.com → スクショ確認 | Day 4-5 |
| M4: AI生成動作 | 速度結果 → テキスト生成確認 | Day 6-7 |
| M5: Maps投稿動作 | テキスト → Maps投稿確認 | Day 8-9 |
| M6: 統合テスト | 全フロー動作確認 | Day 10-11 |
| M7: リリース準備 | アイコン・設定・ドキュメント | Day 12-13 |

---

## 5. 技術調査チェックリスト

### fast.com
- [ ] URLパラメータでテスト開始可能か
- [ ] 結果表示のDOM構造
- [ ] `window` に結果データが露出しているか
- [ ] 自動でテストをトリガーする方法

### Google Maps API
- [ ] Places API でテキスト投稿が可能か
- [ ] 代替：Google Maps URL スキーム
- [ ] API Key の HTTP リファラー制限
- [ ] 拡張機能内での API Key 管理方法

### Chrome Extension
- [ ] Manifest V3 の Content Security Policy
- [ ] Service Worker での API 通信
- [ ] 外部API の許可設定（connect-src）
- [ ] アイコンクリックでの Popup 表示

### AI API
- [ ] Gemini 2.0 Flash の rate limit
- [ ] プロンプト最適化（日本語の自然な投稿文）
- [ ] エラーリトライ戦略
- [ ] コスト見積もり

---

## 6. 次のアクション

1. **fast.com DOM調査** — ブラウザで開いて構造確認
2. **Google Maps API 調査** — ドキュメント確認
3. **拡張機能骨格作成** — Manifest V3 + TS プロジェクト初期化
