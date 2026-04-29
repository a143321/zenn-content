---
title: "WSL2導入手順"
emoji: "🐧"
type: "tech"
topics: ["linux", "windows", "wsl2"]
published: true
---

## WSLとは

WSL（Windows Subsystem for Linux）は、Windows上でLinux環境を動かせる仕組みです。Linux用の開発ツールやコマンドを、仮想マシンを使わずに直接Windowsで利用できるようになります。

## 必要条件

- Windows 10 (バージョン 2004 以降) または Windows 11
- インターネット接続
- 管理者権限 (PCの設定を変更できる権限)

## 導入手順

1. 管理者権限で、PowerShellを開きます

2. インストール可能なLinux一覧を取得する

```powershell
wsl --list --online
```

3. ディストリビューションを指定しないでデフォルトでインストール

```powershell
wsl --install
```

4. 特定のディストリビューションを指定する場合

```powershell
wsl --install -d Ubuntu-22.04
```

:::message
デフォルトでは「Ubuntu」の最新バージョンがインストールされます。特定のバージョンを使いたい場合は `-d` オプションで指定しましょう（例：`Ubuntu-22.04`）。
:::

5. インストール後は、インストールした環境を以下で確認する

```powershell
wsl -l -v
```

```
 NAME            STATE           VERSION
* Ubuntu-22.04    Running         2
  Debian          Stopped         2
```

## wsl.conf の設定（重要）

WSL2 をインストールした後、**`/etc/wsl.conf` の設定は必ず行ってください**。この設定がないと、Node.js や Docker などの開発ツールが不安定になります。

### 設定ファイルの作成

WSL2 を起動し、以下のコマンドで設定ファイルを作成・編集します。

```bash
sudo vi /etc/wsl.conf
```

### 推奨設定

以下の内容をコピーして貼り付けてください。

```ini
[boot]
systemd=true

[user]
default=<your-username>

[interop]
enabled=true
appendWindowsPath=false
```

`<your-username>` は自分の WSL2 ユーザー名に置き換えてください（`whoami` コマンドで確認できます）。

### 各設定の解説

#### `[boot]` セクション

| 設定 | 値 | 必要な理由 |
|:--|:--|:--|
| `systemd` | `true` | Node.js、Docker、バックグラウンドサービスの安定動作に**必須** |

`systemd=true` がないと、`npm` や各種サービスのプロセス管理が不安定になります。

#### `[user]` セクション

| 設定 | 値 | 必要な理由 |
|:--|:--|:--|
| `default` | `<your-username>` | root ではなく一般ユーザーでの作業を標準化 |

省略した場合はインストール時に作成したユーザーが使われますが、明示的に指定しておくと安心です。

#### `[interop]` セクション

| 設定 | 値 | 必要な理由 |
|:--|:--|:--|
| `enabled` | `true` | WSL2 から Windows 側のコマンド（`explorer.exe` 等）を実行可能にする |
| `appendWindowsPath` | `false` | **Windows の PATH が WSL2 に混入するのを防ぐ** |

:::message alert
**`appendWindowsPath=false` は特に重要です。**
この設定がないと、WSL2 の `$PATH` に Windows 側のパスが追加され、WSL2 内の `node` や `npm` が Windows 版と競合してエラーの原因になります。
:::

### 設定の反映

設定を保存した後、**PowerShell**（Windows 側）で以下を実行して WSL2 を再起動します。

```powershell
wsl --shutdown
```

その後、WSL2 を再度起動すれば設定が反映されます。

### 設定の確認

WSL2 内で以下のコマンドを実行して、設定が正しく反映されたか確認します。

```bash
# systemd が動作しているか確認
systemctl is-system-running
# 期待される出力: running

# PATH に Windows のパスが含まれていないか確認
echo $PATH | tr ':' '\n' | grep -c '/mnt/c'
# 期待される出力: 0
```

## デフォルトのディストリビューションを設定する

デフォルトのディストリビューションを切り替える場合には以下のコマンドを使用する

```powershell
wsl --set-default <ディストリビューション名>
```

## WSL環境のエクスポートとインポート（バックアップ・別名インストール）

### エクスポート：WSL環境をバックアップする

```powershell
wsl --export <WSL名> <保存先ファイル名>.tar
```

（例）Ubuntu-22.04 をバックアップ：

```powershell
wsl --export Ubuntu-22.04 D:\backup\ubuntu_backup.tar
```

### インポート：新しい名前で再登録する（別環境として使いたいとき）

```powershell
wsl --import <新しいWSL名> <保存先フォルダ> <エクスポートファイル>.tar
```

（例）「dev」という名前でインポート：

```powershell
wsl --import dev D:\wsl\dev D:\backup\ubuntu_backup.tar
```

:::message
インポート時に保存先フォルダは空である必要があります。
この方法を使うと、同じLinux環境を複数名で共有したり、別名で開発環境を複製したりできます。
:::

## VSCodeの拡張機能であるRemote DevelopmentでWSLを管理しよう

1. VSCodeをインストールします
2. 拡張機能を開き、`Remote Development` をインストールしてください
3. ウィンドウアイコンからWSL2環境を選択・起動できます

## まとめ

WSL2を利用すると、Windows上でLinux環境を構築し、Linuxコマンドを使用することができます。
特に以下の点が便利です：

- VSCode拡張機能「Remote Development」でWSL接続・操作が簡単
- 環境をエクスポート・インポートできるため、複製や共有が容易
- ディストリビューションを切り替えて複数環境を使い分けられる
- **`wsl.conf` を適切に設定することで、開発ツールが安定動作する**

## 参考リンク

- [Microsoft 公式: WSL のインストール](https://learn.microsoft.com/ja-jp/windows/wsl/install)
- [Microsoft 公式: WSL での詳細設定の構成](https://learn.microsoft.com/ja-jp/windows/wsl/wsl-config)
- [WSL2の初期設定: wsl.confの基本設定ガイド](https://zenn.dev/atsushifx/articles/wsl2-setup-wslconf-basics)
