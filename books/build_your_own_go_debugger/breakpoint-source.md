---
title: "ソースコードレベルのブレークポイントの実装"
---

# ソースコードレベルのブレークポイント
前章までメモリアドレスを指定したブレークポイントを実装してきました。ここでは、関数のシンボル、またはファイル名と行数を指定してブレークポイントを設定できるようにします。

## メモリアドレスからソースコードの位置を調べる

### dwarfdump で雰囲気を掴む

まずは、ブレークポイントにヒットした時にソースコードを出力していきます。これを実現するにはメモリアドレスからソースコードの位置を特定する必要があります。
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

`/main` と入力して main.go のファイルを探してみると、以下のような出力がみつかります。ここではメモリアドレスとそれに一致するファイル名、行番号が取得できていることが分かります。したがって、ビルドしたプログラムを実行している時のプログラムカウンタが分かれば、ファイル名と行番号に変換することができます。

```
.debug_line: line number info for a single cu
Source lines (from CU-DIE at .debug_info offset 0x00002505):

            NS new statement, BB new basic block, ET end of text sequence
            PE prologue end, EB epilogue begin
            IS=val ISA number, DI=val discriminator value
<pc>        [lno,col] NS BB ET PE EB IS= DI= uri: "filepath"
0x004ae5a0  [   5, 0] NS uri: "/Users/<username>/lima/sample/cmd/helloworld/main.go"
0x004ae5aa  [   5, 0] NS PE
0x004ae5ae  [   6, 0] NS
0x004ae5b4  [   6, 0]
0x004ae61d  [   7, 0] NS
0x004ae623  [   5, 0] NS
0x004ae62d  [   5, 0] NS ET
```

ビルドしたファイルのデバッグ情報が得られることが分かったので、 Go がサポートしてる DWARF のバージョンを確認しておきます。 objdump コマンドで `--dwarf=info` を指定すると Compilation Unit の Version というフィールドで確認することができます。

以下のコマンドの出力から、 DWARF 4 を利用していることが分かります。詳細は [DWARF 標準](https://dwarfstd.org/download.html)でダウンロードできるので、気になる方はダウンロードして読んでみてください。

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

### メモリアドレスからソースコードの位置を取得する実装
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
	symbolTable *gosym.Table
	dwarfData   *dwarf.Data
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

	dwarfData, err := f.DWARF()
	if err != nil {
		return nil, err
	}

	return &SourceCodeLocator{
		symbolTable: table,
		dwarfData:   dwarfData,
	}, nil

}
```

プログラムカウンタからファイル名と行番号を取得するには、 [gosym.PCToLine](https://pkg.go.dev/debug/gosym#Table.PCToLine) を使います。本来は DWARF の Compilation Units を走査して、対応する DIE(Debugging Information Entry) を探す必要があるのですが、標準パッケージで用意されているメソッドを呼ぶだけでいいので、非常に楽です。

```go:go-debuger/debugger/source_code_locator.go
func (l *SourceCodeLocator) PCToFileLine(pc uint64) (filename string, line int) {
	fn, ln, _ := l.symbolTable.PCToLine(pc)
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

少し長くなりますが、出力用の関数は以下になります。引数に渡した reader がソースコードで、 os.File などで渡されることを想定しています。インターフェースなので他の方でも問題ありません。さらに現在行を強調するために currentLine を引数として受け取ります。lineRange は currentLine から前後何行を表示するのかを指定します。

printSourceCode 関数の最初は、 startLine 行目から endLine 行目まで出力するため、値を簡単に計算しています。

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

出力する文字列を格納するために lines 変数を用意しておきます。その後、ソースコードを bufio.Scanner として1行ずつ読み込んでいきます。
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

Debugger 構造体のメソッドを作成して、プログラムカウンタからファイル名と行番号を取得します。その後、ファイルを開いてソースコードを出力する関数を実行します。

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

	d, err := debugger.NewDebugger(&debugger.Config{
		DebuggeePath: absDebuggeePath,
-	})
+	},
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

## 関数のシンボルを指定してブレークポイントを設定する

### プロローグエンドの解説
まずは関数のプロローグエンドについて理解していきます。
プロローグとは、関数実行前に行う準備のような処理のことを指します。このプロローグが終わるアドレスをプロローグエンドと呼んでいます。
例として Hello, World! プログラムを dwarfdump した結果を再掲します。 `0x004ae5a0` が関数のエントリポイントとなっていますが、 Prologue End (PE) は `0x004ae5aa` になっています。

