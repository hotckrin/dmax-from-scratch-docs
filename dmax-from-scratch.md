## ゼロから作る自動ダメージ最大化シミュレータ

この記事では自動ダメージ最大化シミュレータを作る方法をゼロから解説していきます。
まずは自動ダメージ最大化シミュレータの仕組みを解説し、次に具体的な実装例をハンズオン形式で実践していきます。

文章の構成としては、最適化シミュレーターの解説→実装という順序になっていますが、座学が嫌な人は実装編まで飛ばしても構いません。
そして実装してみて疑問が出てきたときに解説に戻ってくるという読み方でも問題ありません。

ダメージ最大化シミュレータをつくる人が界隈に増えてくれることを願ってこの記事を書きます。
個人的な事情としてだんだんとゲームに使える時間が減ってきておりいずれ失踪すると思っています。
失踪するまえに誰かにダメージ最大化シミュの自作文化を誰かに引き継いでほしいです。


## 自動ダメージ最大化シミュレータとはなにか？

自動ダメージ最大化シミュレータは、ゲーム中でダメージが最大になる構成を自動で見つけてくれるシミュレータです。

モンスターハンターというゲームには、さまざまな装備があり、装備の組み合わせが変わるとモンスターに与えるダメージが変化します。
この与ダメージが最大になる装備の組み合わせを自動で考えてくれるのが自動ダメージ最大化シミュレータです。
自動ダメージ最大化シミュレータでは長いので以降は ダメ最大化シミュ と表記します。

私は過去に2つの自動ダメージ最大化シミュレータを公開してきました。
Monster Hunter Wilds 版のダメ最大化シミュは Web サイトで公開しています。
ダメ最大化シミュの動作イメージを確認したい場合はサイトにアクセスして、右下の最適化ボタンを押してみてください。


## 自動ダメージ最大化シミュレータの仕組み

自動ダメージ最大化シミュレータの仕組みを一言で言うと、「最適化ソルバーに解かせよう！」というものになります。

世の中には最適化ソルバーという素晴らしいツールがあり、これがモンハンの最大ダメージ構成を考える上でのさまざまな難しい問題まとめて解決してくれます。
最適化ソルバーの内部の仕組みは難しいですが、利用するだけであればそれほど難しくはありません。
このドキュメントでは最適化ソルバーの性質と利用方法を見ていきます。

最適化ソルバーには以下のような嬉しい性質があります。

> 最適化ソルバーに対して「装備の条件 と ダメージ計算式」を入力すると、「ダメージが最大になる装備の組み合わせ」が出力される

より一般的に言うと

> 最適化ソルバーに対して「制約条件 と 目的関数」を入力すると、「その制約条件の範囲内で、目的関数が最大 or 最小になるような組み合わせ」が出力される

モンハンの場合、「制約条件」が「装備の条件」に相当し、「目的関数」が「ダメージ計算式」に相当します。

ここで言う装備の条件とは、例えば「頭防具は1つしか装備できない」とか「装飾品はスロットが合うものしか装備できない」のような条件です。
ゲームをプレイする上で暗黙のうちに了解しているさまざまな装備の制約を入力する必要があります。
また、ここで言う「ダメージ計算式」とは例えば「攻撃力〇〇の弓を装備して△△モーションをモンスターに当てると✕✕のダメージになる」のようなダメージを決定する計算式のことです。

最適化ソルバーの嬉しい点は、最適化ソルバーの内部の詳細な仕組みを知る必要がないことです。
つまり、この記事で作っていく最適化シミュ は「難しい部分は最適化ソルバーに丸投げ！」というアプローチで、最大ダメージ装備を探すシミュレータということです。

ただし、最適化ソルバーは一定のフォーマットで入力しなければ最適化問題を解くことができません。
そのため、ダメ最大化シミュを実装するうえでのチャレンジは「装備の条件をどのようにして最適化ソルバーが解ける形式に落とし込むか？」という点になります。
「装備の条件」と「ダメージ計算式」を最適化ソルバーが理解できる形式で入力することができるようになれば、ダメ最大化シミュの完成です。

ダメージ計算式については、すでに計算式がわかっているのであれば最適化ソルバーに入力するのは難しくありません。計算式をそのまま記述するだけです。
(ダメージ計算式において難しいのは、いかにしてダメージ計算式を推測するか？という点です)

問題は、装備の条件をどのようにして最適化ソルバーが解ける形式に落とし込むか、です。以降ではこの方法を見ていきます。


## 装備の条件を数式に落とし込む

ここまでの説明で、モンハンの装備条件を最適化ソルバーが解ける形式に落とし込む必要があることを見てきました。
最適化ソルバーが解ける形式とは数式のことです。
つまり、装備条件を数式として表現することができれば良いことになります。

実際のゲームデータを利用すると複雑になってしまうため、最初はサンプルデータを用意して説明していきます。
(この記事の後半では、実際のゲームデータを利用して最適化シミュを実装していきます。)


```
武器A: 攻撃力 20
武器B: 攻撃力 10
ゲーム内の暗黙の条件: 武器は1つしか利用できない
ダメージ計算式: (ダメージ) = (武器の攻撃力)
```

非常にシンプルなサンプルデータを用意してみました。

ダメージ計算式は武器の攻撃力をそのまま参照するため、攻撃力が高い武器Aのほうが高い与ダメージを得ることができます。<br>
つまり、この場合の最大ダメージ構成は「武器Aを1つ装備する」になります。

ここまでシンプルであれば、もはや最適化ソルバーなどという大袈裟なものを持ち出すまでもなく、直感的に最も強い構成がわかると思います。
しかし、ここでは装備条件を制約式の落とし込む練習のため、あえてこの問題を最適化ソルバーが解ける形式で表現してみましょう。

重要な条件として、武器は1つしか利用できないという条件があるので、これを制約式として表現してみます。

まずダメージ計算式は以下のようになります。

```
(ダメージ) = (武器Aの攻撃力) * (武器Aを利用する) + (武器Bの攻撃力) * (武器Bを利用する)
```

これを数式っぽく表現すると以下のようになります。

```
dmg = A_attack * A_use + B_attack * B_use
```

A_attack は武器Aの攻撃力 20 であり、変化しません。<br>
B_attack は武器Bの攻撃力 10 であり、変化しません。<br>
つまりダメージ計算式は以下のように書けます。

```
dmg = 20 * A_use + 10 * B_use
```

A_use は武器Aを利用するかどうかを表す変数であり、0 または 1 のどちらかの値を取ります。<br>
B_use は武器Bを利用するかどうかを表す変数であり、0 または 1 のどちらかの値を取ります。

ここで、武器は1つしか利用できないという条件があるので、 A_use + B_use は2以上になることはありません。<br>
つまり、A_use + B_use は1以下という制約条件が必要になります。

```
A_use + B_use <= 1
```

最終的に、以下のような装備の制約条件と目的関数が導かれました。
あとは、こちらを最適化ソルバーに入力すると、最大化問題の解として (A_use, B_use) = (1, 0) の組み合わせが出力され、「武器Aを使用したほうが与ダメージが高い」ということがわかります。

