---
title: "Lancers + SMBC の入金検知を Claude API + n8n で組む — 落とし穴 4 点と E2E で見えた構造的限界"
emoji: "💰"
type: "tech"
topics: ["n8n", "claude", "anthropic", "automation", "accounting"]
published: true
---

## はじめに

Claude API + n8n で動かしている経費自動化ワークフロー([前回記事](https://zenn.dev/aiflowlab/articles/n8n-claude-credit-expense-automation))に **入金検知機能**を追加しました。経費編と同じノリで組めると思っていたら、設計の難度が一段上で、4 つの落とし穴に当たりました。

本記事はその設計編です。実装に踏み込む前に、Phase 1.5(入金機能)の設計過程で何が問題になり、どこを諦め、どこを技術で殴れたかを記録します。実装ノードや n8n キャンバスの詳細は次回「実装編」で扱います。

題材は **Lancers / CrowdWorks(プラットフォーム経由入金)+ SMBC 明細表示あり(銀行入金通知)** の組み合わせです。E2E 検証は 8/8 PASS、API コストは Phase 1 + Phase 1.5 合計で約 9 円(Claude Haiku 4.5)。ただし「8/8 PASS」のうち 1 件は **意図的に通した構造的限界の再現テスト**で、これが設計で一番悩んだ箇所です。

なお本記事の業務向け振り返りを [note 設計振り返り記事](https://note.com/aiflowlab/n/n7e6652bb91ef)に出しています。読者ターゲットが違うので、本記事は技術詳細(スキーマ・分岐ロジック・E2E 結果)に絞ります。

---

## 全パターン網羅を捨てる — 入金パターン A/B/C の分類

入金検知の最大の難しさは、**ユーザーごとに入金ルートが大きく違う**ことです。利用銀行 × メール通知設定 × 利用プラットフォーム × 直接案件比率の組み合わせで分岐します。

設計の最初に行ったのは「全パターン網羅は捨てる」決定でした。入金パターンを以下の 3 つに分類し、Phase 1.5 では A + C を主スコープに置きます。

| パターン | 銀行メール本文 | プラットフォーム通知 | 該当例 | Phase 1.5 対応 |
|---|---|---|---|---|
| **A** | 金額 + 振込元あり | あり or なし | SMBC 明細表示あり ユーザーへの入金 | ✓ 銀行メール extraction で対応 |
| **B** | 金額なし(タイミングのみ)| なし | 本業給与 / 個人事業の直接クライアント案件 | ✗ Sample03 候補へ送り |
| **C** | 金額なし or 銀行メール自体なし | あり(Lancers/CW)| 個人事業主の主要ルート | ✓ プラットフォームメール extraction で対応 |

B が Phase 1.5 でカバーできない理由は技術ではなく **入力データそのものが構造的に不足している**ことです。たとえば三菱UFJ銀行・みずほ銀行・SMBC 明細表示なし ユーザーは個人向けの入金通知メールが「ログインして確認してください」しか書いていない、あるいはそもそもメールが届かない。本文に金額がない以上、抽出 LLM をどれだけ強化しても解決しません。

このセグメントは別 Sample(請求書発行 + 月次マッチング駆動の Sample03 候補)で扱う設計にしました。**経費=リアルタイム、入金=月次** という非対称な経理サイクルが現実なので、別アーキテクチャで組む方が筋がいい、という判断です。

graceful degradation の設計も入れています。「銀行メール(A)は取れたら突合せに使う、取れなければスキップして C のみで運用」とすることで、利用銀行に関わらず Phase 1.5 が機能します。筆者の業務用口座が MUFG なので、自分自身が C のみで動くユーザーになっています。

---

## 落とし穴 1: 1 入金で 2 通のメールが届く(報酬確定 vs 振込完了)

Lancers / CrowdWorks では、1 件の入金につき **2 通のメール**が届きます。

- **報酬確定通知**(検収完了時、ジョブ ID + クライアント名 + 報酬総額 + 手数料 + 振込予定日 すべてあり)
- **振込完了通知**(実際の入金時、振込日 + 振込金額のみ。クライアント名・案件名は通常なし)

両方を ledger に記録すると同じ入金が 2 行になります。確定申告のときに二重計上で詰みます。

実装上は Tool Use の input_schema に `payout_kind` enum を入れて、抽出時点で `reward_confirmed` / `transfer_completed` を判別させました。

```json
{
  "name": "record_income",
  "description": "プラットフォーム入金通知メールから入金情報を抽出して記録する",
  "input_schema": {
    "type": "object",
    "properties": {
      "payout_kind": {
        "type": "string",
        "enum": ["reward_confirmed", "transfer_completed"]
      },
      "payment_date": { "type": "string" },
      "gross_amount_jpy": { "type": "number" },
      "fee_jpy": { "type": "number" },
      "net_amount_jpy": { "type": "number" },
      "client_name": { "type": "string" },
      "project_name": { "type": "string" },
      "platform_id": { "type": "string" }
    },
    "required": ["payout_kind", "payment_date", "gross_amount_jpy", "fee_jpy", "net_amount_jpy"]
  }
}
```

(`description` は省略表記、実物は [GitHub](https://github.com/aiflowlab/n8n-claude-samples) を参照)

下流の n8n Switch ノードで `payout_kind === "reward_confirmed"` のみを ledger 追記経路に流し、`transfer_completed` は Slack に「振込手続き完了の確認」だけ通知して帳簿は触らない、という分岐にしています。

税務的には「実現主義(検収完了時点で計上)」と「現金主義(入金時点で計上)」のどちらでも運用可能ですが、本サンプルは実現主義に統一しています。会計方針の切り替えは設定で可能にする想定で、Phase 1.5 ではハードコード。

Tool Use を選んだ理由は、後段で `payout_kind` の enum 値分岐 + 金額の数値計算 + 日付比較を行うため、JSON で型担保された出力が欲しかったからです。判断基準は [Tool Use と prefill の使い分け記事](https://zenn.dev/aiflowlab/articles/claude-tool-use-vs-prefill-selection)にまとめています。

---

## 落とし穴 2: A+C オーバーラップ — 重複検知が原理的に発火しない

SMBC 明細表示あり ユーザーが Lancers から入金を受けた場合、銀行メール(A)と Lancers メール(C)の **両方から同じ入金のデータが取れます**。両方を ledger に書くと、また二重計上です。

Phase 1 経費編から流用した重複検知ロジックがあります。

```text
突合せキー = (取引先半角カナ × 金額 × 日付 ±3 日)
```

これが Phase 1.5 では **構造的に発火しない**ことを E2E で確認しました(`e2e_06_overlap_AC` ケース、8/8 PASS のうちの 1 件)。理由は 3 つで、いずれも仕様の本質に由来します。

| 比較項目 | Lancers reward | SMBC bank income | 一致? |
|---|---|---|---|
| vendor | 株式会社サンプル(クライアント社名)| カ）ランサーズ(プラットフォーム運営の半角カナ)| ✗ |
| amount | 50,000(gross、手数料控除前)| 41,750(net、手数料控除後)| ✗ |
| date | 2026-05-15(検収完了日)| 2026-05-31(銀行入金日)| ✗(**16 日差**)|

3 つ目の日付差は特に重要で、原因は **実現主義と現金主義の会計上の立場の差そのもの**です。Lancers の reward の payment_date は検収完了日(税務上の確定日)、SMBC bank の payment_date は実際の口座入金日。Lancers は月末締めで振込実行されるため、両者は構造的に約 2 週間ズレます。

仮に金額と vendor が一致しても、現状の ±3 日窓では原理的に捕まりません。日付窓を 30 日に広げれば捕まりますが、今度は同月の別取引まで誤マッチします。

結論として、A+C の自動突合は **Phase 1.5 では諦め**、Phase 2(Slack 承認 UI + 取引先マスター突合)で対応する設計に変更しました。現状の振る舞いは、SMBC 銀行入金が来た時点で「net vs gross 差で重複漏れの可能性」警告 Slack を出し、人間レビューに回します。

この「構造的に解けないところを諦めて、警告 Slack で人間レビューに振る」設計は、AI 自動化を売り物にしている人がよく踏む地雷だと思います。「全部 AI で自動」と謳って、後で「あれ?」となる箇所はだいたいこの種類の話です。

---

## 落とし穴 3: 半角カナ表記の取引先名

銀行の振込入金通知では、振込依頼人名は **半角カナ表記**で届きます。

```text
内容: 振込  カ)ランサーズ
内容: 振込  カ)クラウドワークス
内容: 振込  カ)ヤマダーシヨウジ
```

「カ)」は「(株)」の振込業界での略式表記、続く片仮名は社名の振り仮名です。

人間が読めば「ランサーズ」「クラウドワークス」「ヤマダ商事」と即断できますが、自動マッピングするには **取引先マスター(半角カナ → 正式名称)が必須**になります。JavaScript の `String.prototype.normalize("NFKC")` で半角カナ自体は全角に正規化できますが、「カ)」表記を「(株)」に変換するロジックや、振り仮名から実名を引くロジックは別途必要です。

Phase 1.5 では取引先マスターの整備は将来課題として、現状は **半角カナを取引先名としてそのまま記録 + 備考に `⚠ 要レビュー` を入れて Slack 通知**する設計にしました。月末レビュー時に人間が正式名称に直す前提です。

ここでも「AI が下書き、人間が確定」のパターンを採用しています。

---

## 落とし穴 4: 手数料を別行で計上する — ledger スキーマ設計

Lancers のシステム手数料は 16.5%、CrowdWorks は 20%。月の収入が 10 万円なら、手数料だけで 1.6〜2 万円消えます。

報酬確定通知から受取額(net)だけを ledger に記録すると、確定申告時に手数料を経費計上できません。受け取った 41,750 円だけ書いても、税務上は「報酬総額 50,000 円から経費 8,250 円を引いた」という内訳が残らないからです。

解決として、1 入金につき **2 行を ledger に記録する**スキーマにしました。

| transaction_date | vendor | kind | amount_jpy | apportionment | tax_treatment | note |
|---|---|---|---|---|---|---|
| 2026-05-15 | 株式会社サンプル | 収入 | 50,000 | 100% | グロス計上 | platform_id: 1234567 |
| 2026-05-15 | Lancers | 経費(決済手数料) | 8,250 | 100% | 経費計上 | platform_id: 1234567 (16.5%) |

これで「報酬の総額」と「手数料の経費計上」が両方残ります。`platform_id` を共通の備考に入れることで、後から内訳監査するときに突合せられます。

実装は n8n Code ノードで 1 件の Tool Use 出力から 2 行のレコードを生成しています。`runOnceForEachItem` モードで `$input.item` 1 件あたり 2 行を return する形です。

```js
// Code node: split income into 2 ledger rows
const income = $input.item.json;
const ledgerRows = [
  {
    transaction_date: income.payment_date,
    vendor: income.client_name || income.platform_short_name,
    kind: "収入",
    amount_jpy: income.gross_amount_jpy,
    apportionment: 1.0,
    note: `platform_id: ${income.platform_id}`,
  },
  {
    transaction_date: income.payment_date,
    vendor: income.platform_short_name,  // "Lancers" or "CrowdWorks"
    kind: "経費(決済手数料)",
    amount_jpy: income.fee_jpy,
    apportionment: 1.0,
    note: `platform_id: ${income.platform_id} (${income.fee_rate_label})`,
  },
];
return ledgerRows.map(row => ({ json: row }));
```

`fee_jpy = 0` の場合(振込手数料無料月など)は経費行を生成しないように分岐する余地もありますが、Phase 1.5 ではあえて「0 円の経費行」を残してプラットフォームの請求書発行ロジックを可視化する方針にしています。確定申告時のチェック性を優先する選択です。

---

## ハマりポイント

設計とは別レイヤーで、n8n 固有のハマりが 3 つありました。記録のため。

### n8n Switch ノードの型判定

boolean を直接渡すと Switch の条件評価が安定しません。`String(value) === "true"` のように **文字列化してから比較**するのが安全です(n8n 2.18.7 / v3.2 仕様)。これは Phase 1 経費編でも踏んでいた地雷で、Phase 1.5 でも同じ作法を踏襲しました。

### Gmail Trigger の Simplify: false 必須

Gmail Trigger のデフォルト設定では Simplify が ON になっていて、本文(`text`)が返ってきません。Lancers/CW の通知メールは本文に金額・案件 ID が全部入っているので、Simplify を false にしないと抽出元のデータが消えます。Sample 全体の Gmail Trigger を Simplify: false で統一しています。

### Code ノードの multi-item 取りこぼし

Phase 1 で 1 度踏んだ地雷ですが、`runOnceForAllItems`(default)+ `.first()` のコードを書くと、Gmail Trigger が複数メールを一度に返したときに 1 件しか処理されません。`runOnceForEachItem` + `$input.item` で書くか、`runOnceForAllItems` のまま `$input.all()` を使うか、どちらかで対応します。Phase 1.5 の追加 Code ノードは全て `runOnceForEachItem` で書いています。

詳細は別途まとめた [n8n Code ノードの multi-item 対応に関するメモ](https://github.com/aiflowlab/n8n-claude-samples)(GitHub repo)を参照してください。

---

## E2E 検証 — 8/8 PASS の内訳

設計検証の最後に、Phase 1.5 専用の E2E 計画で 8 ケースを実走しました。

| # | ケース | 期待動作 | 結果 |
|---|---|---|---|
| 1 | Lancers 報酬確定 | ledger 2 行(収入 50,000 + 手数料 8,250)+ 入金記録 Slack | ✅ PASS |
| 2 | Lancers 振込完了 | ledger 0 行 + 振込完了確認 Slack | ✅ PASS |
| 3 | CrowdWorks 報酬確定 | ledger 2 行(収入 80,000 + 手数料 16,000) | ✅ PASS |
| 4 | CrowdWorks 振込完了(振込手数料 500)| ledger 0 行 + 振込完了確認 Slack | ✅ PASS |
| 5 | SMBC 公式サンプル | ledger 1 行(その他収入、半角カナ vendor、⚠要レビュー)| ✅ PASS |
| 6 | A+C オーバーラップ | 両方記録(意図的限界の再現) | ✅ PASS |
| 7 | Lancers 報酬確定 ×2 重複 | 2 回目は ±3 日窓で重複検知スキップ | ✅ PASS |
| 8 | 経費メール(Phase 1 回帰) | Anthropic 1 行追記、Phase 1 挙動維持 | ✅ PASS |

ケース 6 が前述の「構造的限界の再現」テストです。A+C 両方記録される設計を明示的に受け入れ、Slack 警告で人間レビューに振るのが正しい動作、として PASS 判定にしています。

ケース 7 の重複検知は同一プラットフォーム内(C 経路内)なら ±3 日窓で正しく発火します。問題は A↔C 間のクロス検出で、これは前述の通り Phase 2 送り。

ケース 8 は Phase 1(経費)既存挙動の回帰テスト。Phase 1.5 の追加で Phase 1 経費が壊れていないことを確認します。Phase 1 / Phase 1.5 を同一ワークフロー内で共存させた(別ワークフローに分けなかった)ため、追加するたびに既存経路を壊さないかを E2E で都度確認する設計にしています。

---

## まとめ

設計編で見えてきた原則を 4 つ。実装の前にこれを言語化できたのが、E2E まで 1 週間でたどり着いた理由だと思います。

- **「全パターン網羅」は捨てる**: 入金ルートはユーザーごとに違いすぎる。網羅性より「典型 2-3 ルートでの参考実装 + 拡張前提」のほうが設計が前に進む
- **AI で完璧を狙わず、人間レビューを前提に組む**: 半角カナ突合せ、A+C オーバーラップ、銀行入金の業務性判定は、いずれも Slack 警告で人間に振る
- **「同じお金が複数経路で来る」を前提にする**: 1 ルート前提だと二重計上する。さらに、実現主義(検収日)と現金主義(入金日)の会計立場の差そのものが日付ラグを生むので、単純な日付窓では捕まらない
- **手数料は別行計上**: 受取額だけ記録するより内訳が残る。確定申告時のチェック性が大きく変わる

次回は実装編として、18 ノード構成の n8n キャンバスと、各ノードの実装ディテール(分岐ロジック、bank_income の Tool 定義、ledger 行生成の Code ノード詳細)に踏み込みます。

---

## 関連リソース

- 経費編(Phase 1): [Claude Haiku 4.5 + n8n でクレカ明細メールを自動経費記録した](https://zenn.dev/aiflowlab/articles/n8n-claude-credit-expense-automation)
- Tool Use と prefill の使い分け: [判断チェックリストと Sample02 での選定例](https://zenn.dev/aiflowlab/articles/claude-tool-use-vs-prefill-selection)
- 業務向け振り返り(note): [「経費の次は入金」を始めたら、設計の壁に 4 つ当たった話](https://note.com/aiflowlab/n/n7e6652bb91ef)
- コード一式(MIT): [aiflowlab/n8n-claude-samples](https://github.com/aiflowlab/n8n-claude-samples)

質問・改善提案は GitHub Issues か X DM([@aiflowlab](https://x.com/aiflowlab))までどうぞ。
