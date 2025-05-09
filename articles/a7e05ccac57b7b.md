---
title: "SMO の working set の選び方"
emoji: "🖖"
type: "tech"
topics:
  - "optimization"
  - "machinllearning"
published: true
published_at: "2020-12-21 14:26"
---

この記事は[数理最適化 Advent Calendar 2020](https://qiita.com/advent-calendar/2020/mo) の 21 日目の記事です。

## 何の話？

SVM の主要な解法のひとつである Sequential Minimal Optimization (SMO) を昔実装して最適化の手法としてすごく勉強になったので、簡単に紹介したいと思います。
全部を説明すると長くなるので、今回は working set selection と呼ばれる部分について説明をします。

この記事の解説は SVM の有名な実装である LIBSVM をベースにしています。
LIBSVM の実装は scikit-learn の裏でも使われているので、一度はお世話になった方が多いでしょう。

基本的には LIBSVM の[ドキュメント](https://www.csie.ntu.edu.tw/~cjlin/papers/libsvm.pdf)や開発者の論文 [Working Set Selection Using Second Order Information for Training Support Vector Machines](https://www.csie.ntu.edu.tw/~cjlin/papers/quadworkset.pdf) の Working Set Selection 1 (WSS 1)[^1] を解説する形になります。

[^1]: この論文で提案されている新手法は WSS 2 の方で WSS 1 は既存手法なのですが、アイデアがわかりやすい WSS 1 の方を解説しています。LIBSVM でも v2.7 までは WSS 1 が採用されていました

## 定式化

ここではサンプル $x_i \in \mathbb{R}^n$ が与えられたときに、そのラベル $y_i \in \{ -1, 1 \}$ を予測する分類問題を扱っています。

### 解きたい問題

SVM の主問題は正則化項と損失関数のトレード・オフを調整するハイパーパラメータ $C$ を使って次のように定式化されています:

$$
\begin{array}{cll}
  \displaystyle \min_{w, b, \xi} & \displaystyle \frac{1}{2} \| w \|^2 + C \sum_i \xi_i & \\\\
  \rm s.t. & y_i (w^\top x_i + b) >= 1 - \xi_i & i = 0, \dots , l \\
  & \xi_i \geq 0 & i = 0, \dots , l
\end{array}
$$

この主問題に対してラグランジュ双対問題を取ると次のようになります:

$$
\begin{array}{cll}
  \displaystyle \min_\alpha & \displaystyle f(\alpha) = \frac{1}{2} \alpha^\top Q \alpha - e^\top \alpha & \\\\
  \rm s.t. & y^\top \alpha = 0 & \\
  & 0 \leq \alpha_i \leq C & i = 0, \dots , l
\end{array}
$$

SMO はこの双対問題を効率良く解くための手法です。

## SMO の基本アイデア

SMO の基本アイデアは次のようなシンプルなものです:

1. 2 つの添字 $i, j$ を選択
2. $\alpha_i, \alpha_j$ だけを変数として他を固定した 2 変数の問題を解く
3. Step 1, 2 を繰り返して最適解に到達したら (or 既定の反復回数を超えたら) 終了

一部の変数以外を固定した簡単な部分問題を繰り返し解くというのは最適化でよくあるやり方で、有名なものだと 1 変数の問題を繰り返し解く coordinate descent という手法があります。

では何故 SVM ではその coordinate descent が使われていないのかというと、双対問題にある $y^\top \alpha = 0$ という制約のせいで 1 変数だけを動かそうとすると制約を破ってしまうのです。そのため、2 変数ずつ動かして少しずつ解を改善していこうという戦略になっています。

## もうちょっと細かい説明

上記のアイデアでもう少しちゃんと詰めなければいけなさそうな

1. 2 つの添字はどうやって選ぶのか？
   - 特にこの部分を Working Set Selection (WSS) と呼んでいます
3. 最適解であることはどうやって判定するのか

あたりを見ていきましょう。

### 最適性の判定

順番が前後しますが、これは添字の選び方にも影響する部分なので先に説明しておきましょう。

SVM は [Karush-Kuhn-Tucker (KKT) 条件](https://ja.wikipedia.org/wiki/%E3%82%AB%E3%83%AB%E3%83%BC%E3%82%B7%E3%83%A5%E3%83%BB%E3%82%AF%E3%83%BC%E3%83%B3%E3%83%BB%E3%82%BF%E3%83%83%E3%82%AB%E3%83%BC%E6%9D%A1%E4%BB%B6)が充たされていれば最適解であることが保証されるので、これを充たす解を見つけることが目標となります。

導出は省略しますが SVM の KKT 条件を書き下すと次のようになります[^2]:

$$
\begin{array}{cc}
  - y_i \nabla f(\alpha)_i \leq b & \forall i \in I_{\rm up}(\alpha), \\
  - y_i \nabla f(\alpha)_i \geq b & \forall i \in I_{\rm low}(\alpha),
\end{array}
$$

where

$$
I_{\rm up}(\alpha) := \{ t \mid \alpha_t < C, y_t = 1 {\rm \ or \ } \alpha_t > 0, y_t = -1 \}, \\
I_{\rm low}(\alpha) := \{ t \mid \alpha_t < C, y_t = -1 {\rm \ or \ } \alpha_t > 0, y_t = 1 \}.
$$

[^2]: この表記は[Working Set Selection Using Second Order Information for Training Support Vector Machines](https://www.csie.ntu.edu.tw/~cjlin/papers/quadworkset.pdf) の 2 節の表記から取ってきたものなので、一般的に使われる式から実装に都合が良いように少し等価な変形を行ったものです。SVM の KKT 条件についてもう少し一般的な表記や導出を知りたい場合は、個人的には[カーネル法入門](https://www.amazon.co.jp/dp/4254128088/)の 4 章あたりを読むのもおすすめです。

細かい導出は論文を参照してもらうとして、ここでは上記の式を真面目に追う必要はありません。

注目すべきポイントは**サンプル (添字) 1 個につき条件式が 1 本ある**というところです。これを踏まえて次の添字の選び方を見ていきましょう。

### 2 つの添字の選び方

最終的なゴールは KKT 条件が全て充たされていることなので、基本となる戦略は**KKT 条件の違反に対して寄与が大きい変数を選ぶ**次のようなものになります:

1. $I_{\rm up}(\alpha)$ の中から KKT 条件の $- y_i \nabla f(\alpha)_i \leq b$ を一番大きく違反する添字 (変数) を選ぶ
2. $I_{\rm low}(\alpha)$ の中から KKT 条件の $- y_i \nabla f(\alpha)_i \geq b$ を一番大きく違反する添字 (変数) を選ぶ
3. 選んだ 2 変数以外を固定した最適化問題を解く

#### ちょっと細かい話

添字を $I_{\rm low}$ と $I_{\rm up}$ の 2 種類に分けたことの意味ですが、これは同時に動かすことができる変数を表しています。
双対問題の $y^\top \alpha = 0$ という制約を充たしながら 2 変数を動かすためには、一方を $y^\top \alpha$ が減る方向に、もう一方を増える方向に動かさなければならないので、同時に選べる 2 変数には少し制限があるというわけです。

## まとめ

~~自分で実装したのが昔過ぎてあとは解説できるほど覚えていない~~あまりに記事が長くなると困るので残りは省略しますが、SMO の概要を知るには working set selection まで理解すれば十分かなと思います

実際に自力で実装して速度を出そうとした場合は、他にも

- 2 変数の最適化問題を効率良く解く手順
- 解が更新されるのに合わせて $\nabla f(\alpha)$ を効率良く更新しながら保持する方法
- 行列 $Q$ の値を全て計算するのは $l^2$ サイズで効率が悪いので、必要になったときにだけ計算して一定数キャッシュしておく実装テクニック

など色々あるので、そのうち続きを書きたいなと思っていますが、興味がある人 (いるのか？) は LIBSVM のドキュメントを読み込んでみてください。
基本的には冒頭で紹介したドキュメントを読み込めば、自分で実装してそれなりの速度を出せるくらいの情報は書かれています。

また、「SMO の基本アイデア」で coordinate descent に少し言及しましたが、実は SVM の定式化からバイアス項 $b$ を除くと、双対問題の制約 $y^\top \alpha = 0$ が消えるので 1 変数ずつ動かすことができるようになります。それに加えて非線形カーネルではなく線形カーネルのみという条件のもとで高速化したものが [LIBLINEAR](https://www.csie.ntu.edu.tw/~cjlin/liblinear/) で、こちらも有名な実装なので興味がある方は調べてみてください。