```
制約条件:
A_use + B_use <= 1
0 <= A_use <= 1 (ただし A_use は整数)
0 <= B_use <= 1 (ただし B_use は整数)

目的関数:
20 * A_use + 10 * B_use
```

さてここまでの流れをみて、「直感的にわかることを冗長で複雑に書いてる」と感じたかもしれません。私もそう思います。<br>
しかし、このような条件を数式に落とし込む手順を発展させていくと、問題が複雑になり人力ではとても解けないような規模の問題になった場合でも最適化ソルバーを利用して問題を解けるようになります。
なので今しばらくお付き合いください。

さて、上記の問題については完成した「制約条件と目的関数」を最適化ソルバーに投げて終了なのですが、最適化ソルバーの性質を理解を深めるためもう少し深堀りしましょう。

次は、上記の制約条件と目的関数をグラフで表現してみましょう。

まず以下の制約条件をグラフ上に図示します。
横軸が A_use の値を表し、縦軸が B_use の値を表しており、制約式の範囲は図のような正方形の領域になります。

```
0 <= A_use <= 1 (ただし A_use は整数)
0 <= B_use <= 1 (ただし B_use は整数)
```

![](images/figure1.png)

ここで、さらに以下の制約式を追加すると、制約式の範囲は図のような三角形の領域になります。

```
A_use + B_use <= 1
```

![](images/figure2.png)

最後に目的関数 `20 * A_use + 10 * B_use` はグラフ上でどのように表現できるかを考えます。
まずは、 `20 * A_use + 10 * B_use = 30` になる点をリストアップしてみます。
(A_use, B_use ) = (0, 3), (1, 1), (2, -1) となります。
これを線でつなぐと、 `20 * A_use + 10 * B_use = 30` を満たす線を引くことができます。

![](images/figure3.png)


次に `20 * A_use + 10 * B_use = 40` となる等高線を引くと以下のようになります。

![](images/figure4.png)

ここで、 20 * A_use + 10 * B_use はダメージ計算式なので、大きければ大きいほどプレイヤーにとって嬉しいです。
なので、等高線をできるだけ右上にズラしていきたいです。
しかし、A_use と B_use には制約条件があり、グラフ上で図示された点しか取ることができません。
よって、A_use と B_use のグラフ上の点のうち、最も等高線が右上になる値の組み合わせ (A_use, B_use) = (1, 0) が、ダメージ計算式を最大化する値の組み合わせということになります。
また、`20 * A_use + 10 * B_use = 20` より最適解によって得られる最大ダメージは 20 であることも分かります。

![](images/figure5.png)

つまり、ダメージ計算式の等高線を引き、等高線が高い方から低い方へ動かしていき、制約条件の表す範囲に重なる等高線まで動かして止めれば最大化問題が解けたことになります。

最適化ソルバーは、内部的にこのようなアプローチで最適化問題を解いています。<br>
実際にはもっと別の高度な仕組みで解いていますが、ダメ最大化シミュを実装するうえでは大雑把なアプローチを理解しておけば十分です。

## 装備の条件を数式に落とし込む 応用編

次は、問題設定をもう少しモンハンの装備条件に近づけてみましょう。

```
武器C: 攻撃力 30, 属性値 20
防具X: 攻撃強化Ⅱ, 属性強化Ⅰ, 体術 x2, 重量1kg
防具Y: 攻撃強化Ⅰ, 属性強化Ⅱ, 体術,    重量2kg

スキルの効果:
攻撃強化Ⅰ: 攻撃力を +10 (装備全体で「攻撃強化Ⅰ」がN個ついている場合、攻撃力の加算値は +10N とする。攻撃強化Ⅱ,属性強化Ⅰ,Ⅱについても同様)
攻撃強化Ⅱ: 攻撃力を +30
属性強化Ⅰ: 属性値を +20
属性強化Ⅱ: 属性値を +30
体術: ダメージ計算には寄与しない

装備の条件: 
(条件1) 防具は0個以上のいくつでも装備できる
(条件2) 体術は最低5個欲しい
(条件3) 防具重量の合計の上限は4kg (4kg 以上の防具は重くて装備できない)

ダメージ計算式:
(ダメージ) = {(武器の攻撃力) + (攻撃強化スキルによる攻撃力加算)}
           + {(武器の属性値) + (属性強化スキルによる属性値加算)}
```

(モンハンには重量制限はなく装備を無限に装備することもできませんが、ここでは条件を式に落とし込む練習のために都合の良い条件を設定しています。)

さて、こうなると直感で解くのは難しくなってきたのではないでしょうか？<br>
それでは先ほどと同じように、上記の条件を制約式として表現し、グラフという簡易的な最適化ソルバーで解いてみましょう。

防具Xを装備する数を X_use, 防具Yを装備する数を Y_use とします。

まず (条件1) を考えます。<br>

> (条件1) 防具は0個以上のいくつでも装備できる

という制約条件を数式で表すと以下のようになります。

```
0 <= X_use (ただし X_use は整数)
0 <= Y_use (ただし Y_use は整数)
```

次に (条件2) を考えます。<br>
> (条件2) 体術は最低5個欲しい

防具Xには体術が2個付いているので、防具Xを X_use 個装備したときの体術の合計は 2 * X_use 個になります。
よって (条件2) を数式で表すと以下のようになります。

```
2 * X_use + 1 * Y_use => 5
```

同様に考えて、 (条件3) 

> (条件3) 防具重量の合計の上限は4kg 

を数式で表すと以下のようになります。
 
```
1 * X_use + 2 * Y_use <= 4
```

次に、以下のダメージ計算式を考えます。

```
ダメージ計算式:
(ダメージ) = {(武器の攻撃力) + (攻撃強化スキルによる攻撃力加算)}
           + {(武器の属性値) + (属性強化スキルによる属性値加算)}

スキルの効果:
攻撃強化Ⅰ: 攻撃力を +10
攻撃強化Ⅱ: 攻撃力を +30
属性強化Ⅰ: 属性値を +20
属性強化Ⅱ: 属性値を +30
```

各スキルの効果を考慮すると、ダメージ計算式は変数 X_use, Y_use を利用して以下のように書き直せます。

```
(ダメージ) = {(武器の攻撃力) + 10 * (攻撃強化Ⅰの個数) + 30 * (攻撃強化Ⅱの個数)}
           + {(武器の属性値) + 20 * (属性強化Ⅰの個数) + 30 * (属性強化Ⅱの個数)}
```

防具Xには攻撃強化Ⅱがついているので、X_use 個装備すると 30 * X_use の攻撃力が加算されます。
防具Yについても同様に考えるとダメージ計算式は以下のように表現できます。

```
dmg = {(30) + 10 * Y_use + 30 * X_use}
    + {(20) + 20 * X_use + 30 * Y_use}
    = 50 + 50 * X_use + 40 * Y_use    
```

ここまでで制約条件と目的関数を変数 X_use, Y_use を利用して表現できました。
制約条件と目的関数をまとめると以下のようになりました。

