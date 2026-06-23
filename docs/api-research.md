# API Research — fast.com DOM & Google Maps API

> FastMap 技術調査結果 (2026-06-23)

---

## 1. fast.com DOM 構造調査

### 1.1 ページ構造

fast.com はシンプルなSPA。Netflix製。

```
<div class="wrapper">
  ├── <div class="logo-container">          ← ロゴ + 言語セレクタ
  └── <div class="speed-container centered">
        ├── <div class="speed-controls-container">
        │     ├── <div id="your-speed-message">  ← 「お使いのインターネットの速度:」
        │     ├── <div class="speed-left-container">
        │     │     └── <div id="speed-value">27</div>       ← 下り速度 (数値のみ)
        │     │     └── <div id="speed-units">Mbps</div>     ← 単位
        │     │     └── <div id="speed-progress-indicator">  ← プログレスサークル
        │     └── <div class="speed-right-container">
        │           └── <div class="speed-units-container">
        └── <div class="test-context-container" id="test-context-container">
              ├── <div id="extra-details-container">          ← 「詳細を表示」後の詳細
              │     ├── <div id="latency-value">56</div>      ← レイテンシ (ms)
              │     ├── <div id="bufferbloat-value">660</div> ← ロード済みレイテンシ
              │     ├── <div id="upload-value">18</div>       ← 上り速度
              │     └── <div id="upload-units">Mbps</div>
              └── <div id="test-info-container">
                    ├── <span id="user-location">Tokyo, JP</span>    ← クライアント位置
                    ├── <span id="user-ip">133.106.32.72</span>     ← IPアドレス
                    ├── <span id="user-isp"></span>                  ← ISP (空の場合あり)
                    └── <span id="server-locations">Yokohama, JP | ...</span> ← サーバー
```

### 1.2 取得可能なデータ

| データ | DOMセレクタ | 型 | 例 |
|---|---|---|---|
| 下り速度 | `#speed-value` | text → number | `"27"` |
| 速度単位 | `#speed-units` | text | `"Mbps"` |
| 上り速度 | `#upload-value` | text → number | `"18"` |
| 上り単位 | `#upload-units` | text | `"Mbps"` |
| レイテンシ(空) | `#latency-value` | text → number | `"56"` |
| レイテンシ(負荷) | `#bufferbloat-value` | text → number | `"660"` |
| 位置情報 | `#user-location` | text | `"Tokyo, JP"` |
| IPアドレス | `#user-ip` | text | `"133.106.32.72"` |
| ISP | `#user-isp` | text | `""` (空の場合あり) |
| サーバー | `#server-locations` | text | `"Yokohama, JP \| ..."` |
| JS変数 | `window.speedtestInd` | number | `47` |

### 1.3 自動テスト開始

- テストは `<div role="button">` (e1) のクリックで開始
- URLパラメータでの直接開始は**不可**（`?start=true`等は効かない）
- `window.speedtestInd` はテスト進行中の値も保持
- 自動開始方法: content script で `document.querySelector('[role="button"]').click()` で可能

### 1.4 スクリーンショット対象

テスト完了後、`.speed-container` をキャプチャすれば十分。
`chrome.tabs.captureVisibleTab()` で全体キャップ → crop でもOK。

### 1.5 注意点

- `#extra-details-container` はデフォルト非表示（`style="display: none"`）
- 「詳細を表示」リンク (`#show-more-details-link`) をクリック後に表示
- レイテンシ/上り速度は詳細表示後のみ取得可能
- ISP (`#user-isp`) は空の場合がある

---

## 2. Google Maps API 調査

### 2.1 投稿（Review）API

**Places API - Place Details** で口コミ(review) は**読み取り**可能。

**口コミの新規投稿（作成）は Google Maps API では不可。**
- Google Maps API には「口コミを投稿する」エンドポイントが存在しない
- 口コミ投稿は Google Maps アプリ/ウェブサイトのUIからのみ可能

### 2.2 代替案

