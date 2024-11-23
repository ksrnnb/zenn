---
title: "ソースコードレベルのブレークポイントの実装"
---

# ソースコードレベルのブレークポイント
今まではメモリアドレスを指定したブレークポイントを実装してきました。ここでは、関数のシンボル、またはファイル名と行数を指定してブレークポイントを設定できるようにします。

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
	reader := l.dwf.Reader()
	for {
		entry, err := reader.Next()
		if err != nil {
			break
		}

		if entry.Tag != dwarf.TagCompileUnit {
			continue
		}

		lineReader, err := l.dwf.LineReader(entry)
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
0x004ae5a0  [   5, 0] NS uri: "/Users/<username>/lima/sample/cmd/helloworld/main.go"
0x004ae5aa  [   5, 0] NS PE
0x004ae5ae  [   6, 0] NS
0x004ae5b4  [   6, 0]
0x004ae61d  [   7, 0] NS
0x004ae623  [   5, 0] NS
0x004ae62d  [   5, 0] NS ET
```

### 関数のシンボルからプロローグのアドレスに変換する
プロローグエンドのアドレスが取得できるようになったら、関数のシンボルからアドレスに変換する処理を実装します。
gosym.Table の LookupFunc メソッドは関数のシンボルから、関数のエントリポイントのアドレスなどをもった gosym.Func 構造体を生成します。これを先ほど実装した getPrologueEndAddress に渡してプロローグエンドのアドレスを取得します。

```go:go-debuger/debugger/source_code_locator.go
func (l *SourceCodeLocator) FuncToAddr(funcSymbol string) (uint64, error) {
	fn := l.st.LookupFunc(funcSymbol)
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
+	var addr uint64
+	var err error
+	if args.Addr != 0 {
+		addr = args.Addr
+	}
+	if args.FunctionSymbol != "" {
+		addr, err = d.locator.FuncToAddr(args.FunctionSymbol)
+		if err != nil {
+			return fmt.Errorf("failed to find symbol %s: %w", args.FunctionSymbol, err)
+		}
+	}
+
+	if addr == 0 {
+		return fmt.Errorf("failed to get breakpoint address. args: %+v", args)
+	}

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
続けて、ファイル名と行番号を指定してブレークポイントを設定できるようにしていきます。

### ファイル名と行番号からアドレスに変換する
ファイル名と行番号からプログラムカウンタに変換する [LineToPC](https://pkg.go.dev/debug/gosym#Table.LineToPC) メソッドが用意されているので、それを利用します。得られたアドレスが関数のエントリポイントだった場合は、ブレークポイントに複数回ヒットすることを防ぐためにプロローグエンドのアドレスを返します。

```go:go-debuger/debugger/source_code_locator.go
func (l *SourceCodeLocator) FileLineToAddr(filename string, line int) (uint64, error) {
	addr, fn, err := l.st.LineToPC(filename, line)
	if err != nil {
		return 0, fmt.Errorf("failed to get addr by filename %s and line %d: %s", filename, line, err)
	}

	if addr == fn.Entry {
		return l.getPrologueEndAddress(fn)
	}

	return addr, nil
}
```

### break コマンドの更新

続いて break コマンドを更新していきます。まずは引数としてファイル名、行番号を受け取れるようにします。

```diff:go-debuger/debugger/debugger.go
type SetBreakpointArgs struct {
	Addr uint64
	// FunctionSymbol is <package name>.<function name> like main.main
	FunctionSymbol string
+	Filename       string
+	Line           int
}
```

ファイル名と行番号が指定された場合は、先ほど定義した FileLineToAddr メソッドを実行してアドレスに変換します。このアドレスを使用してブレークポイントを設定します。

```diff:go-debuger/debugger/debugger.go
func (d *Debugger) SetBreakpoint(args SetBreakpointArgs) error {
	...
	if args.FunctionSymbol != "" {
		...
	}
+	if args.Filename != "" && args.Line != 0 {
+		addr, err = d.locator.FileLineToAddr(args.Filename, args.Line)
+		if err != nil {
+			return fmt.Errorf("failed to find file %s and line %d: %w", args.Filename, args.Line, err)
+		}
+	}
	...
}
```

break コマンドの関数を更新し、ファイル名と行番号を break コマンドの引数として渡せるようにします。

```diff:go-debuger/terminal/command.go
+// setBreakpoint set breakpoint at given address, function or filename and line.
+// address:           break 0xaaaa
+// function:          break main.main
+// filename and line: break /path/to/file 20
func setBreakpoint(dbg *debugger.Debugger, args []string) error {
	...
	if err != nil {
-		return dbg.SetBreakpoint(debugger.SetBreakpointArgs{FunctionSymbol: args[0]})
+		if len(args) == 1 {
+			return dbg.SetBreakpoint(debugger.SetBreakpointArgs{FunctionSymbol: args[0]})
+		} else if len(args) == 2 {
+			line, err := strconv.Atoi(args[1])
+			if err != nil {
+				return fmt.Errorf("failed to parse line number: %w", err)
+			}
+
+			return dbg.SetBreakpoint(debugger.SetBreakpointArgs{
+				Filename: args[0],
+				Line:     line,
+			})
+		} else {
+			return errors.New("length of args must be less than or equal to 2")
+		}
	}
	...
}
```

実装が完了したので、動作を確認してみましょう。
5行目の関数のエントリポイントにブレークポイントを設定した結果が以下になります。意図した位置にブレークポイントが設定できていることが分かります。ファイル名は絶対パスを渡す必要があるので注意してください。

```bash
go run . -path ./cmd/helloworld/

go-debugger> b /Users/<username>/lima/sample/cmd/helloworld/main.go 5

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