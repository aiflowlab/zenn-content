---
title: "Claude Haiku 4.5 + n8n でクレカ明細メールを自動経費記録したら、7/7 精度・1 件約 1 円で運用できた"
emoji: "🧾"
type: "tech"
topics: ["n8n", "claude", "anthropic", "automation", "accounting"]
published: true
---

## はじめに

Claude API + n8n で **経費の自動記録ワークフロー** を実装しました。設計の肝は「既知の取引先はルールで即処理、未知の取引先だけ AI に判定させる」ハイブリッド構造で、精度・コスト・保守性のバランスを取っています。

7 件のテストケース(3 銀行/カード会社・複数経費パターン)で全件 PASS、1 件あたり約 1 円。業務用 Gmail に届くクレカ明細メールを検知して、Claude が取引情報を抽出・分類し、ledger CSV に自動追記 + Slack 通知します。

![n8n ワークフロー全景](https://static.zenn.studio/user-upload/25c206ccb28a-20260513.png)

ソース一式は [aiflowlab/n8n-claude-samples](https://github.com/aiflowlab/n8n-claude-samples) で MIT 公開済みです。

## 設計のコアアイデア:ルール + AI のハイブリッド

このワークフローの肝は、**既知の取引先はルールで処理、未知の取引先だけ AI に判定させる** ハイブリッド構造です。

```
クレカ明細メール
  → 銀行ホワイトリスト照合
  → Claude で取引情報を抽出(Tool Use)
  → vendor_rules.json と照合
    ├─ ルール一致 → カテゴリ/按分率を即決定
    └─ ルール未定義 → Claude で業務関連性 + 按分率を AI 判定
  → 重複チェック → ledger CSV に追記
  → Slack 通知(完了 or 要レビュー)
```

毎月固定で来る SaaS 費用(Anthropic / n8n 等)はルールで確実・高速に処理し、新しい取引先が出てきたときだけ AI に判断を委ねます。

## 前提:Human-in-the-loop 設計

勘定科目・税区分・按分率の最終確定は利用者(または顧問税理士)が行います。本ワークフローが担うのは「取引情報の構造化と記録漏れ防止」までで、税務上の確定は人間側の責任とする設計です。AI 判定行には備考欄に `⚠ 要レビュー` マーカーが残るため、月末・申告前の見直し起点として使えます。

## ワークフロー構成(13 ノード)

| # | ノード | 種別 | 役割 |
|---|---|---|---|
| 1 | Gmail Trigger | Gmail Trigger | 1 分ポーリングで新着メール検知 |
| 2 | Load Configs | Code | bank_senders.json + business_context.md を読み込み |
| 3 | Determine Bank | Code | 送信元ホワイトリスト照合 |
| 4 | Bank Found? | Switch | bank_found で分岐 |
| 5 | Extract (Claude) | HTTP Request | Tool Use で取引情報を抽出 |
| 6 | Parse + Rule Match | Code | 抽出結果パース + vendor_rules.json 照合 |
| 7 | Rule Hit? | Switch | rule_matched で分岐 |
| 8 | Apply Rule | Code | ルールからカテゴリ/按分率を決定 |
| 9 | Classify (Claude) | HTTP Request | business_context を注入して AI 判定 |
| 10 | Format From AI | Code | AI 出力を ledger 形式に整形 |
| 11 | Merge Paths | Merge | Apply Rule / Format From AI の出力を合流 |
| 12 | Append Ledger | Code | 重複チェック + CSV 追記 |
| 13 | Post to Slack | HTTP Request | 完了 / 要レビュー / スキップ通知 |

## 取引情報の抽出:prefill ではなく Tool Use を使う理由

取引情報の抽出には、Claude の **Tool Use(Function Calling)** を使っています。prefill(`{` でアシスタントの出力を JSON に強制する方法)ではなく Tool Use を選んだ理由は 2 つです。

1. **銀行ごとのフォーマット差を吸収しやすい**: 三菱 UFJ / 楽天カード / 三井住友 VPass はそれぞれメールの構造が異なります。「このフィールドから取れ」というルールを書くより、「この情報を探して構造化して」とツールで依頼する方がロバストです
2. **必須フィールドの欠落を検出できる**: Tool Use では `required` フィールドを指定できるため、Claude が必要な情報を見つけられなかったときに明確に失敗させられます

API への呼び出しはこのような形です。

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 512,
  "temperature": 0,
  "system": "...",
  "tools": [
    {
      "name": "record_expense",
      "description": "クレカ明細メールから取引情報を抽出して記録する",
      "input_schema": {
        "type": "object",
        "properties": {
          "transaction_date": {
            "type": "string",
            "description": "取引発生日 (YYYY-MM-DD)"
          },
          "amount_jpy": {
            "type": "number",
            "description": "金額(円、明細記載値そのまま、税込・為替手数料込み)"
          },
          "amount_original": {
            "type": "string",
            "description": "原通貨額(海外利用時、例: 'US$199.00')、国内利用なら空文字"
          },
          "vendor_raw": {
            "type": "string",
            "description": "メール本文に記載された取引先表記そのまま(例: 'ANTHROPIC PBC NEW YORK US')"
          },
          "transaction_category": {
            "type": "string",
            "description": "ご利用区分(海外ショッピング / ショッピング / 1回払い 等)、なければ空文字"
          }
        },
        "required": [
          "transaction_date",
          "amount_jpy",
          "vendor_raw"
        ]
      }
    }
  ],
  "tool_choice": {
    "type": "tool",
    "name": "record_expense"
  },
  "messages": [
    {
      "role": "user",
      "content": "..."
    }
  ]
}
```

`tool_choice: { type: "tool", name: "record_expense" }` で特定ツールの呼び出しを強制します。Claude が他のテキストを出力する余地がなく、構造化抽出に集中させられます。

## 業務関連性の AI 判定:business_context の注入

ルールにない取引先は Claude に判定させます。このとき、**ユーザーの事業内容を記述した設定ファイルを System プロンプトに注入**します。

`config/business_context.md` を起動時に読み込み、Code ノードでプロンプトに差し込む構成です。

```js
// Load Configs ノードで読み込んだ business_context を classify プロンプトに注入
const businessContext = $('Load Configs').item.json.business_context;

const systemPrompt = `あなたは個人事業主の経費判定アシスタントです。
以下の業務コンテキストを参照して、取引が業務関連かどうかと按分率を判定してください。

<business_context>
${businessContext}
</business_context>
...`;
```

このファイルを書き換えるだけで、Web 制作受託・コンサル業・物販事業など異なる業種でも同じワークフローが使えます。

### 出力を JSON に強制する:assistant prefill

Classify では Tool Use ではなく **assistant prefill** を使っています。リクエストの `messages` 末尾にアシスタント側の発話を `{` 一文字で仕込み、Claude の出力をその続きから始めさせる方法です。

```json
"messages": [
  { "role": "user", "content": "..." },
  { "role": "assistant", "content": "{" }
]
```

レスポンス側では先頭の `{` が落ちて返ってくる(prefill 分は応答に含まれない仕様)ため、受信側で接頭辞を補完して `JSON.parse` します。

```js
// Format From AI ノード(runOnceForEachItem)
const text = '{' + (apiResponse.content[0].text || '');
const cls = JSON.parse(text);
```

Extract(抽出)と Classify(分類)で手法を使い分けた理由は次の通りです。

| ノード | 手法 | 選定理由 |
|---|---|---|
| Extract | Tool Use | 銀行ごとのメール構造差を吸収したい / 必須フィールド欠落を検出したい |
| Classify | prefill | 出力スキーマが単一固定(欠落検出は不要)。Tool 定義のオーバーヘッドがない分、レイテンシ・トークン消費が軽い |

Classify は「常に同じ形の JSON が返ってくれば十分」というシンプルな要件なので、prefill で `{` を強制するだけで構造化を確保できます。

### Few-shot が必要な理由

business_context だけでは按分率の判定が保守的に倒れる傾向がありました。「商品が特定できないから 0% にしておこう」という挙動が出たため、Few-shot の例示が必要でした。

System プロンプト内に 4 例をインライン化しています。

```text
<判定例>
例1: vendor_raw=AMAZON.CO.JP / amount_jpy=3300 / transaction_category=ショッピング
出力: {
  "is_business_related": true,
  "vendor": "Amazon",
  "category": "書籍 or 業務用品",
  "recommended_apportionment": 100,
  "reasoning": "金額3,300円は技術書1冊の典型値。business_contextの「業務関連の技術書=100%」に該当する可能性が高い。商品名不明のため確認推奨。",
  "decision_hint": "review_required"
}

例4: vendor_raw=SEIYU NET SUPER / amount_jpy=820 / transaction_category=ショッピング
出力: {
  "is_business_related": false,
  "vendor": "西友ネットスーパー",
  "category": "食品・日用品",
  "recommended_apportionment": 0,
  "reasoning": "食品スーパー。820円・店舗種別から食品・日用品と推測。business_contextの「業務無関係=0%」に該当→スキップ推奨。",
  "decision_hint": "skip_not_business"
}
</判定例>
```

ポイントは **`recommended_apportionment` を「不明だから 0」にしないこと**です。business_context の該当項目の按分率をそのまま出し、確認が必要な場合は `decision_hint: review_required` で人間に委ねる設計にしています。

## 重複検知ロジック

同じ明細メールが重複配信されたとき、または再実行時の二重記録を防ぐため、**取引先 + 金額 + 日付の 3 点照合**で重複チェックをしています。

```js
// Append Ledger ノード内の重複チェック
const existingRows = readLedger();  // 既存 CSV を読み込み
const isDuplicate = existingRows.some(row =>
  row.vendor === newEntry.vendor &&
  row.amount === newEntry.amount &&
  row.date === newEntry.date
);

if (isDuplicate) {
  return [{ json: { status: 'skipped', reason: 'duplicate' } }];
}
```

金額だけ / 日付だけでは偶発的な一致が起こりえます。3 点セットでの照合が現実的なバランスです。

## Slack 通知の 2 パターン

完了通知と要レビュー通知で、Post to Slack ノードがメッセージを切り替えます。

![完了通知](https://static.zenn.studio/user-upload/db10c13f9431-20260513.png)

ルール一致の場合は自動記録完了を報告。

![要レビュー通知](https://static.zenn.studio/user-upload/c7972000884e-20260513.png)

AI 判定の場合は「推奨値で追記済み、内容を確認して」という通知と一緒に **AI の判定理由** を添えています。「なぜそう判断したか」が見えると、確認の手間が大幅に減ります。

## ledger CSV の出力形式

11 カラム構成です。ワークフローが追記する行の例を示します。

**ルール一致(Anthropic):**
```csv
2026-06-15,経費,LLM,Anthropic,Claude Code Max 月額(海外ショッピング),895,デビット(MUFG),100,895,https://console.anthropic.com/settings/billing,Sample02 自動記録(2026-06-15)/ ルール一致(anthropic)
```

**AI 判定・要レビュー(Amazon):**
```csv
2026-06-02,経費,業務関連の技術書,Amazon,ショッピング(1回払い),3300,デビット(MUFG),100,3300,,Sample02 AI判定(2026-06-02)/ ⚠ 要レビュー(AI 推奨)/ 金額3,300円はプログラミング・API設計等の技術書1冊の典型値。商品名がメールから取れないため確認必須。
```

**AI 判定・スキップ(西友ネットスーパー):**
→ 業務無関係と判定されたため ledger への追記なし。Slack にスキップ通知のみ。

カラム定義:
```
日付, 種別, カテゴリ, 取引先, 内容, 金額(税込), 支払/受取方法, 業務按分率(%), 計上額, 証憑URL/ファイル名, 備考
```

`計上額 = 金額 × 業務按分率 / 100` の計算は Append Ledger ノード内で行います。備考欄の `⚠ 要レビュー` マーカーが確定申告時の見直しポイントになります。

## ハマりポイント

n8n + Gmail でハマった 3 点をまとめます。

### 1. Gmail Trigger の Simplify は false に

デフォルト(`Simplify: true`)ではメール本文(`text`)が返らず、snippet のみが渡されます。Claude による情報抽出が機能しないため **`Simplify: false` が必須**です。

```js
// Simplify: false のときのアクセスパス
const from = $json.from.value[0].address;
const body = $json.text;
const date = $json.date;
```

### 2. Code ノードは runOnceForEachItem で書く

デフォルトの `runOnceForAllItems` + `$('NodeName').first()` は複数メール同時着信で最初の 1 件しか処理されません。

```js
// NG: 複数着信で取りこぼす
const data = $('Extract (Claude)').first().json;

// OK: runOnceForEachItem モードで $input.item を使う
const data = $input.item.json;
```

Parse + Rule Match / Apply Rule / Format From AI / Append Ledger の全 Code ノードで徹底しています。

### 3. Switch ノードへは boolean ではなく String() で渡す

n8n 2.18.7(v3.2)の Switch ノードは boolean 値を Rules モードで比較すると一致しないことがあります。

```js
// NG
return [{ json: { bank_found: true } }];

// OK
return [{ json: { bank_found: String(true) } }];  // "true"
```

## テスト結果

| テストケース | 銀行/カード | 経路 | 結果 |
|---|---|---|---|
| Anthropic 月額 ¥15,586 | MUFG | ルール一致 | PASS |
| Anthropic チャージ ¥895 | MUFG | ルール一致 | PASS |
| n8n 月額 ¥3,150 | 楽天カード | ルール一致 | PASS |
| Amazon ¥3,300 | MUFG | AI 判定(要レビュー) | PASS |
| Amazon ¥45,000 | 三井住友 VPass | AI 判定(要レビュー) | PASS |
| お名前.com ¥1,628 | MUFG | AI 判定(要レビュー) | PASS |
| 西友ネットスーパー ¥820 | MUFG | AI 判定(スキップ) | PASS |

- ユニットテスト: 7/7 PASS
- E2E(実 Gmail): 4 経路全疎通(Bank not found / ルール一致 / AI 要レビュー / AI スキップ)
- 1 件あたりコスト: 約 1 円(Haiku 4.5、extract + classify)

## 次のステップ:入金の自動記録(開発中)

現バージョンは経費(支出)の記録に対応しています。次フェーズとして **入金の自動記録** を開発中です。

- ランサーズ / クラウドワークス の報酬確定メール → 売上として自動記録
- 銀行への振込通知メール → 入金ステータスを自動更新
- 手数料は別行で経費計上、グロス / ネットを分けて管理

設計方針は経費編と同じく「送信元ホワイトリスト + AI 判定」のハイブリッドです。個人名からの振込(家族送金・返金等)は AI 判定 + 要レビューに倒し、誤計上を防ぎます。

実装が完成したら「入金編」として別記事で書く予定です。

## まとめ

- **Haiku 4.5 + n8n で 7/7 精度・1 件約 1 円**の経費自動記録ワークフローを構築しました
- **Tool Use で抽出、prefill で分類**という使い分けが、銀行フォーマット差の吸収と JSON 強制の両立に効きました
- **business_context.md の外出し**で、業種を問わず再利用できる設計にしています
- Code ノードの `runOnceForEachItem` と Switch ノードの `String()` 強制は、n8n 固有のハマりポイントとして要注意です

ソース一式は [aiflowlab/n8n-claude-samples](https://github.com/aiflowlab/n8n-claude-samples) に置いてあります(MIT)。設定ファイルのテンプレート・テストケース・ワークフロー JSON 全部入りです。

「設計について話したい」「自分の業務に合わせてカスタマイズしたい」という場合は X([@aiflowlab](https://x.com/aiflowlab))の DM までどうぞ。
