---
title: "実務レベルで最適化のコードを書く時の設計"
emoji: "📄"
type: "tech"
topics:
  - "optimization"
published: true
---

:::message
この記事は数理最適化アドベントカレンダー 2025 12/24 で投稿予定だったけれど間に合わなかったものです。
:::

https://qiita.com/advent-calendar/2025/mathematical-optimization

1 日遅れたら既に代理の記事が投稿されていて、これだけ書いてくれる人がたくさんいるならアドベントカレンダーもうひとつ作っておけば良かったなぁと思うなどしました。

# はじめに

アドベントカレンダーのネタに悩んで過去の記事を見ていたところ、2022 年の岩永さんの記事がふと目に入りました。

https://qiita.com/iwanaga-jiro/items/49e1bab93044cecab05b

そういえば、最適化でソルバーを使う時のコードってサンプルコードや実験用のコードが多いので、こういう記事が増えて皆がどんな設計をしているのか参考にできると嬉しいですよね。

岩永さんの記事でも次のような記述があったので、実務で同じ問題を扱うならこう書くという構成を自分でも用意してみました。

> 本記事では、数理最適化プログラムのクラス設計の一例を紹介しました。本来、クラス設計は自由であるものですが、数理最適化プログラムの特徴を考慮するとテンプレートがあってもよいと思います。

大まかな指針については「上記の記事とだいたい同じで」で話が終わってしまうので、本記事では細かい部分のこだわりを書いていこうと思います。
**あくまで自分ならという構成で思想強め**なので、参考程度に読んでください。

