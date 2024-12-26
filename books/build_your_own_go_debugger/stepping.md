---
title: "ステップ処理の実装"
---

# ステップ処理の実装
本章では、関数内部の処理に移行する step in コマンドや、1行ずつ処理を進める next コマンドなどの実装を進めていきます。

## step in コマンドの実装
step in コマンドは、現在の行に未評価の関数がある場合はその関数内部に移行し、未評価の関数がない場合は次の行に進むような処理となります。

まずは debugger を実装していきます。はじめに、現在のプログラムカウンタを取得し、ファイル名と行番号を取得しておきます。
for ループでファイル名または行番号が変わるまで命令を実行します。ブレークポイントが存在する場合は `stepOverBreakpointIfNeeded` を実行し、存在しない場合は `PtraceSingleStep` を実行して、1命令だけ実行します。

```go:go-debugger/debugger/debugger.go
func (d *Debugger) StepIn() error {
	pc, err := d.getPC()
	if err != nil {
		return err
	}

	initialFilename, initialLine := d.locator.PCToFileLine(pc)

	for {
		pc, err = d.getPC()
		if err != nil {
			return err
		}

		f, l := d.locator.PCToFileLine(pc)
		if f != initialFilename || l != initialLine {
			return d.printSourceCode()
		}

		if _, ok := d.breakpoints[pc]; ok {
			// step over if breakpoint exist
			if err := d.stepOverBreakpointIfNeeded(); err != nil {
				return err
			}
		} else {
			// single step if breakpoint does not exist
			if err := syscall.PtraceSingleStep(d.pid); err != nil {
				return err
			}
			_, err := d.wait()
			if err != nil {
				return err
			}
		}
		// ...
	}
}
```

しかし、この実装だと他の関数の処理に移ったときに、プロローグエンドの手前で処理が停止してしまいます。そのため、命令実行後のプログラムカウンタが関数のエントリポイントだった場合、プロローグエンドまで命令を実行しておきます。

実装は以下のようになります。 for ループの中で命令を実行した後のプログラムカウンタが関数のエントリポイントであるかどうかを確認します。エントリポイントだった場合はその関数のプロローグエンドにブレークポイントを設定してから Continue することで、プロローグエンドまで処理を進めます。

```go:go-debugger/debugger/debugger.go
func (d *Debugger) StepIn() error {
	// ...
	for {
		// ...
		// Get PC again because PC has been changed by step over or single step.
		pc, err = d.getPC()
		if err != nil {
			return err
		}

		// set breakpoint at prologue end and execute Continue
		// if pc is entry point of a function.
		if d.locator.IsFunctionEntrypoint(pc) {
			prologueEnd, err := d.locator.GetPrologueEndAddressByPC(pc)
			if err != nil {
				return err
			}

			var bp *Breakpoint
			if _, ok := d.breakpoints[prologueEnd]; !ok {
				bp, err = NewBreakpoint(d.pid, uintptr(prologueEnd))
				if err != nil {
					return err
				}
			}

			if err := d.Continue(); err != nil {
				return err
			}

			if bp != nil {
				if err := bp.Disable(); err != nil {
					return err
				}
			}

			return nil
		}
	}
}
```

続けて、 Locator のメソッドを定義します。プログラムカウンタから、現在の関数のプロローグエンドを取得する `GetPrologueEndAddressByPC` と、現在のプログラムカウンタが関数のエントリポイントと一致するかどうかを確かめる `IsFunctionEntrypoint` を追加します。

```diff:debugger/source_code_locator.go
type Locator interface {
    ...
	FileLineToAddr(filename string, line int) (uint64, error)
+	GetPrologueEndAddressByPC(pc uint64) (uint64, error)
+	IsFunctionEntrypoint(pc uint64) bool
}
```

`GetPrologueEndAddressByPC` の実装は以下のようになります。すでに `*gosym.Func` からプロローグエンドを取得するメソッドが実装されているので、それを利用する形になります。

