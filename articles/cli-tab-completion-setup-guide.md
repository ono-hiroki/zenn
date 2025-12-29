---
title: "主要CLIツールのTab補完設定ガイド【AWS/Terraform/kubectl/Docker他】"
emoji: "⌨️"
type: "tech"
topics: ["CLI", "shell", "bash", "zsh"]
published: true
---

## はじめに

ターミナルで長いコマンドやオプションを入力するとき、「あのオプション何だっけ...」と手が止まることはありませんか？

Tab補完を設定すれば、コマンドの途中でTabキーを押すだけで候補が表示され、入力ミスも減らせます。この記事では、開発でよく使うCLIツールの補完設定方法をまとめました。

## 補完設定の基本パターン

多くのモダンなCLIツールは、以下のようなパターンで補完スクリプトを生成できます。

```sh
<command> completion <shell>
# 例: kubectl completion bash
```

生成されたスクリプトを `.bashrc` や `.zshrc` に追加することで、補完が有効になります。

## AWS CLI

AWSの各種サービスを操作するCLI。サブコマンドやオプションが膨大なため、補完があると便利です。

**公式ドキュメント**: [Command completion](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-completion.html)

### 設定方法

**Bash**

```sh
# aws_completerのパスを確認
which aws_completer

# ~/.bashrc に追加
complete -C '/usr/local/bin/aws_completer' aws
```

**Zsh**

```sh
# ~/.zshrc に追加
autoload bashcompinit && bashcompinit
autoload -Uz compinit && compinit
complete -C '/usr/local/bin/aws_completer' aws
```

### 補完できるもの

- サブコマンド（`s3`, `ec2`, `lambda` など）
- オプション（`--region`, `--profile` など）
- リソース値（テーブル名、バケット名など）※ AWS CLI v2のみ

## Terraform

インフラをコードで管理するTerraform。`plan`, `apply`, `destroy` などのコマンドを補完できます。

