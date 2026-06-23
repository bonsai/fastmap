# PRD — FastMap: Speed Test → Google Maps 投稿拡張機能

> fast.com の速度テスト結果をスクショし、Google Maps に自動投稿する Chrome 拡張機能

---

## 1. 概要

### プロダクト名
**FastMap** (fastmap-extension)

### 一言説明
fast.com で速度テスト → 結果をスクショ → Google Maps に自動投稿。3クリック以内で完結。

### ターゲットユーザー
- ネット速度を日常的に測定・記録する人
- カフェ・コワーキングスペースの回線速度を共有したい人
- 通信環境の可視化マップを作りたい人

---

## 2. ユーザーフロー（3クリック以内）

```
[Click 1] 拡張機能アイコンクリック
    ↓
    fast.com で速度テスト自動開始（または既に結果表示済みなら即スクショ）
    ↓
[Click 2] 結果画面で「投稿」ボタンクリック（自動生成テキスト確認→編集可）
    ↓
    AIがテキスト生成（速度・場所・回線種別を含む投稿文）
    ↓
[Click 3] Google Maps の該当場所で「投稿」完了
```

### 代替フロー（さらに自動化）
```
[Click 1] 拡張機能アイコンクリック
    ↓
    fast.com 自動テスト → 自動スクショ → AI自動テキスト生成
    ↓
    Google Maps 自動検索 → 該当場所に自動投稿
    ↓
[Click 2] 確認ダイアログ「投稿しますか？」→ OK
    ↓
[Click 3] 投稿完了
```

---

## 3. 機能要件

### 3.1 必須機能 (MVP)

| # | 機能 | 説明 | 自動化度 |
|---|---|---|---|
| F1 | fast.com スクショ | テスト結果画面をキャプチャ | 完全自動 |
| F2 | 位置情報取得 | GPS / IPベースで現在地取得 | 完全自動 |
| F3 | AIテキスト生成 | 速度結果から投稿文を自動生成 | 完全自動 |
| F4 | Google Maps 投稿 | 該当場所の投稿として登録 | 半自動（確認あり） |
| F5 | 3クリック以内 | 上記全工程を3クリックに収める | UX制約 |

### 3.2 拡張機能（Phase 2）

| # | 機能 | 説明 |
|---|---|---|
| F6 | 履歴管理 | 過去のテスト結果をローカル保存・一覧表示 |
| F7 | 回線種別タグ | WiFi / 5G / 4G / 有線を自動判定 |
| F8 | バッチ投稿 | 複数結果を一括投稿 |
| F9 | マップ可視化 | 投稿済み地点を一覧表示するダッシュボード |
| F10 | SNS共有 | X/Twitter への同時投稿 |

---

## 4. 技術アーキテクチャ

### 4.1 全体構成

```
┌─────────────────────────────────────────────┐
│              Chrome Extension               │
│  ┌─────────┐  ┌──────────┐  ┌────────────┐ │
│  │ Popup   │  │ Content  │  │ Background │ │
│  │ UI      │  │ Script   │  │ Service    │ │
│  │         │  │          │  │ Worker     │ │
│  └────┬────┘  └────┬─────┘  └─────┬──────┘ │
│       │            │              │         │
│  [Click 1]   [Screenshot]   [AI + Maps API] │
│       │            │              │         │
└───────┼────────────┼──────────────┼─────────┘
        │            │              │
   ┌────▼────┐  ┌────▼─────┐  ┌────▼──────┐
   │ fast.com│  │ Chrome   │  │ AI API    │
   │ (速度   │  │ Tabs API │  │ (Gemini/  │
   │  テスト)│  │ +        │  │  GPT-4o)  │
   │         │  │ Capture  │  │           │
   └─────────┘  └──────────┘  └───────────┘
                                  │
                            ┌─────▼──────┐
                            │ Google Maps│
                            │ Places API │
                            │ + Maps JS  │
                            └────────────┘
```

### 4.2 技術スタック

| 層 | 技術 | 理由 |
|---|---|---|
| 拡張機能基盤 | Manifest V3 + TypeScript | Chrome最新標準 |
| UIフレームム | React + TailwindCSS | 軽量・高速 |
| スクリーンショット | `chrome.tabs.captureVisibleTab()` | 標準API |
| 位置情報 | `navigator.geolocation` + IP fallback | GPS優先、IP補完 |
| AIテキスト生成 | Gemini API (無料枠) / OpenAI API | コストと品質のバランス |
| Google Maps投稿 | Maps JavaScript API + Places API | 標準の投稿手段 |
| 保存 | `chrome.storage.local` | 拡張機能内ローカル |

### 4.3 API選定

#### AI API（テキスト生成）

