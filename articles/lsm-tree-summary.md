---
title: "LSM ツリーについて学んだので整理する"
emoji: "🌲"
type: "tech"
topics:
  - "database"
published: true
published_at: 2026-03-03 08:30
---

# なぜ LSM ツリーが必要か？
一般的に MySQL などのデータベースでは、インデックスのデータ構造として B ツリーが利用されます。B ツリーは読み取りの速度が速い一方、書き込みが圧倒的に多い状況ではパフォーマンスの面でやや不利となります。書き込みが圧倒的に多い状況とは、例えばログ基盤であったり、 IoT やセンサーデータの取り込み、Web クローラの検索インデックスなど、大量のデータを高速に書き込む必要があるケースになります。

一方で LSM ツリーは上記のようなケースに向いています。ただし B ツリーと比較すると読み取り性能に関してはやや劣るので注意が必要です。読み取り性能をある程度犠牲にして、書き込み性能を最適化しています。とはいえ読み取りも様々な工夫により十分な性能を実現しています。実際に [Bigtable](https://docs.cloud.google.com/bigtable/docs/overview?hl=ja) や [Cassandra](https://cassandra.apache.org/_/index.html) などで LSM ツリーが利用されています。

# LSM ツリーの仕組み
それでは LSM ツリーの仕組みについて見ていきます。
LSM ツリーのデータ構造自体はシンプルです。一度メモリ上にデータを保持し、一定量に達するとファイルに書き出します。このファイルはイミュータブルで、データの編集や削除があったときにファイルを編集することはなく、どんどんファイルが追加されます。

下図に簡単な例を示しています。今回は `key: value` の構造でデータを保存しています。 apple がファイル A に書き込まれている状態で apple の値を更新しようとすると、最初はメモリに apple の値が格納されます。ディスクに Flush するタイミングでは、ファイル A の値を更新することなく、新規のファイル B にそのままデータが追記されます。
apple を削除する場合は tombstone（墓石）という特殊なデータとして保存します。読み取り時は新しいファイルから調べて、キーが見つかったらその値を返します。 tombstone の場合はデータが存在しないと扱って結果を返します。

![LSM ツリーにデータを追記する様子](https://storage.googleapis.com/zenn-user-upload/f30c2484d097-20260304.png)

このデータ構造で高速な書き込み、十分な読み取り性能を実現するために様々なコンポーネントが利用されます。その全体像を以下の図に示します。これらは次節から解説していきます。

![LSM ツリーの全体像](https://storage.googleapis.com/zenn-user-upload/8d14ddc2df32-20260301.png)

## MemTable と WAL
MemTable はインメモリのテーブルで、 SSTable に書き込む前のデータになります。一番簡単に実装するには Map が適しているかと思います。 SSTable はキー順にソートされている必要があるので、 Map を使用する場合は SSTable に書き込むとき（Flush 時）にソートが必要となります。

```
+----------------+
|    MemTable    |
+----------------+
|  apple  : 400  |
|  banana : 100  |
|  orange : 300  |
|      ...       |
+----------------+
```

メモリに書き込んだ時点でソート済みの MemTable を構築したい場合は、[スキップリスト](https://ja.wikipedia.org/wiki/%E3%82%B9%E3%82%AD%E3%83%83%E3%83%97%E3%83%AA%E3%82%B9%E3%83%88)と呼ばれるデータ構造が利用されることがあります。比較的シンプルで面白いデータ構造なので詳細はリンクを確認してみてください。

WAL（Write-Ahead Logging） は、障害などで MemTable に書き込んだ情報が失われないようにログファイルとしてデータを保存します。 Write-Ahead と言う言葉のとおり、 MemTable に書き込むより先にログファイルに書き込まれます。障害が起きたときは、このログファイルに書き込まれた情報から MemTable を再構築することができます。このような WAL の仕組みは PostgreSQL などのリレーショナルデータベースにおいても利用されています（[参考](https://www.postgresql.org/docs/current/wal-intro.html)）。

## SSTable とコンパクション
SSTable（Sorted String Table）は、キー順にソートされたデータになります。このキー順にソートされた SSTable をファイルに書き込んでいきます。以下の図は2個しかデータがありませんが、実際はもっとたくさんのデータが格納されます。

MemTable のサイズがある閾値を超えたタイミングで SSTable としてファイルに書き込んでいくのですが、ファイル数が増えていくと探索するファイル数が多くなってしまい、読み取りのコストが増大します。そこで、コンパクションと呼ばれるマージプロセスによってファイルをまとめます。

![SSTable とコンパクション](https://storage.googleapis.com/zenn-user-upload/c1573f64683b-20260304.png)

このコンパクションにもいくつか手法があるのですが、ここでは単純化した例を紹介します。以下の図のようにレベルという概念を導入します。最初はレベル0の SSTable に書き込まれます。コンパクションする際に下の階層のレベルの SSTable として書き込まれ、そのレベルのファイルも数が増えてきたらさらに下の階層の SSTable としてコンパクションされます。これはあくまで単純化した一例ですが、実際には [Leveled Compaction](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction) などが利用されます。

![コンパクションとレベルの説明図](https://storage.googleapis.com/zenn-user-upload/a8b6b2e9359a-20260301.png)

## Bloom Filter

Bloom Filter は読み取り時の効率を高めるための工夫になります。キーが存在する可能性があるか、あるいはキーが存在しないことが確実であるかを判断できます。

具体的には、大きなビット配列に、キーごとに特有の位置を計算してフラグを立てていきます。以下の図で言うと apple に対して計算した位置が全て 1 であれば apple は存在する可能性があります（偽陽性の可能性あり）。一方で banana に対して計算した位置の一つが0になっているので、 banana は必ず存在しません。

![Bloom Filter](https://storage.googleapis.com/zenn-user-upload/7b8c9dcb88a7-20260301.png)

このようなデータ構造を SSTable ごとに作成し、インメモリに保持することでディスク I/O を削減します。

このときに位置を計算する方法は、理解せずに飛ばしても大丈夫ですが Go で書くと以下のようになります。 `numHashFunc` 個のハッシュ化した値を計算しています。ここでは計算コストを抑えるために2個のハッシュ関数をもとに計算しています。 `indices[i]` が Bloom Filter がもつビット配列の何ビット目か、という位置を表します。

```go
func (bf *BloomFilter) hashIndices(key []byte) []uint {
	// 任意のハッシュ関数2種類で計算する
	h1 := hashWith(fnv.New32(), key)
	h2 := hashWith(fnv.New32a(), key)

	totalBits := uint(len(bf.bits) * 8)
	indices := make([]uint, bf.numHashFunc)
	// Kirsch-Mitzenmacher による手法で、ハッシュ計算を減らすための工夫。
	// k 個の完全に独立したハッシュ関数を使った Bloom Filter と、この手法を使った Bloom Filter は、偽陽性率が漸近的に同じになる
    // https://www.eecs.harvard.edu/~michaelm/postscripts/tr-02-05.pdf
	for i := 0; i < bf.numHashFunc; i++ {
		indices[i] = (uint(h1) + uint(i)*uint(h2)) % totalBits
	}

	return indices
}
```

## Index

前述した通り、Bloom Filter でキーが存在するかもしれないと分かった場合は、 Bloom Filter に対応する SSTable のどの場所を見たらいいのかを、インデックスファイルを参照して調べます。今回はその一種である Sparse Index をみていきます。

SSTable に以下のようにデータが書き込まれた場合、 Sparse Index は一定間隔ごとにキーと位置を保持します。 sparse は dense の対義語で「まばらな」という意味をもちます。すべてのデータを持つのではなく、一部のデータのみインデックスとして保持しているので、英単語の意味から推測しやすいかなと思います。

![Sparse Index](https://storage.googleapis.com/zenn-user-upload/4f9a2335281a-20260301.png)

実際に `cat` の値を調べたい場合の図を以下に示します。

![LSM ツリーからデータを読み取る流れ](https://storage.googleapis.com/zenn-user-upload/66fa0ce00ccc-20260301.png)

最初に MemTable を調べてデータがなかった場合は Bloom Filter で調べます。そこでデータが存在するかもしれないと分かった場合は Sparse Index でどこにあるのかの目安を調べます。SSTable から作成しているので、Sparse Index もキー順にソートされています。したがって cat の場合は apple と gorilla の間にあると推測でき、探索範囲を絞ることができます。

Sparse Index を調べた時も確実にデータがあることは保証されず、データがない場合は次の SSTable に対応する Bloom Filter を調べる、というような流れになります。


# まとめ
ここまで LSM ツリーについて整理してみました。 Bigtable や Cassandra などの仕組みを理解するためにも、 LSM ツリーの学習はおすすめです。簡単に Go で実装してみたので、気になった方は以下を参照してみてください。

https://github.com/ksrnnb/lsm-tree

# 参考文献
- [詳説 データベース](https://www.oreilly.co.jp/books/9784873119540/)

# 宣伝
現在、私が所属しているサイボウズ株式会社では積極的にエンジニア採用を行なっています。以下のリンクから気になる職種があるかどうかだけでも見てもらえると嬉しいです！
興味のある職種があったらぜひご応募ください！

https://cybozu.co.jp/recruit/entry/?job=300+301+302+303+304+305+306+307+308+309+310+367+390+395+397+406&