---
title: "Terraformで Enter を押しても ^M が表示されて進まない問題の解決方法"
emoji: "⌨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Terraform", "Terminal", "Linux", "macOS"]
published: false
---

https://claude.ai/share/115ac0ce-7ade-4403-9b3b-4ed5362af654

## Terraformで Enter を押しても `^M` が表示されて進まない問題

### 現象

Terraform の `apply` などで確認プロンプトが表示され、`yes` と入力して Enter を押すと、`^M` が表示されて処理が進まない。

```
  Enter a value: yes^M
```

### 原因

ターミナルの `icrnl` 設定が無効になっている。

`icrnl` は「入力時に CR（キャリッジリターン）を LF（ラインフィード）に変換する」フラグ。これが無効だと、Enter キーで送られる CR がそのまま `^M` として表示され、プログラムが期待する LF が送られない。

確認コマンド：

```bash
stty -a | grep icrnl
```

`-icrnl` と表示されていたら無効になっている。

### 解決方法

```bash
stty icrnl
```

または、ターミナル設定を一括でデフォルトに戻す：

```bash
stty sane
```

### 補足

この状態は、異常終了したプログラムがターミナル設定を元に戻さなかった場合などに発生することがある。頻発するようであれば、シェルの設定ファイル（`~/.bashrc` や `~/.zshrc`）に `stty icrnl` を追加しておくと良いかもしれない。
