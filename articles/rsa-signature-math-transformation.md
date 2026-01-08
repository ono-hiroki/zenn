---
title: "RSA署名の仕組みを式変形で理解する"
emoji: "✍️"
type: "tech"
topics: ["rsa", "cryptography", "signature", "security"]
published: false
---

## はじめに

RSA暗号の仕組みを理解したら、RSA署名は簡単です。実は、暗号化と署名は同じ数学的原理で動いています。

この記事では、RSA暗号の知識を前提に、署名の仕組みを説明します。

:::message
この記事は「[RSA暗号はなぜ復号で元に戻るのか](https://zenn.dev/sonicmoov/articles/rsa-math-principles)」の続編です。
:::

## 2つの性質

RSA署名が成り立つのは、次の2つの性質のおかげです。

### 性質1：$m^{ed} \equiv m \pmod{n}$

RSA暗号の記事で説明した通り、オイラーの定理により：

$$
m^{ed} \equiv m^{1 + L \cdot k} \equiv m \cdot (m^L)^k \equiv m \cdot 1 \equiv m \pmod{n}
$$

何かを $ed$ 乗して $n$ で割った余りを取ると元に戻る、という性質です。

### 性質2：指数の順序は入れ替えられる

指数法則により：

$$
(m^e)^d \equiv m^{ed} \pmod{n}
$$

$$
(m^d)^e \equiv m^{de} \equiv m^{ed} \pmod{n}
$$

つまり：

$$
(m^e)^d \equiv (m^d)^e \equiv m^{ed} \pmod{n}
$$

$e$ 乗してから $d$ 乗しても、$d$ 乗してから $e$ 乗しても、$n$ で割った余りは同じです。

これは当たり前に見えますが、実は当たり前ではありません。例えば行列の場合、$AB \neq BA$ となることが多く、演算の順序を入れ替えると結果が変わります。RSA では実数の掛け算（可換）を使っているからこそ、$ed = de$ が成り立ち、順序を入れ替えられるのです。

## 暗号化と署名の比較

この2つの性質を使うと、暗号化も署名も同じ原理で説明できます。

### 暗号化→復号

$$
m \rightarrow m^e \bmod n \rightarrow (m^e)^d \bmod n \equiv m^{ed} \equiv m \pmod{n}
$$

公開鍵（$e$）で暗号化して、秘密鍵（$d$）で復号すると元に戻る。

### 署名→検証

$$
m \rightarrow m^d \bmod n \rightarrow (m^d)^e \bmod n \equiv m^{ed} \equiv m \pmod{n}
$$

秘密鍵（$d$）で署名して、公開鍵（$e$）で検証すると元に戻る。

どちらも最終的に $m^{ed} \bmod n$ になるので、元の $m$ に戻ります。

|  | 最初 | 次 | 結果 |
| --- | --- | --- | --- |
| 暗号化→復号 | 公開鍵で $m^e$ | 秘密鍵で $(m^e)^d$ | $m$ |
| 署名→検証 | 秘密鍵で $m^d$ | 公開鍵で $(m^d)^e$ | $m$ |

鍵の使う順番が逆なだけです。

## 署名の式

改めて署名の式を整理します。

### 署名の生成

秘密鍵 $d$ を使って署名 $s$ を作成します。

$$
s = m^d \bmod n
$$

### 署名の検証

公開鍵 $e$ を使って署名 $s$ を検証します。

$$
s^e \bmod n = m' \text{ を計算し、} m' = m \text{ なら署名は正しい}
$$

### 式変形

$$
\begin{aligned}
s^e \bmod n &\equiv (m^d)^e \bmod n & \leftarrow s = m^d \bmod n \\
&\equiv m^{de} \bmod n & \leftarrow \text{指数法則} \\
&\equiv m^{ed} \bmod n & \leftarrow \text{かけ算は順序を入れ替えられる} \\
&\equiv m \bmod n & \leftarrow \text{性質1}
\end{aligned}
$$

## なぜ署名として機能するのか

### 秘密鍵を持つ人だけが署名できる

署名 $s = m^d \bmod n$ を計算するには秘密鍵 $d$ が必要です。$d$ を知らない人は署名を作れません。

### 公開鍵で誰でも検証できる

検証は $s^e \bmod n = m$ を確かめるだけです。公開鍵 $e$ は公開されているので、誰でも検証できます。

### 改ざんを検出できる

メッセージ $m$ を $m'$ に改ざんすると、$s^e \bmod n \neq m'$ となり、検証に失敗します。

## まとめ

RSA署名が成り立つ理由は2つの性質に集約されます。

1. $m^{ed} \equiv m \pmod{n}$（オイラーの定理）
2. $(m^e)^d \equiv (m^d)^e \equiv m^{ed} \pmod{n}$（指数法則）

暗号化と署名は同じ数学的原理で動いており、鍵を使う順番が逆なだけです。

|  | 順番 | 用途 |
| --- | --- | --- |
| 暗号化 | 公開鍵 → 秘密鍵 | 秘密にしたい |
| 署名 | 秘密鍵 → 公開鍵 | 本人確認したい |