ちなみに、ソルバーは [SCIP](https://www.scipopt.org/) を使っています。

# コード設計

説明のベースとなるサンプルコードはこちらです。

https://github.com/sfujiwara/opt-template

だいたい以下のようなディレクトリ構成にしています。

```
.
├── docs
│   └── formulation.ipynb
├── LICENSE
├── optimizers
│   ├── entities.py
│   ├── __init__.py
│   ├── io.py
│   ├── optimizer.py
│   └── visualizer.py
├── pyproject.toml
├── README.md
├── tasks.py
├── testdata
│   └── example01.json
└── uv.lock
```

## ドキュメント

ソルバーを使ったコードは事前に定式化を理解しておかないと単体で読むのは無理なので、必ずリポジトリにはドキュメントを置きましょう。Jupyter Notebook は LaTeX の数式を表示できて GitHub 上でもプレビューが可能なので、定式化ドキュメントを書くのにおすすめです。

## タスクランナー

何を使っても良いけれど、間違いなくあった方が良いです。サンプルでは [Invoke](https://www.pyinvoke.org/) を使っているので `task.py` にタスクを定義しています。

## 最適化モジュール

最適化の部分は `optimizers` ディレクトリ[^1]を切ってモジュールにまとめています。`task.py` を見るとわかりますが、こんな感じで利用します。

```python

from optimizers import io
from optimizers.optimizer import ProductionPlanner

...

planner = ProductionPlanner(problem)
solution = planner.solve()
```

このモジュールには**絶対に最適化と関係ないコードは含めない**ようにします。
たとえば、API サーバーのコードを追加してその中で最適化の処理をしたい場合は次のように実装します。

- 新たに `server` ディレクトリを切ってモジュール化する
- その中で `optimizers` モジュールを呼び出す

実務では最適化モジュールだけは最適化の専門家に実装を依頼するような形で分担することが多いです。

[^1]: ディレクトリ名は気分で変わります

## クラス設計

サンプルコードでは `optimizers/optimizer.py` に `ProductionPlanner` クラスを定義しており、これがソルバーを呼び出す処理を実行します。

ソルバーを呼び出す時は大雑把に

1. 入力データの前処理
2. 変数の追加
3. 制約の追加
4. 目的関数の定義
5. ソルバーの呼び出し

という流れになります[^2]。

[^2]: 制約の追加と目的関数の定義は順番が逆転しても大丈夫ですが、変数の追加は先にやる必要があります

### コンストラクタ

コンストラクタの実装はこんな感じです。入力データの前処理と変数の追加をここでやっています。

```python
def __init__(self, problem: Problem) -> None:
    self.problem = problem

    self.model = pyscipopt.Model("Production Planning")

    self.n_products = len(problem.products)
    self.n_materials = len(problem.materials)

    self._product_id_to_idx = {product.id: i for i, product in enumerate(problem.products)}
    self._material_id_to_idx = {material.id: j for j, material in enumerate(problem.materials)}

    # Product i requires requirements[i][j] units of material j.
    self.requirements = self._preprocess_requirements()

    # Create variables.
    self.x = self._variables_x()
```

[scikit-learn なんかだと入力データはコンストラクタではなくその後の `fit` メソッドで渡す](https://scikit-learn.org/stable/modules/linear_model.html)ので、最適化でも `solve` メソッドを呼び出す時に渡せば良いのではと思うかもしれません。ですが、最適化の場合は入力データをもとに前処理をして必要な値をフィールドにセットしておきたいことが多く、個人的にはここで渡してしまった方が見通しが良いかなと思っています。

たとえば、入力データをもとに ID とインデックスの対応関係を保持する `_product_id_to_idx` dict を作っています。

変数の追加もここでやっているのは諸説ありですが、埋められるフィールドはコンストラクタで埋めちゃった方が良い派なので、フィールドとして持たせるなら[^3]ここでやってしまうのが一番わかりやすいかなと最近は思っています。

変数は複数種類用意するケースもあるので `_variables_xxx` のようにわかりやすい名前を付けて切り出しています。

```python
def _variables_x(self) -> np.ndarray:
    x = np.empty(self.n_products, dtype=object)

    for i in range(self.n_products):
        x[i] = self.model.addVar(vtype="I", name=f"x_{i}")

    return x
```

[^3]: だいたいのモデリングツールは変数追加時に名前をつけるので、わざわざフィールドとして持たせずに `self.model` 経由で名前でアクセスすれば良い説はあります

### solve メソッド

`solve` メソッドでは制約の追加・目的関数の定義・ソルバーの呼び出しをします。

```python
def solve(self) -> Solution:
    # Add constraints.
    self._constraints_stock()

    # Define objective.
    objective = self._objective_gain()
    self.model.setObjective(objective, "maximize")

    # Solve MIP.
    self.model.optimize()

    return self.get_solution()
```

制約の追加も制約の種類ごとに `_constraints_xxx` のような名前で切り出して、制約の追加や削除を手軽に行えるようにしています。ちなみに `xxx` 部分の名前は結構大事で、制約を種類ごとに分けてちゃんと名前を付けておくことで「xxx 制約の話なんだけどさぁ」みたいな感じにコミュニケーションがスムーズになります。

目的関数も同様で、複数項の足し合わせになる場合は項ごとに `_objective_xxx` みたいな感じで切り出しておくと良いでしょう。

## 結果の可視化

サンプルコードでは結果の可視化として `optimizers/visualizer.py` にこんな関数を用意しました。

```python
import pandas as pd

from optimizers.entities import Solution


def print_solution(solution: Solution) -> None:
    """Print the solution."""
    data = {
        "product_id": solution.product_ids,
        "amount": solution.amounts,
    }
    df = pd.DataFrame(data)

    print("===== SOLUTION =====")
    print(df)
```

ソルバーを呼び出す `ProductionPlanner` にメソッドとして持たせるような実装がありがちですが、個人的には完全に切り離したい派閥です[^4]。

実務でやっていると想定外の結果が出てしまった時の調査依頼がちょくちょく発生するのですが、そういうケースでは入力と結果のログだけもらって作業することになります。なので、手元でソルバーを回さなくても入出力だけ渡せば可視化できるように実装しておいた方が何かと便利かと思います。

[^4]: というか、メソッドとして持たせる実装をして最近後悔しました

# まとめ

書いてみたらイマイチまとまりのない文章になってしまったので若干もやもやしているのですが、詳しいことはサンプルコードを見て頂けると。

今後自分でもテンプレートとして利用するつもりで用意したので、たぶん定期的にメンテすると思います。まだ手探りでやっているので、ある日見たらごっそり構成が変わっているとかあるかもしれませんが、その時は察してください。