| 候補 | 無料枠 | 品質 | 推奨 |
|---|---|---|---|
| Gemini 2.0 Flash | 1500 req/day | 高 | **第一候補** |
| GPT-4o-mini | 制限あり | 高 | 第二候補 |
| Claude Haiku | 制限あり | 中 | 第三候補 |

→ **Gemini 2.0 Flash** をデフォルトとし、ユーザーがAPIキーを切り替え可能

#### Google Maps API

| API | 用途 | 費用 |
|---|---|---|
| Maps JS API | 地図表示 | 月$200無料枠 |
| Places API | 場所検索・投稿 | 月$200無料枠 |
| Geocoding API | 座標→住所変換 | 月$200無料枠 |

---

## 5. データモデル

### 5.1 SpeedTestResult

```typescript
interface SpeedTestResult {
  id: string;                    // UUID
  timestamp: number;             // Unix ms
  downloadMbps: number;          // 下り速度
  uploadMbps: number;            // 上り速度
  latencyMs: number;             // レイテンシ
  jitterMs: number;              // ジッター
  server: string;                // テストサーバー
  connectionType: string;        // wifi / cellular / ethernet
  location: {
    lat: number;
    lng: number;
    accuracy: number;            // GPS精度(m)
    source: 'gps' | 'ip' | 'manual';
  };
  screenshot?: string;           // base64 data URL
  generatedText?: string;        // AI生成テキスト
  postedPlaceId?: string;        // Google Maps投稿先Place ID
  status: 'captured' | 'text_generated' | 'posted' | 'failed';
}
```

### 5.2 設定

```typescript
interface ExtensionSettings {
  aiProvider: 'gemini' | 'openai' | 'claude';
  geminiApiKey?: string;
  openaiApiKey?: string;
  claudeApiKey?: string;
  autoScreenshot: boolean;       // テスト完了後自動スクショ
  autoGenerateText: boolean;     // AIテキスト自動生成
  autoPost: boolean;             // 確認なしで投稿（危険）
  textTemplate: string;          // テキストテンプレート
  defaultMapZoom: number;        // マップ初期ズーム
}
```

---

## 6. 画面設計

### 6.1 Popup UI（メイン）

```
┌──────────────────────────────┐
│  🚀 FastMap                  │
│                              │
│  ┌──────────────────────┐    │
│  │  ⬇️ 45.2 Mbps       │    │
│  │  ⬆️ 12.8 Mbps       │    │
│  │  📡 遅延 23ms       │    │
│  │  📍 東京都渋谷区    │    │
│  └──────────────────────┘    │
│                              │
│  [📸 スクショして投稿]       │  ← Click 1
│                              │
│  ┌──────────────────────┐    │
│  │ 📝 生成テキスト:      │    │
│  │ 「渋谷のカフェで速度  │    │
│  │  テスト。下り45Mbps、 │    │
│  │  上り12Mbps。WiFi快適 │    │
│  │  です！」             │    │
│  └──────────────────────┘    │
│                              │
│  [✏️ 編集] [🗺️ 投稿]        │  ← Click 2-3
│                              │
│  ⚙️ 設定  │  📋 履歴         │
└──────────────────────────────┘
```

### 6.2 フローチャート

```
[拡張機能アイコンクリック]
        │
        ▼
  fast.com タブは開いている？
   ├─ Yes → 結果は表示済み？
   │         ├─ Yes → [F1] スクショ取得
   │         └─ No  → fast.com でテスト実行 → [F1] スクショ
   └─ No  → fast.com を新規タブで開く → テスト実行 → [F1] スクショ
        │
        ▼
  [F2] 位置情報取得（GPS → IP fallback）
        │
        ▼
  [F3] AI テキスト生成（Gemini API）
        │
        ▼
  ユーザーにテキスト確認・編集
        │
        ▼
  [F4] Google Maps で該当場所検索
        │
        ▼
  ユーザーが場所を選択（または自動選択）
        │
        ▼
  投稿確認ダイアログ
        │
        ▼
  投稿完了 ✅
```

---

## 7. 実装フェーズ

### Phase 1: MVP（1-2週間）
- [ ] Chrome拡張機能の骨格（Manifest V3 + TS）
- [ ] fast.com スクリーンショット機能
- [ ] 位置情報取得（GPS + IP）
- [ ] Gemini API でテキスト生成
- [ ] Google Maps 投稿（手動選択）
- [ ] Popup UI（結果表示 + 投稿ボタン）

### Phase 2: 自動化強化（1週間）
- [ ] fast.com 自動テスト実行
- [ ] 自動テキスト生成 → 確認 → 自動投稿フロー
- [ ] 回線種別自動判定
- [ ] エラーハンドリング