| 方法 | 実現性 | 備考 |
|---|---|---|
| **A. Google Maps URL スキーム** | ✅ 高 | `https://www.google.com/maps/search/?api=1&query=lat,lng` で地図を開く |
| **B. Google Maps の Place ページを開く** | ✅ 高 | `https://www.google.com/maps/place/?q=place_id:ChIJ...` |
| **C. ユーザーが手動で投稿** | ✅ 確実 | 拡張機能が投稿画面まで自動遷移、ユーザーがコピペ |
| **D. Maps JavaScript API で地図表示** | ✅ 高 | 地図表示 + マーカー配置まではAPIで可能 |

### 2.3 推奨アプローチ

**A + C のハイブリッド**:
1. 拡張機能が位置情報を取得
2. Google Maps の Place ページを自動開く（URLパラメータで検索）
3. ユーザーが投稿画面でAI生成テキストをペースト
4. クリップボードにコピーボタンで対応

**将来的な改善**:
- Google の「Local Guides」プログラムAPI（非公開）
- Google Maps のレビュー投稿API（存在しない）

### 2.4 必要なAPI

| API | 用途 | 費用 |
|---|---|---|
| Maps JS API | 地図表示（オプション） | 月$200無料 |
| Geocoding API | 座標→住所 | 月$200無料 |
| Places API (Nearby Search) | 位置から場所検索 | 月$200無料 |

---

## 3. スクリーンショット方法

### 3.1 Chrome Extension API

```typescript
// 現在のタブをキャプチャ
const dataUrl = await chrome.tabs.captureVisibleTab(null, {
  format: 'png',
  quality: 100
});
// → "data:image/png;base64,iVBORw0KGgo..." が返る
```

### 3.2 キャプチャ対象

- fast.com の結果画面全体をキャプチャ
- 必要に応じて `.speed-container` 部分のみクロップ
- クロップは Canvas API で実装可能

---

## 4. AI テキスト生成

### 4.1 プロンプト設計（案）

```
以下の情報からGoogle Mapsの口コミ投稿文を100字以内で生成してください。

- 場所: {{location}}
- 下り速度: {{download}} Mbps
- 上り速度: {{upload}} Mbps
- レイテンシ: {{latency}} ms
- 回線種別: {{connectionType}}

カジュアルで自然な日本語で。速度が速い場合はポジティブに、遅い場合は正直に。
```

### 4.2 使用API

- **Gemini 2.0 Flash** — 無料枠 1500req/day、日本語品質良好
- フォールバック: テンプレートベースのローカル生成（API Key未設定時）

---

## 5. 結論と推奨

### 自動テスト → 投稿のフロー（確定）

```
1. [Click 1] 拡張機能アイコンクリック
   → content script が fast.com タブで DOM から速度データ抽出
   → 同時に位置情報取得 (GPS → IP fallback)
   → 同時にスクリーンショット取得

2. [AI処理] Gemini API でテキスト自動生成
   → Popup に結果表示、ユーザーが編集可能

3. [Click 2] 「投稿」ボタンクリック
   → Google Maps の該当場所ページを新規タブで開く
   → テキストをクリップボードにコピー

4. [Click 3] Google Maps でユーザーがペーストして投稿
```

### 技術的課題

| 課題 | 深刻度 | 対策 |
|---|---|---|
| Google Maps API で投稿不可 | 高 | URLスキームで地図を開く + ユーザー手動投稿 |
| fast.com 自動テスト | 低 | content script で click トリガー |
| 詳細データ取得 | 低 | 「詳細を表示」を自動クリック |
| Gemini API 無料枠 | 低 | ローカルテンプレート fallback |

### 次のアクション

- [ ] 拡張機能骨格作成 (Manifest V3 + TS + Vite)
- [ ] content script: fast.com DOM スクレイピング
- [ ] 位置情報取得モジュール
- [ ] Gemini API クライアント
- [ ] Google Maps URL スキーム生成
- [ ] Popup UI (React + TailwindCSS)
- [ ] GitHub Actions CI
- [ ] Playwright E2E テスト
