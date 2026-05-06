# Threads投稿自動化システム

Claude Code (web版) + Routines で、毎日Threadsの投稿案を3件自動生成し、
Googleスプレッドシートに記録する仕組みです。
スプレッドシートに記録された投稿文は、別途設定したGAS(Google Apps Script)が
時刻トリガーで読み込み、Threadsへ自動投稿します。

## 全体フロー

```
[Claude Code Routine] (毎日18:00など)
    ↓
1. 競合調査スキル(Xから人気投稿を分析)
    ↓
2. バズる投稿文作成スキル(3案生成)
    ↓
3. スプレッドシート記録スキル(翌日分として「未着手」で登録)
    ↓
[Googleスプレッドシート]
    ↓ (ステータスを「投稿予定」に手動変更 or 自動承認)
    ↓
[GAS時刻トリガー] (07:00 / 12:00 / 19:00)
    ↓
[Threadsへ自動投稿] → ステータスを「投稿完了」に更新
```

## ディレクトリ構成

```
threads-routine/
├── config/
│   └── settings.yaml          # ← テーマ・スプレッドシートURLなどを編集
├── skills/
│   ├── competitor-research/   # 競合調査スキル
│   ├── viral-post-writer/     # 投稿文生成スキル
│   └── sheet-logger/          # スプレッドシート記録スキル
└── output/                    # 実行結果(自動生成)
    ├── research/
    └── posts/
```

---

## セットアップ手順

### STEP 1. このリポジトリをGitHubに作成

Claude Code (web版)はGitHubリポジトリと連携するため、まずGitHubに本フォルダをpushしてください。

### STEP 2. 設定ファイルを編集

`config/settings.yaml` を開いて以下を変更:

- `theme.genre` などテーマ情報(ジャンル変更はここだけ)
- `spreadsheet.url` に既存スプレッドシートのURL貼り付け
- `research.keywords` にX検索したいキーワード

### STEP 3. スプレッドシートの準備

記録先スプレッドシートを開き、1行目に以下のヘッダーを入れておく:

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| 投稿予定日 | 投稿予定時間 | プラットフォーム | 投稿テキスト案 | 画像・動画リンク | ステータス |

スプレッドシートの共有設定は **「制限付き(自分のみ)」のまま** でOKです。
共有リンクをONにする必要はありません。

### STEP 4. Google Drive コネクタの認証【推奨方式】

スプレッドシートを **非公開のまま** Routineから書き込ませるには、
**Google Drive コネクタ** を使うのが最も簡単・安全です。

#### なぜこの方式が推奨か
- スプレッドシートを公開する必要がない(自分のみアクセス可のまま使える)
- サービスアカウントのJSONキー管理が不要
- 認証情報がGitHubに漏れるリスクがない
- 1クリックで設定完了

#### 設定手順

1. **claude.ai の Settings → Connectors を開く**
2. **「Google Drive」を有効化** し、Googleアカウントで認証(ブラウザで「許可」を押す)
3. 認証完了後、Routine作成時に **Connectors欄で「Google Drive」にチェック**

これだけで、Routineは **あなた本人として** スプレッドシートにアクセスできます。
スプレッドシートの共有設定は変更不要です。

### STEP 5. Routine作成 (claude.ai/code/routines)

「New routine」から以下を設定:

**プロンプト:**
```
config/settings.yaml の設定に従って、以下を順に実行してください。

1. competitor-research スキルを実行し、Xでキーワード検索を行って
   人気投稿のパターンを output/research/ に保存

2. viral-post-writer スキルを実行し、調査結果を元に
   Threads投稿案を3件生成して output/posts/ に保存

3. sheet-logger スキルを実行し、生成された3件を
   設定ファイルで指定されたGoogleスプレッドシートに記録

実行後、生成結果のサマリーをコミットしてpushしてください。
失敗時は output/error.md にエラー内容を記録してください。
```

**リポジトリ:** 本リポジトリを選択

**コネクタ:** Google Drive を有効化(STEP 4で認証済み)