```
制約条件:
0 <= X_use (ただし X_use は整数)
0 <= Y_use (ただし Y_use は整数)
2 * X_use + 1 * Y_use => 5
1 * X_use + 2 * Y_use <= 4

目的関数:
50 + 50 * X_use + 40 * Y_use
```

それでは次にグラフに図示し、最適解を求めてみましょう。

まず、制約条件をグラフに図示すると以下のようになります。<br>
青い三角形の領域が制約条件を満たす解の範囲です。
さらに、整数の制約も考慮すると解の候補は `(X_use, Y_use) = (2,1), (4,0)` のいずれかに絞れます。

![](images/figure6.png)

次に先ほどと同様に目的関数の等高線を考慮すると、`50 + 50 * X_use + 40 * Y_use = 250` の場合に制約条件を満たし、かつ目的関数が最大となることがわかります。
よって、制約条件を満たす解のうち最大ダメージを実現する解は (4,0) であり、その最大ダメージは 250 であることがわかりました。

![](images/figure7.png)

このセクションでは、問題を制約条件と目的関数に落とし込み、さらに作図により最適解を求めました。
後のセクションでは、この問題をプログラムとして表現し、最適化ソルバーに解かせるところまで実施します。

実際の最適化シミュを実装するうえでは、「作図により最適解を求める」部分は最適化ソルバーの仕事になるため、この部分は不要になります。
しかし、作図部分の動作原理は最適化シミュレータの最も簡単な動作を模倣したものなので、どのような問題であれば最適化ソルバーは高速で解くことができるのかについての示唆を得ることができます。
次のセクションでは、最適化ソルバーが解ける問題や解きやすい問題の性質について見ていきます。

## どうやって式に落とし込めばよいか？

(書きかけ. 数学アレルギーがない人向け)

ここまでは具体例を出して説明してきましたが、そもそもダメージ最大化問題を解くというのはどのような取り組みでしょうか？
実際の問題に対応できるようにここで言語化しておきましょう。

まず、モンハンにおけるダメージ最大化を最適化問題として見たときの解の候補とはなんでしょうか？
それは、データ上存在するすべての装備をそれぞれ何個使うかを表現したベクトルです。
このベクトルが、制約条件を満たしており、かつそのベクトルを目的関数に適用したときのダメージの値が、他のどの解候補 (装備ベクトルの組み合わせ) よりも高い場合、そのベクトルはダメージ最大化問題における最適解である、と言えます。

前のセクションでは、問題における装備条件とダメージ計算式を、それぞれ制約条件と目的関数という形で、最適化ソルバーが解ける形式に落とし込んできました。
この定式化は以下のルールに従う必要があります。

1. 変数と定数からなる式によって制約条件を表現すること
2. 制約条件が最適解を持つこと

定式化の方針がわからない場合は以下のような問いを立ててください。

> (1) 最適解を求める上で不変の値はなにか？ (定数はなにか)<br>
> (2) 最適解を求める上で変化する値はなにか？ (変数はなにか)

例えば、先程の例で登場した「防具Dには体術Lv2がついている」という条件の場合、
「防具Bを何個装備するか？」は最大ダメージ構成を探すときに変化する値、すなわち変数です。ここでは変数 d_use と定義しました。

「防具Bに体術スキルLv2がついているというデータ」は最大ダメージ構成を探すうえで変化しない値、すなわち定数です。

結果として、変数 d_use, e_use を利用して以下のような制約条件式に落とし込むことができました。

```
5 <= 2 * d_use + 1 * e_use
```

## シミュを高速化するコツ

最適化ソルバーの性質として「変数同士の掛け算が少ないほど問題を高速に解ける」という性質があります。<br>
例えば、変数 x と変数 y がある場合、

> - `xy` という式よりも、 `3x + 2y` のような式のほうが高速に解ける
> - `3x^2 + xy` という式よりも、 `3x + 3x + 2x + y` のような式のほうが高速に解ける

ということです。変数の数自体はどちらも同じですが、後者のほうが変数同士の掛け算が少なく、より高速に解くことができます。

変数同士の掛け算を減らした究極系、つまり変数同士の積がない式は一次式と呼ばれています。またこのような式の性質を線形性と呼ばれることがあります。
(厳密には線形性ではありませんが、工学などの応用系の分野では一次式を雑に「線形である」と呼ぶことがあります。)

つまり、もしあなたがモンハンのダメージ最大化問題をこの一次式の形に落とし込むことができたなら、より高速なシミュが作れるかもしれないということです！
(私は無理でした。問題の性質的に無理ではないかと思っています。)
ただし、変数同士の積を減らすために補助変数を導入すると、補助変数と制約式が増えるため必ずしも高速化するとは限らない点に注意が必要です。

とはいえ、立式する際の方向性として「変数同士の積を減らす」という意識を持っておくことは重要です。
実際のモンハンのデータを式に落とし込む際に、変数同士の掛け算を減らせないか？という視点を意識すると、より高速な最適化シミュを作れる可能性が上がります。

逆に、変数同士の掛け算が多すぎると計算が重くなってしまい、実用に耐えないほど遅いダメ最大化シミュになってしまう可能性が高くなります。


```note
実際の例として、サンブレイクの最適化シミュはワイルズの最適化シミュよりも最適解が見つかるのに時間がかかります。
サンブレイクもワイルズも仕組み的には同じ実装をしており、どちらも同じ最適化ソルバーを利用しています。
しかしサンブレイクのほうが攻撃力と属性値に対して乗算で効果があるスキルが多いため、変数同士の掛け算が多くなり結果として処理が重くなっています。
また防具のデータ数が多い点もサンブレイクのほうが処理が重い原因の1つです。
```



## 変数同士の掛け算ができてしまってもOK

線形計画法に慣れていると「最適化ソルバーは線形でない問題は解けない」という思い込みがあるかもしれません。
(私は以前は漠然とそう思っていました)

実際には、問題が線形になっていなくても (つまり、非線形であっても) 解けるソルバーはあります。
本ドキュメントの最適化シミュではSCIPというソルバーを利用します。
SCIPソルバーは無料で利用できる (Apache 2.0 License) ソルバーでありながら、MINLPという形式の問題を解くことができます。
MINLP (Mixed Integer Nonlinear Programming) は混合整数非線形計画問題のことで、制約条件式や目的関数が非線形であっても解くことができますし、変数に整数の制約が追加されても (つまり混合整数問題であっても) 解くことができるちおう優れものです。

つまり何が言いたいかというと「式に落とし込む際に変数同士の掛け算ができてしまっても、それを解けるソルバーはあるので諦めず実装してみましょう」ということです。<br>
（ただし、問題の規模によっては実用に耐えないほど遅くなる可能性があります）

こちらの詳細については数学的なトピックになります。
詳細を知らなくてもダメ最大化シミュは実装できるので、この記事では触れません。

興味のある方は以下のようなキーワードで調べてみてください。
- 1次式
- 線形計画問題
- 非線形計画問題
- 非線形混合整数計画問題


## 最適化シミュを自作する

ここまでの説明で、モンハンに模した装備条件やダメージ計算式を、最適化ソルバーが解ける形式に落とし込む方法を見てきました。
前のセクションではグラフを利用して解きましたが、次は実際にプログラムを書いて最適化ソルバーに解かせてみましょう。

