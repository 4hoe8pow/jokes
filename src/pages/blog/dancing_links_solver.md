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

# 1. 緒言

Dancing Links（以下、DLXと略記）は、Donald E. Knuthにより提唱されたAlgorithm Xを効率的に実装するためのデータ構造であり、Exact Cover問題に対する高速な解法を提供する。
本稿では、数独（ナンプレ）をExact Cover問題として定式化し、DLXを用いた解法をC#により実装する過程を、理論的基盤の視点から考察したい。

# 2. Exact Cover問題と数独の定式化

Exact Cover問題とは、$0$ – $1$ 行列 $A = (a_{ij})$ に対して、行の部分集合 $S$ を選び、任意の列 $j$ について

$$
\sum_{i \in S} a_{ij} = 1
$$

を満たすような $S$ を求める問題である。すなわち、各列を重複なく一意にカバーする行の組合せを探索する問題である。

数独における制約は、以下の4点に要約され、すべてExact Cover形式に帰着できる：

- 各マス $(r, c)$ には、ちょうど1つの数字 $n$ を配置する（マス制約）
- 各行 $r$ には、数字 $1$〜$9$ が各1回現れる（行制約）
- 各列 $c$ には、数字 $1$〜$9$ が各1回現れる（列制約）
- 各 $3 \times 3$ ブロック $b$ には、数字 $1$〜$9$ が各1回現れる（ブロック制約）

これらの制約は、各 $(r, c, n)$ の組に対して、対応する4つの列インデックスに変換され、合計 $9 \times 9 \times 9 = 729$ 行、$4 \times 81 = 324$ 列からなる $0$–$1$行列として構築される。

# 3. Dancing Linksの構造と操作

DLXは、上述の行列における“1”の位置を双方向連結リストのノードとして管理する手法である。各ノードは左右（同一行）および上下（同一列）にポインタを有し、列ヘッダにあたる`ColumnNode`を中心に、すべてのノードが環状に接続される。

この構造により、列の`Cover`／`Uncover`操作がポインタの付け替えのみで $\mathcal{O}(n)$ 時間にて実現され、バックトラックを伴う再帰探索において高速な復元が可能となる。

## 3.1 Cover／Uncoverの実装

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

`Cover`は列ノードを環状リストから除外し、同列に属する行の他ノードも順次削除する。`Uncover`はその逆操作であり、探索木におけるバックトラック時に必要となる。

# 4. C#実装における行列構築

以下は、C#においてExact Cover行列を構築する手続きである：

```csharp
const int totalColumns = 4 * 81;
var columnList = new ColumnNode[totalColumns];
var head = new ColumnNode("head");

// 列ノードの初期化
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

// 各(r, c, n)に対して行を構築
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

各$(r, c, n)$ に対して生成される列インデックスは、以下の式により決定される：

- マス制約: $i_1 = 9r + c$

- 行制約: $i_2 = 81 + 9r + (n - 1)$

- 列制約: $i_3 = 162 + 9c + (n - 1)$

- ブロック制約: $i_4 = 243 + 9b + (n - 1)$

ここで $b = 3 \left\lfloor \frac{r}{3} \right\rfloor + \left\lfloor \frac{c}{3} \right\rfloor$ とする。

# 5. 探索アルゴリズム（Algorithm X）

再帰的探索は、最小サイズの列を選択し、その列に対応する各行を試行的に選びながら、順次`Cover`操作を適用することで進行する。解が見つからなければ`Uncover`により状態を巻き戻す。

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

# 6. 解の復元

最終的に得られたノード群から、$(r, c, n)$ を復元し、9×9の盤面を再構成する。

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

# 7. 結論

DLXとAlgorithm Xの組合せは、Exact Cover問題として定式化可能な数独に対して、極めて効率的な解法を提供する。
特に、`Cover`／`Uncover`の $\mathcal{O}(n)$ 操作と最小列選択ヒューリスティックにより、探索空間を効果的に削減できる点が注目される。

### 参考文献

1. Knuth, D. E., Dancing Links, 2000.
2. Knuth, D. E., The Art of Computer Programming, Volume 4A: Combinatorial Algorithms, 2011.
3. Russell, S. J., Norvig, P., Artificial Intelligence: A Modern Approach, 4th ed., 2020.
