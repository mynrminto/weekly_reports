# ハンズオン：毎週月曜、自分専用の「論文ラジオ」が自動で届く仕組みを作る

所要 **30分** ／ 対象: 普段 Claude Code を使っている方

## 完成するもの

毎週決まった時刻に、**あなたが何もしなくても**次が自動で起きます。

1. PubMed から、あなたの専門領域の**先週の新着論文**を検索して数本を選ぶ
2. 2人の話者による**対話形式のラジオ台本**を書く
3. Gemini の音声合成で **MP3** にする
4. 要点をまとめた**インフォグラフィック画像**を作る
5. GitHub に保存し、**Notion に音声つきのページ**として掲載する
6. 完了メールが届く

---

## ⚠️ 事前準備（当日までに済ませてください）

30分で終わらせるため、**アカウント作成は事前に**お願いします。当日作ると間に合いません。

- [ ] **GitHub アカウント**（[github.com/signup](https://github.com/signup)）— メール認証まで完了させる
- [ ] **Notion アカウント**（[notion.so](https://www.notion.so/)）— 無料プランで可
- [ ] **Google アカウント** — Gemini API キーの取得に使用
- [ ] **Claude の Pro / Max プラン** — Routine（定期実行）に必要
- [ ] 手元の Claude Code から `/login` で claude.ai にログイン済み

---

## タイムテーブル

| 時間 | ステップ |
|---|---|
| 0:00–0:03 | 全体像の説明 |
| 0:03–0:07 | **STEP 1** リポジトリを作る |
| 0:07–0:11 | **STEP 2** GitHub と Claude をつなぐ |
| 0:11–0:15 | **STEP 3** コネクタ（PubMed / Notion）を接続 |
| 0:15–0:19 | **STEP 4** Gemini API キーを取得 |
| 0:19–0:25 | **STEP 5** Routine を設定 |
| 0:25–0:30 | **STEP 6** テスト実行して結果を見る |

---

## STEP 1. リポジトリを作る（4分）

テンプレートから、自分のリポジトリを作ります。

1. テンプレートリポジトリのページを開く（講師が URL を共有します）
2. 緑色の **「Use this template」** → **「Create a new repository」** をクリック
3. リポジトリ名を入力（例: `weekly-radio`）
4. **Public** を選択 ← ⚠️ **重要**
5. **「Create repository」** をクリック

> **なぜ Public なのか**: Notion に音声を埋め込むとき、GitHub の
> raw URL（`raw.githubusercontent.com/...`）から音声ファイルを取り込みます。
> Private だとこの URL に Notion 側からアクセスできず、音声が再生できません。
> 公開したくない内容は扱わないでください。

---

## STEP 2. GitHub と Claude をつなぐ（4分）

Claude のクラウドセッションが、あなたのリポジトリを clone / push できるようにします。
**どちらか一方**でOKです。

### 方法A: ターミナルから（Claude Code ユーザーはこちらが速い）

手元のターミナルで:

```
/web-setup
```

ローカルの `gh` CLI のトークンが Claude アカウントに同期されます。**10秒で終わります。**

### 方法B: ブラウザから（GitHub App を導入）

1. [claude.ai/code](https://claude.ai/code) を開く
2. 画面の案内に従って **Claude GitHub App** を認可する
3. インストール先リポジトリを選ぶ（**All repositories** か、STEP 1 で作ったリポジトリ）

> **A と B の違い**: どちらでも clone / push はできます。
> GitHub App（方法B）を入れると、これに加えて**PR のイベントに反応する**機能
> （CI 失敗の自動修正など）が使えるようになります。今日の内容だけなら A で十分です。

---

## STEP 3. コネクタ（PubMed / Notion）を接続（4分）

[claude.ai/customize/connectors](https://claude.ai/customize/connectors) を開き、2つ追加します。

| コネクタ | 手順 | 所要 |
|---|---|---|
| **PubMed** | 一覧から追加するだけ（**認証不要**） | 10秒 |
| **Notion** | 追加 → Notion のログイン画面 → **どのページへのアクセスを許可するか選ぶ** | 2分 |

> **Notion の認可画面での注意**: 「ページを選択」で、書き込みを許可するページを
> **必ず1つ以上**選んでください。ここで何も選ばないと、後で
> 「掲載先データベースを作れない」というエラーになります。

---

## STEP 4. Gemini API キーを取得（4分）

音声合成に使います。

1. [aistudio.google.com/apikey](https://aistudio.google.com/apikey) を開く
2. Google アカウントでログイン
3. **API キーを作成** をクリック（画面の表記は "Create API key" などの場合があります）
4. 表示されたキーをコピー ← **この後すぐ使います**

> キーは無料枠で使えます。クレジットカード登録は不要です。
> **キーは他人に見せないでください。** Slack やスライドに貼らないよう注意。

---

## STEP 5. Routine を設定（6分）

ここが本番です。[claude.ai/code/routines](https://claude.ai/code/routines) を開き、
**「New routine」** をクリック。

### 5-1. 名前とプロンプト

- **名前**: `週刊ラジオ（毎週月曜 7:00）` など
- **Instructions**: リポジトリ内の `handson/template/routine-prompt.md` にある
  コードブロックの中身を**そのまま貼り付け**（編集不要）

### 5-2. リポジトリを選ぶ

STEP 1 で作ったリポジトリを追加します。

### 5-3. 環境変数に Gemini キーを設定 ← ⚠️ つまずきポイント

1. Instructions 欄の下にある**雲のアイコン**（`Default` と表示）をクリック
2. 環境の一覧が出るので、環境の上にカーソルを乗せ、右に出る**歯車アイコン**をクリック
3. **Update cloud environment** ダイアログの**環境変数**欄に、次の1行を入力:

   ```
   GEMINI_API_KEY=ここにSTEP4でコピーしたキー
   ```

   > **引用符で囲まないでください。** `"..."` を付けると、引用符ごと値として保存されます。

4. **Network access** は **Trusted のまま変更不要**
5. **Save changes** をクリック

> **なぜネットワーク設定を触らなくていいのか**（今日いちばん面白いところ）
>
> - **Gemini** は `*.googleapis.com` が既定の許可リストに入っているので、そのまま通ります
> - **PubMed / Notion** はコネクタ経由で、通信は Anthropic のサーバ側で行われるため、
>   そもそもこの許可リストの対象外です
>
> つまり「鍵は環境変数、外部サービスはコネクタ」と役割を分けたので、追加設定が不要になっています。

### 5-4. スケジュール

**Select a trigger** で **Schedule** を選び、**Weekly / 月曜 / 7:00** に設定。

> 時刻は**あなたのローカル時間**で入力すれば自動変換されます。UTC 換算は不要です。
> 実際の実行は数分ずれることがあります（負荷分散のため）。

### 5-5. コネクタと権限 ← ⚠️ 最重要

フォーム下部の2つのタブを必ず確認します。

- **Connectors**: **PubMed** と **Notion** が含まれていることを確認
  （既定で全部入るので、不要なものは外してOK）
- **Permissions**: **「Allow unrestricted branch pushes」を必ず有効化**

> **これを忘れると必ず失敗します。** Routine は既定では `claude/` で始まる名前の
> ブランチにしか push できません。今回のパイプラインは `main` に push するので、
> この許可がないと毎週 push の段階で止まります。

最後に **Create** をクリック。

---

## STEP 6. テスト実行（5分）

来週まで待たずに、その場で動かします。

1. 作成した Routine の詳細ページで **「Run now」** をクリック
2. 実行中のセッションが開くので、Claude の作業を眺める

**5分では完走しません**（音声生成まで含めて10〜15分ほどかかります）。
途中で解散しても、セッションはクラウドで動き続け、**終わればメールが届きます。**

### 確認ポイント

- [ ] PubMed 検索が走り、論文が選ばれた
- [ ] `radio.mp3` が生成された（= Gemini キーが正しく設定できている）
- [ ] GitHub の `reports/<日付>/` に成果物が push された
- [ ] Notion にページができ、**音声がページ内で再生できる**

> ⚠️ 実行一覧の**緑色の表示は「セッションが異常終了しなかった」という意味**で、
> 「やりたいことが成功した」という意味ではありません。中身を必ず開いて確認してください。

---

## 自分のテーマに変える（持ち帰り課題）

リポジトリの **`prompts/weekly_radio_prompt.md`** を開き、冒頭の**設定欄だけ**を書き換えます。

```yaml
show_name: "週刊・循環器ラジオ"
pubmed_query: >-
  (heart failure OR atrial fibrillation) AND (cardiology OR cardiovascular)
topic_count: 3
hosts:
  - { role: "進行役", name: "ナオ",     voice: "Puck" }
  - { role: "解説役", name: "リク先生", voice: "Kore" }
```

編集して push すれば、**次回の実行から自動で反映されます。**
Routine のプロンプトを貼り直す必要はありません（「リポジトリの手順書を読んで従え」という
構造にしてあるためです）。

変更を Claude Code に頼むのが手軽です:

```
prompts/weekly_radio_prompt.md の設定欄を、私の専門領域である〇〇に合わせて書き換えて push して
```

---

## うまくいかないとき

| 症状 | 原因と対処 |
|---|---|
| push が拒否される | **Allow unrestricted branch pushes** が無効。Routine 編集 → Permissions で有効化 |
| 音声ができない | `GEMINI_API_KEY` の設定漏れ、または値を引用符で囲んでいる。環境設定を再確認 |
| Notion にページができない | Notion 連携時にページを選んでいない。[connectors](https://claude.ai/customize/connectors) から再認可 |
| Notion で音声が再生できない | リポジトリが **Private**。Public にするか、リポジトリを作り直す |
| 論文が0件 | その週に新着が無い領域。検索クエリを広げる（手順書が自動で14日に広げる設計です） |
| コネクタのツールが無い | Routine の **Connectors** タブで PubMed / Notion が外れている |

## 費用について

- **Gemini API**: 無料枠の範囲内（週1回の音声合成なら十分収まります）
- **Claude**: Pro / Max の通常の利用枠を消費します。Routine には1日あたりの実行回数上限もあります
- **GitHub / Notion**: 無料プランで可（Notion 無料プランのファイル上限に合わせ、MP3 は 5MB 未満に収める設計です）
