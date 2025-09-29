---
layout: ../../layouts/BlogLayout.astro
title: 無向グラフの生成関数
description: グラフ理論とRust 1.87
tags: ["rust"]
time: 2
featured: false
timestamp: 2025-08-26T10:10:31+00:00
filename: treegen
---

## 前提

```log
$ rustc -Vv
rustc 1.87.0 (17067e9ac 2025-05-09)
binary: rustc
commit-hash: 17067e9ac6d7ecb70e50f92c1944e545188d2359
commit-date: 2025-05-09
host: x86_64-pc-windows-msvc
release: 1.87.0
LLVM version: 20.1.1
```

## 実装

```rust
use rand::Rng;
use rand::prelude::IndexedMutRandom;
use rand::prelude::SliceRandom;
use rand::rng;
use std::cmp::Ordering;
use std::collections::BinaryHeap;

/// ノードの役割
#[derive(Debug, PartialEq, Clone, Copy)]
pub enum NodeType {
    Start,
    Goal,
    EventPlaceholder,
    Empty,
}

#[derive(Debug, Clone)]
pub struct Edge {
    pub to: usize,   // 接続先のノードID
    pub weight: u32, // コスト
}

#[derive(Debug, Clone)]
pub struct Node {
    pub id: usize,
    pub node_type: NodeType,
    pub edges: Vec<Edge>,
}

impl Node {
    fn new(id: usize) -> Self {
        Node {
            id,
            node_type: NodeType::Empty,
            edges: Vec::new(),
        }
    }
    fn degree(&self) -> usize {
        self.edges.len()
    }
}

/// グラフの構造と特性を定義するためのパラメータ群。
#[derive(Debug, Clone)]
pub struct GraphParams {
    /// 始点から最も近い終点までの最短経路長（コスト）の最小値を指定します。
    /// 最短経路問題に関連する制約です。グラフ生成後にダイクストラ法を用いて
    /// Startノードから各Goalノードへの最短経路コストを計算し、その最小値がこの値を
    /// 下回らないことを保証します。
    /// グラフの広がりや、タスク達成に必要な最低限の労力を担保します。
    pub min_steps: u32,

    /// グラフを構成する頂点（Vertex）の総数。
    /// グラフの「位数（Order）」とも呼ばれる、グラフの基本的なサイズを定義します。
    /// 多くのグラフアルゴリズムの計算量は、頂点数 `N`（または `|V|`）と辺数 `E`（または `|E|`）の
    /// 多項式で表現されるため、この値は生成パフォーマンスに直接影響します。
    pub max_nodes: usize,

    /// グラフ内の任意の頂点が持つことができる辺（Edge）の最大数。
    /// 頂点の「次数（Degree）」に対する上限 `Δ(G)` を設ける制約です。
    /// この値を小さく設定すると、辺の数が少なくなり「スパースグラフ（疎なグラフ）」が
    /// 生成されやすくなります。特に `max_degree = 2` の場合、生成されるグラフは
    /// パスグラフと閉路グラフの組み合わせに限定されます。
    /// 逆に値を大きくすると、デンスグラフ（密なグラフ）の生成が可能になります。
    pub max_degree: usize,

    /// 「行き止まり」同士を接続して閉路を形成する割合（0.0～1.0）。
    /// グラフの木構造（バックボーン）を生成した後、次数が1の頂点（葉頂点、Leaf）同士を
    /// 新たな辺で接続し、閉路（Cycle）を導入する処理の強度を制御します。
    /// - `0.0` の場合: グラフは木（Tree）となり、閉路は存在せず、辺の数は `N-1` となります。
    /// - `> 0.0` の場合: 辺が追加され、グラフの「環状数（Cyclomatic Number）」、
    ///   すなわち `E - N + 1`（連結グラフの場合）が増加します。
    ///   このパラメータは、グラフの「木らしさ（Treeness）」を調整する役割を持ち、
    ///   値が高いほどメッシュ状の複雑なトポロジーに近づきます。
    pub dead_end_merge_ratio: f64,

    /// グラフの「複雑さ」を制御する抽象的な係数。
    /// 生成されるグラフの構造的特性に影響を与えるためのヒューリスティックな値。
    /// Goalノードの数やEventPlaceholderノードの発生率を内生的に決定するために使用されます。
    /// 値を高くすると、攻略すべき目標地点が増えたり、イベントが発生する場所が増えたりして、探索がより複雑になります。
    pub complexity_factor: f64,
}

#[derive(Debug)]
pub struct Graph {
    pub nodes: Vec<Node>,
}

impl Graph {
    /// パラメータ構造体を受け取って新しいグラフを生成する
    pub fn new(params: &GraphParams) -> Option<Graph> {
        const MAX_ATTEMPTS: usize = 100; // min_stepsを満たすグラフ生成の最大試行回数

        for _ in 0..MAX_ATTEMPTS {
            if let Some(graph) = generate_graph_internal(
                params.max_nodes,
                params.max_degree,
                params.dead_end_merge_ratio,
                params.complexity_factor,
            ) {
                // 生成されたグラフが min_steps を満たすか検証
                let start_node_id = graph
                    .nodes
                    .iter()
                    .find(|n| n.node_type == NodeType::Start)
                    .map(|n| n.id);

                if let Some(start_id) = start_node_id {
                    let distances = dijkstra(&graph, start_id);
                    let min_dist_to_goal = graph
                        .nodes
                        .iter()
                        .filter(|n| n.node_type == NodeType::Goal)
                        .filter_map(|n| distances[n.id])
                        .min()
                        .unwrap_or(u32::MAX);

                    if min_dist_to_goal >= params.min_steps {
                        return Some(graph); // 成功
                    }
                }
            }
        }
        None // 最大試行回数内に条件を満たすグラフを生成できなかった
    }

    /// 2ノード間に双方向のエッジを追加する
    fn add_edge(&mut self, from: usize, to: usize, weight: u32) {
        self.nodes[from].edges.push(Edge { to, weight });
        self.nodes[to].edges.push(Edge { to: from, weight });
    }

    /// グラフの構造をターミナルに分かりやすく表示する
    pub fn display_as_tree(&self) {
        println!("\n--- Graph Structure (Tree View) ---");
        if let Some(start_node) = self.nodes.iter().find(|n| n.node_type == NodeType::Start) {
            let mut visited = vec![false; self.nodes.len()];

            // ルート（Startノード）の表示
            println!("● Node {} (Start)", start_node.id);

            // 再帰的に枝を表示開始
            self.display_branch_recursive(start_node.id, &mut visited, String::new());
        } else {
            println!("No start node found to begin the tree display.");
        }
        println!("-----------------------------------\n");
    }

    /// `display_as_tree`のための再帰的なヘルパー関数
    fn display_branch_recursive(&self, node_id: usize, visited: &mut Vec<bool>, prefix: String) {
        // 現在のノードを訪問済みにする
        visited[node_id] = true;

        // 表示すべき接続先（まだ訪問していないノード）をフィルタリング
        let neighbors_to_visit: Vec<&Edge> = self.nodes[node_id]
            .edges
            .iter()
            .filter(|edge| !visited[edge.to])
            .collect();

        let num_neighbors = neighbors_to_visit.len();

        for (i, edge) in neighbors_to_visit.iter().enumerate() {
            let target_node = &self.nodes[edge.to];

            // 最後の枝かどうかで記号を変える
            let connector = if i == num_neighbors - 1 {
                "└──"
            } else {
                "├──"
            };

            // 次の階層のプレフィックスを準備
            let next_prefix = if i == num_neighbors - 1 {
                prefix.clone() + "    "
            } else {
                prefix.clone() + "|   "
            };

            // 接続先のノード情報を表示
            let symbol = match target_node.node_type {
                NodeType::Start | NodeType::Goal => "●",
                _ => "○",
            };
            let type_str = format!("{:?}", target_node.node_type);
            println!(
                "{}{} [w:{}] {} Node {} ({})",
                prefix, connector, edge.weight, symbol, target_node.id, type_str
            );

            // 再帰呼び出しでさらに深く探索
            self.display_branch_recursive(target_node.id, visited, next_prefix);
        }
    }
}

/// グラフ生成の内部ロジック
fn generate_graph_internal(
    max_nodes: usize,
    max_degree: usize,
    dead_end_merge_ratio: f64,
    complexity_factor: f64,
) -> Option<Graph> {
    if max_nodes < 2 || max_degree < 1 {
        return None;
    }

    let mut rng = rng();
    let mut graph = Graph {
        nodes: (0..max_nodes).map(Node::new).collect(),
    };

    // --- Step 1: complexity_factor からパラメータを内生的に決定 ---
    let calculated_goals = 1.0 + (max_nodes as f64 * complexity_factor * 0.1).floor();
    let num_goals = (calculated_goals as usize).min(max_nodes.saturating_sub(1));
    let event_rate = (0.5 * complexity_factor).min(1.0);

    // --- Step 2: ノードの役割を割り当て ---
    let mut node_indices: Vec<usize> = (0..max_nodes).collect();
    node_indices.shuffle(&mut rng);

    // Startノード
    let start_id = node_indices.pop().unwrap();
    graph.nodes[start_id].node_type = NodeType::Start;

    // Goalノード
    for _ in 0..num_goals {
        if let Some(id) = node_indices.pop() {
            graph.nodes[id].node_type = NodeType::Goal;
        }
    }

    // EventPlaceholderノード
    let num_events = (node_indices.len() as f64 * event_rate).floor() as usize;
    for _ in 0..num_events {
        if let Some(id) = node_indices.pop() {
            graph.nodes[id].node_type = NodeType::EventPlaceholder;
        }
    }

    // --- Step 3: 木構造（バックボーン）を構築 ---
    let mut unvisited_nodes: Vec<usize> = (0..max_nodes).collect();
    unvisited_nodes.shuffle(&mut rng);

    // 最初にランダムなノードを取り出し、訪問済みリスト（木の一部）に追加
    let mut visited_nodes: Vec<usize> = Vec::with_capacity(max_nodes);
    if let Some(first_node) = unvisited_nodes.pop() {
        visited_nodes.push(first_node);
    } else {
        return None; // ノードがない場合は終了
    }

    let mut edges_count = 0;

    // 未訪問のノードがなくなるまでループ
    while let Some(u) = unvisited_nodes.pop() {
        // u を接続するための接続先を、既に訪問済みのノードからランダムに探す
        // max_degree 制約が厳しい場合、一度で接続先が見つからない可能性があるため、
        // ある程度の回数試行する。
        const MAX_CONNECTION_ATTEMPTS: usize = 10; // 接続試行の上限
        let mut connected = false;

        for _ in 0..MAX_CONNECTION_ATTEMPTS {
            // 既に木に属しているノードからランダムにvを選ぶ
            let v = *visited_nodes.choose_mut(&mut rng).unwrap();

            // 次数制約をチェック
            if graph.nodes[u].degree() < max_degree && graph.nodes[v].degree() < max_degree {
                let weight = assign_weight(&mut rng);
                graph.add_edge(u, v, weight);
                edges_count += 1;
                connected = true;
                break; // 接続に成功したので試行ループを抜ける
            }
        }

        if !connected {
            // 何度試行しても接続できなかった場合。
            // よりロバストにするなら、次数に空きがあるノードを全探索するなどの
            // フォールバック処理も考えられるが、多くの場合これでうまくいく。
            // ここでは生成失敗とする。
            return None;
        }

        // uを訪問済みリストに追加
        visited_nodes.push(u);
    }
    if edges_count < max_nodes - 1 {
        return None;
    }

    // --- Step 4: 行き止まりを結合して閉路を作る ---
    if dead_end_merge_ratio > 0.0 {
        let mut dead_ends: Vec<usize> = graph
            .nodes
            .iter()
            .filter(|n| n.degree() == 1)
            .map(|n| n.id)
            .collect();
        dead_ends.shuffle(&mut rng);

        let num_merges = ((dead_ends.len() / 2) as f64 * dead_end_merge_ratio).floor() as usize;

        for i in 0..num_merges {
            let u = dead_ends[2 * i];
            let v = dead_ends[2 * i + 1];

            let is_connected = graph.nodes[u].edges.iter().any(|edge| edge.to == v);
            if !is_connected
                && graph.nodes[u].degree() < max_degree
                && graph.nodes[v].degree() < max_degree
            {
                let weight = assign_weight(&mut rng);
                graph.add_edge(u, v, weight);
            }
        }
    }

    Some(graph)
}

/// 確率に基づいてエッジの重みを割り当て
fn assign_weight<R: Rng + ?Sized>(rng: &mut R) -> u32 {
    match rng.random::<f64>() {
        x if x < 0.70 => 1, // 70%
        x if x < 0.90 => 2, // 20%
        x if x < 0.98 => 3, // 8%
        _ => 4,             // 2%
    }
}

// --- Dijkstra法 ---
#[derive(Copy, Clone, Eq, PartialEq)]
struct State {
    cost: u32,
    position: usize,
}

impl Ord for State {
    fn cmp(&self, other: &Self) -> Ordering {
        other
            .cost
            .cmp(&self.cost)
            .then_with(|| self.position.cmp(&other.position))
    }
}

impl PartialOrd for State {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

/// 指定した始点から全ノードへの最短経路コストを計算
fn dijkstra(graph: &Graph, start_node_id: usize) -> Vec<Option<u32>> {
    let mut dist: Vec<Option<u32>> = vec![None; graph.nodes.len()];
    let mut pq = BinaryHeap::new();

    dist[start_node_id] = Some(0);
    pq.push(State {
        cost: 0,
        position: start_node_id,
    });

    while let Some(State { cost, position }) = pq.pop() {
        if let Some(d) = dist[position] {
            if cost > d {
                continue;
            }
        }

        for edge in &graph.nodes[position].edges {
            let next = State {
                cost: cost + edge.weight,
                position: edge.to,
            };
            if dist[next.position].is_none() || next.cost < dist[next.position].unwrap() {
                pq.push(next);
                dist[next.position] = Some(next.cost);
            }
        }
    }
    dist
}

fn main() {
    // パラメータを構造体で定義
    let params = GraphParams {
        min_steps: 10,
        max_nodes: 50,
        max_degree: 4,
        dead_end_merge_ratio: 0.2,
        complexity_factor: 0.5,
    };

    println!("Generating graph with parameters: {:?}", params);

    // 構造体を渡してグラフを生成
    if let Some(graph) = Graph::new(&params) {
        println!(
            "Successfully generated a graph with {} nodes.",
            graph.nodes.len()
        );
        let start_node = graph
            .nodes
            .iter()
            .find(|n| n.node_type == NodeType::Start)
            .unwrap();
        let goal_nodes_count = graph
            .nodes
            .iter()
            .filter(|n| n.node_type == NodeType::Goal)
            .count();
        println!("- Start node: {}", start_node.id);
        println!("- Goal nodes: {}", goal_nodes_count);

        let distances = dijkstra(&graph, start_node.id);
        let min_dist = graph
            .nodes
            .iter()
            .filter(|n| n.node_type == NodeType::Goal)
            .filter_map(|n| distances[n.id])
            .min()
            .unwrap_or(u32::MAX);
        println!("- Minimum steps to a goal: {}", min_dist);

        graph.display_as_tree();
    } else {
        println!(
            "Failed to generate a graph that meets the criteria within the specified attempts."
        );
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::VecDeque;

    fn is_connected(graph: &Graph) -> bool {
        if graph.nodes.is_empty() {
            return true;
        }
        let mut visited = vec![false; graph.nodes.len()];
        let mut queue = VecDeque::new();
        queue.push_back(0);
        visited[0] = true;
        let mut count = 1;
        while let Some(u) = queue.pop_front() {
            for edge in &graph.nodes[u].edges {
                if !visited[edge.to] {
                    visited[edge.to] = true;
                    queue.push_back(edge.to);
                    count += 1;
                }
            }
        }
        count == graph.nodes.len()
    }

    #[test]
    fn test_graph_generation_scenarios() {
        // テストケースをパラメータ構造体のベクターとして定義
        let test_cases = vec![
            (
                "【基本】標準的なパラメータ",
                GraphParams {
                    min_steps: 5,
                    max_nodes: 30,
                    max_degree: 4,
                    dead_end_merge_ratio: 0.3,
                    complexity_factor: 0.5,
                },
            ),
            (
                "【純粋な木】閉路なし",
                GraphParams {
                    min_steps: 0,
                    max_nodes: 25,
                    max_degree: 5,
                    dead_end_merge_ratio: 0.0,
                    complexity_factor: 0.2,
                },
            ),
            (
                "【閉路が多い】行き止まりをすべて結合",
                GraphParams {
                    min_steps: 0,
                    max_nodes: 25,
                    max_degree: 5,
                    dead_end_merge_ratio: 1.0,
                    complexity_factor: 0.4,
                },
            ),
            (
                "【線形/低密度】次数が2に制限される",
                GraphParams {
                    min_steps: 10,
                    max_nodes: 40,
                    max_degree: 2,
                    dead_end_merge_ratio: 0.1,
                    complexity_factor: 0.3,
                },
            ),
            (
                "【高密度】次数上限が高い",
                GraphParams {
                    min_steps: 3,
                    max_nodes: 50,
                    max_degree: 10,
                    dead_end_merge_ratio: 0.5,
                    complexity_factor: 0.6,
                },
            ),
            (
                "【大規模】ノード数が多い",
                GraphParams {
                    min_steps: 10,
                    max_nodes: 100,
                    max_degree: 5,
                    dead_end_merge_ratio: 0.2,
                    complexity_factor: 0.5,
                },
            ),
            (
                "【高難度】min_stepsの要求が厳しい",
                GraphParams {
                    min_steps: 20,
                    max_nodes: 50,
                    max_degree: 3,
                    dead_end_merge_ratio: 0.1,
                    complexity_factor: 0.7,
                },
            ),
        ];

        for (name, params) in test_cases {
            println!("--- Testing Scenario: {} ---", name);

            if let Some(graph) = Graph::new(&params) {
                assert_eq!(
                    graph.nodes.len(),
                    params.max_nodes,
                    "[{}] Node count mismatch",
                    name
                );
                assert!(is_connected(&graph), "[{}] Graph is not connected", name);
                let start_nodes = graph
                    .nodes
                    .iter()
                    .filter(|n| n.node_type == NodeType::Start)
                    .count();
                assert_eq!(
                    start_nodes, 1,
                    "[{}] There should be exactly one start node",
                    name
                );
                let goal_nodes = graph
                    .nodes
                    .iter()
                    .filter(|n| n.node_type == NodeType::Goal)
                    .count();
                assert!(
                    goal_nodes >= 1,
                    "[{}] There should be at least one goal node",
                    name
                );

                for node in &graph.nodes {
                    assert!(
                        node.degree() <= params.max_degree,
                        "[{}] Node {} exceeds max degree (degree: {}, max: {})",
                        name,
                        node.id,
                        node.degree(),
                        params.max_degree
                    );
                }

                let total_degree: usize = graph.nodes.iter().map(|n| n.degree()).sum();
                let num_edges = total_degree / 2;
                if params.dead_end_merge_ratio == 0.0 {
                    assert_eq!(
                        num_edges,
                        params.max_nodes - 1,
                        "[{}] A pure tree must have N-1 edges",
                        name
                    );
                } else {
                    assert!(
                        num_edges >= params.max_nodes - 1,
                        "[{}] With merging, edges should be >= N-1",
                        name
                    );
                }

                let start_id = graph
                    .nodes
                    .iter()
                    .find(|n| n.node_type == NodeType::Start)
                    .unwrap()
                    .id;
                let distances = dijkstra(&graph, start_id);
                let min_dist_to_goal = graph
                    .nodes
                    .iter()
                    .filter(|n| n.node_type == NodeType::Goal)
                    .filter_map(|n| distances[n.id])
                    .min()
                    .unwrap_or(u32::MAX);
                assert!(
                    min_dist_to_goal >= params.min_steps,
                    "[{}] Graph does not meet min_steps requirement. (min_dist: {}, expected: >= {})",
                    name,
                    min_dist_to_goal,
                    params.min_steps
                );
            } else {
                println!(
                    "Warning: [{}] Failed to generate a graph that meets the criteria within the specified attempts.",
                    name
                );
            }
        }
    }

    #[test]
    fn test_generation_failure_on_bad_params() {
        let params_too_few_nodes = GraphParams {
            min_steps: 0,
            max_nodes: 1,
            max_degree: 1,
            dead_end_merge_ratio: 0.0,
            complexity_factor: 0.5,
        };
        assert!(
            Graph::new(&params_too_few_nodes).is_none(),
            "Should fail with too few nodes"
        );

        let params_zero_degree = GraphParams {
            min_steps: 0,
            max_nodes: 10,
            max_degree: 0,
            dead_end_merge_ratio: 0.0,
            complexity_factor: 0.5,
        };
        assert!(
            Graph::new(&params_zero_degree).is_none(),
            "Should fail with max_degree of 0"
        );
    }
}

#[cfg(test)]
mod enhanced_tests {
    use super::*;
    use std::collections::HashMap;
    use std::collections::VecDeque;

    /// グラフの連結性をBFSで確認
    fn is_connected(graph: &Graph) -> bool {
        if graph.nodes.is_empty() {
            return true;
        }
        let mut visited = vec![false; graph.nodes.len()];
        let mut queue = VecDeque::new();
        queue.push_back(0);
        visited[0] = true;
        let mut count = 1;
        while let Some(u) = queue.pop_front() {
            for edge in &graph.nodes[u].edges {
                if !visited[edge.to] {
                    visited[edge.to] = true;
                    queue.push_back(edge.to);
                    count += 1;
                }
            }
        }
        count == graph.nodes.len()
    }

    #[test]
    fn test_bidirectional_edges() {
        let params = GraphParams {
            min_steps: 0,
            max_nodes: 50,
            max_degree: 5,
            dead_end_merge_ratio: 0.5,
            complexity_factor: 0.5,
        };
        let graph = Graph::new(&params).expect("Graph generation failed");

        for node in &graph.nodes {
            for edge in &node.edges {
                let reciprocal = &graph.nodes[edge.to].edges;
                assert!(
                    reciprocal.iter().any(|e| e.to == node.id),
                    "Edge from {} to {} is not bidirectional",
                    node.id,
                    edge.to
                );
            }
        }
    }

    #[test]
    fn test_multiple_goal_shortest_path() {
        let params = GraphParams {
            min_steps: 5,
            max_nodes: 30,
            max_degree: 4,
            dead_end_merge_ratio: 0.2,
            complexity_factor: 0.8,
        };
        let graph = Graph::new(&params).expect("Graph generation failed");
        let start_id = graph
            .nodes
            .iter()
            .find(|n| n.node_type == NodeType::Start)
            .unwrap()
            .id;
        let distances = dijkstra(&graph, start_id);

        let goal_nodes: Vec<&Node> = graph
            .nodes
            .iter()
            .filter(|n| n.node_type == NodeType::Goal)
            .collect();
        assert!(!goal_nodes.is_empty(), "Should have at least one goal node");

        for goal in goal_nodes {
            let dist = distances[goal.id].unwrap();
            assert!(
                dist >= params.min_steps,
                "Goal {} distance {} < min_steps {}",
                goal.id,
                dist,
                params.min_steps
            );
        }
    }

    #[test]
    fn test_goal_distance_statistics() {
        let params = GraphParams {
            min_steps: 5,
            max_nodes: 50,
            max_degree: 4,
            dead_end_merge_ratio: 0.2,
            complexity_factor: 0.5,
        };

        if let Some(graph) = Graph::new(&params) {
            let start_id = graph
                .nodes
                .iter()
                .find(|n| n.node_type == NodeType::Start)
                .unwrap()
                .id;

            let distances = dijkstra(&graph, start_id);

            let goal_distances: Vec<u32> = graph
                .nodes
                .iter()
                .filter(|n| n.node_type == NodeType::Goal)
                .filter_map(|n| distances[n.id])
                .collect();

            assert!(!goal_distances.is_empty(), "No goal nodes found");

            let max_dist = *goal_distances.iter().max().unwrap();
            let avg_dist = goal_distances.iter().sum::<u32>() as f64 / goal_distances.len() as f64;

            println!(
                "Goal nodes distance stats - max: {}, avg: {:.2}, min_steps required: {}",
                max_dist, avg_dist, params.min_steps
            );

            assert!(
                max_dist <= params.max_nodes as u32 * 4,
                "Max distance too large"
            );
            assert!(
                avg_dist >= params.min_steps as f64,
                "Average distance below min_steps"
            );
        } else {
            panic!("Graph generation failed");
        }
    }

    #[test]
    fn test_edge_weight_distribution() {
        let mut counter: HashMap<u32, usize> = HashMap::new();
        let mut rng = rng();
        for _ in 0..10000 {
            let w = assign_weight(&mut rng);
            *counter.entry(w).or_default() += 1;
        }
        // 基本的な範囲確認
        assert!(counter.contains_key(&1), "Weight 1 missing");
        assert!(counter.contains_key(&2), "Weight 2 missing");
        assert!(counter.contains_key(&3), "Weight 3 missing");
        assert!(counter.contains_key(&4), "Weight 4 missing");
        // おおよその分布確認（誤差10%以内）
        let total: f64 = counter.values().sum::<usize>() as f64;
        let p1 = counter[&1] as f64 / total;
        let p2 = counter[&2] as f64 / total;
        let p3 = counter[&3] as f64 / total;
        let p4 = counter[&4] as f64 / total;
        assert!((p1 - 0.7).abs() < 0.1, "Weight 1 probability out of range");
        assert!((p2 - 0.2).abs() < 0.05, "Weight 2 probability out of range");
        assert!(
            (p3 - 0.08).abs() < 0.03,
            "Weight 3 probability out of range"
        );
        assert!(
            (p4 - 0.02).abs() < 0.02,
            "Weight 4 probability out of range"
        );
    }

    #[test]
    fn test_random_generation_consistency() {
        let params = GraphParams {
            min_steps: 5,
            max_nodes: 30,
            max_degree: 4,
            dead_end_merge_ratio: 0.3,
            complexity_factor: 0.5,
        };
        for _ in 0..10 {
            let graph = Graph::new(&params).expect("Graph generation failed");
            assert!(is_connected(&graph));
            for node in &graph.nodes {
                assert!(node.degree() <= params.max_degree);
            }
        }
    }

    #[test]
    fn test_large_scale_graph() {
        let params = GraphParams {
            min_steps: 10,
            max_nodes: 1000,
            max_degree: 5,
            dead_end_merge_ratio: 0.2,
            complexity_factor: 0.5,
        };
        let graph = Graph::new(&params).expect("Failed to generate large graph");
        assert_eq!(graph.nodes.len(), 1000);
        assert!(is_connected(&graph));
    }
}
```
