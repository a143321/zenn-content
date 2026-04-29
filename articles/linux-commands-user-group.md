---
title: "Linuxコマンドまとめ：ユーザー・グループ編"
emoji: "🐧"
type: "tech"
topics: ["linux", "lpic", "command", "初心者"]
published: false
---

## はじめに

LPIC Level 1 の学習中に整理した、ユーザー・グループ管理に関するLinuxコマンドのまとめです。
実際にコマンドを実行しながら学んだ内容を、自分用のリファレンスとしてまとめています。

---

## 現在のユーザーを確認する

### `whoami` — 自分のユーザー名を表示

```bash
$ whoami
user
```

最もシンプル。「今、誰としてログインしているか？」を確認するコマンド。

### `id` — 詳細なユーザー情報を表示

```bash
$ id
uid=1000(user) gid=1000(user) groups=1000(user),4(adm),27(sudo),1001(docker)
```

| 項目 | 意味 |
|---|---|
| `uid` | ユーザーID（一意の番号） |
| `gid` | プライマリグループID |
| `groups` | 所属する全グループ（補助グループ含む） |

### `who` / `w` — ログイン中のユーザーを確認

```bash
$ who          # ログイン中のユーザーと端末を表示
$ w            # who + 何をしているかも表示
```

---

## ユーザー一覧を確認する

### `/etc/passwd` — 全ユーザーの定義ファイル

```bash
$ cat /etc/passwd
```

各行のフォーマット：

```
user:x:1000:1000:コメント:/home/user:/bin/bash
 ①   ②  ③    ④     ⑤         ⑥          ⑦
```

| # | フィールド | 説明 |
|---|---|---|
| ① | ユーザー名 | ログイン名 |
| ② | パスワード | `x` = `/etc/shadow` に格納 |
| ③ | UID | ユーザーID |
| ④ | GID | プライマリグループID |
| ⑤ | GECOS | コメント（フルネーム等） |
| ⑥ | ホームディレクトリ | ログイン時の作業ディレクトリ |
| ⑦ | ログインシェル | 使用するシェル |

### 人間のユーザーだけを抽出する

```bash
# UID 1000以上のユーザー（= 一般ユーザー）を表示
$ awk -F: '$3 >= 1000 && $1 != "nobody" { print $1, $3 }' /etc/passwd
```

:::message
**UIDの区分**
- `0` = root（管理者）
- `1〜999` = システムユーザー（サービス用）
- `1000〜` = 一般ユーザー（人間）
:::

---

## グループを確認する

### `groups` — 自分の所属グループを表示

```bash
$ groups
user adm cdrom sudo dip plugdev users docker
```

### `/etc/group` — 全グループの定義ファイル

```bash
$ cat /etc/group
```

各行のフォーマット：

```
sudo:x:27:user
 ①  ② ③   ④
```

| # | フィールド | 説明 |
|---|---|---|
| ① | グループ名 | |
| ② | パスワード | 通常 `x` |
| ③ | GID | グループID |
| ④ | メンバー | カンマ区切りのユーザー一覧 |

### よく見るグループの意味

| グループ | 意味 |
|---|---|
| `sudo` | `sudo` コマンドで管理者権限を実行可能 |
| `docker` | `sudo` なしで Docker コマンドを実行可能 |
| `adm` | システムログ（`/var/log`）の閲覧が可能 |
| `www-data` | Webサーバー（Apache/Nginx）のプロセス用 |

---

## ユーザーの作成・削除

### `useradd` — ユーザーを作成

```bash
# 基本
$ sudo useradd newuser

# ホームディレクトリも作成（-m）、シェル指定（-s）
$ sudo useradd -m -s /bin/bash newuser

# グループも指定（-G で補助グループ追加）
$ sudo useradd -m -s /bin/bash -G sudo,docker newuser
```

| オプション | 意味 |
|---|---|
| `-m` | ホームディレクトリを自動作成 |
| `-s` | ログインシェルを指定 |
| `-G` | 補助グループを指定（カンマ区切り） |
| `-g` | プライマリグループを指定 |
| `-d` | ホームディレクトリのパスを指定 |

### `userdel` — ユーザーを削除

```bash
$ sudo userdel newuser          # ユーザーのみ削除
$ sudo userdel -r newuser       # ホームディレクトリも削除
```

### `usermod` — ユーザー情報を変更

```bash
# グループに追加（-a -G は「追加」、-G だけだと「置き換え」なので注意！）
$ sudo usermod -aG docker user

# ログインシェルを変更
$ sudo usermod -s /bin/zsh user
```

:::message alert
**要注意**: `usermod -G` を `-a` なしで使うと、既存の補助グループが全て外れます！
必ず `-aG`（追加モード）を使いましょう。
:::

---

## グループの作成・削除

### `groupadd` / `groupdel`

```bash
$ sudo groupadd developers      # グループ作成
$ sudo groupdel developers      # グループ削除
```

---

## パスワード管理

### `passwd` — パスワードを変更

```bash
$ passwd                 # 自分のパスワードを変更
$ sudo passwd newuser    # 他のユーザーのパスワードを変更（root権限必要）
```

### `/etc/shadow` — パスワードの保存先

```bash
$ sudo cat /etc/shadow   # root権限が必要
```

:::message
`/etc/passwd` にはパスワードは保存されていません（`x` と表示される）。
実際のハッシュ化されたパスワードは `/etc/shadow` に格納されています。
:::

---

## まとめ：コマンド早見表

| やりたいこと | コマンド |
|---|---|
| 自分のユーザー名を確認 | `whoami` |
| 自分の詳細情報を確認 | `id` |
| 自分のグループを確認 | `groups` |
| ログイン中のユーザーを確認 | `who` / `w` |
| 全ユーザー一覧 | `cat /etc/passwd` |
| 全グループ一覧 | `cat /etc/group` |
| ユーザー作成 | `sudo useradd -m -s /bin/bash ユーザー名` |
| ユーザー削除 | `sudo userdel -r ユーザー名` |
| グループに追加 | `sudo usermod -aG グループ名 ユーザー名` |
| パスワード変更 | `passwd` / `sudo passwd ユーザー名` |

---

## 関連ファイル一覧

| ファイル | 内容 | 権限 |
|---|---|---|
| `/etc/passwd` | ユーザー定義 | 誰でも読める |
| `/etc/shadow` | パスワード（ハッシュ） | root のみ |
| `/etc/group` | グループ定義 | 誰でも読める |
| `/etc/gshadow` | グループパスワード | root のみ |
