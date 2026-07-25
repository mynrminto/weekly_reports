# 週刊・小児腎臓病ラジオ — 週次パイプライン手順書

これは毎週月曜の自動実行(Routine)が新しいセッションに渡す**正典手順**です。
実行セッションはこのファイルの指示に従い、PubMed の新着から小児腎臓病トピックを選び、
ラジオ台本・音声(MP3)・論文要約・インフォグラフィックを生成し、Notion 掲載と
Google Drive 保存、リポジトリへのコミットまで行います。

## 前提
- リポジトリ: `mynrminto/weekly_reports`、ブランチ `claude/pediatric-kidney-disease-radio-k11ma7`
- 利用コネクタ(MCP): PubMed / Google Drive / Notion
- 環境変数 `GEMINI_API_KEY`（未設定なら音声はスキップし、その旨を成果物と通知に明記）
- Notion 掲載先データベース(data_source_id): **`1a961a49-2238-4dcb-87fc-53c23ffcb5d7`**
  （「週刊・小児腎臓病ラジオ（各号）」DB。ハブページ: https://app.notion.com/p/3a84bd470a818169afbcefb2f3b7f11b ）

## 手順

### 0. 準備
- `git fetch origin <branch> && git checkout <branch> && git pull` で最新化。
- `pip install -r scripts/requirements.txt`。
- 当日(JST)の日付を `DATE`（YYYY-MM-DD）とし、`reports/<DATE>/` を作業ディレクトリにする。

### 1. PubMed 検索（直近7日の新着）
`mcp__PubMed__search_articles` を次で実行:
- `query`:
  ```
  (pediatric OR paediatric OR children OR childhood) AND (kidney disease OR nephrology OR nephrotic OR nephritis OR renal)
  ```
- `datetype`: `edat`（PubMed 収載日）
- `date_from`: DATE の 7 日前 / `date_to`: DATE
- `sort`: `pub_date` / `max_results`: 30
- ヒットが多い場合や質を優先したい場合は、`AND (Review[Publication Type] OR Randomized Controlled Trial[Publication Type] OR Guideline[Publication Type])`
  や主要誌フィルタで追加抽出して候補を絞る。
- 各候補は `mcp__PubMed__get_article_metadata` でタイトル/著者/誌名/日付/DOI/抄録を取得。

### 2. 選定（3〜5 本）
- 臨床的インパクト・新規性・小児腎臓病領域との関連度で 3〜5 本を選ぶ。
- 症例報告・純粋な基礎のみ・関連薄のものは優先度を下げる。
- 選定結果を `reports/<DATE>/articles.json` に保存（各: pmid, title, journal, date, doi, url, one_line）。

### 3. ラジオ台本 `reports/<DATE>/script.md` と読み上げ用 `script.txt`
- 日本語・会話調のラジオ番組風。構成:
  1. オープニング（番組名「週刊・小児腎臓病ラジオ」、今週の日付、今週のテーマ一言）
  2. 各トピック（1本ずつ）: 何がわかったか→なぜ重要か→臨床的含意。誌名と発表時期に触れ、
     **PMID と DOI を口頭でも述べる**（例:「PMID 12345678」）。
  3. クロージング（今週のまとめ、来週予告的な一言、出典は概要欄参照の案内）
- 5〜8 分相当（およそ 1,600〜2,600 字）。医学的な断定は避け、原著参照を促す。
- 読み上げ用 `script.txt` は Markdown 記号や URL を除いた素のテキスト（段落は空行区切り）。

### 4. 音声 MP3
- `python scripts/tts_gemini.py reports/<DATE>/script.txt reports/<DATE>/radio.mp3`
- 終了コード 2（キー未設定）の場合は音声をスキップし、以降のリンク欄に「音声: キー未設定のため未生成」と記す。

### 5. インフォグラフィック
- `reports/<DATE>/infographic.html` を**自己完結 HTML**（インライン CSS、外部リクエストなし）で作成。
  内容: 番組名/日付、今週の新着件数・紹介本数などの統計、各論文の要点カード（タイトル・誌名・一言要約・PMID）。
  配色・可読性は `dataviz` スキルの指針に沿う。ダーク背景・幅約 1120–1200px を推奨。
- `python scripts/render_infographic.py reports/<DATE>/infographic.html reports/<DATE>/infographic.png`

### 6. Google Drive 保存
- `mcp__Google_Drive__create_file` で `radio.mp3`（あれば）と `infographic.png` を
  `pediatric-kidney-radio/<DATE>/` 相当の名前でアップロード。
- 閲覧可能な共有リンクを取得し `reports/<DATE>/links.txt` に記録。

### 7. Notion 掲載
- `mcp__Notion__notion-create-attachment` に `infographic.html` を **inline content** で渡し、
  返る `markdown_source`（`file-upload://…`）を取得。
- `mcp__Notion__notion-create-pages`（parent = `data_source_id: 1a961a49-2238-4dcb-87fc-53c23ffcb5d7`）で当週ページを作成:
  - properties: 週(タイトル=「YYYY-MM-DD 週」)、公開日、トピック数、PMIDs、MP3(URL)、インフォグラフィックPNG(URL)
  - content（Notion-flavored Markdown）: 冒頭に `<embed src="file-upload://…">` でインフォグラフィックを埋め込み、
    続いて各論文の要約（見出し=タイトル、誌名・日付・PMID/DOI リンク・3〜5 行要約）、末尾に MP3 リンク。
  - 実装時に NFM 仕様 `notion://docs/enhanced-markdown-spec` を参照（推測で書かない）。

### 8. コミット & プッシュ
- `reports/<DATE>/`（script.md, script.txt, articles.json, infographic.html, links.txt）をコミット。
  ※ MP3/PNG は Drive 保存が主。リポジトリを軽量に保つためコミットは任意（数 MB 未満なら可）。
- `git push -u origin <branch>`（失敗時は指数バックオフで最大 4 回）。

### 9. 最終メッセージ（= 完了通知メール本文になる）
次を簡潔にまとめて出力:
- 今週紹介した 3〜5 本の見出し
- Notion ページ URL
- MP3 の Drive リンク（未生成ならその旨）
- 補足（キー未設定などの注意があれば）
