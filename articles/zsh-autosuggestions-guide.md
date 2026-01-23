---
title: "zsh-autosuggestions でコマンド入力を爆速化する"
emoji: "💡"
type: "tech"
topics: ["zsh", "shell", "CLI", "Mac", "Linux"]
published: true
---

## はじめに

ターミナルで過去に実行したコマンドを再度打とうとして、「あれ、あのオプション何だっけ」と履歴を遡った経験はありませんか？

**zsh-autosuggestions** は、入力中のコマンドに対して履歴から候補を薄いグレーで表示してくれるプラグインです。→キーを押すだけで候補を確定できるため、コマンド入力が格段に速くなります。

デモは[公式のasciinema](https://asciinema.org/a/37390)で確認できます。

この記事では、インストールから実践的なカスタマイズまでを解説します。

## インストール方法

### 方法1: Homebrew（macOS）

```sh
brew install zsh-autosuggestions
```

インストール後、`~/.zshrc` に以下を追加します。

```sh
source $(brew --prefix)/share/zsh-autosuggestions/zsh-autosuggestions.zsh
```

### 方法2: Git clone（macOS / Linux）

プラグイン用のディレクトリを作成し、リポジトリをクローンします。

```sh
mkdir -p ~/.zsh/plugins
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/plugins/zsh-autosuggestions
```

`~/.zshrc` に以下を追加します。

```sh
source ~/.zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
```

### 方法3: Oh My Zsh（プラグインとして追加）

Oh My Zshを使っている場合は、プラグインディレクトリにクローンします。

```sh
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

`~/.zshrc` の `plugins` に追加します。

```sh
plugins=(
  git
  zsh-autosuggestions  # 追加
)
```

### インストール確認

設定を反映します。

```sh
source ~/.zshrc
```

何か文字を入力して、薄いグレーでサジェストが表示されれば成功です。

## 基本的な使い方

### サジェストの確定

| 操作 | キー | 説明 |
|------|------|------|
| 全体を確定 | `→` または `End` | サジェスト全体を受け入れる |
| 単語単位で確定 | `Ctrl + →` | 次の単語まで受け入れる |
| 無視して入力 | そのまま入力 | サジェストは消えて通常入力になる |

### 実践例

```sh
# "docker" と入力すると、過去の履歴から候補が表示される
$ docker compose up -d  # ← 薄いグレーで表示

# → キーを押すと確定
$ docker compose up -d  # ← 通常の色になり確定
```

部分的に採用したい場合は、`Ctrl + →` で単語単位で確定できます。

```sh
$ docker  # ← "docker compose up -d" がサジェストされている状態

# Ctrl + → を1回押す
$ docker compose  # ← "compose" まで確定

# あとは自分で続きを入力
$ docker compose logs -f
```

## カスタマイズ

### サジェストの色を変更する

デフォルトの薄いグレーが見づらい場合、色を変更できます。

```sh
# ~/.zshrc に追加
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=#888888"
```

よく使われる設定例：

```sh
# 明るいグレー（ダークテーマ向け）
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=#888888"

# 暗いグレー（ライトテーマ向け）
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=#555555"

# 色 + 太字
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=cyan,bold"

# 色 + 下線
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=yellow,underline"
```

使える色名：`black`, `red`, `green`, `yellow`, `blue`, `magenta`, `cyan`, `white`

### サジェストのソースを変更する

サジェストの候補をどこから取得するかを設定できます。

```sh
# デフォルト: 履歴から取得
ZSH_AUTOSUGGEST_STRATEGY=(history)

# 履歴 + 補完システムから取得（より多くの候補）
ZSH_AUTOSUGGEST_STRATEGY=(history completion)

# 最近の履歴を優先しつつ、マッチするものを全体から探す
ZSH_AUTOSUGGEST_STRATEGY=(match_prev_cmd history)
```

| 戦略 | 説明 |
|------|------|
| `history` | コマンド履歴から検索（デフォルト） |
| `completion` | zshの補完システムから候補を取得 |
| `match_prev_cmd` | 直前のコマンドと同じ状況での履歴を優先 |

`match_prev_cmd` は、例えば `git add` の後に `git commit` を提案するなど、コマンドの文脈を考慮したサジェストをしてくれます。

### 確定キーを変更する

デフォルトの `→` 以外に、独自のキーバインドを追加できます。

```sh
# Ctrl + Space でサジェストを確定
bindkey '^ ' autosuggest-accept
```

よく使われるウィジェット：

| ウィジェット | 動作 |
|------------|------|
| `autosuggest-accept` | サジェストを確定 |
| `autosuggest-execute` | サジェストを確定して即実行 |
| `autosuggest-clear` | サジェストをクリア |
| `autosuggest-fetch` | サジェストを再取得 |

### バッファサイズの制限

非常に長いコマンドがサジェストされると見づらくなるため、制限をかけられます。

```sh
# 100文字を超えるサジェストは表示しない
ZSH_AUTOSUGGEST_BUFFER_MAX_SIZE=100
```

### 特定のウィジェットでサジェストをクリア

特定の操作をしたときにサジェストを消したい場合は、以下のように設定します。

```sh
# Ctrl+C でサジェストをクリア
ZSH_AUTOSUGGEST_CLEAR_WIDGETS+=(kill-line)
```

## 私の設定

参考までに、私が使っている設定を紹介します。

```sh
# ~/.zshrc

# zsh-autosuggestions
source ~/.zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh

# 履歴 + 直前のコマンドを考慮
ZSH_AUTOSUGGEST_STRATEGY=(match_prev_cmd history)

# 色（ダークテーマ用）
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=#666666"
```

シンプルな設定ですが、これだけでも十分に便利に使えています。

## トラブルシューティング

### サジェストが表示されない

1. プラグインが正しく読み込まれているか確認

```sh
# zsh-autosuggestions が読み込まれているか確認
echo $ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE
# 何か出力されればOK
```

2. `source ~/.zshrc` を実行して再読み込み

3. 履歴ファイルが存在するか確認

```sh
ls -la ~/.zsh_history
```

### サジェストの色が見えない

ターミナルのカラースキームとの相性問題の可能性があります。`ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE` で色を調整してください。

### ペースト時にサジェストが邪魔になる

大量のテキストをペーストするとき、サジェストが干渉することがあります。以下の設定で緩和できます。

```sh
# ペースト時にサジェストを無効化
ZSH_AUTOSUGGEST_IGNORE_WIDGETS+=(bracketed-paste)
```

## 関連プラグイン

zsh-autosuggestions と一緒に使うと便利なプラグインを紹介します。

| プラグイン | 説明 |
|-----------|------|
| [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting) | コマンドを色付けして、有効なコマンドかどうかを視覚的に確認できる |
| [zsh-completions](https://github.com/zsh-users/zsh-completions) | 追加の補完定義を提供 |

## まとめ

zsh-autosuggestions を使うことで：

- **履歴から候補が自動表示** され、長いコマンドを覚える必要がない
- **→キーで即確定** できるため、入力が速くなる
- **カスタマイズ可能** で、自分好みの見た目や挙動に調整できる

一度設定すれば、あとは意識せずに使えるプラグインです。まだ導入していない方はぜひ試してみてください。

## 参考リンク

- [zsh-autosuggestions 公式リポジトリ](https://github.com/zsh-users/zsh-autosuggestions)
- [zsh-autosuggestions Wiki](https://github.com/zsh-users/zsh-autosuggestions/wiki)
