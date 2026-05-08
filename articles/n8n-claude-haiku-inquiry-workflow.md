---
title: "Claude Haiku 4.5 + n8n で問い合わせ対応ワークフローを作ったら、100% 精度・1 件 0.5 円で運用できた"
emoji: "📨"
type: "tech"
topics: ["n8n", "claude", "anthropic", "automation", "python"]
published: false
---

## はじめに

Claude API + n8n で何が作れるかの実例として、**問い合わせメール対応ワークフロー** を実装しました。10 件のテストケースで 4 軸分類すべて 100% 一致、1 件あたり約 0.5 円で運用できる構成です。

![分類精度テストの実行結果](https://static.zenn.studio/user-upload/0bb11ada9b6a-20260508.png)

ワークフローの中身は次の 3 段構えです。

1. **4 軸で分類**: `category` / `urgency` / `sentiment` / `recommended_response_deadline` を Claude に判定させる
2. **緊急度別にドラフト生成**: 高(謝罪先行) / 中(受領確認) / 低(感謝返礼)の 3 ルートでトーンを書き分け
3. **Slack 通知**: 緊急度別の絵文字つきで、元メッセージと AI ドラフトを並べて投稿

本記事では、設計判断・プロンプト本文・n8n の分岐ロジック・Slack 整形までを、コードと一緒に共有します。ソース一式は [aiflowlab/n8n-claude-samples](https://github.com/aiflowlab/n8n-claude-samples) で MIT 公開済みです。

## ワークフロー全体像

10 ノード構成です。Webhook で受信 → Claude で分類 → 緊急度で 3 ルートに分岐 → 緊急度別ドラフト生成 → Slack 通知 → 呼び出し元へ JSON 返却、という流れになっています。

![n8n キャンバス全景](https://static.zenn.studio/user-upload/7c3988e633b3-20260508.png)

入口の Webhook は `POST /webhook/sample01` で、JSON ボディ(`sender_name` / `sender_email` / `subject` / `body`)を受けます。出口は Slack 通知に加えて、呼び出し元にも分類結果 + ドラフトを返す二重出力にしておくと、別システムから呼び出すときに使い回しやすくなります。

## 3 つの設計判断

実装で意識した設計判断を 3 つだけ挙げます。

### 1. モデルは Haiku 4.5 を採用

問い合わせの分類は決められた enum への振り分けで、創造性より一貫性が要求されるタスクです。Sonnet / Opus はオーバースペックなので、Haiku 4.5 で十分です。料金は入力 $1 / MTok、出力 $5 / MTok($1 = 約 156 円)。10 件で約 2 円、1 件あたり約 0.2 円で分類が回せました(ドラフト生成込みでも約 0.5 円)。

### 2. 分類とドラフト生成を別々のノードに分けた

「分類して、その内容に合わせてドラフトも書いて」を 1 回の API 呼び出しで済ませるのは魅力的に見えますが、System プロンプトが肥大化して各タスクの精度が落ちます。今回は **分類 1 ノード + ドラフト 3 ノード(緊急度別)** という構成にしました。

メリットは 3 つあります。

- 各タスクのプロンプトを個別に最適化できる
- どの段階で間違えたかが切り分けやすい
- ドラフト生成は緊急度別に呼ぶので、緊急じゃない問い合わせのドラフトは「呼ばない」選択もできる

### 3. urgency 判定は「迷ったら 1 段階上」

業務影響として **対応漏れ(false negative)** のほうが「不要に急いだ(false positive)」より痛い、という前提に倒しました。プロンプトに明示的に書いておくと、境界事例での判定が安定します。

## 分類プロンプトの中身

主役のプロンプトです。System に 4 軸の enum を完全列挙し、JSON 出力を強制します。

![分類ノードのプロンプト前半](https://static.zenn.studio/user-upload/9e3b0fe7b4a3-20260508.png)

![分類ノードのプロンプト後半](https://static.zenn.studio/user-upload/7c9ec4c8fec6-20260508.png)

System プロンプトを抜粋します(完全版は [リポジトリの classify.md](https://github.com/aiflowlab/n8n-claude-samples/blob/main/samples/sample01/prompts/classify.md))。

```text
あなたは BtoB 企業の顧客対応窓口で、受信した問い合わせメールを分類するアシスタントです。

入力された問い合わせを以下の 4 軸で分類し、必ず JSON のみを出力してください。

<分類軸>
- category: "質問" / "苦情" / "見積もり依頼" / "予約・問い合わせ" / "その他"
- urgency: "高" / "中" / "低"
- sentiment: "ポジティブ" / "ニュートラル" / "ネガティブ"
- recommended_response_deadline: "2時間以内" / "24時間以内" / "3営業日以内"
</分類軸>

<判定の指針>
- urgency と recommended_response_deadline は対応関係を持たせる(高→2時間以内 等)
- 苦情・トラブル系は sentiment がネガティブで urgency が高くなる傾向
- 判断に迷ったら 1 段階上の緊急度を採用(false negative を避ける)
</判定の指針>
```

ポイントは次の 4 つです。

1. **XML タグで入力を構造化**: 公式ガイドの "Use XML tags" に従って各フィールドを明示
2. **Assistant prefill `{` で JSON 強制**: 前置きや説明文の混入を防ぐ
3. **`temperature: 0`**: 同一入力 → 同一出力を担保
4. **enum 値の完全列挙**: 「至急」「緊急」のような表記ゆれを防ぐ

API 呼び出しの実体は次のような形です。

```python
payload = {
    "model": "claude-haiku-4-5-20251001",
    "max_tokens": 256,
    "temperature": 0,
    "system": SYSTEM_PROMPT,
    "messages": [
        {"role": "user", "content": user_prompt},
        {"role": "assistant", "content": "{"},
    ],
}
```

返ってきた `text` の頭に prefill した `{` を結合してから `json.loads` すれば、ほぼ確実にパースできます。

## n8n の Switch ノードで 3 ルートに分岐

分類結果の `urgency` を見て、後段のドラフト生成ノードを高 / 中 / 低の 3 ルートに振り分けます。

![Switch ノードの 3 ルール](https://static.zenn.studio/user-upload/00f23417f96d-20260508.png)

Rules モードで `{{ $json.classification.urgency }}` を文字列等価で比較し、Output Name を「高」「中」「低」に命名します。各 Output に Draft Urgent / Normal / Simple の 3 ノードを接続するだけで、後続のドラフト生成ノードは「自分の役割の入力」だけを受け取る設計になります。

`switch by urgency` のような単純な振り分けは、コード(Function ノード)で書くより Switch ノードに寄せたほうが、後から見た人に意図が伝わりやすいです。

## ドラフト生成プロンプト(緊急度別)

ドラフト生成は緊急度別に 3 つの System プロンプトを使い分けます。

| ルート | トーン |
|---|---|
| Urgent(高) | 謝罪先行 → 本日中の具体アクション → 代替案提示 |
| Normal(中) | 受領確認 → 不足情報の質問 → 1 営業日以内の見積予告 |
| Simple(低) | 感謝への返礼 → 簡潔な事実回答 |

「1 つのプロンプトで `if urgency == 高 then ...` のように条件分岐させる」案も検討しましたが、品質が安定しません。**人間に書かせるなら別人に振り分けるレベルの差** を、プロンプトでも分けたほうが結果的にうまくいきます。

3 ルート分のプロンプト本文は [draft.md](https://github.com/aiflowlab/n8n-claude-samples/blob/main/samples/sample01/prompts/draft.md) で公開しています。

## Slack 通知の整形(mrkdwn)

Slack 投稿は Code ノードで mrkdwn を組み立てます。緊急度別の絵文字(`:red_circle:` / `:large_yellow_circle:` / `:large_green_circle:`)を冒頭に付け、元メッセージとドラフトを並べて投稿します。

![Slack 通知の 3 色並び](https://static.zenn.studio/user-upload/a1ec2d9dab43-20260508.png)

絵文字は実運用でも効きます。チャンネルを開いた瞬間に「赤丸が並んでる = 急ぎ案件溜まってる」が一目で分かるので、トリアージのコストがほぼゼロになります。

## 検証結果(精度・トークン・コスト)

`scripts/test_classify.py` で 10 件のテストケースを実行した結果です。

| 指標 | 値 |
|---|---|
| All-fields match | **10 / 10(100%)** |
| 入力トークン合計 | 9,995 |
| 出力トークン合計 | 541 |
| 推定コスト | **$0.0127(約 2 円)** |
| 1 件あたり(分類のみ) | 約 0.2 円 |

10 件には次のような **境界ケース** を含めましたが、すべて期待通りに分類されました。

- 短文だけのメール(「資料ください」のみ)
- カタログ請求(穏やかだけど低緊急)
- マイルドな批判(怒鳴ってないネガティブ)
- 苦情と見積依頼の混在(後述)

混在ケースは特に印象的だったので、Slack 出力の実例を載せておきます。

![苦情 + 見積依頼の混在ケース](https://static.zenn.studio/user-upload/fb9a6a3abf64-20260508.png)

「先月納品物の誤字を指摘 + 来期分の見積依頼」の 1 通を、ドラフト側で **謝罪 + 見積を翌日午前中に送る** という 2 アクションに展開してくれました。人間オペレータが急いでさばくと前半の苦情だけに反応して見積依頼を見落としやすいパターンなので、ここを LLM で拾えるのは実用上のメリットが大きいです。

## ハマったポイント

n8n 運用で踏んだ落とし穴を 3 つ。

### 1. import:workflow しただけでは active にならない

`docker exec` 経由で `n8n import:workflow --input=...` を流しても、ワークフローは **Draft 状態で入る** だけです。Webhook を受けるには n8n UI 上で **Unpublish → Publish** を踏む必要がありました。CLI で active 化するオプションは現状無さそうなので、再インポートのたびに UI 操作が要ります。

### 2. `$env` を参照したいなら `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`

ワークフロー JSON で Slack Webhook URL を `$env.SLACK_WEBHOOK_URL_SAMPLE01` のように参照するなら、Docker 起動時の環境変数に `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` を入れる必要があります。デフォルトの true だと Code ノード等から `$env` 参照がブロックされます。

### 3. ターミナルの末尾出力が消えることがある

`python3 scripts/test_classify.py` の最終行(SUMMARY)が WSL 上で稀に消えます。`python3 -u`(unbuffered)で実行するか、`clear` してから再実行で出ました。スクショ撮影の直前で詰まったので共有しておきます。

## まとめ

- **Haiku 4.5 + n8n で 1 件 0.5 円・精度 100%** の問い合わせ対応ワークフローが組めました
- **分類とドラフトを分離** + **緊急度別にプロンプトを分ける** 設計が、品質の安定に効きました
- ソース一式は [aiflowlab/n8n-claude-samples](https://github.com/aiflowlab/n8n-claude-samples) に置いてあります(MIT)。プロンプト・テストケース・ワークフロー JSON 全部入りです

「自分の業務にも当てはめたい」「この設計判断、別の選択肢ない?」といった質問・突っ込みがあれば、X([@aiflowlab](https://x.com/aiflowlab))の DM までお気軽にどうぞ。