```
.debug_line: line number info for a single cu
Source lines (from CU-DIE at .debug_info offset 0x00002505):

            NS new statement, BB new basic block, ET end of text sequence
            PE prologue end, EB epilogue begin
            IS=val ISA number, DI=val discriminator value
<pc>        [lno,col] NS BB ET PE EB IS= DI= uri: "filepath"
0x004ae5a0  [   5, 0] NS uri: "/Users/<username>/lima/sample/cmd/helloworld/main.go"
0x004ae5aa  [   5, 0] NS PE
0x004ae5ae  [   6, 0] NS
0x004ae5b4  [   6, 0]
0x004ae61d  [   7, 0] NS
0x004ae623  [   5, 0] NS
0x004ae62d  [   5, 0] NS ET
```

さらに objdump した結果も見てみます。関数のエントリポイント `4ae5a0` では rsp レジスタの値をチェックして、条件を満たした場合は `4ae5a4` の命令で `4ae623` に jump しています。ここでスタックに十分なメモリ容量が確保されているかをチェックしています。その後、 `4ae628` の命令で関数のエントリポイントに戻り、 rsp レジスタが条件を満たさなくなったら rbp, rsp のレジスタを操作します。この一連の処理がプロローグとなり、`4ae5aa` がプロローグエンドになります。

```
00000000004ae5a0 <main.main>:
  4ae5a0:	49 3b 66 10          	cmp    rsp,QWORD PTR [r14+0x10]
  4ae5a4:	76 7d                	jbe    4ae623 <main.main+0x83>
  4ae5a6:	55                   	push   rbp
  4ae5a7:	48 89 e5             	mov    rbp,rsp
  4ae5aa:	48 83 ec 48          	sub    rsp,0x48
  ...
  4ae623:	e8 18 3e fc ff       	call   472440 <runtime.morestack_noctxt.abi0>
  4ae628:	e9 73 ff ff ff       	jmp    4ae5a0 <main.main>
```

以上から、関数のエントリポイント `4ae5a0` にブレークポイントを設定した場合、 Continue するとプロローグの処理の中でもう一度 `4ae5a0` にヒットしてしまいます。これを避けるために、関数名を指定してブレークポイントを設定するときはプロローグエンドに指定するのが望ましいです。 