### Phase 3: 拡張機能（1-2週間）
- [ ] 履歴管理（chrome.storage.local）
- [ ] ダッシュボード（投稿済み地点マップ）
- [ ] 設定画面（API プロバイダ切り替え）
- [ ] SNS同時投稿

---

## 8. リスクと対策

| リスク | 影響 | 対策 |
|---|---|---|
| Google Maps API の投稿制限 | 高 | Places API のレビュー申請 + フォールバック |
| Gemini API 無料枠超過 | 中 | キャッシュ + ローカルテンプレート fallback |
| fast.com UI変更 | 中 | セレクタの抽象化 + バージョン管理 |
| GPS精度不足（屋内） | 中 | IPベース補完 + 手動位置選択 |
| Chrome Manifest V3 制約 | 低 | Service Worker で対応 |

---

## 9. 成功指標 (KPI)

| 指標 | 目標値 |
|---|---|
| 完了までのクリック数 | ≤ 3 |
| 投稿完了までの時間 | ≤ 30秒 |
| AIテキスト生成成功率 | ≥ 95% |
| ユーザーがテキストを編集する割合 | ≤ 30%（自動生成の品質指標） |

---

## 10. オープンな質問

1. Google Maps の「投稿」は Places API で可能か？→ 要検証（口コミ投稿APIは限定的）
2. fast.com の自動テスト開始は可能か？→ 要検証（UI操作の自動化）
3. 無料枠で十分か？→ Gemini 1500req/day なら個人利用は十分
4. 複数ブラウザ対応が必要か？→ Chrome のみで開始、Edge対応はPhase 3

---

## 11. テスト戦略

### 11.1 使用ツール

| ツール | 用途 | 備考 |
|---|---|---|
| **DevTools (Chrome)** | 拡張機能のデバッグ、Console確認、Network監視 | 開発中のメイン |
| **Edge (msedge)** | Edgeブラウザでの互換性テスト | Edge版として動作確認 |
| **Playwright** | E2Eテスト自動化 | headless + devtool protocol |
| **Vitest** | ユニットテスト | 各モジュールの単体検証 |
| **Chrome DevTools Protocol (CDP)** | スクリーンショット自動取得の検証 | content script のテスト |

### 11.2 テストレベル

| レベル | 内容 | ツール |
|---|---|---|
| Unit | screenshot.ts, geolocation.ts, ai.ts, maps.ts | Vitest |
| Integration | スクショ→AI生成→Maps投稿のフロー統合 | Vitest + Playwright |
| E2E | 拡張機能のインストール→実行→投稿完了まで | Playwright + CDP |
| Cross-browser | Edgeでの動作確認 | Playwright (Chromium/Edge) |

### 11.3 テスト環境

- Chrome: 開発用（拡張機能を `chrome://extensions` からアンパックドロード）
- Edge: 互換性検証（`edge://extensions` から同様ロード）
- CI: GitHub Actions の ubuntu-latest + Playwright (headless)

---

## 12. CI/CD パイプライン (GitHub Actions)

### 12.1 Workflow 構成

| Workflow | トリガー | 処理 |
|---|---|---|
| `ci.yml` | push / PR | lint → typecheck → unit test → build |
| `e2e.yml` | PR / main merge | Playwright E2E テスト (headless) |
| `release.yml` | tag push (`v*`) | build → zip → GitHub Release 作成 |
| `edge-test.yml` | PR (手動) | Edge (msedge) での E2E テスト |

### 12.2 リリースフロー

```
[開発] → [PR] → [CI: lint+test+build]
                            ↓
                    [main merge]
                            ↓
                    [E2E テスト]
                            ↓
                    [git tag v0.1.0]
                            ↓
                    [release.yml 発火]
                            ↓
                    [GitHub Release 作成]
                    (fastmap-v0.1.0.zip)
```

### 12.3 デプロイターゲット

| ターゲット | 方法 | タイミング |
|---|---|---|
| GitHub Release | `gh release create` | tag push時 |
| Chrome Web Store | 手動アップロード（将来） | 安定版リリース時 |
| Edge Add-ons | 手動アップロード（将来） | 安定版リリース時 |

---

## 13. セキュリティ

### API Key 管理
- 拡張機能内にAPI Keyを直接埋め込まない
- サーバーサイドプロキシ（`server/`）経由でAPI呼び出し
- GitHub Secrets でCI/CDのキーを管理

### Content Security Policy
- Manifest V3 の CSP に準拠
- `connect-src` に必要なAPI エンドポイントのみ許可
- `unsafe-eval` は使用しない

### ユーザーデータ
- 位置情報はローカルのみ（`chrome.storage.local`）
- 外部サーバーへの送信は最小限（AI API のみ）
- スクリーンショットはローカル保存のみ
