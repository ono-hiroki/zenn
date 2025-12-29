---
title: "もうコマンドを打ち間違えない！主要CLIツールのTab補完設定まとめ"
emoji: "⌨️"
type: "tech"
topics: ["CLI", "shell", "bash", "zsh"]
published: false
---

## はじめに

ターミナルで長いコマンドやオプションを入力するとき、「あのオプション何だっけ...」と手が止まることはありませんか？

Tab補完を設定すれば、コマンドの途中でTabキーを押すだけで候補が表示され、入力ミスも減らせます。この記事では、開発でよく使うCLIツールの補完設定方法をまとめました。

## 補完設定の基本パターン

多くのモダンなCLIツールは、以下のようなパターンで補完スクリプトを生成できます。

```bash
<command> completion <shell>
# 例: kubectl completion bash
```

生成されたスクリプトを `.bashrc` や `.zshrc` に追加することで、補完が有効になります。

---

## AWS CLI

AWSの各種サービスを操作するCLI。サブコマンドやオプションが膨大なので、補完があると非常に便利です。

### 公式ドキュメント
https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-completion.html

### 設定方法

**Bash**
```bash
# aws_completerのパスを確認
which aws_completer

# ~/.bashrc に追加
complete -C '/usr/local/bin/aws_completer' aws
```

**Zsh**
```bash
# ~/.zshrc に追加
autoload bashcompinit && bashcompinit
autoload -Uz compinit && compinit
complete -C '/usr/local/bin/aws_completer' aws
```

### 補完できるもの
- サブコマンド（`s3`, `ec2`, `lambda` など）
- オプション（`--region`, `--profile` など）
- リソース値（テーブル名、バケット名など）※ AWS CLI v2のみ

---

## Terraform

インフラをコードで管理するTerraform。`plan`, `apply`, `destroy` などのコマンドや状態管理系のサブコマンドを補完できます。

### 公式ドキュメント
https://developer.hashicorp.com/terraform/cli/commands

### 設定方法

```bash
# 自動でシェルの設定ファイルに追加される
terraform -install-autocomplete

# シェルを再起動、または設定を再読み込み
source ~/.bashrc  # or ~/.zshrc
```

### アンインストール
```bash
terraform -uninstall-autocomplete
```

---

## kubectl

Kubernetesクラスタを操作するCLI。リソース名やnamespace、Pod名まで補完してくれるので、長い名前を覚える必要がなくなります。

### 公式ドキュメント
https://kubernetes.io/docs/reference/kubectl/generated/kubectl_completion/

### 設定方法

**Bash**
```bash
# 現在のシェルで有効化
source <(kubectl completion bash)

# 永続化（~/.bashrc に追加）
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```

**Zsh**
```bash
# 現在のシェルで有効化
source <(kubectl completion zsh)

# 永続化（~/.zshrc に追加）
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
```

**エイリアスにも補完を効かせる**
```bash
alias k=kubectl
complete -o default -F __start_kubectl k
```

### 補完できるもの
- サブコマンド（`get`, `describe`, `apply` など）
- リソースタイプ（`pods`, `services`, `deployments` など）
- リソース名（実際のPod名、Service名など）
- namespace名
- オプション

---

## Docker

コンテナ操作に必須のDocker CLI。コンテナ名やイメージ名も補完対象になります。

### 公式ドキュメント
https://docs.docker.com/engine/cli/completion/

### 設定方法

**Bash**
```bash
# bash-completionのインストール（未インストールの場合）
sudo apt install bash-completion  # Debian/Ubuntu
brew install bash-completion      # macOS

# 補完スクリプトを生成して配置
docker completion bash > /etc/bash_completion.d/docker
# または
docker completion bash | sudo tee /etc/bash_completion.d/docker
```