最適化問題の設定としては、先程の制約条件と目的関数をそのまま利用します。

```
制約条件:
0 <= X_use (ただし X_use は整数)
0 <= Y_use (ただし Y_use は整数)
2 * X_use + 1 * Y_use => 5
1 * X_use + 2 * Y_use <= 4

目的関数:
50 + 50 * X_use + 40 * Y_use    
```

ここから、制約条件と目的関数をプログラムで表現していきます。
制約条件と目的関数をプログラムで表現するためにはモデリングツールを利用します。

モデリングツールを利用して制約条件と目的関数を表現すると、最適化問題を1つのモデルとして出力することができます。
あとは、このモデルを最適化ソルバーに入力すればソルバーが自動的に最適化問題を解いてくれます。

> 1. 問題を制約条件と目的関数に変換
> 2. 制約条件と目的関数をモデリングツールで表現
> 3. モデリングツールが最適化問題のモデルを出力
> 4. 最適化ソルバーに入力
> 5. 最適化ソルバーが最適解を発見


という流れになります。


まずは、プログラミングを実行する環境を構築しましょう。

このドキュメントはプログラミングをしたことがない人が、最適化シミュを自作できるようなドキュメントを目指して書いています。
なので、まずは環境構築というのがいかに大変かということを説明しておきます。

プログラミングにおける最大の難関は環境を構築することと言っても過言ではありません。
実際それくらい環境構築は大変です。
なぜ、環境構築が大変なのかというと、プログラミングでものを作る際には多くの外部ツールに依存して作ることになるからです。
実際に自分で作る部分が1%で残り99%は他のコードを流用するようなイメージです。
環境構築は99%の他のコードの依存関係を解決する仕事なので、大変になりがちです。

ということで、がんばって環境構築をしていきましょう。

環境は以下になります。

| 項目                                | 内容                       | 必須 |
|-------------------------------------|----------------------------| ---- |
| OS                                  | ubuntu 22.04.1 LTS         |      |
| 実行環境                            | Python 3.8.16              | ◯   |
| モデリングツールのライブラリ        | pyomo 6.4.4                | ◯   |
| Python version とライブラリ管理     | uv                         |      |
| 最適化ソルバー                      | SCIP version 8.0.3         | ◯   |

OS を ubuntu としているのは主に最適化ソルバー SCIP の実行環境のためです。<br>
SCIP はC/C++で実装されており、バイナリ形式のファイルを実行する形になるため、OSを揃えておいたほうがトラブルは少ないと思います。
一応 SCIP のパッケージとしては Linux/Windows/MacOS/Raspberry の環境のコンパイル済みパッケージが提供されているのでどの環境でも動かせると思いますが、この記事では試していません。<br>
(ソースコードからビルドする際には Linux を推奨します。昔 Windows 上でソースからビルドしましたがかなり大変でした。)<br>
Python はバージョンが揃っていればどのOSでも問題なく動くと思います。

この記事では Windosw11 環境で実行環境を構築していきます。
その他のOSの方は、上記の表の必須の項目を準備してください。(ChatGPT, Gemini, Claude などに聞けば教えてくれると思います。)


wip: 環境構築したときのメモ
claude との相談履歴: https://claude.ai/share/a050b41a-1a56-4d99-bee7-7fa6f8ec6938
(まっさらな ubuntu 環境を wsl 上に新規構築する手順が書いてある)

最終的な実行方法
```sh
# 対話シェル
$ /home/hoge/work/scipoptsuite-8.0.3/build_default/bin/scip
SCIP> read /home/hoge/work/dmax-from-scratch/dmax-practice/linear-problem.nl
SCIP> optimize
SCIP> display solution

# ワンライナー
$ /home/hoge/work/scipoptsuite-8.0.3/build_default/bin/scip -f linear-problem.nl

```

scip 環境構築
```sh
# コンパイル済み scip をインストールするためのスクリプトをダウンロードする
$ wget https://www.scipopt.org/download/release/SCIPOptSuite-8.0.3-Linux-ubuntu.sh

# ダウンロードしたスクリプトを実行して、コンパイル済み scip をインストールする
$ sh SCIPOptSuite-8.0.3-Linux-ubuntu.sh
Do you accept the license? [yn]:
y

Saying no will install in: "/home/dmax-scratch" [Yn]:
Y


# scip の依存関係インストール (libgsl23 は入らなかったので libgsl27 に変更)
$ sudo apt install -y gcc g++ gfortran liblapack3 libtbb2 libcliquer1 libopenblas-dev patchelf libgsl-dev libgsl27

# scip 動作確認

# 対話型で起動して実行
$ /home/dmax-scratch/SCIPOptSuite-8.0.3-Linux/bin/scip

# read コマンドで問題ファイルを読み込みます
# read <問題ファイルへのパス>
SCIP> read /home/dmax-scratch/dmax-practice-problem.nl

read problem </home/dmax-scratch/dmax-practice-problem.nl>
============

original problem has 3 variables (0 bin, 2 int, 0 impl, 1 cont) and 2 constraints

# optimize コマンドで読み込んだ問題を最適化します
SCIP> optimize

solution violates original bounds of variable <objconstant> [50,50] solution value <0>
all 1 solutions given by solution candidate storage are infeasible

feasible solution found by trivial heuristic after 0.0 seconds, objective value 5.000000e+01
presolving:
(round 1, fast)       1 del vars, 0 del conss, 0 add conss, 3 chg bounds, 0 chg sides, 0 chg coeffs, 0 upgd conss, 0 impls, 0 clqs
(round 2, fast)       1 del vars, 0 del conss, 0 add conss, 3 chg bounds, 1 chg sides, 1 chg coeffs, 0 upgd conss, 0 impls, 0 clqs
(round 3, fast)       1 del vars, 0 del conss, 0 add conss, 4 chg bounds, 1 chg sides, 1 chg coeffs, 0 upgd conss, 0 impls, 0 clqs
(round 4, fast)       2 del vars, 0 del conss, 0 add conss, 4 chg bounds, 1 chg sides, 1 chg coeffs, 0 upgd conss, 1 impls, 0 clqs
   (0.0s) running MILP presolver
   (0.0s) MILP presolver (2 rounds): 0 aggregations, 2 fixings, 0 bound changes
presolving (5 rounds: 5 fast, 1 medium, 1 exhaustive):
 4 deleted vars, 2 deleted constraints, 0 added constraints, 4 tightened bounds, 0 added holes, 1 changed sides, 1 changed coefficients
 1 implications, 0 cliques
transformed 1/2 original solutions to the transformed problem space
Presolving Time: 0.00

SCIP Status        : problem is solved [optimal solution found]
Solving Time (sec) : 0.01
Solving Nodes      : 0
Primal Bound       : +1.90000000000000e+02 (2 solutions)
Dual Bound         : +1.90000000000000e+02
Gap                : 0.00 %

# 最適解を表示します
SCIP> display solution

objective value:                                  190
Ax_use                                              2   (obj:50)
Ay_use                                              1   (obj:40)
objconstant                                        50   (obj:1)

# ワンライナーで実行
# 上記の対話型で実行した内容を一発で実行できます
$ /home/dmax-scratch/SCIPOptSuite-8.0.3-Linux/bin/scip -f /home/dmax-scratch/dmax-practice-problem.nl


# version が 8.0.3 か確認
$ /home/dmax-scratch/SCIPOptSuite-8.0.3-Linux/bin/scip --version
SCIP version 8.0.3 [precision: 8 byte] [memory: block] [mode: optimized] [LP solver: Soplex 6.0.3] [GitHash: 62fab8a2e3]
Copyright (C) 2002-2022 Konrad-Zuse-Zentrum fuer Informationstechnik Berlin (ZIB)

External libraries:
  Soplex 6.0.3         Linear Programming Solver developed at Zuse Institute Berlin (soplex.zib.de) [GitHash: f900e3d0]
  CppAD 20180000.0     Algorithmic Differentiation of C++ algorithms developed by B. Bell (github.com/coin-or/CppAD)
  ZLIB 1.2.11          General purpose compression library by J. Gailly and M. Adler (zlib.net)
  GMP 6.2.1            GNU Multiple Precision Arithmetic Library developed by T. Granlund (gmplib.org)
  ZIMPL 3.5.3          Zuse Institute Mathematical Programming Language developed by T. Koch (zimpl.zib.de)
  AMPL/MP 4e2d45c4     AMPL .nl file reader library (github.com/ampl/mp)
  PaPILO 2.1.2         parallel presolve for integer and linear optimization (github.com/scipopt/papilo) [GitHash: 2fe2543]
  bliss 0.77           Computing Graph Automorphism Groups by T. Junttila and P. Kaski (www.tcs.hut.fi/Software/bliss/)
  Ipopt 3.13.2         Interior Point Optimizer developed by A. Waechter et.al. (github.com/coin-or/Ipopt)

Compiler: gcc 9.4.0

Build options:
 ARCH=x86_64
 OSTYPE=Linux-4.19.0-21-amd64
 COMP=GNU 9.4.0
 BUILD=Release
 DEBUGSOL=OFF
 EXPRINT=cppad
 SYM=bliss
 GMP=ON
 IPOPT=ON
 WORHP=OFF
 LPS=spx
 LPSCHECK=OFF
 NOBLKBUFMEM=OFF
 NOBLKMEM=OFF
 NOBUFMEM=OFF
 THREADSAFE=ON
 READLINE=off
 SANITIZE_ADDRESS=OFF
 SANITIZE_MEMORY=OFF
 SANITIZE_UNDEFINED=OFF
 SANITIZE_THREAD=OFF
 SHARED=ON
 VERSION=8.0.3.0
 API_VERSION=104
 ZIMPL=ON
 ZLIB=ON

```

