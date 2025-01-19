---
title: "Ebitengine でブロック崩しをつくる"
emoji: "🧱"
type: "tech"
topics:
  - "go"
published: false
---

# はじめに
本記事では、 Ebitengine を使ってゲームを作る方法を簡単に学ぶため、ブロック崩しを作成します。

# 開発環境構築
## Go のインストール
公式ドキュメントに従って、 Go をインストールします。
https://go.dev/doc/install

以下のコマンドを実行し、バージョンが出力されていれば OK です。

```bash
$ go version
# go version go1.23.1 darwin/arm64
```

## C コンパイラのインストール（macOS, Linux の場合）
Ebitengine のドキュメントを参考に、 C コンパイラをインストールします。
https://ebitengine.org/en/documents/install.html

## Ebitengine の動作確認
以下のコマンドを実行して、 Gopher くんが回転していることを確認してください。

```bash
$ go run github.com/hajimehoshi/ebiten/v2/examples/rotate@latest
```

![回転している Gopher くん](https://storage.googleapis.com/zenn-user-upload/ae62343075ed-20250115.png)

# Hello, World の出力

## プロジェクトの作成
まずは、ブロック崩し用のプロジェクトを作成します。

```bash
# 適当なディレクトリに移動してから下記コマンドを実行
$ mkdir brickbreaker
$ cd brickbreaker

# <username> を適当に置き換える
$ go mod init github.com/<username>/brickbreaker
$ touch main.go
```

## Hello, World プログラムの作成

main.go ファイルを以下のように編集します。コードの詳細は次の章で解説するので、いったん理解しなくても大丈夫です。

```go:main.go
package main

import (
	"log"

	"github.com/hajimehoshi/ebiten/v2"
	"github.com/hajimehoshi/ebiten/v2/ebitenutil"
)

type Game struct{}

func (g *Game) Update() error {
	return nil
}

func (g *Game) Draw(screen *ebiten.Image) {
	ebitenutil.DebugPrint(screen, "Hello, World!")
}

func (g *Game) Layout(outsideWidth, outsideHeight int) (screenWidth, screenHeight int) {
	return 320, 240
}

func main() {
	ebiten.SetWindowSize(640, 480)
	ebiten.SetWindowTitle("Hello, World!")
	if err := ebiten.RunGame(&Game{}); err != nil {
		log.Fatal(err)
	}
}
```

## 実行

コードが書けたら、下記コマンドを実行して実行します。

```bash
$ go mod tidy

$ go run .
```

以下のような画面が出力されると成功です。

![Hello, World](https://storage.googleapis.com/zenn-user-upload/f8e6aae4f374-20250115.png)

## コードの解説

ここから先ほど書いた Hello, World のコードを解説していきます。

### Ebitengine パッケージのインポート

import 文で、 Ebitengine のパッケージをインポートしています。インポートすると `ebiten.Xxx` や `ebitenutil.Xxx` という書き方で、 Ebitengine で定義された関数などを利用することができます。

```go
package main

import (
	"log"

	"github.com/hajimehoshi/ebiten/v2"
	"github.com/hajimehoshi/ebiten/v2/ebitenutil"
)
```

### Game 構造体
Game 構造体は以下のようになります。これは [ebiten.Game](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#Game) インターフェースの実装になります（= Update, Draw, Layout メソッドを実装する）。

ここで、 Update メソッドは 「tick」ごとに実行されます。 tick とは、更新の時間単位で、デフォルトで1/60秒になります。すなわち1秒間に60回 Update メソッドが実行されます。 Update メソッド内でブロック崩しのボールやプレイヤーの移動の処理を書くことになりますが、ここでは何もせずに nil を返します。

Update に対して Draw メソッドは「フレーム」ごとに実行されます。フレームとは、画面をレンダリングに要する時間単位のことで、ディスプレイのリフレッシュレートごとに異なる値となります。例えばリフレッシュレートが 120 Hz のディスプレイを使用している場合は、1秒間に120回 Draw メソッドが実行されます。ここでは ebitenutil.DebugPrint 関数を実行して Hello, World で出力します。

これらのメソッドに関してはドキュメントの [How to code works](https://ebitengine.org/en/tour/hello_world.html#How_the_code_works) などを参照してください。
```go
type Game struct{}

func (g *Game) Update() error {
	return nil
}

func (g *Game) Draw(screen *ebiten.Image) {
	ebitenutil.DebugPrint(screen, "Hello, World!")
}

func (g *Game) Layout(outsideWidth, outsideHeight int) (screenWidth, screenHeight int) {
	return 320, 240
}
```

### main 関数

main 関数では、はじめにウィンドウサイズ、タイトルを設定します。その後 ebiten.RunGame でゲームを起動します。引数に Game 構造体のポインタを渡すことで、定期的に Game 構造体の Update メソッドや Draw メソッドが実行されます。
```go
func main() {
	ebiten.SetWindowSize(640, 480)
	ebiten.SetWindowTitle("Hello, World!")
	if err := ebiten.RunGame(&Game{}); err != nil {
		log.Fatal(err)
	}
}
```

# ブロックの描画

ひととおり基礎は掴めたかと思うので、ここからブロック崩しを作っていきたいと思います。まずはブロックを描画していきます。

## ブロック構造体の生成


ブロックに関するコードを書くために、ファイルを作成します。

```bash
touch block.go
```

コードは以下になります。 Block 構造体が今回描画するブロックを表しています。ブロックを描画する時の座標 (x, y) や、ブロックの幅、高さ、色などの情報を持っています。
また、ブロックの描画に必要な情報を定数として定義しています。

```go:blcok.go
package main

import "image/color"

const (
	blockRowNums    = 5  // ブロックを何行ならべるか
	blockCloumnNums = 10 // ブロックを何列ならべるか
	blockPadding    = 10 // ブロックとブロックの間の間隔
	blockTopOffset  = 50 // 画面の上端とブロックの間の間隔
	blockWidth      = 60 // ブロックの幅
	blockHeight     = 20 // ブロックの高さ
)

type Block struct {
	x, y      float64     // ブロックの描画位置 (x, y)。左上が (0, 0) になる
	width     float64     // ブロックの幅
	height    float64     // ブロックの高さ
	isVisible bool        // ブロックが見えるかどうか。ボールが衝突したら false になる
	color     color.Color // ブロックの色
}
```

![変数の説明図](https://storage.googleapis.com/zenn-user-upload/786ef52f8116-20250119.png)

Block 構造体が定義できたら、ブロックのスライスを生成する関数を作成します。 blockRowNums と blockCloumnNums で指定した行列分の Block 構造体を生成してスライスに追加します。

```go:blcok.go
// ゲーム開始した時のブロックを生成する
func generateInitialBlocks() []*Block {
	var blocks []*Block
	// 指定した行、列の分だけ Block 構造体を作成してスライスに追加する
	for row := 0; row < blockRowNums; row++ {
		for col := 0; col < blockCloumnNums; col++ {
			block := &Block{
				x:         float64(col)*(blockWidth+blockPadding) + blockPadding,    // 左端からの距離
				y:         float64(row)*(blockHeight+blockPadding) + blockTopOffset, // 上端からの距離
				width:     blockWidth,
				height:    blockHeight,
				isVisible: true, // 初期化時は true にしてみえるようにする
				color: color.RGBA{
					R: uint8(200 - row*30), // 何行目かで色を少し変える
					G: uint8(200 - row*30), // 何行目かで色を少し変える
					B: 255,
					A: 255,
				},
			}
			blocks = append(blocks, block) // スライスに追加
		}
	}

	return blocks
}
```

## 描画

まずは Game 構造体を更新します。 NewGame 関数でブロックを生成して Game 構造体を初期化します。

```diff:main.go
+const (
+	gameScreenWidth  = 640 // ゲームのウィンドウの横幅
+	gameScreenHeight = 480 // ゲームのウィンドウの縦幅
+)

-type Game struct{}
+type Game struct {
+	blocks []*Block
+}

+func NewGame() *Game {
+	g := &Game{}
+	g.blocks = generateInitialBlocks()
+
+	return g
+}
```

Draw メソッドを更新して、 Game 構造体がもっている Block を描画します。

```diff:main.go
func (g *Game) Draw(screen *ebiten.Image) {
-	ebitenutil.DebugPrint(screen, "Hello, World!")
+	// ブロックの描画
+	for _, block := range g.blocks {
+		// isVisible == false の Block（ボールが衝突した場合）は表示しない
+		if block.isVisible {
+			blk := ebiten.NewImage(int(block.width), int(block.height)) // 画像の生成
+			blk.Fill(block.color)                                       // 色の指定
+			var opts ebiten.DrawImageOptions                            // オプションの宣言
+			opts.GeoM.Translate(block.x, block.y)                       // 描画位置を指定
+			screen.DrawImage(blk, &opts)                                // 画像を指定したオプションで描画
+		}
+	}
}
```

Layout メソッドを更新して、幅、高さを指定します。
```diff:main.go
func (g *Game) Layout(outsideWidth, outsideHeight int) (screenWidth, screenHeight int) {
-	return 320, 240
+	return gameScreenWidth, gameScreenHeight
}
```

main 関数では、 NewGame 関数で Game 構造体を初期化してから `ebiten.RunGame` を実行します。

```diff:main.go
func main() {
-	ebiten.SetWindowSize(640, 480)
-	ebiten.SetWindowTitle("Hello, World!")
-	if err := ebiten.RunGame(&Game{}); err != nil {
+	g := NewGame() // ゲームの初期化
+	ebiten.SetWindowSize(gameScreenWidth, gameScreenHeight)
+	ebiten.SetWindowTitle("ブロック崩し")
+	if err := ebiten.RunGame(g); err != nil {
		log.Fatal(err)
	}
}
```

## 実行
コードが書けたら実行します。

```bash
$ go run .
```

以下のようにブロックが描画されていたら成功です。

![ブロックの描画](https://storage.googleapis.com/zenn-user-upload/da737fe4a526-20250115.png)

# プレイヤーの描画

続いて、プレイヤーを作成していくのでファイルを作成します。

```bash
touch player.go
```

コードは以下のようになります。プレイヤーの幅と高さ、描画位置を指定して初期化します。

```go:player.go
package main

// プレイヤーの設定
const (
	playerWidth    = 80                                  // プレイヤーの幅
	playerHeight   = 10                                  // プレイヤーの高さ
	initialPlayerX = (gameScreenWidth - playerWidth) / 2 // プレイヤーの初期位置
	initialPlayerY = gameScreenHeight - playerHeight     // プレイヤーの初期位置
)

type Player struct {
	x, y   float64
	width  float64
	height float64
}

func NewPlayer() *Player {
	return &Player{
		x:      initialPlayerX,
		y:      initialPlayerY,
		width:  playerWidth,
		height: playerHeight,
	}
}
```

Game 初期化時に Player も初期化します。

```diff:main.go
type Game struct {
	blocks []*Block
+	player *Player
}

func NewGame() *Game {
	g := &Game{}
	g.blocks = generateInitialBlocks()
+	g.player = NewPlayer()

	return g
}
```

Draw メソッドを更新して、プレイヤーも描画します。プレイヤーの描画はブロックの描画とほぼ同じ処理になります。

```diff:main.go
func (g *Game) Draw(screen *ebiten.Image) {
+	// プレイヤーの描画
+	player := ebiten.NewImage(int(playerWidth), int(playerHeight))
+	player.Fill(color.White)
+	var playerOpts ebiten.DrawImageOptions
+	playerOpts.GeoM.Translate(g.player.x, g.player.y)
+	screen.DrawImage(player, &playerOpts)

	// ブロックの描画
	...
}
```

ここでプレイヤーが画面下部に描画されていることを確認しましょう。

![プレイヤーの描画](https://storage.googleapis.com/zenn-user-upload/915c89e84d23-20250117.png)


# プレイヤーの移動

プレイヤーが描画できたので、次に横方向に移動できるようにしていきます。まずはプレイヤーを動かすスピードを定義しておきます。

```diff:player.go
// プレイヤーの設定
const (
	playerWidth        = 80 // プレイヤーの幅
	...
+	playerSpeed = 6         // プレイヤーのスピード
)

type Player struct {
	...
+	speed  float64
}


func NewPlayer() *Player {
	return &Player{
		x:      initialPlayerX,
		...
+		speed:  playerSpeed,
	}
}
```

speed を定義できたら、 Update メソッドを実装します。 IsKeyPressed 関数は指定したキーを押しているかどうかを返します。左矢印キーを押している間はプレイヤーの x 座標を減らしつづけ（左方向に移動）、0より小さくなる場合は0に設定することで画面の左端よりも左側に移動しないようにします。右側も同様のロジックになります。

```go:main.go
func (g *Game) Update() error {
	// プレイヤーの移動
	if ebiten.IsKeyPressed(ebiten.KeyLeft) {
		g.player.x -= g.player.speed
		if g.player.x < 0 {
			g.player.x = 0
		}
	}
	if ebiten.IsKeyPressed(ebiten.KeyRight) {
		g.player.x += g.player.speed
		if g.player.x > gameScreenWidth-playerWidth {
			g.player.x = gameScreenWidth - playerWidth
		}
	}

	return nil
}
```

それでは矢印キーを押してプレイヤーを移動できるかどうか確認してみてください。

```bash
go run .
```

![プレイヤーが移動している画面](https://storage.googleapis.com/zenn-user-upload/6f7eb3656841-20250119.png)


# ボールの描画

つづいてボールを描画していきます。

## 描画

ボールを描画するため、ファイルを作成します。

```bash
touch ball.go
```

コードは以下のようになります。

```go:ball.go
package main

import (
	"image/color"
	"math"

	"github.com/hajimehoshi/ebiten/v2"
	"github.com/hajimehoshi/ebiten/v2/vector"
)

// ボールの設定
const (
	ballRadius   = 5                    // ボールの半径
	ballSpeed    = 5                    // ボールが移動するスピード
	ballInitialX = gameScreenWidth / 2  // ボールの初期位置
	ballInitialY = gameScreenHeight / 2 // ボールの初期位置
)

type Ball struct {
	x, y           float64
	radius         float64
	speedX, speedY float64
}

func NewBall() *Ball {
	return &Ball{
		x:      ballInitialX,
		y:      ballInitialY,
		radius: ballRadius,
		// 右下斜め45度に向かって、速さ ballSpeed で進む
		speedX: ballSpeed * math.Cos(math.Pi/4),
		speedY: ballSpeed * math.Sin(math.Pi/4),
	}
}
```

Game 構造体を初期化するときに Ball も初期化します。

```diff:main.go
type Game struct {
	blocks []*Block
	player *Player
+	ball   *Ball
}

func NewGame() *Game {
	g := &Game{}
	g.blocks = generateInitialBlocks()
	g.player = NewPlayer()
+	g.ball = NewBall()

	return g
}
```

描画する処理は以下のようになります。完璧な円を描くのは難しいので、円に近い多角形を描画します。やや処理が複雑なので、中身は理解できなくても、この関数は円を描画するんだなと感じていただければ OK です。

```go:main.go
// 指定した座標にボールを描画する。円に近くなるように多角形を描いている。
func DrawBall(screen *ebiten.Image, ball *Ball, clr color.Color) {
	segments := 72
	for i := 0; i < segments; i++ {
		angle1 := float64(i) / float64(segments) * 2 * math.Pi
		angle2 := float64(i+1) / float64(segments) * 2 * math.Pi
		x1 := ball.x + ball.radius*math.Cos(angle1)
		y1 := ball.y + ball.radius*math.Sin(angle1)
		x2 := ball.x + ball.radius*math.Cos(angle2)
		y2 := ball.y + ball.radius*math.Sin(angle2)
		vector.StrokeLine(screen, float32(x1), float32(y1), float32(x2), float32(y2), 1, clr, false)
	}
}
```

定義した DrawBall 関数を Draw メソッド内で実行します。

```go:main.go
func (g *Game) Draw(screen *ebiten.Image) {
	// プレイヤーの描画
	...

	// ボールの描画
	DrawBall(screen, g.ball, color.White)

	// ブロックの描画
	...
}
```

ここでボールが描画されていることを確認しましょう。

```bash
go run .
```

![ボールの描画](https://storage.googleapis.com/zenn-user-upload/b3002bb6202d-20250118.png)


## ボールの移動

次に、ボールを動かしていきます。ボールのスピードに従って座標を更新した後に、ボールと壁との間で衝突判定を行います。衝突した場合はスピードの向きを反転させて逆方向に移動するようにします。

ボールが下端に到達した場合はゲームオーバーとなるので、初期化してゲームを再開します。

```diff:main.go
func (g *Game) Update() error {
	...
	if ebiten.IsKeyPressed(ebiten.KeyRight) {
		...
	}

+	// ボールの移動
+	g.ball.x += g.ball.speedX
+	g.ball.y += g.ball.speedY
+
+	// ボールと壁との衝突判定
+	// 左の壁：g.ball.x < g.ball.radius
+	// 右の壁：g.ball.x > gameScreenWidth-g.ball.radius
+	if g.ball.x < g.ball.radius || g.ball.x > gameScreenWidth-g.ball.radius {
+		g.ball.speedX *= -1
+	}
+	// 上の壁：g.ball.y < g.ball.radius
+	if g.ball.y < g.ball.radius {
+		g.ball.speedY *= -1
+	}
+	// 下の壁はゲームオーバー
+	if g.ball.y > gameScreenHeight-g.ball.radius {
+		g.initialize() // ゲームオーバーになったら、ゲームを初期化する。
+	}

	return nil
}
```

initialize メソッドは以下のようになります。 NewGame の時も同じ処理になるので書きかえておきます。

```diff:go
func NewGame() *Game {
	g := &Game{}
-	g.blocks = generateInitialBlocks()
-	g.player = NewPlayer()
-	g.ball = NewBall()
+	g.initialize()

	return g
}

+func (g *Game) initialize() {
+	g.blocks = generateInitialBlocks()
+	g.player = NewPlayer()
+	g.ball = NewBall()
+}
```

ここで動作を確認しておきます。ボールが右下に向かって移動して、下端に到達したら初期位置に戻って右下に移動する、という処理が無限に続いていれば成功です。

![ボールが動いている画面](https://storage.googleapis.com/zenn-user-upload/bde2e5184b47-20250119.png)

# 衝突処理

ここまで実装できたら、あとは衝突処理を実装することでブロック崩しが完成になります。

## プレイヤーとボールの衝突

プレイヤーとボールの衝突を実装します。ボールの y 軸方向の速度を反転させることで衝突を表現します。

```diff:main.go
func (g *Game) Update() error {
	...

+	// プレイヤーとボールの衝突判定
+	if (g.ball.y+g.ball.radius >= g.player.y) && (g.ball.y+g.ball.radius <= g.player.y+g.player.height) {
+		if g.ball.x >= g.player.x && g.ball.x <= g.player.x+g.player.width {
+			// ボールの速度を更新
+			g.ball.speedY *= -1
+
+			// ボールをプレイヤーの上に位置させる
+			g.ball.y = g.player.y - g.ball.radius - 1
+		}
+	}

	return nil
}
```

## ブロックとボールの衝突

最後に、ブロックとボールの衝突を実装します。ボールがブロックに衝突した時も y 軸方向の速度を反転させます。また、ブロックの場合は衝突したら消える必要があるので isVisible を false にします。

```diff:main.go
func (g *Game) Update() error {
	...

+	// ブロックとボールの衝突判定
+	for _, block := range g.blocks {
+		if block.isVisible {
+			if g.ball.x >= block.x && g.ball.x <= block.x+block.width &&
+				g.ball.y-g.ball.radius <= block.y+block.height && g.ball.y+g.ball.radius >= block.y {
+				g.ball.speedY *= -1
+				block.isVisible = false
+				break
+			}
+		}
+	}

	return nil
}
```

それでは実際に動かしてみましょう。これでブロック崩しは完成です。お疲れ様でした！

![完成したブロック崩し](https://storage.googleapis.com/zenn-user-upload/8807edaf71d3-20250118.png)

# おまけ

余裕のある方は、いくつか改善の余地があるのでチャレンジしてみましょう。

## 跳ね返る方向の調整
ボールがプレイヤーと衝突したときは、y 軸方向のスピードを反転させるだけでしたが、これだと毎回同じ方向に跳ね返るのでゲーム性があまりありません。そこで、プレイヤーに衝突した位置に応じて跳ね返る向きを変えると、ゲーム性が上がります。

## スタート処理の改善
現在、毎回自動的にゲームが開始していますが、あるキー（例えばスペースキー）を押したときにゲームを開始できるようにすると、より落ち着いてゲームをプレイできます。

## ゲームクリア画面
ゲームをクリアした際には達成感を味わって欲しいので、クリア画面を実装するのもおすすめです。