**公式ドキュメント**: [CLI Commands](https://developer.hashicorp.com/terraform/cli/commands)

### 設定方法

```sh
# 自動でシェルの設定ファイルに追加される
terraform -install-autocomplete

# シェルを再起動、または設定を再読み込み
source ~/.bashrc  # または ~/.zshrc
```

### 補完できるもの

- サブコマンド（`plan`, `apply`, `destroy` など）
- オプション

## kubectl

Kubernetesクラスタを操作するCLI。リソース名やNamespace、Pod名まで補完できるため、長い名前を覚える必要がなくなります。

**公式ドキュメント**: [kubectl completion](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_completion/)

### 設定方法

**Bash**

```sh
# 現在のシェルで有効化
source <(kubectl completion bash)

# 永続化（~/.bashrc に追加）
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```

**Zsh**

```sh
# 現在のシェルで有効化
source <(kubectl completion zsh)

# 永続化（~/.zshrc に追加）
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
```

### エイリアスにも補完を効かせる

**Bash**

```sh
alias k=kubectl
complete -o default -F __start_kubectl k
```

**Zsh**

```sh
alias k=kubectl
compdef k=kubectl
```

### 補完できるもの

- サブコマンド（`get`, `describe`, `apply` など）
- リソースタイプ（`pods`, `services`, `deployments` など）
- リソース名（実際のPod名、Service名など）
- Namespace名
- オプション

## Docker

コンテナ操作に必須のDocker CLI。コンテナ名やイメージ名も補完対象になります。

**公式ドキュメント**: [Completion](https://docs.docker.com/engine/cli/completion/)

### 設定方法

**Bash**

```sh
# bash-completionのインストール（未インストールの場合）
sudo apt install bash-completion  # Debian/Ubuntu
brew install bash-completion      # macOS

# 補完スクリプトを生成して配置
sudo sh -c 'docker completion bash > /etc/bash_completion.d/docker'
```

**Zsh**

```sh
# 補完ディレクトリを作成
mkdir -p ~/.docker/completions

# 補完スクリプトを生成
docker completion zsh > ~/.docker/completions/_docker

# ~/.zshrc に追加
cat <<'EOF' >> ~/.zshrc
FPATH="$HOME/.docker/completions:$FPATH"
autoload -Uz compinit
compinit
EOF
```

### 補完できるもの

- サブコマンド（`run`, `build`, `ps` など）
- オプション
- コンテナID・コンテナ名
- イメージ名・タグ
- ネットワーク名
- ボリューム名

## Git

バージョン管理の定番。ブランチ名やリモート名の補完は日常的に役立ちます。

**公式ドキュメント**: [Git in Bash](https://git-scm.com/book/en/v2/Appendix-A:-Git-in-Other-Environments-Git-in-Bash)

### 設定方法

**Bash**

```sh
# 補完スクリプトをダウンロード
curl -o ~/.git-completion.bash \
  https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash

# ~/.bashrc に追加
echo 'source ~/.git-completion.bash' >> ~/.bashrc
```

**Zsh（Oh My Zshを使う場合）**

```sh
# ~/.zshrc の plugins に git を追加
plugins=(git)
```

**Zsh（手動設定）**

```sh
# 補完スクリプトをダウンロード
mkdir -p ~/.zsh
curl -o ~/.zsh/_git \
  https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.zsh
curl -o ~/.zsh/git-completion.bash \
  https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash

# ~/.zshrc に追加
fpath=(~/.zsh $fpath)
autoload -Uz compinit && compinit
```

### 補完できるもの

- サブコマンド（`commit`, `push`, `checkout` など）
- ブランチ名（ローカル・リモート）
- タグ名
- リモート名
- ファイルパス
- オプション

## GitHub CLI (gh)

GitHubをコマンドラインから操作できるCLI。PR番号やリポジトリ名まで補完できます。

**公式ドキュメント**: [gh completion](https://cli.github.com/manual/gh_completion)

### 設定方法

**Bash**

```sh
# ~/.bashrc に追加
eval "$(gh completion -s bash)"
```

**Zsh**

```sh
# ~/.zshrc に追加
eval "$(gh completion -s zsh)"
```

### 補完できるもの

- サブコマンド（`pr`, `issue`, `repo` など）
- オプション
- リポジトリ名
- PR番号・Issue番号
- ブランチ名

## Taskfile (task)

Makefileの代替として人気のタスクランナー。定義したタスク名を補完できます。

**公式ドキュメント**: [Installation - Completion](https://taskfile.dev/installation/#setup-completions)

### 設定方法

**Bash**

```sh
# 現在のシェルで有効化
eval "$(task --completion bash)"

# 永続化
echo 'eval "$(task --completion bash)"' >> ~/.bashrc
```

**Zsh**

```sh
# 現在のシェルで有効化
eval "$(task --completion zsh)"

# 永続化
echo 'eval "$(task --completion zsh)"' >> ~/.zshrc
```

### 補完できるもの

- タスク名（Taskfile.ymlで定義したもの）
- オプション

## npm

Node.jsのパッケージマネージャー。スクリプト名やサブコマンドを補完できます。

**公式ドキュメント**: [npm-completion](https://docs.npmjs.com/cli/commands/npm-completion)

### 設定方法

**Bash**

```sh
npm completion >> ~/.bashrc
```

**Zsh**

```sh
npm completion >> ~/.zshrc
```

### 補完できるもの

- サブコマンド（`install`, `run`, `publish` など）
- `npm run` のスクリプト名
- オプション

## まとめ

| ツール | 補完コマンド | 対応シェル |
|--------|-------------|-----------|
| AWS CLI | `complete -C aws_completer aws` | Bash, Zsh |
| Terraform | `terraform -install-autocomplete` | Bash, Zsh |
| kubectl | `kubectl completion <shell>` | Bash, Zsh |
| Docker | `docker completion <shell>` | Bash, Zsh |
| Git | スクリプトをsource | Bash, Zsh |
| GitHub CLI | `gh completion -s <shell>` | Bash, Zsh |
| Taskfile | `task --completion <shell>` | Bash, Zsh |
| npm | `npm completion` | Bash, Zsh |

Tab補完を設定することで：

- **タイプミスが減る** - 候補から選択するだけ
- **コマンドを覚える必要がない** - Tabを押せば候補が出る
- **作業スピードが上がる** - 長いリソース名も一発入力

一度設定してしまえばあとは快適に使えるので、ぜひ普段使っているツールから試してみてください。