```go:debugger/source_code_locator.go
func (l *SourceCodeLocator) GetPrologueEndAddressByPC(pc uint64) (uint64, error) {
	_, _, fn := l.st.PCToLine(pc)
	if fn == nil {
		return 0, fmt.Errorf("faield to get prologue end address of 0x%x", pc)
	}

	return l.getPrologueEndAddress(fn)
}
```

`IsFunctionEntrypoint` の実装は以下のようになります。

```go:debugger/source_code_locator.go
func (l *SourceCodeLocator) IsFunctionEntrypoint(pc uint64) bool {
	_, _, fn := l.st.PCToLine(pc)
	if fn == nil {
		return false
	}

	if fn.Entry == pc {
		return true
	}

	return false
}
```

最後に step in のコマンドを追加します。

```diff:terminal/command.go
func NewCommands() *Commands {
	return &Commands{
		cmds: []command{
            ...
+			{
+				aliases: []string{"stepin", "si"},
+				cmdFn:   stepIn,
+			},
		},
	}
}

+func stepIn(dbg *debugger.Debugger, args []string) error {
+	return dbg.StepIn()
+}
```
## step in 動作確認
step in コマンドの動作確認を行うにあたって、もう少し処理を複雑にしたいので、ファイルを追加します。

```diff
go-debugger/
  └── cmd
+     └── function
+         ├── function.go
+         └── main.go
```

それぞれのファイルの実装は以下のようになります。ステップ処理を確認しやすいように適当に関数を定義しておきます。

```go:cmd/function/function.go
package main

func fn1() int {
	a := 1
	b := 2
	return fn2(a, b)
}

func fn2(a, b int) int {
	a += 1
	b += 1
	return fn3(a, b)
}

func fn3(a, b int) int {
	return a * b
}
```

```go:cmd/function/main.go
package main

import "fmt"

func main() {
	result := fn1()

	fmt.Printf("result: %d\n", result)
}
```

それでは、動作確認してみます。一度適当なところにブレークポイントを設定して Continue した後に、 step in を何度か実行した結果を以下に示します。

```bash
go run . -path ./cmd/function/

# go-debugger> b main.main
# go-debugger> c
# 
# hit breakpoint at 0x4b02ae
#   1 package main
#   2 
#   3 import "fmt"
#   4 
# > 5 func main() {
#   6     result := fn1()
#   7 
#   8     fmt.Printf("result: %d\n", result)
#   9 }
# 
# go-debugger> si
#   1 package main
#   2 
#   3 import "fmt"
#   4 
#   5 func main() {
# > 6     result := fn1()
#   7 
#   8     fmt.Printf("result: %d\n", result)
#   9 }
# 
# go-debugger> si
# hit breakpoint at 0x4b018a
#   1 package main
#   2 
# > 3 func fn1() int {
#   4     a := 1
#   5     b := 2
#   6     return fn2(a, b)
#   7 }
#   8 
# 
# go-debugger> si
#   1 package main
#   2 
#   3 func fn1() int {
# > 4     a := 1
#   5     b := 2
#   6     return fn2(a, b)
#   7 }
#   8 
#   9 func fn2(a, b int) int {
```

## step out コマンドの実装
step in が実装できたので、次は step out を実装していきます。 step out は現在の関数の呼び出し元に戻る操作になります。

まずは任意のメモリアドレスに格納されている値を読み取る処理を書くためにファイルを追加します。

```diff
go-debugger/
  └── debugger
+     └── memory.go
```

実装は以下のようになります。 PtracePeekData で任意のアドレスの値のデータを読み取ります。

```go:debugger/memory.go
package debugger

import (
	"encoding/binary"
	"fmt"
	"syscall"
)

func readMemory(pid int, addr uint64) (uint64, error) {
	// data is 8 byte to store uint64 value
	data := make([]byte, 8)
	_, err := syscall.PtracePeekData(pid, uintptr(addr), data)
	if err != nil {
		return 0, fmt.Errorf("failed to readMemory: %s", err)
	}

	return binary.LittleEndian.Uint64(data), nil
}
```