**Zsh**
```bash
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

**Fish**
```bash
mkdir -p ~/.config/fish/completions
docker completion fish > ~/.config/fish/completions/docker.fish
```

### 補完できるもの
- サブコマンド
- オプション
- コンテナID・コンテナ名
- イメージ名・タグ
- ネットワーク名
- ボリューム名

---

## Git

バージョン管理の定番。ブランチ名やリモート名の補完は日常的に使う機能です。

### 公式ドキュメント
https://git-scm.com/book/en/v2/Appendix-A%3A-Git-in-Other-Environments-Git-in-Zsh

### 設定方法

**Bash**
```bash
# 補完スクリプトをダウンロード
curl -o ~/.git-completion.bash \
  https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash

# ~/.bashrc に追加
echo 'source ~/.git-completion.bash' >> ~/.bashrc
```

**Zsh（oh-my-zshを使う場合）**
```bash
# ~/.zshrc の plugins に git を追加
plugins=(git)
```

**Zsh（手動設定）**
```bash
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

---

## GitHub CLI (gh)

GitHubをコマンドラインから操作できるCLI。PR番号やリポジトリ名まで補完してくれます。

### 公式ドキュメント
https://cli.github.com/manual/gh_completion

### 設定方法

**Bash**
```bash
# ~/.bashrc に追加
eval "$(gh completion -s bash)"
```

**Zsh**
```bash
# ~/.zshrc に追加
eval "$(gh completion -s zsh)"
```

**Fish**
```bash
gh completion -s fish > ~/.config/fish/completions/gh.fish
```

### 補完できるもの
- サブコマンド（`pr`, `issue`, `repo` など）
- オプション
- リポジトリ名
- PR番号・Issue番号
- ブランチ名

---

## Taskfile (task)

Makefileの代替として人気のタスクランナー。タスク名を補完できます。

### 公式ドキュメント
https://taskfile.dev/docs/installation#completion

### 設定方法

**Bash**
```bash
# 現在のシェルで有効化
eval "$(task --completion bash)"

# 永続化
echo 'eval "$(task --completion bash)"' >> ~/.bashrc
```

**Zsh**
```bash
# 現在のシェルで有効化
eval "$(task --completion zsh)"

# 永続化
echo 'eval "$(task --completion zsh)"' >> ~/.zshrc
```

**Fish**
```bash
task --completion fish > ~/.config/fish/completions/task.fish
```

### 補完できるもの
- タスク名（Taskfile.ymlで定義したもの）
- オプション

---

## npm

Node.jsのパッケージマネージャー。スクリプト名やサブコマンドを補完できます。

### 公式ドキュメント
https://docs.npmjs.com/cli/commands/npm-completion

### 設定方法

**Bash**
```bash
npm completion >> ~/.bashrc
```

**Zsh**
```bash
npm completion >> ~/.zshrc
```

### 補完できるもの
- サブコマンド（`install`, `run`, `publish` など）
- `npm run` のスクリプト名
- オプション

---

## まとめ

| ツール | 補完コマンド | 対応シェル |
|--------|-------------|-----------|
| AWS CLI | `complete -C aws_completer aws` | Bash, Zsh |
| Terraform | `terraform -install-autocomplete` | Bash, Zsh |
| kubectl | `kubectl completion <shell>` | Bash, Zsh, Fish, PowerShell |
| Docker | `docker completion <shell>` | Bash, Zsh, Fish |
| Git | スクリプトをsource | Bash, Zsh |
| GitHub CLI | `gh completion -s <shell>` | Bash, Zsh, Fish, PowerShell |
| Taskfile | `task --completion <shell>` | Bash, Zsh, Fish, PowerShell |
| npm | `npm completion` | Bash, Zsh |

補完を設定することで：

- **タイプミスが減る** - 候補から選択するだけ
- **コマンドを覚える必要がない** - Tabを押せば候補が出る
- **作業スピードが上がる** - 長いリソース名も一発入力

一度設定してしまえばあとは快適に使えるので、ぜひ普段使っているツールから試してみてください！