それでは pyomo によるモデルを出力するためのコードを書いていきましょう。

まずは最適化モデルを定義していくためのベースとなる Model クラスのインスタンスを作成します。

```py
from pyomo.environ import *

if __name__ == '__main__':
    # 線形計画問題を解くためのPyomoモデルを定義します
    
    # モデル定義
    mdl = ConcreteModel(name="dmax-practice", doc="dmax-practice: ゼロから作るモンハン最適化シミュレータ")
```

pyomo の Model クラスには以下の2種類があります。

| モデルの種類 | 用途 |
| ---- | ---- |
| ConcreteModel | モデル定義の際にパラメータの値が分かる場合に利用 |
| AbstractModel | モデル定義の際にパラメータの値が分からず、最適化を実行する段階で判明する場合に利用 |

最適化モデルにおける変数は最適化処理で求める値なのでモデルを定義する際にはわかりません。
変数以外のパラメータ (定数) についてはモデルを定義する際に分かる場合と、最適化を実行する段階になるまで分からない場合の2種類があります。
モデルを定義する際にパラメータが分かる場合には ConcreteModel を利用します。
最適化を実行する段階になるまでパラメータの値が分からない場合には AbstractModel を利用します。

今回のようなモンハンの最適化シミュレーターの場合、パラメータとは具体的な装備やスキル等のゲーム内データにあたります。
これらは、ゲーム内データやユーザの入力を元にモデルを定義する際に分かるものなので、今回は ConcreteModel を採用します。


Model クラスのインスタンス `mdl` を作成できたので、以降はこの `mdl` に変数やパラメータ、制約条件、目的関数を定義していきます。

制約条件を表現するためには変数とパラメータが必要です。今回のようなシンプルな問題ではパラメータは制約条件に直書きすればいいですが、変数は最適化処理で変化しうるので先に定義しておく必要があります。
なので、まずは変数を以下のように宣言しましょう。

```py
from pyomo.environ import *

if __name__ == '__main__':
    # 線形計画問題を解くためのPyomoモデルを定義します
    
    # モデル定義
    mdl = ConcreteModel(name="dmax-practice", doc="dmax-practice: ゼロから作るモンハン最適化シミュレータ")

    # 変数定義
    # X_use: 非負整数変数
    mdl.X_use = Var(within=NonNegativeIntegers, initialize=0)
    
    # Y_use: 非負整数変数  
    mdl.Y_use = Var(within=NonNegativeIntegers, initialize=0)
```

pyomo の変数は `Var` クラスで定義します。

`Var` クラスの主な引数は以下の通りです。

| 引数 | 説明 | 
|------|------|
| `domain` または `within` | 変数の取りうる値の範囲を指定 (実数、整数など) |
| `initialize` | 変数の初期値を指定 |

