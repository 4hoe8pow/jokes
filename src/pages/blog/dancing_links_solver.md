---
layout: ../../layouts/BlogLayout.astro
title: ナンプレのダンシングリンクによる解法
description: Dancing Links（DLX）とAlgorithm Xを用いたナンプレ（数独）解法の学術的解説
tags: ["ナンプレ", "ダンシングリンク", "DLX", "Algorithm X", "C#"]
time: 15
featured: false
timestamp: 2025-05-23T10:00:00+09:00
filename: dancing_links_solver
---

# はじめに

ダンシングリンク（Dancing Links, DLX）は、Donald E. Knuthが提唱したAlgorithm Xを効率的に実装する手法であり、Exact Cover問題を高速に解決できる。ナンプレ（数独）は各行・列・3×3ブロックに1から9までの数字が一度ずつ出現する制約を持ち、Exact Cover問題として定式化可能である。本稿では、提示のC#実装をベースに、行列の構築からCover／Uncover操作、再帰探索までの一連の流れを学術的視点で解説する。

## Exact Cover問題とAlgorithm X

Exact Cover問題は、0–1行列 $A = (a_{ij})$ に対し、行の部分集合 $S$ を選び、すべての列 $j$ について

$$
\sum_{i \in S} a_{ij} = 1
$$

を満たす $S$ を求める問題です。すなわち、各列をちょうど1回だけカバーする行の集合を選びます。

ナンプレ（数独）は、次の制約をExact Cover問題として表現できます：
- 各マス $(r, c)$ に数字 $n$ を1つだけ入れる
- 各行 $r$ に各数字 $n$ が1回だけ現れる
- 各列 $c$ に各数字 $n$ が1回だけ現れる
- 各ブロック $b$ に各数字 $n$ が1回だけ現れる

これらは、0–1行列の各行が $(r, c, n)$ の組み合わせ、各列が上記4制約に対応する形で構築されます。

## ダンシングリンクの実装原理

ダンシングリンク（Dancing Links）は、0–1行列の各“1”をノードとして双方向連結リストで管理し、列・行の両方向に素早くアクセスできるようにしたデータ構造です。各ノードは上下左右のポインタを持ち、列ごとに`ColumnNode`、行ごとに`DLXNode`として接続されます。これにより、行や列の「カバー」「アンカバー」操作をノードのポインタ付け替えだけでO(1)で実現できます。

## C#コードの構造概要

提示の実装は以下の主要クラスで構成される：

- `DLXNode`／`ColumnNode`：双方向連結リストノード
- `DLXMatrix`：Exact Cover行列の構築とCover／Uncover操作
- `DLXSolver`：Algorithm Xの再帰探索および行シャッフル
- `KandokuGenerator`：DLXを用いた解生成とマスク処理

以降、行列構築から探索まで順を追って詳述する。

## 1. Exact Cover行列の構築

```csharp
const int totalColumns = 4 * 81;
var columnList = new ColumnNode[totalColumns];
var head = new ColumnNode("head");

// ヘッダを起点に324個の列ノードを生成し、双方向連結リスト化
foreach (int columnIndex in Enumerable.Range(0, totalColumns))
{
    var columnNode = new ColumnNode(columnIndex.ToString());
    columnList[columnIndex] = columnNode;
    previousColumn.Right = columnNode;
    columnNode.Left = previousColumn;
    previousColumn = columnNode;
}
previousColumn.Right = head;
head.Left = previousColumn;
foreach (var (row, col, num) in
  from row in Enumerable.Range(0, 9)
  from col in Enumerable.Range(0, 9)
  from num in Enumerable.Range(1, 9)
  select (row, col, num))
{
    int block = row / 3 * 3 + (col / 3);
    int[] columnIndices = [
      row * 9 + col,
      81 + row * 9 + (num - 1),
      2 * 81 + col * 9 + (num - 1),
      3 * 81 + block * 9 + (num - 1)
    ];
    AddDLXRow(columnList, row, col, num, columnIndices);
}
```

