---
title: "AppSheet × GAS で AI 画像解析アプリを作る（Vertex AI 連携）"
emoji: "📱"
type: "tech"
topics: ["appsheet", "gas", "vertexai", "gemini", "nocode"]
published: true
---

## はじめに

前回の記事で、GAS から Vertex AI (Gemini) を呼び出す画像解析スクリプトを作成しました。

https://zenn.dev/northgarden/articles/gas-vertex-ai-image-analyzer

今回はその続編として、**AppSheet から GAS を呼び出して AI 画像解析を自動化** する方法を解説します。

### この記事で作るもの

**スマホで写真を撮る → AI が自動解析 → 結果をスプレッドシートに保存** するアプリです。

- AppSheet の Form で画像を選択・送信
- Automation（Bot）で GAS を自動呼び出し
- Vertex AI (Gemini) の回答をスプレッドシートに書き戻し
- 保存後に詳細画面へ自動遷移

### 前提条件

- 前回の記事の GAS プロジェクトがセットアップ済み
- AppSheet の基本操作を理解していること

---

## 1. スプレッドシートの準備

AppSheet のデータソースとなるスプレッドシートを作成します。今回は **2つのシート** を用意します。

### 回答シート

AI 画像解析のリクエストと結果を格納するメインシートです。

![回答シート](/images/appsheet-gas-image-analyzer/data_source_response_sheet.png)

| カラム名 | 用途 |
|---|---|
| `回答ID` | 一意のキー（AppSheet で `UNIQUEID()` を自動生成） |
| `カテゴリー` | プロンプトマスターと紐付けるカテゴリー |
| `件名` | ユーザーが入力する件名 |
| `回答結果` | Vertex AI からの回答テキスト（GAS が書き込む） |
| `画像ファイルパス` | AppSheet で撮影・選択した画像のパス |

### プロンプトマスター

カテゴリーごとにプロンプトを管理するマスターシートです。

![プロンプトマスター](/images/appsheet-gas-image-analyzer/data_source_prompt_master_sheet.png)

| カラム名 | 用途 |
|---|---|
| `カテゴリー` | キー（回答シートと紐付け） |
| `プロンプト` | AI に送信する指示文 |

:::message
プロンプトをマスター化しておくと、カテゴリーごとに異なる指示文を使い分けられます。例えば「デフォルト」カテゴリーには「価格のみを抽出してください。円は不要です。」のようなプロンプトを設定できます。
:::

---

## 2. AppSheet アプリの作成

### アプリの作成手順

