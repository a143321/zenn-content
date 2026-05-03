---
title: "GAS × Vertex AI (Gemini) で画像解析スクリプトを作る"
emoji: "📷"
type: "tech"
topics: ["gas", "vertexai", "gemini", "googlecloud", "appsheet"]
published: true
---

## はじめに

Google Apps Script (GAS) から Vertex AI の Gemini モデルを使って、Google Drive 上の画像を解析するスクリプトを作りました。

AppSheet と連携することで、**スマホで写真を撮る → AI が自動で画像を解析 → 結果をスプレッドシートに保存** というワークフローが実現できます。

### この記事で作るもの

- GAS から Vertex AI (Gemini) を呼び出す画像解析スクリプト
- サービスアカウント認証による安全な API 接続
- AppSheet から呼び出し可能な公開 API

### 対象読者

- GAS で AI を使ってみたい方
- AppSheet と GAS を連携させたい方
- Vertex AI のサービスアカウント認証を GAS で実装したい方

### リポジトリ

https://github.com/a143321/gas-vertex-ai-image-to-text

:::message alert
この記事は **2026年5月時点** の情報です。使用モデル `gemini-2.5-flash` は **2026年10月16日以降に廃止予定** です。もし動作しない場合は、`config.gs` の `DEFAULT_MODEL` を最新の Gemini モデル名に変更してください。利用可能なモデルは [Vertex AI のモデル一覧](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models) で確認できます。
:::

---

## アーキテクチャ

全体の構成は以下の通りです。

```
AppSheet → GAS (api.gs) → Vertex AI (Gemini)
                ↓
           Google Drive（画像取得）
```

1. AppSheet（またはGASエディタ）からスクリプトを実行
2. GAS が Google Drive から画像を取得し、Base64 に変換
3. サービスアカウント認証で Vertex AI API にリクエスト
4. Gemini が画像を解析し、テキストを返す

---

## 前提条件

- Google Cloud プロジェクトを持っていること
- Google Apps Script の基本操作ができること

---

## 1. Google Cloud の準備

### Vertex AI API の有効化