上記C#コードでは、まず324個（9×9のマス、行、列、ブロックごとに81個ずつ）の`ColumnNode`を生成し、円環状の双方向連結リストとして接続します。次に、各マス・数字の組み合わせごとに4つの制約（マス、行、列、ブロック）を満たす列インデックスを計算し、`AddDLXRow`で該当する列ノードに`DLXNode`を追加します。これにより、各ノードが上下左右・列ノードと連結され、ダンシングリンクの基盤が構築されます。このとき、各 $(r, c, n)$ の組み合わせに対し、列インデックスは次のように計算されます：

- マス制約: $i_1 = 9r + c$
- 行制約: $i_2 = 81 + 9r + (n-1)$
- 列制約: $i_3 = 2 \times 81 + 9c + (n-1)$
- ブロック制約: $i_4 = 3 \times 81 + 9b + (n-1)$

ここで $b = 3 \times \left\lfloor \frac{r}{3} \right\rfloor + \left\lfloor \frac{c}{3} \right\rfloor$ です。

## 2. Cover／Uncover操作

```csharp
public static void Cover(ColumnNode col)
{
    col.Right.Left = col.Left;
    col.Left.Right = col.Right;
    for (DLXNode row = col.Down; row != col; row = row.Down)
        for (DLXNode j = row.Right; j != row; j = j.Right)
        {
            j.Down.Up = j.Up;
            j.Up.Down = j.Down;
            j.Column.Size--;
        }
}

public static void Uncover(ColumnNode col)
{
    for (DLXNode row = col.Up; row != col; row = row.Up)
        for (DLXNode j = row.Left; j != row; j = j.Left)
        {
            j.Column.Size++;
            j.Down.Up = j;
            j.Up.Down = j;
        }
    col.Right.Left = col;
    col.Left.Right = col;
}
```

`Cover`メソッドは、指定した列ノードをリストから除外し、その列に属するすべての行ノードも他の列から除外します。これはノードのポインタ操作のみで行われ、物理的な削除・復元を伴わないため、探索時のバックトラックが高速です。`Uncover`はこの逆操作で、除外したノードを元の位置に戻します。これがダンシングリンクの最大の特徴であり、Algorithm Xの効率的な実装を可能にしています。

## 3. 再帰探索とダンシングリンクの活用
```csharp
public bool Search(List<DLXNode> solution)
{
    if (header.Right == header) return true;
    ColumnNode c = (ColumnNode)header.Right;
    for (ColumnNode j = (ColumnNode)c.Right; j != header; j = (ColumnNode)j.Right)
        if (j.Size < c.Size) c = j;
    DLXMatrix.Cover(c);
    var rows = new List<DLXNode>();
    for (DLXNode r = c.Down; r != c; r = r.Down)
        rows.Add(r);
    Shuffle(rows);
    foreach (var r in rows)
    {
        solution.Add(r);
        for (var j = r.Right; j != r; j = j.Right)
            DLXMatrix.Cover(j.Column);
        if (Search(solution)) return true;
        solution.RemoveAt(solution.Count - 1);
        for (var j = r.Left; j != r; j = j.Left)
            DLXMatrix.Uncover(j.Column);
    }
    DLXMatrix.Uncover(c);
    return false;
}
```


`Search`メソッドでは、最小サイズの列（制約が最も厳しい列）を選び、その列に属する各行を順に選択して再帰的に探索します。行を選択するたびに関連する列を`Cover`し、解が見つからなければ`Uncover`で元に戻します。ダンシングリンク構造により、これらの操作が高速かつ安全に行えるため、膨大な組み合わせの中から効率的に解を探索できます。

## 4. 解の再構成

```csharp
return solution.Aggregate(new string[9, 9], (result, node) =>
{
    int id = node.RowID;
    int r = id / 81;
    int c = id / 9 % 9;
    int n = (id % 9) + 1;
    result[r, c] = ((KandokuSymbol)n).ToString();
    return result;
});
```

## まとめ
DLXとAlgorithm Xを組み合わせた本実装は、ナンプレをExact Cover問題としてモデル化し、O(1)のCover/Uncover操作と最小列選択ヒューリスティックで高速に解を探索する。学術的にも実用的にも優れたアプローチである。

### 参考文献

1. Knuth, D. E., Dancing Links, 2000.
2. Knuth, D. E., The Art of Computer Programming, Volume 4A: Combinatorial Algorithms, 2011.
3. Russell, S. J., Norvig, P., Artificial Intelligence: A Modern Approach, 4th ed., 2020.
