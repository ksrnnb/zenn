---
title: "メモリアドレスからソースコードへの変換"
---

# ブレークポイントにヒットした時にコードを出力する
本章では、ブレークポイントにヒットした時に、ブレークポイントとその前後のコードを出力するようにしていきます。

## メモリアドレスからソースコードの位置を調べる

### DWARF の雰囲気を掴む

ブレークポイントにヒットした時にソースコードを出力するには、メモリアドレスからソースコードの位置を特定する必要があります。
メモリアドレスからソースコードの位置を特定するには、 [DWARF](https://dwarfstd.org/) というデバッグ情報のフォーマットを利用します。

DWARF の雰囲気を掴むために dwarfdump を使ってみます。コマンドがない場合はインストールしておきます。

```bash
sudo apt install -y dwarfdump
```

以下のコマンドで helloworld プログラムをビルドします。

```bash
go build -o helloworld.o -gcflags "all=-N -l" ./cmd/helloworld/
```

dwarfdump でビルドしたプログラムのデバッグ情報を出力します。 `-l` オプションで .debug_line の情報だけ出力するようにします。

```bash
dwarfdump -l helloworld.o | less
```

`/main` と入力して main.go のファイルを探してみると、以下のような結果が出力されます。

```
.debug_line: line number info for a single cu
Source lines (from CU-DIE at .debug_info offset 0x00002505):

            NS new statement, BB new basic block, ET end of text sequence
            PE prologue end, EB epilogue begin
            IS=val ISA number, DI=val discriminator value
<pc>        [lno,col] NS BB ET PE EB IS= DI= uri: "filepath"
0x004ae5a0  [   5, 0] NS uri: "/Users/<username>/lima/go-debugger/cmd/helloworld/main.go"
0x004ae5aa  [   5, 0] NS PE
0x004ae5ae  [   6, 0] NS
0x004ae5b4  [   6, 0]
0x004ae61d  [   7, 0] NS
0x004ae623  [   5, 0] NS
0x004ae62d  [   5, 0] NS ET
```

後半部分が今回読み解きたい部分なのですが、読み方はヘッダーに記載があります。知りたい情報は pc（メモリアドレス） と lno（行番号）、そして uri（ファイルのパス）になります。
たとえばメモリアドレスが 0x004ae5ae の場合は、 `/Users/<username>/lima/go-debugger/cmd/helloworld/main.go` の6行目に相当します。

```
<pc>        [lno,col] NS BB ET PE EB IS= DI= uri: "filepath"
```

ビルドしたファイルのデバッグ情報が得られることが分かったので、 Go がサポートしてる DWARF のバージョンを確認しておきます。 objdump コマンドで `--dwarf=info` を指定すると Compilation Unit の Version というフィールドで確認することができます。

以下のコマンドの出力から、 DWARF 4 を利用していることが分かります。仕様の詳細は [DWARF 標準](https://dwarfstd.org/download.html)でダウンロードできるので、気になる方はダウンロードして読んでみてください。
また、仕様のように細かい情報は不要で DWARF に入門したいという方には [Introduction to the DWARF Debugging Format](https://dwarfstd.org/doc/Debugging%20using%20DWARF-2012.pdf) がおすすめです。

```bash
objdump --dwarf=info helloworld.o | less

# helloworld.o:     file format elf64-x86-64

# Contents of the .debug_info section:

#   Compilation Unit @ offset 0:
#    Length:        0x2ab (32-bit)
#    Version:       4
#    Abbrev Offset: 0
#    Pointer Size:  8
```

### メモリアドレスからソースコードの位置を取得する
メモリアドレスとソースコードのファイル名と行番号の対応が取得できることが分かったので、ソースコードの出力を実装していきます。

最初にファイルを追加します。

```diff
go-debugger/
  └── debugger
     ├── breakpoint.go
     ├── debugger.go
     ├── register.go
+    └── source_code_locator.go
```

プログラムカウンタからファイル名、行番号に変換するメソッドを実装していくので、 interface を定義しておきます。その実装は SourceCodeLocator になり、 gosym.Table と dwarf.Data のポインタをフィールドとして持ちます。 Go は [debug/dwarf](https://pkg.go.dev/debug/dwarf) や [debug/gosym](https://pkg.go.dev/debug/gosym) など、デバッグに便利な標準パッケージが用意されているのでそれらを利用していきます。

```go:go-debuger/debugger/source_code_locator.go
package debugger

import (
	"debug/dwarf"
	"debug/elf"
	"debug/gosym"
	"errors"
)

type Locator interface {
	PCToFileLine(pc uint64) (filename string, line int)
}

// SourceCodeLocator converts memory address to file name and line number.
type SourceCodeLocator struct {
	st *gosym.Table
	dwf   *dwarf.Data
}
```

SourceCodeLocator の初期化処理は以下のようになります。この処理はコメントにも書いていますが、 [pclntab_test.go](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/debug/gosym/pclntab_test.go;l=86) のコードを参考にしています。

```go:go-debuger/debugger/source_code_locator.go
// This implementation is based on the process in pclntab_test.go file.
// https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/debug/gosym/pclntab_test.go;l=86
func NewSourceCodeLocator(debuggeePath string) (*SourceCodeLocator, error) {
	f, err := elf.Open(debuggeePath)
	if err != nil {
		return nil, err
	}
	defer f.Close()

	s := f.Section(".gosymtab")
	if s == nil {
		return nil, errors.New(".gosymtab section is not found")
	}

	symdata, err := s.Data()
	if err != nil {
		return nil, err
	}

	pclndata, err := f.Section(".gopclntab").Data()
	if err != nil {
		return nil, err
	}

	pcln := gosym.NewLineTable(pclndata, f.Section(".text").Addr)

	table, err := gosym.NewTable(symdata, pcln)
	if err != nil {
		return nil, err
	}

	dwf, err := f.DWARF()
	if err != nil {
		return nil, err
	}

	return &SourceCodeLocator{
		st: table,
		dwf:   dwf,
	}, nil

}
```

プログラムカウンタからファイル名と行番号を取得するには、 [PCToLine](https://pkg.go.dev/debug/gosym#Table.PCToLine) メソッドを使います。本来は DWARF の Compilation Units を走査して、対応する DIE(Debugging Information Entry) を探す必要があるのですが、標準パッケージで用意されているメソッドを呼ぶだけでいいので、非常に楽です。

```go:go-debuger/debugger/source_code_locator.go
func (l *SourceCodeLocator) PCToFileLine(pc uint64) (filename string, line int) {
	fn, ln, _ := l.st.PCToLine(pc)
	return fn, ln
}
```

## 現在のソースコードの位置を出力する

次に、出力用の関数を実装するファイルを作成します。

```diff
go-debugger/
  └── debugger
     ├── breakpoint.go
     ├── debugger.go
     ├── register.go
     ├── source_code_locator.go
+    └── source_code_printer.go
```

出力用の関数の一部が以下になります。引数に渡した reader がソースコードで、 os.File などで渡されることを想定しています。インターフェースなので他の型でも問題ありません。さらに現在行を強調するために currentLine を引数として受け取ります。lineRange は currentLine から前後何行を表示するのかを指定します。

printSourceCode 関数では、 startLine 行目から endLine 行目まで出力するため、値を簡単に計算しています。

```go:go-debuger/debugger/source_code_printer.go
package debugger

import (
	"bufio"
	"fmt"
	"io"
)

// how many lines from given line number
const lineRange = 5

// printSourceCode prints out source code passed as a reader, clealy emphasize the currrent line.
func printSourceCode(reader io.Reader, currentLine int) {
	startLine := 1
	if currentLine > lineRange {
		startLine = currentLine - lineRange
	}
	endLine := currentLine + lineRange
	scanLine := 1

    // ...
}
```

続いての処理では、出力する文字列を格納するために lines 変数を用意しておきます。その後、ソースコードを bufio.Scanner として1行ずつ読み込んでいきます。
startLine までは何もしないで continue して、 endLine を超えた場合は break します。 `startLine <= scanLine <= endLine` を満たす場合は、 scanLine 行目のソースコードを lines に格納します。 scanLine が currentLine と一致する場合は強調するために `> ` を先頭に追加します。

Scan が終了したら、 lines 変数をもとにソースコードを出力していきます。

```go:go-debuger/debugger/source_code_printer.go
func printSourceCode(reader io.Reader, currentLine int) {
	// ...
	var lines []string
	scanner := bufio.NewScanner(reader)
	for scanner.Scan() {
		if scanLine < startLine {
			scanLine++
			continue
		}
		if scanLine > endLine {
			break
		}

		text := scanner.Text()
		if scanLine == currentLine {
			text = fmt.Sprintf("> %d %s", scanLine, text)
		} else {
			text = fmt.Sprintf("  %d %s", scanLine, text)
		}
		lines = append(lines, text)
		scanLine++
	}

	for _, text := range lines {
		fmt.Printf("%s\n", text)
	}
}
```

あとはこれらを使っていきましょう。まずは Debugger 構造体のメソッドを更新します。

```diff:go-debuger/debugger/debugger.go
type Debugger struct {
	config      *Config
	pid         int
	breakpoints map[uint64]*Breakpoint
+	locator     Locator
}

- func NewDebugger(config *Config) (*Debugger, error) {
+ func NewDebugger(config *Config, locator Locator) (*Debugger, error) {
	d := &Debugger{
		config:      config,
		breakpoints: make(map[uint64]*Breakpoint),
+		locator:     locator,
	}
	if err := d.Launch(); err != nil {
		return nil, err
	}

	return d, nil
}
```

ソースコードを出力するための Debugger 構造体のメソッドを作成します。プログラムカウンタからファイル名と行番号を取得し、ファイルを開いてソースコードを出力する関数を実行します。

```go:go-debuger/debugger/debugger.go
func (d *Debugger) printSourceCode() error {
	pc, err := d.getPC()
	if err != nil {
		return err
	}

	filename, line := d.locator.PCToFileLine(pc)
	f, err := os.Open(filename)
	if err != nil {
		return err
	}
	defer f.Close()

	printSourceCode(f, line)

	return nil

}
```

ブレークポイントにヒットした時に d.printSourceCode を実行するようにします。

```diff:go-debuger/debugger/debugger.go
func (d *Debugger) onBreakpointHit() error {
    ...

	fmt.Printf("hit breakpoint at 0x%x\n", previousPC)

+	if err := d.printSourceCode(); err != nil {
+		return err
+	}

	return nil
}
```

あとは Debugger 初期化時に SourceCodeLocator を渡します。

```diff:go-debuger/main.go
func main() {
	...
	defer cleanup()

+	locator, err := debugger.NewSourceCodeLocator(absDebuggeePath)
+	if err != nil {
+		fmt.Fprintf(os.Stderr, "faield to initialize source code locator: %s", err)
+		return
+	}

-	d, err := debugger.NewDebugger(&debugger.Config{
-		DebuggeePath: absDebuggeePath,
-	})
+	d, err := debugger.NewDebugger(
+		&debugger.Config{
+			DebuggeePath: absDebuggeePath,
+		},
+		locator,
+	)
	...
}
```

それでは動作確認してみます。ブレークポイントにヒットした時にソースコードが表示されるようになりました。

```bash
go run . -path ./cmd/helloworld/

# go-debugger> b 4ae618

# go-debugger> c
# hit breakpoint at 0x4ae618
#   1 package main
#   2 
#   3 import "fmt"
#   4 
#   5 func main() {
# > 6     fmt.Println("Hello, World!")
#   7 }

# go-debugger> c
# Hello, World!
# go-debugger gracefully shut down
```
