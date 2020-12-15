# 算数マスターたかし

算数マスターとは，素早く正確に足し算や引き算，さらには掛け算や割り算などを計算することができる人材です．

わたしは機械学習で計算マスターを作ることにしました．

学習モデルの名前は「たかし」です．たかしくんと呼んでください．

##  たかし学習の環境構築
たかしの学習には`PyTorch==1.7.0`が必要です．
pipやanacondaでインストールしてください．
Docker環境がある場合は，Dockerを使うと楽かもしれません．

##  Dockerによるたかしの学習の環境構築
たかしの学習環境は仮想環境を利用しても作ることができます．
たかしはバーチャルな存在なのです．

### コンテナのビルド
以下のコマンドでたかしの学習用コンテナを作ることができます．
```
$docker build . -t takashi
$docker run -it -v /path/to/SansuMasterTakashi/takashi:/takashi takashi
```

### コンテナの再開
```
$docker exec -it <<CONTAINER ID>> bash
```

# 問題集の準備
たかしくんが解く練習ワークとテストを生成します．`/takashi/src`フォルダに移動してください．
```
$python3 create_datasets.py
```
で問題集を作成できますが，いくつかオプションをつけることができます．
`python3 create_datasets.py -h`とするとオプション一覧を見ることができます．
ここでは掛け算と割り算も含めた計算マスターたかしを作りたいので、
```
$python3 create_datasets.py --include_kakezan --include_warizan
```
としてワークを作成します．

たかしくんには賢くなってほしいので，ワークの問題とテスト問題をいっぱい解いてもらいます．
この問題数はオプションで増やすこともできます．
| 項目 | 値 |
| ------------- | ------------- |
| 訓練用データセットのサンプル数（ワークの収録問題数） | 10,000 |
| テスト用データセットのサンプル数（算数テストの問題数） | 1,000 |

作成先のフォルダは`takashi/data/datasets`がデフォルトになっています．確認しましょう．

確認すると，次のようになっているはずです．左側が式，右側が答えです．

毎回ランダムに生成されるので，実際の式や答えは違っていると思います．
```
6*4-5	19
5+4+1+2	12
4*6-3	21
8*9*8	576
8/4+4	6
(3-1)*(8+6)	28
7+2	9
1+6-3	4
4*3	12
8+8-5	11
...
```
いい感じに小学校っぽい算数の問題が生成されていることがわかります．
* 答えや計算過程で0以下が登場することはない．
* 割り算で小数点がでることはない．
* 割り算と掛け算は先に計算する．ただし括弧があれば括弧の中を先に計算する．
* 答えが大きすぎるものは望ましくない．
これらが（だいたい）満たされた問題が生成されていると思います．
なお，中括弧や大括弧は括弧には対応していません．

なお，オプションによっては難しくすることもできます．
```
$python3 create_datasets.py --max_depth 5 --include_kakezan --include_warizan
```
とすると，括弧の深度が最大5となり人間にとってもクッソめんどくさい式が生成できます．
```
(9*6/6-(7/(3/3))+7-4+6+7)/(1/1)	18
(4+5)/(9/3)*5*4/5-2-4	6
3-(4-(5-(9/3)))+3	4
5+2+9+4-(9/3)	17
5+7-((8-3-3)/1+2)	8
6+9/3+9+3*2+8-(1/1+1)-(8-4)	26
5+9	14
5*9	45
4+4+7/((4+7-(7-3))/(6/(7-1)))	9
(3-((3+2)/(1+4)))*(1+3+7*6+2/2)-4	90
...
```
# ワークの前処理（トークン化）
たかしは式のままではダメで，数字列にしないと学習してくれません．
たかしはわがままだと思います．


言ってもしょうがないので，下のような変換表を用いて式と答えを数字列に変換します．

| 文字 | 数字 |
| ------------- | ------------- |
| 0 | 0 |
| 1 | 1 |
| 2 | 2 |
| 3 | 3 |
| 4 | 4 |
| 5 | 5 |
| 6 | 6 |
| 7 | 7 |
| 8 | 8 |
| 9 | 9 |
| `__PAD__` | 10 |
| + | 11 |
| - | 12 |
| * | 13 |
| / | 14 |
| ( | 15 |
| ) | 16 |
| `__BOS__` | 17 |

ただし，`__PAD__`はパディング（行列にして処理するときに，式によって長さが異なると不都合なので，これで余ってる部分を埋める），`__BOS__`はBeginOfSentenceで，系列変換のデコーダーに最初に与える記号です．

例えば，式`(2+5)*9`と答え`63`は次のように変換されます（`__PAD__`は省略します）

| ( | 2 | + | 5 | ) | * | 9 |
|---|---|---|---|---|---|---|
|15 | 2 |11 | 5 |16 | 13| 9 |

|           | 6 | 3 |
|-----------|---|---|
| `__BOS__` | 6 | 3 |

`src`フォルダ内の`preprocess.py`によって前処理を行います．
```
$python3 preprocess.py
```
`-h`でオプション一覧を見ることができます．

# たかしの学習
デフォルトでは次のような条件でたかしは学習します．
エポック数を200にすると精度がよくなりました．

やっぱりワークは100週より200週したほうがいいですね．

| 項目 | 値 |
| ------------- | ------------- |
| エポック数（ワークを何週するか？） | 100 |
| バッチ数（たかしが同時に解く問題数） | 100 |
| 学習率 | 0.001 |
| 損失関数 | クロスエントロピー誤差 |
| オプティマイザ | Adam |

たかしはLSTMというRNNの一種を2つ使って構成されています．
というのも，1つ目は式を読み取る（＝理解する）部分であり，
2つ目は答えを書き出す（＝答える）部分です．

それぞれエンコーダー，デコーダーと言います．
Attentionは使っていません．シンプルなseq2seqモデルです．

まず，数字列はembedding層によって200次元のベクトルに変換されます．
そしてエンコーダー（LSTM）によってベクトル列を前方からエンコードされ，最終的に固定長のベクトル（128次元）になります．

その後，エンコード結果のベクトルと__BOS__記号が与えられると，
デコーダー（LSTM）によって答えが1文字ずつ生成されていくというながれになっています．

`src`フォルダ内の`train.py`によって学習を行います．
```
$python3 train.py
```
`-h`でオプション一覧を見ることができます．
epoch数を200にするにはこんな感じにするとできると思います．
```
$python3 train.py --epoch=200
```

# たかしのテスト
いよいよテスト当日です．

たかしの算数力はどれくらいになっているのでしょうか．
`src`フォルダ内の`test.py`によってテストを行います．
```
$python3 test.py
```

結果はどうなりましたか？

たかしはちゃんと算数マスターになれたでしょうか？
それほど重い学習ではない（CPUでも数分とか）なので，ぜひ条件を変えて試してみてください．

めざせ算数マスター！