delve はこのあたりのことを考慮して実装されています。delve のケースを詳しく知りたい方は GopherCon EU 2018 で Alessandro Arzill 氏が発表した [Internal Architecture of Delve, a Debugger For Go](https://www.youtube.com/watch?v=IKnTr7Zms1k&t=1568s) を参考にしてみてください。

### プロローグエンドを取得する実装

それでは関数のシンボルを指定してブレークポイントを設定できるようにしていきます。まずは、プロローグエンドのアドレスを取得する処理を実装していきます。
dwarf.Data を走査して Compilation Unit のエントリを探します。 Compilation Unit が見つかったら、関数のエントリポイントが一致する LineEntry を探して、 Prologue End に一致したらそのアドレスを返します。


```go:go-debuger/debugger/source_code_locator.go
func (l *SourceCodeLocator) getPrologueEndAddress(fn *gosym.Func) (uint64, error) {
	reader := l.dwarfData.Reader()
	for {
		entry, err := reader.Next()
		if err != nil {
			break
		}

		if entry.Tag != dwarf.TagCompileUnit {
			continue
		}

		lineReader, err := l.dwarfData.LineReader(entry)
		if err != nil {
			return 0, err
		}

		var lineEntry dwarf.LineEntry
		for lineReader.Next(&lineEntry) == nil {
			if lineEntry.Address == fn.Entry {
				for lineReader.Next(&lineEntry) == nil {
					if lineEntry.PrologueEnd {
						return lineEntry.Address, nil
					}
				}
			}
		}
	}

	return 0, fmt.Errorf("faield to get prologue end address for function %s", fn.Name)
}
```

この処理を dwarfdump したデータと比べながらみていきます。 `dwarfdump helloworld.o | less` を実行した後に `/4ae5a0` と入力して検索すると、以下のような出力が得られます。 `DW_TAG_compile_unit` がタグで、 `DW_AT_low_pc` が関数のエントリポイントに一致します。他の行をみてみると `DW_TAG_subprogram` などのタグがあることが分かります。 `entry.Tag != dwarf.TagCompileUnit` の処理では、エントリのタグが `DW_TAG_compile_unit` であるかどうかを確認しています。

```
COMPILE_UNIT<header overall offset = 0x000024fa>:
< 0><0x0000000b>  DW_TAG_compile_unit
                    DW_AT_name                  main
                    DW_AT_language              DW_LANG_Go
                    DW_AT_stmt_list             0x00000b7c
                    DW_AT_low_pc                0x004ae5a0
                    DW_AT_ranges                0x000002e0
  ranges: 2 at .debug_ranges offset 736 (0x000002e0) (32 bytes)
   [ 0] range entry    0x00000000 0x0000008d
   [ 1] range end      0x00000000 0x00000000
                    DW_AT_comp_dir              .
                    DW_AT_producer              Go cmd/compile go1.23.2; -N -l regabi
                    <Unknown AT value 0x2905>   main

LOCAL_SYMBOLS:
< 1><0x0000004f>    DW_TAG_subprogram
                      DW_AT_name                  main.main
                      DW_AT_low_pc                0x004ae5a0
                      DW_AT_high_pc               0x004ae62d
```

dwarf.LineEntry は `.debug_line` セクションの各行に対応しています。 `DW_TAG_compile_unit` タグのエントリが見つかったら関数のエントリポイントと一致する LineEntry を探し（`if lineEntry.Address == fn.Entry`）、一致した場合は Prologue End の行を探す（`if lineEntry.PrologueEnd`）、といった処理になります。関数のエントリポイントと一致する LineEntry が見つからなかった場合は違う関数などの Compilation Unit なので、無視して次の Compilation Unit を探します。

```
.debug_line: line number info for a single cu
Source lines (from CU-DIE at .debug_info offset 0x00002505):

            NS new statement, BB new basic block, ET end of text sequence
            PE prologue end, EB epilogue begin
            IS=val ISA number, DI=val discriminator value
<pc>        [lno,col] NS BB ET PE EB IS= DI= uri: "filepath"
0x004ae5a0  [   5, 0] NS uri: "/Users/kyota/lima/sample/cmd/helloworld/main.go"
0x004ae5aa  [   5, 0] NS PE
0x004ae5ae  [   6, 0] NS
0x004ae5b4  [   6, 0]
0x004ae61d  [   7, 0] NS
0x004ae623  [   5, 0] NS
0x004ae62d  [   5, 0] NS ET
```

### 関数のシンボルからプロローグのアドレスに変換する
プロローグエンドのアドレスが取得できるようになったら、関数のシンボルからアドレスに変換する処理を実装します。
gosym.Table の LookupFunc メソッドは関数のシンボルから、関数のエントリーポイントのアドレスなどをもった gosym.Func 構造体を生成します。これを先ほど実装した getPrologueEndAddress に渡してプロローグエンドのアドレスを取得します。

```go:go-debuger/debugger/source_code_locator.go
func (l *SourceCodeLocator) FuncToAddr(funcSymbol string) (uint64, error) {
	fn := l.symbolTable.LookupFunc(funcSymbol)
	if fn == nil {
		return 0, fmt.Errorf("failed to find function: %s", funcSymbol)
	}

	peAddr, err := l.getPrologueEndAddress(fn)
	if err != nil {
		return 0, err
	}

	return peAddr, nil
}
```

### break コマンドの更新
あとは break コマンドで関数のシンボル名を受け取れるようにしていきます。まずは引数の型を定義し、アドレス以外の値も受け取るようにします。

```go:go-debuger/debugger/debugger.go
type SetBreakpointArgs struct {
	Addr uint64
	// FunctionSymbol is <package name>.<function name> like main.main
	FunctionSymbol string
}
```

定義した引数の型を使用して、関数のシンボルが渡されたときはアドレスに変換する処理を入れます。

```diff:go-debuger/debugger/debugger.go
- func (d *Debugger) SetBreakpoint(addr uint64) error {
+ func (d *Debugger) SetBreakpoint(args SetBreakpointArgs) error {
+	if args.Addr == 0 && args.FunctionSymbol == "" {
+		return fmt.Errorf("address or function symbol must be given, but both are empty")
+	}
+
+	var addr uint64
+	var err error
+	if args.Addr != 0 {
+		addr = args.Addr
+	}
+	if args.FunctionSymbol != "" {
+		addr, err = d.locator.FuncToAddr(args.FunctionSymbol)
+		if err != nil {
+			return err
+		}
+	}
+
	bp, err := NewBreakpoint(d.pid, uintptr(addr))
	...
}
```

command の処理も更新します。

```diff:go-debuger/terminal/command.go
func setBreakpoint(dbg *debugger.Debugger, args []string) error {
	...
	if err != nil {
-		return errors.New("breakpoint address must be parsed as uint64")
+		return dbg.SetBreakpoint(debugger.SetBreakpointArgs{FunctionSymbol: args[0]})
	}

-	return dbg.SetBreakpoint(addr)
+	return dbg.SetBreakpoint(debugger.SetBreakpointArgs{Addr: addr})
}
```

それでは動作確認してみましょう。関数のシンボルを指定（`b main.main`）してブレークポイントが設定できました。

```bash
go run . -path ./cmd/helloworld/

# go-debugger> b main.main

# go-debugger> c
# hit breakpoint at 0x4ae5aa
#   1 package main
#   2 
#   3 import "fmt"
#   4 
# > 5 func main() {
#   6     fmt.Println("Hello, World!")
#   7 }

# go-debugger> c
# Hello, World!
# go-debugger gracefully shut down
```

## ファイル名と行番号を指定してブレークポイントを設定する
最後にファイル名と行番号を指定してブレークポイントを設定します。