---
title: "Claude API の Tool Use と prefill、どう使い分ける? 判断チェックリストと Sample02 での選定例"
emoji: "⚖️"
type: "tech"
topics: ["claude", "anthropic", "toolcalling", "promptengineering", "n8n"]
published: false
---

## はじめに

Claude API で構造化出力(JSON など)を得るとき、選択肢は主に 2 つあります。

- **Tool Use**: `tools` パラメータでスキーマを定義し、Claude にツール呼び出しの形で出力させる
- **assistant prefill**: `messages` の末尾に `{` を仕込んで、続きを JSON として生成させる

両方とも「Claude に決まった形の出力を返してもらう」手段ですが、**どっちを選ぶかの判断基準は意外と言語化されていません**。公式ドキュメントは「使い方」中心で、「いつ Tool Use を選び、いつ prefill を選ぶか」の選定論はあまりまとまっていません。

本記事では、個人事業向け業務自動化サンプル(Sample02: n8n + Claude による経費自動記録ワークフロー)の実装を題材に、Tool Use と prefill の **使い分け判断基準** をチェックリスト化します。

- Sample02 のソース: [aiflowlab/n8n-claude-samples](https://github.com/aiflowlab/n8n-claude-samples)
- Sample02 の実装紹介(経費編): [Claude Haiku 4.5 + n8n でクレカ明細メールを自動経費記録したら、7/7 精度・1 件約 1 円で運用できた](https://zenn.dev/aiflowlab/articles/n8n-claude-credit-expense-automation)

## 挙動の違い

両者の決定的な違いは、**レスポンス構造**と**型担保の強さ**にあります。

### Tool Use

`tools` パラメータでスキーマを宣言し、`tool_choice` で特定ツール呼び出しを強制します。

```json
{
  "model": "claude-haiku-4-5-20251001",
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
            "description": "メール本文に記載された取引先表記そのまま"
          },
          "transaction_category": {
            "type": "string",
            "description": "ご利用区分(海外ショッピング / ショッピング / 1回払い 等)、なければ空文字"
          }
        },
        "required": ["transaction_date", "amount_jpy", "vendor_raw"]
      }
    }
  ],
  "tool_choice": { "type": "tool", "name": "record_expense" },
  "messages": [{ "role": "user", "content": "..." }]
}
```

レスポンスは `content` 配列の `tool_use` ブロックに格納されます。

```js
const toolUseBlock = response.content.find(b => b.type === 'tool_use');
const data = toolUseBlock.input;  // 既にパース済みオブジェクト
```

特徴:

- `required` 指定で必須フィールドの欠落を検出可能
- 型(`number` / `string` / `enum`)を強制
- 返り値は既にパース済みオブジェクト(`JSON.parse` 不要)
- スキーマがリクエストにもレスポンスにも含まれるため、トークン消費 + レイテンシは prefill より重い

### prefill

`messages` の末尾に `{ "role": "assistant", "content": "{" }` を入れて、Claude にその続きから生成させます。

```json
{
  "model": "claude-haiku-4-5-20251001",
  "system": "出力は次の JSON 形式で...",
  "messages": [
    { "role": "user", "content": "..." },
    { "role": "assistant", "content": "{" }
  ]
}
```

レスポンスでは prefill 分の `{` が **落ちて返ってくる**(prefill 分は応答に含まれない仕様)ため、受信側で接頭辞を補完します。

```js
const text = '{' + response.content[0].text;
const data = JSON.parse(text);
```

特徴:

- 出力構造は system プロンプトでテキスト指示するだけ(スキーマ宣言なし)
- 型担保はゼロ — `JSON.parse` がパースできれば成功、できなければ失敗するだけ
- Tool 定義のオーバーヘッドがないので軽量
- スキーマ仕様が system 内に閉じる(リクエスト構造はシンプル)

### 比較サマリ

| 観点 | Tool Use | prefill |
|---|---|---|
| 型担保(`number` / `string` / `enum`) | ◎ | ✗ |
| 必須フィールドの欠落検出 | ◎(`required`) | ✗(受信側で手書き) |
| トークン消費 / レイテンシ | 重 | 軽 |
| スキーマの管理場所 | リクエスト構造 | system プロンプト |
| 失敗時の挙動 | スキーマ違反で自動リトライ余地あり | `JSON.parse` 失敗 |
| パース処理 | 不要(`.input` がオブジェクト) | `'{' + text` を `JSON.parse` |

## 判断基準(チェックリスト)

以下のチェックで分岐します。**1 つでも左(Tool Use)に倒れる項目があれば Tool Use を選ぶ** のが安全側です。

| 項目 | Tool Use が向く | prefill が向く |
|---|---|---|
| 出力フィールドのうち、欠落したら後段で例外を出すべきものがある | ◯ | ✗ |
| 後段で数値計算 / 日付比較 / enum 分岐を行う | ◯ | ✗ |
| 出力スキーマが入力次第で変わる(条件分岐がある) | ◯ | ✗ |
| `required` 指定で欠落を即検出したい | ◯ | ✗ |
| Claude 側に「フィールドの意味」をツール定義経由で伝えたい | ◯ | ✗ |
| 出力スキーマが単一固定 + 自由文字列フィールド中心 | ✗ | ◯ |
| 高頻度呼び出しでレイテンシ / コストが気になる | ✗ | ◯ |
| 同じスキーマ定義を複数箇所(リクエスト + system)に書きたくない | ✗ | ◯ |

実用判断: **「数値や日付を後段で機械処理するなら Tool Use、文字列分類で人間が読むだけなら prefill」** がざっくりした目安です。

## ケーススタディ A: Sample02 Extract がなぜ Tool Use か

Sample02 の Extract ノードは、クレカ明細メールから取引情報を抽出します。

```json
{
  "transaction_date": "2026-05-15",
  "amount_jpy": 895,
  "amount_original": "US$8.95",
  "vendor_raw": "ANTHROPIC PBC NEW YORK US",
  "transaction_category": "海外ショッピング"
}
```

このノードで Tool Use を選んだ理由は 3 つあります。

### 1. 後段で数値計算が走る

`amount_jpy` は次のノードで `業務按分率 × 計上額` の計算に使われます。`"895"`(文字列)で返ってくると JavaScript の暗黙変換で罠を踏む可能性があるため、`type: "number"` で型担保しています。

### 2. 銀行フォーマット差を吸収したい

三菱 UFJ / 楽天カード / 三井住友 VPass は明細メールのフォーマットが異なります。「このフィールドはここから取れ」というルールを書くより、「この情報を探して構造化して」とツールで依頼する方がロバストです。Tool Use の `description` フィールドで「取引発生日 (YYYY-MM-DD)」のようにフィールドごとの意味を Claude に伝えられます。

```json
"transaction_date": {
  "type": "string",
  "description": "取引発生日 (YYYY-MM-DD)"
}
```

### 3. 必須フィールドの欠落を検出したい

`transaction_date` / `amount_jpy` / `vendor_raw` が取れないメール(プロモーション混入など)は後段で確実に弾きたい。`required` 指定で「この 3 つが揃わないと失敗」を強制できます。

```json
"required": ["transaction_date", "amount_jpy", "vendor_raw"]
```

prefill だと、これらの担保を全部 `JSON.parse` 後の手書きバリデーションで書く必要があり、コードが膨らみます。

## ケーススタディ B: Sample02 Classify がなぜ prefill か

Sample02 の Classify ノードは、未知の取引先について「業務関連か / 按分率は何%か」を AI に判定させます。

```json
{
  "is_business_related": true,
  "vendor": "Amazon",
  "category": "業務関連の技術書",
  "recommended_apportionment": 100,
  "reasoning": "金額3,300円は技術書1冊の典型値...",
  "decision_hint": "review_required"
}
```

このノードで prefill を選んだ理由は 3 つあります。

### 1. 出力スキーマが単一固定 + 文字列中心

分類タスクなので、入力に関わらず常に同じフィールドが返れば良い。`recommended_apportionment` だけ数値ですが 1 フィールドのみで、`JSON.parse` 後の `typeof` チェックで十分担保できます。

### 2. system プロンプトに詳細仕様が既にある

判定ロジック(`business_context`、判定例の Few-shot 4 件)が system プロンプト内に分厚く書かれているため、ここに「出力 JSON はこの形」を加えても自然です。Tool Use のスキーマ定義との重複定義を避けられます。

### 3. レイテンシ / コストの軽量化

Sample02 では未知ベンダーは全数 Classify に流れる設計です。呼び出し頻度がそこそこあるので、Tool 定義のオーバーヘッドを削れるなら削りたい。

レスポンスのパースは `'{' + content[0].text` を `JSON.parse` するだけ:

```js
// Format From AI ノード(n8n、runOnceForEachItem)
const text = '{' + (apiResponse.content[0].text || '');
const cls = JSON.parse(text);
```

`|| ''` は念のための防御です。`apiResponse.content[0].text` が undefined のとき、JavaScript は `'{' + undefined` を `"{undefined"` という文字列に暗黙変換します(`undefined` が文字列化されて連結される仕様)。どちらにせよ `JSON.parse` は失敗しますが、`"{"` の方がエラーログから「API レスポンスが空だった」と判別しやすい — というレベルのガードです。本質的な空応答対策は別途、リトライやスキップで設計してください。

## よくある誤用パターン

実装で踏みやすい誤用を 4 つ挙げます。

### (a) prefill で型担保したつもり

system に「出力は `amount` フィールドを number で」と書けば、人間的には型を指示したように見えますが、**Claude は自然言語の指示として解釈するだけ** です。`amount: "895"`(文字列)で返ってくることが普通にあります。

対策: 型担保が必要なら Tool Use を使うか、prefill 採用なら受信側で `Number(data.amount)` 強制変換 + `isNaN` チェックを書く。

### (b) Tool Use の過剰利用でレイテンシ悪化

Classify のような「シンプルな分類タスク」で Tool Use を使うと、Tool 定義の往復で 200〜500ms レベルのレイテンシ増があります。高頻度呼び出しでは効いてきます。

対策: 「型担保が要らない」「スキーマ単一固定」「呼び出し頻度高い」の 3 条件を満たすなら prefill 検討。

### (c) `JSON.parse` 失敗時のリカバリ未設計

prefill は `JSON.parse` 失敗すると後段で例外。Tool Use もスキーマ違反で `stop_reason` が `tool_use` 以外で返ることがあります。

対策: 失敗時のフォールバック(リトライ / スキップ + ログ / human-in-the-loop)を設計段階で決める。Sample02 では「Extract 失敗 → Slack に要レビュー通知 + スキップ」設計にしています。

### (d) prefill の応答に `{` が含まれていると勘違い

prefill した `{` は **応答に含まれない仕様** です。受信側で `'{' + response.content[0].text` と接頭辞を補完しないと、不正な JSON になります。公式ドキュメントには書いてあるけれど、初回実装で必ずハマるポイントです。

## まとめ + 関連話題

- Tool Use と prefill は **型担保とレイテンシのトレードオフ**。型担保が要るなら Tool Use、要らないなら prefill 検討
- 判断は **後段の処理(数値計算 / 日付比較 / enum 分岐)が要るか** が最大の分岐点
- prefill は「`{` が応答に含まれない」「型担保ゼロ」の 2 点で誤用を踏みやすい

関連話題(別記事化候補):

- **prompt caching**: 同じ system プロンプトを繰り返し送る場合、cache hit でトークン消費を大幅削減できる。Tool Use のスキーマ部分も cacheable
- **Few-shot**: prefill 採用時、出力品質の保守的バイアス(「商品名が取れない → 按分率 0% にしておこう」のような保守化)を Few-shot で補正できる。Sample02 Classify では 4 例埋め込みで解消

Sample02 のソース一式は [aiflowlab/n8n-claude-samples](https://github.com/aiflowlab/n8n-claude-samples) で MIT 公開しています。実際に動かしたい・自分のプロジェクトに合わせて改造したい方はどうぞ。

## アイデア・フィードバック募集

「自分のプロジェクトでこういうケースに迷っている」「この判断軸が抜けている」「Sample02 のこの実装、自分ならこうする」といった声を [GitHub Issues](https://github.com/aiflowlab/n8n-claude-samples/issues) または X DM([@aiflowlab](https://x.com/aiflowlab)) で気軽にどうぞ。次の Sample 設計に取り込ませてもらいます。
