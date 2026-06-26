---
name: sns-cycle
description: X(Twitter)投稿パイプラインの /loop 1サイクルを統括するスキル。Post Analyst→Trend Scout→Draft Writer→/goal達成判定→Typefullyキュー登録→Thumbnail生成→Slack承認依頼までを1回実行する。/loop 1d /sns-cycle で日次自動巡回。サブエージェント(trend-scout/draft-writer/thumbnail-maker/post-analyst)を Agent ツールで起動し、/goal の採点・KPI集計・外部API(OpenAI Image / Typefully / Slack)呼び出しを自分で行う。
---

# /sns-cycle — 投稿パイプライン1サイクル統括（/loop × /goal）

設計の `/loop`（量を出す自動巡回）と `/goal`（基準で絞る判定）を**1回ぶん**実行する統括スキル。
**コードは持たず**、サブエージェント呼び出し・採点ロジック・外部API curl をこの手順で組み立てる。

> 安全原則：**自動公開しない**。Typefully には「下書き」を作るだけ。X への公開は人間が Typefully 上で行う（「予約まで自動・公開だけ承認」）。

---

## 0. 準備

1. **環境変数を読む**（API 呼び出しの前に必ず）。プロジェクト直下に `.env` があれば読み込む：
   ```bash
   set -a; [ -f ./.env ] && . ./.env; set +a
   ```
2. 動作モードを決める：
   - `LIVE=1` かつ `TYPEFULLY_API_KEY` あり → **LIVE**（Typefully に下書き登録・反応計測する）
   - それ以外 → **DRY**（下書き案と承認依頼のみ。計測/KPI/Typefully登録はスキップ）
   - `OPENAI_API_KEY` あり → サムネ画像を生成。無ければ画像はスキップしプロンプトのみ残す。
3. `state/goal.json`（ゴール単一ソース）を Read。`account` がプレースホルダ（`@your_handle` 等）のままなら、その旨を出力に注記して続行（DRY 想定）。
4. 実行IDを決める：`<YYYY-MM-DD>-<manual|loop>`（同日複数なら連番）。
   生成物の保存先 `workflow/runs/<実行ID>/` を用意する。

---

## 1. アウターループ：Post Analyst（反応分析）

- **LIVE のとき**：Typefully Analytics API から直近公開分の反応（impressions / engagement / 時刻 / 形式）を取得し、`post-analyst` サブエージェントに渡して起動。
  - 戻り JSON の `recommended_mode`（exploit/explore）を受け取る。
  - Analyst は `state/playbook.md` に学びを追記する。
- **DRY のとき**：計測はスキップ。`recommended_mode` は KPI 履歴が無いので **exploit**（実証済みフォーマット重視）を既定とする。
- Typefully の認証ヘッダ名・メトリクスのフィールド名は実運用前に公式ドキュメントで最終確認（README の TODO）。

---

## 2. /loop 前段：Trend Scout → Draft Writer

### 2.1 Trend Scout（ネタ収集）
- `trend-scout` を Agent ツールで起動。渡す情報：
  - `mode` = §1 の `recommended_mode`
  - 必要件数 = `goal.json.policy.posts_per_day × 1.5`（最低 3、設計の「1サイクル3案生成」に合わせる）
- 戻り JSON の `ideas`（スコア付き候補）を受け取る。

### 2.2 Draft Writer（下書き作成）
- Scout の `ideas` 全件（または上位数件）を `draft-writer` に渡して起動。渡す情報：
  - 採用ネタ、`goal.json.voice`
  - **重複回避参照**：LIVE なら Typefully の直近公開/予約済みテキストを渡す。DRY なら無し（`no-reference`）。
- 戻り JSON の `drafts`（投稿テキスト）を受け取る。

---

## 3. /goal 達成判定（採点 → 絞り込み）

各 draft を、元ネタ（Scout の scores）と KPI 進捗で採点する。

### 3.1 fit_score（0–1）
```
fit = 0.4 * goal_fit + 0.2 * voice + 0.2 * freshness + 0.2 * (1 - risk)
```
- `goal_fit / voice / freshness / risk` は Scout の該当 idea の `scores` を使う。
- **mode 加点**（README の運用）：
  - **exploit** → `format` が `tips` または `thread` の draft に +0.05
  - **explore** → `freshness` が高い（≥0.7）draft に +0.05
- 文字数オーバー（どれかの tweet が 280 超）や重複フラグが立つ draft は除外（または減点して下位へ）。

### 3.2 選抜
- **しきい値 0.6 以上**を承認候補とする。
- そのうち上位 `goal.json.policy.posts_per_day`（既定 2）件を「承認へ送る対象」に選ぶ。
- 0.6 未満／本数オーバー分は **保留（HOLD）**。silently に捨てず、出力に「保留 N 件」と明記する。
- KPI 補正：直近 engagement_rate が目標帯を下回る→exploit 寄り（既に §1 で mode 反映済み）。

---

## 4. サムネイル生成（承認対象のみ）

> ⚠️ **実測で確定した方針：`images/generations` を使う。`images/edits`（参照画像つき編集）は使わない。**
> `edits` で参照画像 `character.png` を渡すと、日本語タイトルが「今すぐチャンネル登録してね」等の定番フレーズに化けて**投稿内容と一致しない**。`generations` に日本語で文字を明示指定すると**指定どおりの日本語が正確に描かれる**（ChatGPT UI と同じ）。
> 代償としてマスコットの参照画像は使えないため、`thumbnail-maker` が `goal.json.thumbnail.character` を**文章でプロンプトに書き起こして**再現する。

承認へ送る draft それぞれについて：