readMemory 関数を使って、呼び出し元のアドレスを取得する関数を定義します。
呼び出し元の関数のアドレスは、 `RBP レジスタの値 + 8` となります。これは [System V Application Binary Interface](https://gitlab.com/x86-psABIs/x86-64-ABI) の Figure 3.3 に記載があるので、興味のある方は参照してみてください。

```go:debugger/debugger.go
func (d *Debugger) getCallerAddress() (uint64, error) {
	rbp, err := readRegister(d.pid, Rbp)
	if err != nil {
		return 0, err
	}

	// rbp + 8 means caller address.
	// see the section 3.2.2 The Stack Frame of https://gitlab.com/x86-psABIs/x86-64-ABI
	return readMemory(d.pid, rbp+8)
}
```

step out の実装は以下のようになります。呼び出し元のアドレスにブレークポイントを設定して Continue することで、呼び出し元の関数に戻ります。 Continue した後は必要に応じてブレークポイントを削除します。

```go:debugger/debugger.go
func (d *Debugger) StepOut() error {
	callerAddress, err := d.getCallerAddress()
	if err != nil {
		return err
	}

	_, ok := d.breakpoints[callerAddress]
	if !ok {
		if err := d.SetBreakpoint(SetBreakpointArgs{Addr: callerAddress}); err != nil {
			return err
		}
	}

	if err := d.Continue(); err != nil {
		return err
	}

	if !ok {
		if err := d.removeBreakpoint(callerAddress); err != nil {
			return err
		}
	}

	return nil
}
```

ブレークポイントを削除する関数の実装は以下になります。 Breakpoint の map にブレークポイントが存在していたら Disable を実行して元の命令に戻してから、 map から削除します。

```go:debugger/debugger.go
func (d *Debugger) removeBreakpoint(addr uint64) error {
	bp, ok := d.breakpoints[addr]
	if !ok {
		return nil
	}

	if err := bp.Disable(); err != nil {
		return err
	}

	delete(d.breakpoints, addr)

	return nil
}
```

最後に step out のコマンドを追加します。

```diff:terminal/command.go
func NewCommands() *Commands {
	return &Commands{
		cmds: []command{
            ...
+			{
+				aliases: []string{"stepout", "so"},
+				cmdFn:   stepOut,
+			},
		},
	}
}

+func stepOut(dbg *debugger.Debugger, args []string) error {
+	return dbg.StepOut()
+}
```

## step out の動作確認

動作確認すると、以下のようになります。適当な位置にブレークポイントを設定して Continue し、 fn1 関数の中に入るために何度か step in を実行します。その後 step out を実行して呼び出し元に戻っていることが分かります。

```bash
go run . -path ./cmd/function/
# go-debugger> b main.main
# 
# go-debugger> c
# hit breakpoint at 0x4b02ae
#   1 package main
#   2 
#   3 import "fmt"
#   4 
# > 5 func main() {
#   6     result := fn1()
#   7 
#   8     fmt.Printf("result: %d\n", result)
#   9 }
# 
# go-debugger> si
#   1 package main
#   2 
#   3 import "fmt"
#   4 
#   5 func main() {
# > 6     result := fn1()
#   7 
#   8     fmt.Printf("result: %d\n", result)
#   9 }
# 
# go-debugger> si
# hit breakpoint at 0x4b018a
#   1 package main
#   2 
# > 3 func fn1() int {
#   4     a := 1
#   5     b := 2
#   6     return fn2(a, b)
#   7 }
#   8 
# 
# go-debugger> so
# hit breakpoint at 0x4b02b7
#   1 package main
#   2 
#   3 import "fmt"
#   4 
#   5 func main() {
# > 6     result := fn1()
#   7 
#   8     fmt.Printf("result: %d\n", result)
#   9 }
```

## next コマンドの実装

最後に next コマンドを実行していきます。ここでは next と呼んでいますが step over とも呼ばれます。このコマンドは1行だけ次の行に処理を進めます。
ここではシンプルな手法で next コマンドを実装します。具体的には、現在の関数のすべての行と、呼び出し元のアドレスにブレークポイントを設定して Continue することで1行だけ処理を進めます。

### GetFunction メソッドの実装 
まずは関数の情報を取得する GetFunction メソッドを実装していきます。 Function 構造体は関数名と、関数の開始アドレスと終了アドレス、そしてそれぞれの行番号をもちます。

```diff:debugger/source_code_locator.go
type Locator interface {
	...
+	GetFunction(pc uint64) (Function, error)
}

+type Function struct {
+	Name      string
+	Entry     uint64
+	End       uint64
+	EntryLine int
+	EndLine   int
+}
```

関数のすべての行にブレークポイントを設定するために、現在の関数が開始するアドレスと終了するアドレスを取得する必要があります。そのためにまずはタグが DW_TAG_subprogram となるエントリを探します。 DW_TAG_subprogram タグを持つエントリの DW_AT_low_pc と DW_AT_high_pc 属性がそれぞれ関数の開始アドレス、終了アドレスとなります。引数で渡した pc がそれぞれの間に収まる値の場合は、現在の関数の開始アドレスと終了アドレスになります。


```go:debugger/source_code_locator.go
func (l *SourceCodeLocator) getLowPCAndHighPC(pc uint64) (lowPC uint64, highPC uint64, err error) {
	reader := l.dwf.Reader()
	for {
		entry, err := reader.Next()
		if err != nil {
			break
		}

		if entry == nil {
			break
		}

		if entry.Tag != dwarf.TagSubprogram {
			continue
		}

		lowPC, _ = entry.Val(dwarf.AttrLowpc).(uint64)
		highPC, _ = entry.Val(dwarf.AttrHighpc).(uint64)

		if pc >= lowPC && pc <= highPC {
			return lowPC, highPC, nil
		}
	}

	return 0, 0, nil
}
```

つぎに関数の開始行番号、終了行番号を取得して　Function 型の構造体を返す GetFunction メソッドを実装します。
まずはじめに必要な情報を取得しておきます。

```go:debugger/source_code_locator.go
func (l *SourceCodeLocator) GetFunction(pc uint64) (Function, error) {
	filename, _, fnc := l.st.PCToLine(pc)
	reader := l.dwf.Reader()

	lowPC, highPC, err := l.getLowPCAndHighPC(pc)
	if err != nil {
		return Function{}, err
	}
	// ...
}
```

続けて、 DW_TAG_compile_unit タグをもつエントリを探して、行番号に関する情報を取得するために LineReader を生成します。

```go:debugger/source_code_locator.go
func (l *SourceCodeLocator) GetFunction(pc uint64) (Function, error) {
	// ...
	for {
		entry, err := reader.Next()
		if err != nil {
			break
		}

		if entry == nil {
			break
		}

		if entry.Tag != dwarf.TagCompileUnit {
			continue
		}

		lineReader, err := l.dwf.LineReader(entry)
		if err != nil {
			return Function{}, err
		}
		// ...
	}
}
```

その後、 LineEntry を走査していき、関数の開始行と終了行を取得します。

```go:debugger/source_code_locator.go
func (l *SourceCodeLocator) GetFunction(pc uint64) (Function, error) {
	// ...
	for {
		// ...
		var lineEntry dwarf.LineEntry

		entryLine := math.MaxInt
		endLine := math.MinInt

		for {
			if err := lineReader.Next(&lineEntry); err != nil {
				break
			}

			if lineEntry.File.Name != filename {
				continue
			}

			if !lineEntry.IsStmt {
				continue
			}

			if lineEntry.Address >= lowPC && lineEntry.Address <= highPC {
				if entryLine > lineEntry.Line {
					entryLine = lineEntry.Line
				}

				if endLine < lineEntry.Line {
					endLine = lineEntry.Line
				}
			}
		}
```

関数の開始行と終了行が取得できたら Function 構造体として返します。

```go:debugger/source_code_locator.go
func (l *SourceCodeLocator) GetFunction(pc uint64) (Function, error) {
	// ...
	for {
		// ...
		if entryLine != math.MaxInt && endLine != math.MinInt {
			return Function{
				Name:      fnc.Name,
				Entry:     lowPC,
				End:       highPC,
				EntryLine: entryLine,
				EndLine:   endLine,
			}, nil
		}
	}

	return Function{}, fmt.Errorf("failed to get function of pc %0x", pc)
}
```

### Next メソッドの実装

Debugger の Next メソッドを実装していきます。まずは先ほど実装した GetFunction を利用して現在の関数の情報を取得します。

```go:debugger/debugger.go
func (d *Debugger) Next() error {
	pc, err := d.getPC()
	if err != nil {
		return err
	}

	fnc, err := d.locator.GetFunction(pc)
	if err != nil {
		return err
	}
	// ...
}
```

つづけて現在の関数の行全て（現在行を除く）にブレークポイントを設定します。またブレークポイントは一時的ですぐに削除するので、削除するときのためにブレークポイントを設定したアドレスを保持しておきます。

```go:debugger/debugger.go
func (d *Debugger) Next() error {
	// ...
	filename, currentLine := d.locator.PCToFileLine(pc)
	var deletingBreakpointAddresses []uint64
	for l := fnc.EntryLine; l <= fnc.EndLine; l++ {
		if l == currentLine {
			continue
		}

		addr, err := d.locator.FileLineToAddr(filename, l)
		if err != nil {
			// ignore error because empty line
			continue
		}

		_, ok := d.breakpoints[addr]
		if ok {
			continue
		}

		if err := d.SetBreakpoint(SetBreakpointArgs{Addr: addr}); err != nil {
			return err
		}

		deletingBreakpointAddresses = append(deletingBreakpointAddresses, addr)
	}
	// ...
}
```

さらに現在の関数の呼び出し元にもブレークポイントを設定します。その後 Continue を実行することで1行処理が進みます。最後に一時的に設定したブレークポイントを全て削除します。

```go:debugger/debugger.go
func (d *Debugger) Next() error {
	// ...
	callerAddress, err := d.getCallerAddress()
	if err != nil {
		return err
	}

	_, ok := d.breakpoints[callerAddress]
	if !ok {
		if err := d.SetBreakpoint(SetBreakpointArgs{Addr: callerAddress}); err != nil {
			return err
		}

		deletingBreakpointAddresses = append(deletingBreakpointAddresses, callerAddress)
	}

	if err := d.Continue(); err != nil {
		return err
	}

	for _, addr := range deletingBreakpointAddresses {
		if err := d.removeBreakpoint(addr); err != nil {
			return fmt.Errorf("failed to disable breakpoint in next command: %w", err)
		}
	}

	return nil
}
```

最後にコマンドを追加しておきます。

```diff:terminal/command.go
func NewCommands() *Commands {
	return &Commands{
		cmds: []command{
			...
+			{
+				aliases: []string{"next", "n"},
+				cmdFn:   next,
+			},
		},
	}
}

+func next(dbg *debugger.Debugger, args []string) error {
+	return dbg.Next()
+}
```

それでは、動作確認してみましょう。適当な行にブレークポイントを設定してから next コマンドを実行すると、1行ずつ処理が進んでいることが分かります。

```bash
go run . -path ./cmd/function/
# go-debugger> b main.main
# 
# go-debugger> c
# hit breakpoint at 0x4b02ae
#   1 package main
#   2 
#   3 import "fmt"
#   4 
# > 5 func main() {
#   6     result := fn1()
#   7 
#   8     fmt.Printf("result: %d\n", result)
#   9 }
# 
# go-debugger> n
# hit breakpoint at 0x4b02b2
#   1 package main
#   2 
#   3 import "fmt"
#   4 
#   5 func main() {
# > 6     result := fn1()
#   7 
#   8     fmt.Printf("result: %d\n", result)
#   9 }
# 
# go-debugger> n
# hit breakpoint at 0x4b02bc
#   3 import "fmt"
#   4 
#   5 func main() {
#   6     result := fn1()
#   7 
# > 8     fmt.Printf("result: %d\n", result)
#   9 }
# 
# go-debugger> q
# go-debugger gracefully shut down
```