引数 `domain (within)` では変数の取り得る値の範囲を指定します。取りうる値の範囲は Set と呼び、 Set としては `Reals` (実数), `Integers` (整数), `NonNegativeIntegers` (非負整数), `Boolean` (真偽値) などを指定できます。
定義済みの Set の一覧については [Pyomo ドキュメント](https://pyomo.readthedocs.io/en/stable/explanation/modeling/math_programming/sets.html#predefined-virtual-sets) を参照してください。
`within` は `domain` のエイリアスなのでどちらでも良いです。`domain` を指定しなかった場合のデフォルト値は `Reals` (実数) になります。

今回の問題の場合、`X_use` と `Y_use` はともに非負整数という条件があるので `NonNegativeIntegers` を指定します。

```
0 <= X_use (ただし X_use は整数)
0 <= Y_use (ただし Y_use は整数)
```

引数 `initialize` では変数の初期値を指定します。初期値としては数値もしくは初期値を返すための関数を指定できます。
`initialize` で指定された値は最適化処理を行う際に解を探索する開始点として利用されます。
今回の問題の場合は非負整数であれば何でも良いので、初期値として0を指定しました。

このように作成した変数を `mdl.X_use = ` という形でモデルの属性として追加しています。
Python ではドットに続く名前すべてを属性と呼びます。属性は他のプログラミング言語におけるメンバ変数、フィールド、メソッド、プロパティ等に相当します。

```py
    mdl.X_use = Var(within=NonNegativeIntegers, initialize=0)
```

この `mdl.X_use = ` の部分に馴染みがない方もいるかもしれません。ここでは Model クラスにおそらく存在しないであろう `X_use` という属性を勝手に定義しています。
これは Python の動的な属性追加という機能を利用しています。
Python は実行時にオブジェクトに対して新しい属性を追加することができ、ここではそのように追加された属性を動的属性と呼んでいます。

pyomo では変数のようなモデルを定義する要素を、動的属性によって追加することができます。
変数だけではなく、パラメータ、制約条件、目的関数もすべて動的属性によってモデルに追加することができます。
このような設計により pyomo ではモデルにその変数や制約条件等が紐づいていることを直感的に表現することができます。

ここまでで、モデルに必要な変数を追加することができました。
次はこれらの変数を利用して制約条件をモデルに追加していきましょう。ここでも動的属性を利用してモデルに追加していきます。

```
制約条件:
2 * X_use + 1 * Y_use => 5
1 * X_use + 2 * Y_use <= 4
```

上記の制約条件は以下のようにモデルに追加できます。

```py
from pyomo.environ import *

if __name__ == '__main__':
    # モデル定義
    mdl = ConcreteModel(name="dmax-practice", doc="dmax-practice: ゼロから作るモンハン最適化シミュレータ")
    
    # 変数定義
    mdl.X_use = Var(within=NonNegativeIntegers, initialize=0)
    mdl.Y_use = Var(within=NonNegativeIntegers, initialize=0)
    
    # 制約条件定義
    # 制約1: 2 * X_use + 1 * Y_use >= 5
    def constraint_1(mdl):
        return 2 * mdl.X_use + 1 * mdl.Y_use >= 5
    
    mdl.const_1 = Constraint(rule=constraint_1)
    
    # 制約2: 1 * X_use + 2 * Y_use <= 4
    def constraint_2(mdl):
        return 1 * mdl.X_use + 2 * mdl.Y_use <= 4
    
    mdl.const_2 = Constraint(rule=constraint_2)
```

pyomo で制約条件を定義する際には `Constraint` クラスを利用します。`Constraint` クラスの引数 `rule` には制約条件の式を返す関数を指定します。
`rule` 引数に渡している `constraint_1` の定義を見ると、`mdl.X_use` と `mdl.Y_use` を利用して制約条件の不等式が定義されており、直感的に制約条件を定義できていることが分かると思います。

`rule` 引数に渡す関数の第1引数は常にモデルオブジェクトを受け取ります。そのため関数 `constrait_1` を定義する際には以下のように `mdl` と記述して、第1引数にモデルを受け取るように定義する必要があります。

```py
    def constraint_1(mdl):
        return 2 * mdl.X_use + 1 * mdl.Y_use >= 5
```

このようにして作成した制約条件を表す Constraint クラスのインスタンスを `mdl.const_1 = ` という形で、動的属性によりモデルに追加しています。

```py
    mdl.const_1 = Constraint(rule=constraint_1)
```

ここまでで、変数と制約条件をモデルに追加できました。最後に目的関数をモデルに追加します。

```
50 + 50 * X_use + 40 * Y_use    
```

上記の目的関数は以下のようにモデルに追加できます。

```py
from pyomo.environ import *

if __name__ == '__main__':
    # モデル定義
    mdl = ConcreteModel(name="dmax-practice", doc="dmax-practice: ゼロから作るモンハン最適化シミュレータ")
    
    # 変数定義
    mdl.X_use = Var(within=NonNegativeIntegers, initialize=0)
    mdl.Y_use = Var(within=NonNegativeIntegers, initialize=0)
    
    # 制約条件定義
    def constraint_1(mdl):
        return 2 * mdl.X_use + 1 * mdl.Y_use >= 5
    
    mdl.const_1 = Constraint(rule=constraint_1)
    
    def constraint_2(mdl):
        return 1 * mdl.X_use + 2 * mdl.Y_use <= 4
    
    mdl.const_2 = Constraint(rule=constraint_2)
    
    # 目的関数定義: 50 + 50 * X_use + 40 * Y_use を最大化
    def objective_function(mdl):
        return 50 + 50 * mdl.X_use + 40 * mdl.Y_use
    
    mdl.OBJ = Objective(rule=objective_function, sense=maximize)
```

pyomo で目的関数を定義する際には Objective クラスを利用します。 Objective クラスの取り扱いは Constraint とほぼ同じです。

引数 `rule` には目的関数の式を返す関数を指定します。Constraint の場合には、制約条件をを返すために等式や不等式を返す関数を指定していましたが、Objective の場合には、目的関数の式を返すため等号や不等号は登場しません。

引数 `sense` には最適化の方向性として目的関数を最大化したいのか最小化したいのかを指定します。デフォルトは `sense=minimize` (最小化) なので、今回のような最大化の場合には明示的に `sense=maximize` と指定する必要があります。

このようにして作成した目的関数を、動的属性によりモデルに追加します。

```py
    mdl.OBJ = Objective(rule=objective_function, sense=maximize)
```

ここまでで、モデルの定義がすべて完了しました。最後にモデルをファイルに出力しましょう。
(ここで出力するファイルは後に、SCIP ソルバーに入力することで最適解を得ることができます)

```py
from pyomo.environ import *

if __name__ == '__main__':
    # モデル定義
    mdl = ConcreteModel(name="dmax-practice", doc="dmax-practice: ゼロから作るモンハン最適化シミュレータ")
    
    # 変数定義
    mdl.X_use = Var(within=NonNegativeIntegers, initialize=0)
    mdl.Y_use = Var(within=NonNegativeIntegers, initialize=0)
    
    # 制約条件定義
    def constraint_1(mdl):
        return 2 * mdl.X_use + 1 * mdl.Y_use >= 5
    
    mdl.const_1 = Constraint(rule=constraint_1)
    
    def constraint_2(mdl):
        return 1 * mdl.X_use + 2 * mdl.Y_use <= 4
    
    mdl.const_2 = Constraint(rule=constraint_2)
    
    # 目的関数定義
    def objective_function(mdl):
        return 50 + 50 * mdl.X_use + 40 * mdl.Y_use
    
    mdl.OBJ = Objective(rule=objective_function, sense=maximize)
    
    # 問題ファイルを出力
    # symbolic_solver_labels を有効化して変数名等の情報を保持
    mdl.write("dmax-practice-problem.nl", format="nl", io_options={'symbolic_solver_labels': True})
    print("最適化問題のモデルをファイルを出力しました")
```

以下のように、モデルインスタンスの `write` メソッド `mdl.write()` を利用することで、定義したモデルをファイルに出力することができます。

```py
    mdl.write("dmax-practice-problem.nl", format="nl", io_options={'symbolic_solver_labels': True})
```

第1引数にはファイル名を指定します。

引数 `format` ではファイルの形式を指定できます。ここでは多くの最適化問題とソルバーに対応している `nl` (AMPL Nonlinear) 形式を指定します。

引数 `io_options` では出力のオプションを指定できます。`'symbolic_solver_labels': True` によって、人間が読める形式のラベルがファイルに保存されるようになります。
デバッグを行う際にファイルが人間が読めると便利なので指定しておきましょう。


以上で、最適化問題のモデルをファイルを出力するためのプログラムが完成しました。
ターミナルから以下のようにして実行してみましょう。
`dmax-practice-problem.*` という名前のファイルが3つ出力されていれば成功です。

```sh
# プログラムを実行
$ uv run main.py
最適化問題のモデルをファイルを出力しました

# ls コマンドでファイルが出力されたかどうか確認
$ ls -lh | grep dmax-practice
-rw-r--r-- 1 hoge hoge   12 Jul 21 11:40 dmax-practice-problem.col
-rw-r--r-- 1 hoge hoge  747 Jul 21 11:40 dmax-practice-problem.nl
-rw-r--r-- 1 hoge hoge   20 Jul 21 11:40 dmax-practice-problem.row
```

出力されたファイルを眺めてみましょう。

```sh
# .nl ファイルの中身を確認 (出力は長いので省略)
$ cat dmax-practice-problem.nl

# .col ファイルに変数の名前が保存されていることを確認
$ cat dmax-practice-problem.col
X_use
Y_use

# .row ファイルに制約式や目的関数の名前が保存されていることを確認
$ cat dmax-practice-problem.row
const_1
const_2
OBJ
```

`.nl` 形式のファイルがメインのモデルファイルです。最適化ソルバーに入力する問題ファイルは `.nl` 形式のファイルになります。
では残りの `.col` や `.row` 形式のファイルは何かというと、python プログラム中で指定した変数、制約式、目的関数の名前が出力されるファイルになります。
`io_options={'symbolic_solver_labels': True}` を指定することによって、`.col` や `.row` 形式のファイルが出力されるようになります。
これらのファイルがあると最適化問題の結果やモデルファイルが人間の読める形式で出力されるようになるため、結果の表示やデバッグに役立ちます。


次はいよいよ、最適化ソルバーに最適化問題を解かせてみます。
python プログラムに出力されたモデルのメインファイル `dmax-practice-problem.nl` をSCIPソルバーに入力して最適化してみましょう。

まず、SCIPソルバーを対話モードで起動します。


```sh
dmax-scratch@DESKTOP-BP23A1J:~$ /home/dmax-scratch/SCIPOptSuite-8.0.3-Linux/bin/scip
SCIP version 8.0.3 [precision: 8 byte] [memory: block] [mode: optimized] [LP solver: Soplex 6.0.3] [GitHash: 62fab8a2e3]
Copyright (C) 2002-2022 Konrad-Zuse-Zentrum fuer Informationstechnik Berlin (ZIB)

External libraries:
  Soplex 6.0.3         Linear Programming Solver developed at Zuse Institute Berlin (soplex.zib.de) [GitHash: f900e3d0]
  CppAD 20180000.0     Algorithmic Differentiation of C++ algorithms developed by B. Bell (github.com/coin-or/CppAD)
  ZLIB 1.2.11          General purpose compression library by J. Gailly and M. Adler (zlib.net)
  GMP 6.2.1            GNU Multiple Precision Arithmetic Library developed by T. Granlund (gmplib.org)
  ZIMPL 3.5.3          Zuse Institute Mathematical Programming Language developed by T. Koch (zimpl.zib.de)
  AMPL/MP 4e2d45c4     AMPL .nl file reader library (github.com/ampl/mp)
  PaPILO 2.1.2         parallel presolve for integer and linear optimization (github.com/scipopt/papilo) [GitHash: 2fe2543]
  bliss 0.77           Computing Graph Automorphism Groups by T. Junttila and P. Kaski (www.tcs.hut.fi/Software/bliss/)
  Ipopt 3.13.2         Interior Point Optimizer developed by A. Waechter et.al. (github.com/coin-or/Ipopt)

user parameter file <scip.set> not found - using default parameters

SCIP>
```

次に、以下のように `read` コマンドを実行してモデルファイル `dmax-practice-problem.nl` をSCIPソルバーに入力します。
読込結果から3つの変数 (variables) と2つの制約式 (constraints) があることがわかります。
python プログラム中で定義した変数は `X_use` と `Y_use` の2つのみですが、`.nl` 形式では目的関数の定数項を変数として表現しているため変数が1つ増えています。

```sh

SCIP> read dmax-practice-problem.nl

read problem <dmax-practice-problem.nl>
============

original problem has 3 variables (0 bin, 2 int, 0 impl, 1 cont) and 2 constraints
```

次に、以下のように `optimize` コマンドを実行して最適化を実行してください。
出力結果の `SCIP Status        : problem is solved [optimal solution found]` から最適化が完了し、最適解が見つかったことが分かります。

```sh
SCIP> optimize

solution violates original bounds of variable <objconstant> [50,50] solution value <0>
all 1 solutions given by solution candidate storage are infeasible

presolving:
(round 1, fast)       2 del vars, 1 del conss, 0 add conss, 2 chg bounds, 0 chg sides, 0 chg coeffs, 0 upgd conss, 0 impls, 0 clqs
(round 2, fast)       2 del vars, 2 del conss, 0 add conss, 3 chg bounds, 0 chg sides, 0 chg coeffs, 0 upgd conss, 0 impls, 0 clqs
presolving (3 rounds: 3 fast, 1 medium, 1 exhaustive):
 3 deleted vars, 2 deleted constraints, 0 added constraints, 3 tightened bounds, 0 added holes, 0 changed sides, 0 changed coefficients
 0 implications, 0 cliques
transformed 1/1 original solutions to the transformed problem space
Presolving Time: 0.00

SCIP Status        : problem is solved [optimal solution found]
Solving Time (sec) : 0.00
Solving Nodes      : 0
Primal Bound       : +2.50000000000000e+02 (1 solutions)
Dual Bound         : +2.50000000000000e+02
Gap                : 0.00 %
```

最後に、以下のように `display solution` を実行して、得られた最適解を表示してください。
結果から目的関数を最大化する最適解は `X_use=4`, `Y_use=0`  (0なので表示されていない) の場合であり、目的関数の最大値は250になることが分かります。

```sh
SCIP> display solution

objective value:                                  250
X_use                                               4   (obj:50)
objconstant                                        50   (obj:1)
```

以上の結果から、ゲーム内の条件が以下のような形式だった場合に、そのゲーム内で実現できる最大ダメージは250であり、実現するための装備は「防具Xのみ4つ装備」であることがわかります。

```
武器C: 攻撃力 30, 属性値 20
防具X: 攻撃強化Ⅱ, 属性強化Ⅰ, 体術 x2, 重量1kg
防具Y: 攻撃強化Ⅰ, 属性強化Ⅱ, 体術,    重量2kg

スキルの効果:
攻撃強化Ⅰ: 攻撃力を +10 (装備全体で「攻撃強化Ⅰ」がN個ついている場合、攻撃力の加算値は +10N とする。攻撃強化Ⅱ,属性強化Ⅰ,Ⅱについても同様)
攻撃強化Ⅱ: 攻撃力を +30
属性強化Ⅰ: 属性値を +20
属性強化Ⅱ: 属性値を +30
体術: ダメージ計算には寄与しない

装備の条件: 
(条件1) 防具は0個以上のいくつでも装備できる
(条件2) 体術は最低5個欲しい
(条件3) 防具重量の合計の上限は4kg (4kg 以上の防具は重くて装備できない)

ダメージ計算式:
(ダメージ) = {(武器の攻撃力) + (攻撃強化スキルによる攻撃力加算)}
           + {(武器の属性値) + (属性強化スキルによる属性値加算)}
```

ここまでの流れで、最適化ソルバーにゲーム内の最適化問題を解かせることができました。
流れをまとめると
「ゲーム内の条件を制約式と目的関数に落とし込み、Python プログラムによって最適化問題をモデルとしてファイルに出力し、モデルファイルを最適化ソルバーに読み込ませて最適解を得る」という流れになっていました。



var, rule, index を深堀りしたい。

`rule` 関数の戻り値としては基本的に制約式を返す必要があります。制約式は `<=`, `>=`, `==` を含む関係式により定義します。
したがって、もし「防具の合計数が必ず1でなければならない」という制約がある場合は、以下のような `==` による制約式を定義することになるでしょう。

```py
    def constraint_armor_leq(mdl):
        return mdl.X_use + mdl.Y_use == 1
```














## モンハンワイルズのデータで最適化シミュを自作する

次はいよいよ、実際のモンハンワイルズのデータを利用して最適化シミュレータを作成していきます。

この章で実装する最適化シミュの名前は dmax-mini と呼ぶことにします。

dmax-mini では装備やスキルの種類の数は絞りますが、実際の最適化シミュ (DMAX) で考慮している処理はすべて実装していきます。
なので、こちらの dmax-mini をベースに対応するデータを拡張していけば DMAX と同じ最適化シミュが実装できるようになります。

ステップ
1. 防具 + 護石
2. 防具 + 護石 + 装飾品
3. 防具 + 護石 + 装飾品 + スキル下限レベル


ワイルズの最適化シミュレータを作成するうえでの課題は以下です。


- 装備の制約条件
  - 各装備は1個以下しか装備できない (装備するか、装備しないかの2択)
  - 各部位ごとに装備できる防具は1個以下
  - 配列の考え方
- グループスキルの考え方
- 装飾品の制約条件
  - 装飾品は装飾品スロットLv以下のものしか着けられない
- スキルの制約条件
  - 同じスキルのうち、1つのレベルだけが有効になる
- ダメージ計算式
  - スキルが発動したときにダメージ計算式に影響を及ぼすことを式で表現する
  - 加算スキルの表現方法
  - 乗算スキルの表現方法
- 会心率の制約条件
  - 会心率は 100% を超えることはできない
- ユーザが指定したスキルレベルの制約条件
  - ユーザが指定したスキルレベルの下限以上になる装備構成を出力したい
- 定数の管理


ここから必要なデータを整理する
- 武器装飾品
- 防具装飾品
- 攻撃系スキル
- 会心系スキル
- 属性系スキル


装飾品の考え方
簡単のために Lv1装飾品は Lv2 に装着できないとする

armor_a := Lv1スロットが1つ, Lv2スロットが2つ
armor_b := Lv1スロットが2つ, Lv2スロットが1つ

deco_j := Lv1装飾品
deco_k := Lv1装飾品 

deco_l := Lv2装飾品
deco_m := Lv2装飾品 

1 * use_armor_a  + 2 * use_armor_b >= num_deco_j + num_deco_k
2 * use_armor_a  + 1 * use_armor_b >= num_deco_l + num_deco_m

Lv1装飾品は Lv2 にも装着できる

3 * use_armor_a  + 3 * use_armor_b >= num_deco_j + num_deco_k                                # <- 1スロ装飾品は Lv2スロットにも装着できる
2 * use_armor_a  + 1 * use_armor_b >= (num_deco_l + num_deco_m) + (num_deco_j + num_deco_k)  # <- 1スロ装飾品が Lv2スロットを使ったら、その分Lv2スロットの枠を消費すべき

3 * use_armor_a  + 3 * use_armor_b >= (num_deco_j + num_deco_k) + (num_deco_l + num_deco_m)
2 * use_armor_a  + 1 * use_armor_b >= (num_deco_l + num_deco_m)                            


1スロはlv2にもはまるので上の式に足しておく

困った点
・lv2がレベルにハマるよにみえる→でも下の式で抑えられているからok

・実際はlv2スロは1と2で取り合うべきなので上の式か下の式のどちらかに1と2の両方を追加して取り合うようにすべき
上下のどちらの式を修正しても問題ないように見える
ほげ — 昨日 20:10
アプローチ
lv2スロには1も2もはいる
←上か下の式を修正
ただこれだと1と2によるスロットの取り合いが発生しない
←増やした差分ポイント分は2も消費するので上の式に2をいれる
or
←増やした差分を1が消費するのはいいが、lv2は1と2で取り合うべきなので下の式に1を追加する
ほげ — 10:04
↑これ不等号逆でも成り立つは嘘だった

下のlv2スロの制約式にスロ1装飾品をいれることで、スロ2とスロ1の枠の取り合いを起こそうとすると、実際にはlv1スロがまだ余っているのに、lv2スロ側の制約をうけて スロ1が装着できなくなってしまう



use_armor_a * 2  + use_armor_b * 2 * b_s1 >= num_deco_j + num_deco_k























## 変数同士の掛け算は解くのが遅くなる

## 書きたいこと

- ダメージ最大化問題における最適解とはなにか？
  - 装備を何個使うかを表したベクトル です
  - 全探索すればいい → 無理 → 最適化ソルバーへ

- もっと概要が一目瞭然にわかるように書きたい
  - 基本方針は「最適化ソルバーにおまかせ！」
  - 線形でなくても解けるよ！
  - 式に落とし込むことがすべて
  - ダメージ計算式を推測する、キャッチアップするのが困難


- どういう制約式なら解けるのか？
  - 変数と定数で表現する
  - 閉じていないといけない
  - 線形なら軽い
  - 非線形でも解けるが重くなっていく
  - 重すぎると有限時間内に解けなくなる
    - 重さ対決! サンブレイク vs ワイルズ

- 条件を数式に落とし込む
  - 装備は 0, 1 変数で表現 (-1 とか 2とかない)
  - もう少し別の例
    - スキルの on/off や必須スキル
  - 装飾品の考え方
  - 必須スキル指定


- 環境構築が最大の難関
  - 環境の概要図を書く (問題ファイル作成は python + pyomo 作成した問題を解くのが scip)
- dmax-practice
  - pyomo -> 問題ファイル -> scip ソルバ -> 出力
- 実データ簡易版 
  - データ準備、入力準備
  - dmax コード書いて動作まで確認
  - dmax コードの解説
    - 数式？ 行列はなくても 説明できるかも？

- 泥臭い部分
  - ダメージ計算式を割り出す
  - データを整形して入力する
  - 便利なUIを実装する
  - ソルバをどこで動かすか？

- グラフ化したこと と 線形性、非線形の説明があまりつながっていない

- web サービス化編
- web サービス化 wasm 編