1. [AppSheet](https://www.appsheet.com/) にログイン
2. **Create** → **App** → **Start with existing data** を選択
3. 先ほど作成したスプレッドシートを選択
4. AppSheet がシート構造を自動認識してアプリを生成

### データソースの確認

生成されたアプリの「回答」テーブルのデータソース設定を確認します。

![データソース設定](/images/appsheet-gas-image-analyzer/datasource_spreadsheet.png)

**確認ポイント:**
- **Table name**: `回答`
- **Source Path**: スプレッドシート名
- **Worksheet Name**: `回答`
- **Are updates allowed?**: Updates, Adds, Deletes すべてチェック

---

## 3. テーブル設定

### 回答テーブルのカラム定義

テーブルの各カラムの型と設定を確認・調整します。

![テーブルカラム設定](/images/appsheet-gas-image-analyzer/table_columns.png)

| NAME | TYPE | KEY? | INITIAL VALUE | 備考 |
|---|---|---|---|---|
| `_RowNumber` | Number | | | システム列 |
| `回答ID` | Text | ✅ | `UNIQUEID()` | 自動採番 |
| `カテゴリー` | Enum | | `"デフォルト"` | プロンプトマスターに対応 |
| `件名` | Text | | | ユーザー入力（LABEL） |
| `回答結果` | Text | | `"AI回答取得までお待ちください。"` | GAS から書き込まれる |
| `画像ファイルパス` | Image | | | 写真を撮影・選択 |
| `回答_件名_価格` | Text | | 下記の式 | 一覧表示用（任意） |

:::message
`回答結果` の INITIAL VALUE に「AI回答取得までお待ちください。」を設定しておくと、ユーザーが送信直後に結果を確認しても処理中であることがわかります。
:::

:::details 回答_件名_価格 の FORMULA（任意）
一覧画面で「件名 : 価格」を素早く確認するための仮想カラムです。AI 回答がまだ返っていない場合は「不明」と表示します。

```
IF([回答結果] = "AI回答取得までお待ちください。",
  CONCATENATE([件名], " : 不明"),
  CONCATENATE([件名], " : ", [回答結果], "円")
)
```
:::

### 画像カラムの詳細設定

`画像ファイルパス` カラムの Type を **Image** に設定し、保存先フォルダを確認します。

![画像カラム設定](/images/appsheet-gas-image-analyzer/image_file_path_column_setting.png)

**確認ポイント:**
- **Type**: `Image`
- **Image/File folder path**: `"Images"`（画像の保存先フォルダ名）

:::message
この `Image/File folder path` の値が、GAS に渡す `folderId` の中のサブフォルダになります。AppSheet はこのフォルダに画像を自動保存します。
:::

---

## 4. Automation（Bot）の設定

ここが AppSheet × GAS 連携の核心部分です。Automation を使って、データ登録時に GAS を自動呼び出しします。

### Bot の全体構成

**Bots → 画像分析Bot** を作成します。全体の流れは以下の通りです。

![Bot 全体構成](/images/appsheet-gas-image-analyzer/Bots設定_Event_Trigger設定.png)

```
EVENT: 画像登録時（回答テーブルへの Add / Update）
  ↓
PROCESS:
  Step 1: AI画像分析（Call a script）
  Step 2: 分析結果を保存（Set row values）
```

### Event（トリガー）の設定

右側の Settings パネルで以下を設定します。

| 項目 | 設定値 |
|---|---|
| **Event name** | `画像登録時` |
| **Event source** | `App` |
| **Table** | `回答` |
| **Data change type** | `Adds` ✅ / `Updates` ✅ |

:::message
**Updates にもチェックを入れている理由:** 件名やカテゴリーを修正して再度解析させたい場合に、行を更新するだけで Bot が再実行されます。
:::

---

## 5. GAS 関数の呼び出し設定

### Step 1: Call a script（AI画像分析）

Bot の Process で **Call a script** ステップを追加し、GAS 関数を設定します。

![Call a script 設定](/images/appsheet-gas-image-analyzer/Bots設定_API呼び出し部分.png)

| 項目 | 設定値 |
|---|---|
| **Apps Script Project** | `gas-vertex-ai-image-analyzer`（GAS プロジェクト名） |
| **Function Name** | `analyzeImageByPathVertexAi` |

#### Function Parameters のマッピング

GAS 関数の各パラメータに AppSheet の値をマッピングします。

| パラメータ | 設定値 |
|---|---|
| `userPrompt` | `ANY(SELECT(プロンプトマスター[プロンプト], [_THISROW].[カテゴリー] = [カテゴリー]))` |
| `folderId` | `"your-appsheet-project-folder-id"`（※自分のフォルダ ID に置き換え） |
| `filePath` | `[画像ファイルパス]` |
| `model` | `"gemini-2.5-flash"` |
| `systemPrompt` | `""` |

:::message
**userPrompt** の `SELECT` 式がポイントです。プロンプトマスターテーブルから、現在の行のカテゴリーに一致するプロンプトを動的に取得しています。`[_THISROW].[カテゴリー]` が現在行の値、`[カテゴリー]` がプロンプトマスター側の値です。
:::

:::message alert
**folderId は AppSheet のプロジェクトフォルダの ID を指定してください。**

AppSheet は撮影した画像を `Images/` サブフォルダに保存し、ファイルパスは `Images/xxxxx.jpg` の形式で記録されます。GAS の `getFileFromFilePath` はこのパスをルートフォルダから辿るため、`Images/` の親フォルダ（= プロジェクトフォルダ）を指定する必要があります。

```
Google Drive
└── AppSheet/
    └── AI画像分析アプリ/     ← ★ この ID を folderId に設定
        ├── Images/           ← AppSheet が画像を自動保存
        │   └── xxxxx.jpg
        └── AI画像分析結果     ← スプレッドシート
```

`Images/` フォルダの ID を指定すると、パスの `Images/` 部分が解決できずエラーになります。
:::

#### Return Value の設定

| 項目 | 設定値 |
|---|---|
| **Return Value** | ON |
| **Type** | `String` |
| **Specific type** | `LongText` |

### Step 2: 分析結果を保存（Set row values）

GAS からの戻り値をスプレッドシートに書き戻します。

![分析結果を保存](/images/appsheet-gas-image-analyzer/Bots設定_AI回答をデータソースに保存する.png)

| 項目 | 設定値 |
|---|---|
| **Action** | `Set row values` |
| **Set these column(s)** | `回答結果` = `[AI画像分析].[Output]`（Step 1 の戻り値） |

:::message
`[AI画像分析].[Output]` の `[AI画像分析]` は Step 1 の名前、`.[Output]` は Call a script の戻り値を参照するプロパティです。この書式を忘れると値が取得できないので注意してください。
:::

---

## 6. ビューの設定

### Action: 保存後に詳細画面に遷移

Form で保存した後、自動的に詳細画面に遷移するアクションを作成します。

![保存後に詳細画面に遷移](/images/appsheet-gas-image-analyzer/Actions設定_保存後に詳細画面に遷移.png)

| 項目 | 設定値 |
|---|---|
| **Action name** | `保存後に詳細画面に遷移` |
| **For a record of this table** | `回答` |
| **Do this** | `App: go to another view within this app` |
| **Target** | `LINKTOROW([回答ID], "回答_Detail")` |
| **Position** | `Hide` |

### Form ビューの Event Actions 設定

回答_Form の **Event Actions** に、保存後のアクションを設定します。

![Form Saved 設定](/images/appsheet-gas-image-analyzer/回答_FormにおけるEvent_Actions設定_From_Saved.png)

| 項目 | 設定値 |
|---|---|
| **Form Saved** | `保存後に詳細画面に遷移` |

:::message alert
**ハマりポイント:** Action で `LINKTOROW` を設定しただけでは、Form の保存後は一覧に戻ってしまいます。必ず **Form ビューの Event Actions → Form Saved** でアクションを紐付けてください。
:::

---

## 7. 動作確認

### 全体の流れ

1. AppSheet で **＋ボタン** をタップ → Form が開く
2. **件名**を入力、**カテゴリー**を選択、**画像**を撮影または選択
3. **保存** をタップ
4. 自動的に **詳細画面** に遷移
5. 数秒待つと Bot が起動 → GAS が Vertex AI を呼び出し
6. **回答結果** に AI の解析結果が表示される

### 画面の様子

**Form 画面** — 件名・カテゴリー・画像を入力して保存します。

![Form 画面](/images/appsheet-gas-image-analyzer/アプリ_Form画面.png)

**詳細画面** — 保存後に自動遷移し、AI の回答結果が表示されます。

![詳細画面](/images/appsheet-gas-image-analyzer/アプリ_詳細画面.png)

**一覧画面** — 画像サムネイルと「件名 : 価格」が一目で確認できます。

![一覧画面](/images/appsheet-gas-image-analyzer/アプリ_一覧画面.png)

:::message
初回実行時は GAS の OAuth2 認証でトークンを取得するため、少し時間がかかる場合があります。
:::

---

## おわりに

AppSheet × GAS × Vertex AI (Gemini) で、ノーコード AI 画像解析アプリを構築しました。

### 今回のポイント

- **Automation（Bot）** で GAS を自動呼び出し
- **Call a script** で GAS 関数のパラメータをマッピング
- **プロンプトマスター** でプロンプトを動的に切り替え
- **Form Saved イベント** で保存後の画面遷移を制御

AppSheet の設定は UI ベースで変わりやすいですが、設定の「意図」を理解しておけば、UI が変わっても対応できます。

### 関連記事

- [GAS × Vertex AI (Gemini) で画像解析スクリプトを作る](https://zenn.dev/northgarden/articles/gas-vertex-ai-image-analyzer)

### 参考リンク

- [GitHub リポジトリ](https://github.com/a143321/gas-vertex-ai-image-to-text)
- [AppSheet Automation ドキュメント](https://support.google.com/appsheet/answer/10105373)