1. [Google Cloud Console](https://console.cloud.google.com/) を開く
2. **API とサービス** → **ライブラリ** で「Vertex AI API」を検索
3. **有効にする** をクリック

### サービスアカウントの作成

GAS から Vertex AI を呼び出すために、サービスアカウントを作成します。

1. **IAM と管理** → **サービスアカウント** に移動
2. **サービスアカウントを作成** をクリック
3. 名前を入力（例: `gas-vertex-ai-image-analyzer`）

![サービスアカウントの作成](/images/gas-vertex-ai-image-analyzer/create_service_account.png)

4. **Vertex AI ユーザー**（`roles/aiplatform.user`）ロールを付与

![ロールの設定](/images/gas-vertex-ai-image-analyzer/set_role.png)

:::message
**管理者ロールではなくユーザーロールで十分です。** 今回は推論（API リクエスト）のみ行うため、最小権限の原則に従います。
:::

サービスアカウントが作成されました。

![作成完了](/images/gas-vertex-ai-image-analyzer/created_service_account.png)

### JSON キーのダウンロード

1. 作成したサービスアカウントをクリック
2. **「キー」タブ** → **「鍵を追加」** → **「新しい鍵を作成」**
3. **JSON** を選択 → ダウンロード

![JSONキーの作成](/images/gas-vertex-ai-image-analyzer/create_key.png)

:::message alert
ダウンロードした JSON キーは、スクリプトプロパティに登録したら**すぐに削除**してください。GitHub にコミットすることのないよう注意しましょう。
:::

---

## 2. GAS プロジェクトのセットアップ

### プロジェクトの作成

[Google Apps Script](https://script.google.com/) で新規プロジェクトを作成します。

### OAuth2 ライブラリの追加

サービスアカウント認証に OAuth2 ライブラリを使用します。

1. GAS エディタで **ライブラリ（＋）** をクリック
2. 以下のスクリプト ID を入力：

```
1B7FSrk5Zi6L1rSxxTDgDEUsPzlukDsi4KGuTMorsTQHhGBzBkMun4iDF
```

3. 最新バージョンを選択 → **追加**

### スクリプトプロパティの設定

**プロジェクトの設定 → スクリプトプロパティ** に以下を登録します。

| プロパティ | 値 | 必須 |
|---|---|---|
| `VERTEX_AI_PROJECT_ID` | Google Cloud プロジェクト ID | ✅ |
| `VERTEX_AI_CLIENT_EMAIL` | JSON キーの `client_email` | ✅ |
| `VERTEX_AI_PRIVATE_KEY` | JSON キーの `private_key` | ✅ |
| `VERTEX_AI_LOCATION` | リージョン（デフォルト: `us-central1`） | |

:::details VERTEX_AI_PRIVATE_KEY の貼り付けについて
JSON キーの `private_key` の値は `-----BEGIN PRIVATE KEY-----\n...` のような形式になっています。**そのままコピー＆ペースト** してください。改行文字（`\n`）はコード側で自動変換するため、エスケープされた状態のまま貼り付けて問題ありません。
:::

---

## 3. コードの実装

ファイル構成は以下の通りです。各ファイルの役割を明確に分離しています。

```
src/
├── config.gs       # 設定管理
├── auth.gs         # OAuth2 認証
├── drive.gs        # Drive ファイル操作
├── vertex_ai.gs    # Vertex AI API 通信
├── api.gs          # 公開 API（エントリポイント）
└── test.gs         # テスト用
```

### config.gs — 設定管理

スクリプトプロパティからの値取得と、デフォルト値の管理を行います。

デフォルトモデルは `gemini-2.5-flash` を使用しています。高速かつ低コストで、画像解析には十分な性能です。

:::message
モデルの廃止日や最新バージョンについては、記事冒頭の注意書きを参照してください。`DEFAULT_MODEL` を変更するだけで新しいモデルに切り替えられます。
:::

```javascript
const DEFAULT_MODEL = "gemini-2.5-flash";
const DEFAULT_LOCATION = "us-central1";

function getProjectId() {
  return _getRequiredProperty("VERTEX_AI_PROJECT_ID");
}

function getClientEmail() {
  return _getRequiredProperty("VERTEX_AI_CLIENT_EMAIL");
}

function getPrivateKey() {
  const raw = _getRequiredProperty("VERTEX_AI_PRIVATE_KEY");
  return raw.replace(/\\n/g, "\n");
}

function getLocation() {
  return _getOptionalProperty("VERTEX_AI_LOCATION", DEFAULT_LOCATION);
}

function getModel(model) {
  return model || DEFAULT_MODEL;
}
```

:::message
`getPrivateKey()` で文字列 `\n`（バックスラッシュ + n）を実際の改行文字に変換しています。スクリプトプロパティに保存された秘密鍵にはエスケープされた改行が含まれているため、この変換が必要です。
:::

### auth.gs — サービスアカウント認証

OAuth2 ライブラリを使い、サービスアカウントで Vertex AI の認証トークンを取得します。

```javascript
function getOAuthService() {
  return OAuth2.createService("VertexAI")
    .setTokenUrl("https://oauth2.googleapis.com/token")
    .setPrivateKey(getPrivateKey())
    .setIssuer(getClientEmail())
    .setPropertyStore(PropertiesService.getScriptProperties())
    .setScope("https://www.googleapis.com/auth/cloud-platform");
}

function getAccessToken() {
  const service = getOAuthService();
  if (!service.hasAccess()) {
    throw new Error("OAuth2 認証に失敗しました: " + service.getLastError());
  }
  return service.getAccessToken();
}
```

### drive.gs — Google Drive ファイル操作

AppSheet から渡されるファイルパス（例: `photos/image.jpg`）を、ルートフォルダ ID を起点にして Drive 上のファイルに解決し、画像データを Base64 に変換します。

Gemini API に画像を渡す方法には **URL 指定**と **Base64 インライン**の2つがあります。今回は Base64 を採用しています。

| 方法 | メリット | デメリット |
|---|---|---|
| URL 指定 | シンプル | 画像を公開 URL にする必要がある |
| **Base64 インライン** | **画像を非公開のまま送信できる** | データサイズが大きくなる |

Google Drive の画像はデフォルトで非公開です。URL 方式だと画像を全世界に公開することになるため、業務データを扱う場合は **Base64 インラインが必須** です。

```javascript
function getFileFromFilePath(rootFolderId, filePath) {
  const pathParts = filePath.split("/");
  const fileName = pathParts.pop();
  let currentFolder = DriveApp.getFolderById(rootFolderId);

  // フォルダ階層をたどる
  for (const part of pathParts) {
    const folders = currentFolder.getFoldersByName(part);
    if (!folders.hasNext()) {
      throw new Error(`フォルダが見つかりません: ${part}`);
    }
    currentFolder = folders.next();
  }

  const files = currentFolder.getFilesByName(fileName);
  if (!files.hasNext()) {
    throw new Error(`ファイルが見つかりません: ${fileName}`);
  }
  return files.next();
}

// Drive ファイル → Base64 エンコードされた画像データに変換
function getImageDataFromDrive(file) {
  const blob = file.getBlob();
  return {
    fileId: file.getId(),
    fileName: file.getName(),
    base64: Utilities.base64Encode(blob.getBytes()),
    mimeType: blob.getContentType(),
  };
}
```

:::message
`Utilities.base64Encode(blob.getBytes())` が画像のバイナリを Base64 文字列に変換するポイントです。この文字列が Vertex AI API の `inlineData.data` フィールドに渡されます。
:::

### vertex_ai.gs — Vertex AI API 通信

API リクエストの構築と送信を行うコアモジュールです。主な処理は以下の3ステップです。

1. **エンドポイント URL の構築** — プロジェクト ID・リージョン・モデル名から API URL を組み立てる
2. **リクエストボディの構築** — 画像（Base64）とプロンプトを Gemini API の `contents` 形式に変換
3. **API 呼び出し** — Bearer トークンで認証し、レスポンスからテキストを抽出

```javascript
function generateTextFromImages(userPrompt, imageDataList, model, systemPrompt) {
  // バリデーション & パラメータ初期化
  model = (model && model.trim()) ? model.trim() : DEFAULT_MODEL;

  // エンドポイント URL を構築
  const projectId = getProjectId();
  const location = getLocation();
  const url = `https://${location}-aiplatform.googleapis.com/v1/projects/${projectId}/locations/${location}/publishers/google/models/${model}:generateContent`;

  // 画像 + プロンプトを contents 配列に変換
  const imageParts = imageDataList.map(img => ({
    inlineData: { mimeType: img.mimeType, data: img.base64 }
  }));
  const contents = [{
    role: "user",
    parts: [...imageParts, { text: userPrompt }]
  }];

  // API 呼び出し
  const response = UrlFetchApp.fetch(url, {
    method: "post",
    contentType: "application/json",
    headers: { Authorization: "Bearer " + getAccessToken() },
    payload: JSON.stringify({ contents }),
    muteHttpExceptions: true,
  });

  if (response.getResponseCode() !== 200) {
    throw new Error(`Vertex AI API エラー: ${response.getContentText()}`);
  }

  const json = JSON.parse(response.getContentText());
  return json.candidates[0].content.parts[0].text;
}
```

:::message
エンドポイント URL のフォーマットは `https://{LOCATION}-aiplatform.googleapis.com/v1/projects/{PROJECT_ID}/locations/{LOCATION}/publishers/google/models/{MODEL}:generateContent` です。`config.gs` の設定値が自動的に埋め込まれます。
:::

### api.gs — 公開 API

AppSheet から直接呼び出せるエントリポイントです。

```javascript
// 単一画像の解析
function analyzeImageByPathVertexAi(userPrompt, folderId, filePath, model, systemPrompt) {
  const result = _analyzeImagesByPath({
    userPrompt, folderId,
    filePaths: [filePath],
    model, systemPrompt
  });
  return result.response;
}
```

:::details 複数画像の解析（最大3枚）
```javascript
function analyzeMultipleImagesByPathVertexAi(
  userPrompt, folderId, photoPath, addPhoto1Path, addPhoto2Path, model, systemPrompt
) {
  const filePaths = [photoPath, addPhoto1Path, addPhoto2Path].filter(p => p);
  const result = _analyzeImagesByPath({
    userPrompt, folderId, filePaths, model, systemPrompt
  });

  return {
    response: result.response,
    photoId: result.files[0]?.id || "",
    photoName: result.files[0]?.name || "",
    addPhoto1Id: result.files[1]?.id || "",
    addPhoto1Name: result.files[1]?.name || "",
    addPhoto2Id: result.files[2]?.id || "",
    addPhoto2Name: result.files[2]?.name || "",
  };
}
```
:::

---

## 4. 動作確認

### セットアップ確認

GAS エディタで `checkSetup` を実行して、スクリプトプロパティが正しく設定されているか確認します。

### テスト実行

`test.gs` にテスト用の関数を用意しています。フォルダ ID とファイルパスを自分の環境に合わせて変更し、GAS エディタから実行してください。

```javascript
// テスト用定数（自分の環境に合わせて変更）
const TEST_FOLDER_ID = "your-folder-id";
const TEST_FILE_PATH = "your-image.jpg";
const TEST_PROMPT = "この画像に写っているものを教えてください";

function testSingleImage() {
  const result = analyzeImageByPathVertexAi(
    TEST_PROMPT, TEST_FOLDER_ID, TEST_FILE_PATH, "", ""
  );
  Logger.log(result);
}
```

---

## 5. トラブルシューティング

| エラー | 原因と対処 |
|---|---|
| `スクリプトプロパティが設定されていません` | 必須プロパティが未設定 → プロジェクトの設定を確認 |
| `OAuth2 認証に失敗しました` | キーの貼り付けミス → `PRIVATE_KEY` を再確認 |
| `Vertex AI API エラー（401）` | トークン期限切れ → `resetOAuthService()` を実行 |
| `Vertex AI API エラー（403）` | 権限不足 → サービスアカウントの IAM ロールを確認 |
| `フォルダが見つかりません` | フォルダ ID またはパスが間違っている |

---

## おわりに

GAS × Vertex AI (Gemini) で画像解析スクリプトを実装しました。

サービスアカウント認証は少しセットアップが手間ですが、一度設定すれば API キーの管理が不要になり、セキュアな運用が可能です。

AppSheet と組み合わせることで、ノーコードで AI 画像解析アプリが作れるので、ぜひ試してみてください。

### 参考リンク

- [GitHub リポジトリ](https://github.com/a143321/gas-vertex-ai-image-to-text)
- [Vertex AI Gemini API リファレンス](https://cloud.google.com/vertex-ai/docs/generative-ai/model-reference/gemini)
- [OAuth2 for Apps Script](https://github.com/googleworkspace/apps-script-oauth2)
- [Google Apps Script開発をローカルで行うためのツール「clasp」入門 - Qiita](https://qiita.com/yonaka15/items/c63653af71eb8e20c2bb)
- [Google Apps Script + OAuthライブラリで API 操作を行う - Qiita](https://qiita.com/TakeshiNickOsanai/items/62810b0e96bf37bd0eca)
