---
layout: ../../layouts/BlogLayout.astro
title: RustでDiffの実装
description: 最長共通部分列
tags: ["Rust"]
time: 5
featured: true
timestamp: 2025-03-20T07:39:03+00:00
filename: diff_lcs
---

## はじめに

ファイル間の差分検出はソフトウェア開発に不可欠な機能です。  
本稿では、Rustで最長共通部分列（Longest Common Subsequence; LCS）アルゴリズムを活用し、行単位で差分を検出するツールを実装する方法を紹介します。

---

## LCSアルゴリズムの概要

LCSは、2つのシーケンス $$A = a_1 a_2 \cdots a_n$$ と $$B = b_1 b_2 \cdots b_m$$ の中で、順序を保ちながら共通する最長の部分列を求めます。

動的計画法により、以下の再帰式でテーブルを構築します。

$$
dp[i][j] =
\begin{cases}
dp[i-1][j-1] + 1 & \text{if } a_i = b_j \\
\max(dp[i-1][j], dp[i][j-1]) & \text{otherwise}
\end{cases}
$$

ここで、$$dp[i][j]$$ は $$A$$ の先頭から $$i$$ 文字、$$B$$ の先頭から $$j$$ 文字までのLCSの長さを表します。

### 計算量

このアルゴリズムの計算量は、

$$
\mathcal{O}(n \times m)
$$

となり、空間計算量も同様に $$\mathcal{O}(n \times m)$$ です。

---

## 実装解説

### ファイル読み込み関数

```rust
use std::fs::File;
use std::io::{self, BufRead, BufReader};

fn read_lines(path: &str) -> io::Result<Vec<String>> {
    let file = File::open(path)?;
    let reader = BufReader::new(file);
    reader.lines().collect()
}
```

### LCSテーブル構築

```rust
fn build_lcs_table(a: &[&str], b: &[&str]) -> Vec<Vec<usize>> {
    let n = a.len();
    let m = b.len();
    let mut dp = vec![vec![0; m + 1]; n + 1];
    for i in 1..=n {
        for j in 1..=m {
            dp[i][j] = if a[i - 1] == b[j - 1] {
                dp[i - 1][j - 1] + 1
            } else {
                dp[i - 1][j].max(dp[i][j - 1])
            };
        }
    }
    dp
}
```

### 差分生成のためのバックトラック

```rust
fn backtrack<'a>(
    dp: &[Vec<usize>],
    a: &[&'a str],
    b: &[&'a str],
    i: isize,
    j: isize,
    out: &mut Vec<String>,
) {
    if i > 0 && j > 0 && a[i as usize - 1] == b[j as usize - 1] {
        backtrack(dp, a, b, i - 1, j - 1, out);
        out.push(format!("  {}", a[i as usize - 1]));
    } else if j > 0 && (i == 0 || dp[i as usize][j as usize - 1] >= dp[i as usize - 1][j as usize]) {
        backtrack(dp, a, b, i, j - 1, out);
        out.push(format!("+ {}", b[j as usize - 1]));
    } else if i > 0 {
        backtrack(dp, a, b, i - 1, j, out);
        out.push(format!("- {}", a[i as usize - 1]));
    }
}
```

### Diff本体

```rust
fn diff(file1: &str, file2: &str) -> io::Result<Vec<String>> {
    let a_lines = read_lines(file1)?;
    let b_lines = read_lines(file2)?;
    let a: Vec<&str> = a_lines.iter().map(String::as_str).collect();
    let b: Vec<&str> = b_lines.iter().map(String::as_str).collect();

    let dp = build_lcs_table(&a, &b);
    let mut res = Vec::with_capacity(a.len() + b.len());
    backtrack(&dp, &a, &b, a.len() as isize, b.len() as isize, &mut res);
    Ok(res)
}
```