1. `thumbnail-maker` を起動し、`goal.json.thumbnail` を反映した**日本語の生成プロンプト**（`items[].prompt` / `title_lines` / `subcopy` / `size`）を得る。タイトルは必ず投稿内容を表す言葉であること。
2. `OPENAI_API_KEY` があれば OpenAI Image API（**generations**）を呼び、`workflow/runs/<実行ID>/thumb-<n>.png` に保存する。
   - **jq は環境に無い前提。Python で b64_json を抽出する。**
   - **生成サイズのまま保存し、リサイズ・トリミングは一切しない。**
   - 失敗してもサイクルは止めず、画像スキップ＋プロンプト保存で続行。`gpt-image-2` が一時的に HTTP 520 を返すことがあるので最大3回リトライ。
   - 例（プロンプトは一時ファイルに書いてから渡す。`$PROMPT_FILE` / `$OUT` / `$N` は適宜置換）：
     ```bash
     python - "$PROMPT_FILE" "$OUT/thumb-$N.json" "$OUT/thumb-$N.png" <<'PY'
     import sys, json, base64, os, urllib.request, urllib.error
     prompt = open(sys.argv[1], encoding='utf-8').read()
     body = json.dumps({"model":"gpt-image-2","prompt":prompt,
                        "size":"1536x1024","n":1,"quality":"high"}).encode()
     req = urllib.request.Request("https://api.openai.com/v1/images/generations",
         data=body, headers={"Authorization":"Bearer "+os.environ["OPENAI_API_KEY"],
                             "Content-Type":"application/json"})
     try:
         d = json.loads(urllib.request.urlopen(req, timeout=300).read())
         open(sys.argv[3],'wb').write(base64.b64decode(d['data'][0]['b64_json']))
         print("saved", sys.argv[3])
     except urllib.error.HTTPError as e:
         print("HTTP", e.code, e.read()[:300].decode('utf-8','replace'))
     PY
     ```
   - モデル/サイズは `goal.json.thumbnail` の値を優先。`size` を API が拒否したら横長の対応サイズ（`1536x1024`）にフォールバック。
3. `OPENAI_API_KEY` が無い → 画像はスキップし、プロンプトだけ `drafts.md` に残す。
4. 生成後は画像を見て、**タイトル文字が投稿内容と一致しているか**を必ず確認する。化けていたら同じ generations で1〜2回再生成する。

---

## 5. Typefully キュー登録（LIVE のみ）

- **LIVE のとき**：承認対象の各 draft を **Typefully API で下書き（draft）として作成**する。
  - スレッドは Typefully のスレッド分割で登録。任意で予約時刻（`schedule-date`）を付与。
  - 下書きは**人が公開操作するまで X に出ない** = これが承認待ち状態の実体。
  - 認証ヘッダ名は公式ドキュメントで確認（README TODO）。例：
    ```bash
    curl -s https://api.typefully.com/v1/drafts/ \
      -H "X-API-KEY: $TYPEFULLY_API_KEY" \
      -H "Content-Type: application/json" \
      -d '{"content":"<本文（スレッドは改行4つで分割）>","threadify":false,"schedule-date":null}'
    ```
  - 返ってきた draft の URL/ID を控え、Slack 通知のディープリンクに使う。
- **DRY のとき**：Typefully 登録はスキップ。承認依頼には「下書き案（ローカル保存）」のみ載せる。

---

## 6. 生成物の保存（毎回）

`workflow/runs/<実行ID>/` に必ず残す（レビュー用の単一の置き場）：
- `drafts.md` — 承認対象＋保留の下書き本文、fit_score、mode、（あれば）Typefully リンク、サムネプロンプト
- `thumb-<n>.png` — 生成できたサムネイル

加えて `workflow/progress/<YYYY-MM-DD>.md` に1行サマリを追記：
- 例：`- <実行ID> mode=exploit 生成3 承認2 保留1 thumb=2 typefully=LIVE`

---

## 7. 公開承認依頼（Slack）

- 接続済み Slack MCP で `goal.json.notify.slack_channel`（既定 `#sns-approvals`）へダイジェスト送信。
  - 内容：承認対象の下書き本文（要約）＋ fit_score ＋ **Typefully ディープリンク**（LIVE）＋ サムネ有無。
  - 文面は「公開は Typefully 上で人間が実行してください」と明記。
- **無応答時の挙動**：`goal.json.notify.no_response`（既定 `HOLD`）に従い **自動公開しない**。
  鮮度切れネタは関連ウィンドウ経過で失効/繰越。時間依存ネタは予約枠接近時にリマインド。
- Slack MCP が使えない場合は、承認依頼ダイジェストを出力本文に表示してフォールバック。

---

## 8. サイクル完了サマリ（ユーザーへの出力）

最後に簡潔に報告する：
- mode（exploit/explore）と判断根拠（KPI 対比 or DRY 既定）
- 生成 N / 承認 M / 保留 K 件、各承認 draft の fit_score
- Typefully 登録：LIVE/DRY、登録できた件数とリンク
- サムネ：生成 / スキップ（理由）
- 保存先 `workflow/runs/<実行ID>/` のパス
- 次サイクルへの申し送り（playbook に追記した学び、繰越ネタ）

---

## モード別チェックリスト

| 項目 | DRY (LIVE!=1 or キー無) | LIVE |
|---|---|---|
| Post Analyst 計測 | スキップ（mode=exploit 既定） | Typefully Analytics → 分析 → playbook 追記 |
| 重複参照 | no-reference | Typefully 直近公開を Writer に渡す |
| Typefully 登録 | スキップ | draft 作成（公開はしない） |
| サムネ生成 | OPENAI_API_KEY 次第 | OPENAI_API_KEY 次第 |
| Slack 承認依頼 | 下書き案のみ | 下書き＋Typefully リンク |
| 自動公開 | しない | **しない** |