**トリガー:** Schedule → 毎日 18:00 JST など (翌日朝7時投稿の前日に生成)

### STEP 6. 動作確認

Routine作成後、まず手動で「Run now」して動作確認:
- `output/research/YYYY-MM-DD.md` が作られるか
- `output/posts/YYYY-MM-DD.md` に3案あるか
- スプレッドシートに3行追加されるか(ステータス=「未着手」)

---

## Threadsへの自動投稿(GAS側の設定)

スプレッドシートに記録された投稿文は、Google Apps Script(GAS)を使って
Threadsへ自動投稿します。**この部分はClaude Codeの管轄外** で、別途設定が必要です。

### 必要な準備

1. **Threads Graph API のアクセストークンを取得**
   - Meta for Developers でアプリを作成
   - Threads APIの権限を有効化
   - 長期アクセストークンを発行

2. **対象スプレッドシートを開き、拡張機能 → Apps Script を起動**

3. **GASでThreads投稿スクリプトを実装**
   - スプレッドシートを読み込み、ステータスが「未着手」(または「投稿予定」)の行を検索
   - 投稿予定日時が現在時刻以前かチェック
   - Threads Graph API でテキスト投稿
   - 投稿成功時にステータスを「投稿完了」に更新

4. **時刻トリガーを設定**
   - GASエディタの「トリガー」アイコンから新規トリガー作成
   - 関数: 投稿実行関数を選択
   - イベントのソース: 時間主導型
   - 頻度: 1時間おき (07:00 / 12:00 / 19:00 を全カバーするため)

### Claude Code側のスキルとの連携ポイント

- スプレッドシートのヘッダー名・列順は **本READMEのSTEP 3と完全一致** させること
  (GAS側のコードもこのヘッダー名を前提に書く)
- ステータスの値は以下で揃える:
  - `未着手` … Claude Codeが書き込んだ初期状態
  - `投稿完了` … GASが投稿成功時に更新
  - `投稿失敗` … GASが投稿失敗時に更新(任意)

### 注意点

- Threads Graph APIは2段階リクエスト(コンテナ作成 → 公開)が必要
- アクセストークンは60日で失効するため、定期的にリフレッシュする処理が必要
- GASの実行時間制限(6分/回)に注意

GAS側の実装は、別途Claudeに「上記仕様でGASスクリプトを書いて」と依頼すれば作成可能です。

---

## カスタマイズ

### テーマを変えたい
`config/settings.yaml` の `theme:` セクションを書き換えるだけ。
スキル本体の修正は不要。

### 投稿件数を増やしたい
`generation.posts_per_day` を変更し、`schedule_times` に時刻を追加。
ただし `viral-post-writer` の3案役割分担(共感型/ノウハウ型/逆張り型)は3件想定なので、
4件以上にする場合はスキル内の役割表を拡張してください。

### Threads以外も対応したい
`spreadsheet.platforms` のような項目を追加し、各スキルでループ処理にする
(現状はThreads単独で最適化)。

---

## 注意点

- **X側のスクレイピング制約**: WebFetchでXが取れない日が出たら、
  Apify等のMCPサーバを追加すると安定します
- **画像・動画**: Claude Codeでは自動生成しません。必要な場合は
  手動で別途用意してスプレッドシートのE列に追記してください
- **Routinesの最小実行間隔は1時間**。1日3回以上の実行は不可
- **Pro/Max/Team/Enterpriseプラン** で Claude Code on the web 有効が必要

## トラブルシュート

| 症状 | 対処 |
|---|---|
| Xが取得できない | `output/research/`にエラー記録される。Nitterまたは過去レポートが使われる |
| スプレッドシートに書き込めない | Google Driveコネクタの認証を再確認、URLが正しいか確認 |
| ジャンルがブレる | `theme.genre` を具体的に書く(「AI」より「生成AIの業務活用」) |
| GASがThreadsに投稿できない | アクセストークンの有効期限を確認、API権限を確認 |
