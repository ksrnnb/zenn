---
title: "アドレス指定によるブレークポイントの実装（後編）"
---

# ブレークポイントヒット後に元の命令に戻す
ブレークポイントにヒットしたあとに処理を継続する場合、元々実行する予定だった命令に戻す必要があります。しかし、単純に同じアドレスに元の値を入れるだけではダメで、 INT3 命令を実行した分、 RIP レジスタ（プログラムカウンタ）の値も元に戻す必要があります。そこで、レジスタの値を読み書きする処理を実装していきます。

## レジスタの読み書き

レジスタの読み書きを行う処理は register.go にまとめていきます。
```diff
go-debugger/
  └── debugger
     ├── breakpoint.go
     ├── debugger.go
+    └── register.go
```

レジスタの値を読み取る処理は以下のようになります。 syscall.PtraceRegs 構造体に値を読み取り、引数で渡した register に一致するフィールドの値を reflect パッケージを利用して取得します。今回は Rbp, Rip, Rsp の3つのレジスタのみ利用します。

```go:go-debugger/debugger/register.go
package debugger

import (
	"fmt"
	"reflect"
	"syscall"
)

type Register string

const (
	Rbp Register = "Rbp" // base pointer
	Rip Register = "Rip" // instruction pointer (= program counter)
	Rsp Register = "Rsp" // stack pointer
)

func readRegister(pid int, register Register) (uint64, error) {
	regs := &syscall.PtraceRegs{}
	if err := syscall.PtraceGetRegs(pid, regs); err != nil {
		return 0, fmt.Errorf("failed to get registers of pid %d: %w", pid, err)
	}

	v := reflect.ValueOf(regs).Elem()
	field := v.FieldByName(string(register))
	if !field.IsValid() {
		return 0, fmt.Errorf("no '%s' field in sys.PtraceRegs", register)
	}
	if field.Kind() != reflect.Uint64 {
		return 0, fmt.Errorf("field %s is not of type uint64", register)
	}

	return field.Uint(), nil
}
```

値の書き込みは以下のようになります。一度レジスタの値を読み取った後、引数で指定した register のみ更新して、 syscall.PtraceSetRegs で更新します。

```go:go-debugger/debugger/register.go
func writeRegister(pid int, register Register, value uint64) error {
	regs := &syscall.PtraceRegs{}
	if err := syscall.PtraceGetRegs(pid, regs); err != nil {
		return err
	}

	v := reflect.ValueOf(regs).Elem()
	field := v.FieldByName(string(register))
	if !field.IsValid() {
		return fmt.Errorf("no '%s' field in syscall.PtraceRegs", register)
	}
	if field.Kind() != reflect.Uint64 {
		return fmt.Errorf("field %s is not of type uint64", register)
	}
	if !field.CanSet() {
		return fmt.Errorf("field %s cannot set", register)
	}
	field.SetUint(value)

	return syscall.PtraceSetRegs(pid, regs)
}
```

## ブレークポイントの無効化

ブレークポイントを一時的に無効にする必要があるので、 Disable メソッドを実装します。このメソッドでは、 INT3 命令で上書きした命令を元の命令に戻します。元の命令に戻す際に、先頭の1バイトのみ変更するようにします。
先頭の1バイトのみ変更する理由は、意図せずブレークポイントを削除することを防ぐためです。 PtracePeekData/PtracePokeData でやりとりするデータのサイズは8バイトですが、1命令のサイズは3バイトだったり5バイトだったりとバラバラになります。そのため、元の命令のデータを8バイトそのまま書き込もうとすると、ブレークポイントを設定していたのにそれも元の命令に戻してしまう、といったことになってしまいます。コードのコメントに具体例を載せてあるので、適宜参照してください。

```go:go-debugger/debuger/breakpoint.go
// Disable updates the instruction at the address of the breakpoint to the original instruction
// before overwriting it with the INT3 instruction
func (bp *Breakpoint) Disable() error {
	buf := make([]byte, 8)
	_, err := syscall.PtracePeekData(bp.pid, bp.addr, buf)
	if err != nil {
		return err
	}

	data := binary.LittleEndian.Uint64(buf)
	originalData := binary.LittleEndian.Uint64(bp.originalInstruction)
	// newData replaces only the first byte of data with originalData
	// example)
	//   data:         0xfffec9cc68ec83cc
	//   originalData: 0xfffec9e868ec8348
	//   newData:      0xfffec9cc68ec8348
	newData := (data & ^uint64(0xff)) | (originalData & 0xff)
	newInstruction := make([]byte, 8)
	binary.LittleEndian.PutUint64(newInstruction, newData)

	_, err = syscall.PtracePokeData(bp.pid, bp.addr, newInstruction)
	if err != nil {
		return err
	}

	bp.isEnabled = false
	return nil
}
```

## continue の処理改善

まず、現在のプログラムカウンタ（インストラクションポインタ）を読み書きするためのメソッドを実装しておきます。プログラムカウンタは、次に実行する命令のメモリアドレスのことです。 [Intel® 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) の Volume 1 の `3.5 INSTRUCTION POINTER` に記載がありますので、より正確な情報はそちらをご参照ください。

```go:go-debugger/debuger/debugger.go
func (d *Debugger) getPC() (uint64, error) {
	return readRegister(d.pid, Rip)
}

func (d *Debugger) setPC(pc uint64) error {
	return writeRegister(d.pid, Rip, pc)
}
```

続いて、ブレークポイントにヒットした時の処理を実装します。プログラムカウンタの値を読み取って、 INT3 命令を実行した分だけアドレスの値から引きます（1を引く）。その値をプログラムカウンタに設定することで、 INT3 命令を実行する手前まで処理を戻すことができます。

```go:go-debugger/debugger/debugger.go
func (d *Debugger) onBreakpointHit() error {
	pc, err := d.getPC()
	if err != nil {
		return err
	}

	// PC is incremented by 1 when the INT3 instruction is executed,
	// so PC is restored when the breakpoint is hit.
	previousPC := pc - 1
	if err := d.setPC(previousPC); err != nil {
		return err
	}

	fmt.Printf("hit breakpoint at 0x%x\n", previousPC)

	return nil
}
```

onBreakpointHit でプログラムカウンタを元に戻しても、ブレークポイントは設定されたままなので、ブレークポイントを通過する処理を実装します。

はじめに、現在のプログラムカウンタの値を読み取って、設定したブレークポイントがあるかどうかを確認します。存在していた場合はブレークポイントが有効かどうかを確認します。有効だった場合は Disable メソッドで一時的に無効化（INT3 命令を元に戻す）します。その後 syscall.PtraceSingleStep で1命令だけ実行して、ブレークポイントを再度有効にします。

```go:go-debugger/debugger/debugger.go
func (d *Debugger) stepOverBreakpointIfNeeded() error {
	pc, err := d.getPC()
	if err != nil {
		return err
	}

	bp, ok := d.breakpoints[pc]
	if !ok {
		return nil
	}

	if !bp.IsEnabled() {
		return nil
	}

	if err := bp.Disable(); err != nil {
		return err
	}

	if err := syscall.PtraceSingleStep(d.pid); err != nil {
		return err
	}

	if _, err := d.wait(); err != nil {
		return err
	}

	if err := bp.Enable(); err != nil {
		return err
	}

	return nil
}
```

Continue メソッドで先ほど定義した stepOverBreakpointIfNeeded メソッドを実行し、 continue コマンドを実行した時のプログラムカウンタがブレークポイント上の場合は一時的にブレークポイントを無効にして、1命令だけ実行して、ブレークポイントを再度有効にします。その後 syscall.PtraceCont メソッドを実行することで continue します。

ブレークポイントにヒットしたときは、 d.onBreakpointHit メソッドで INT3 命令で進んでしまったプログラムカウンタを元に戻します。

```diff:go-debugger/debugger/debugger.go
func (d *Debugger) Continue() error {
+	if err := d.stepOverBreakpointIfNeeded(); err != nil {
+		return fmt.Errorf("failed to step over breakpoint: %w", err)
+	}
+
	if err := syscall.PtraceCont(d.pid, 0); err != nil {
		return fmt.Errorf("faield to execute ptrace cont: %w", err)
	}

	...

	if ws.Stopped() {
		switch ws.StopSignal() {
		case syscall.SIGTRAP:
-			fmt.Println("hit breakpoint!")
+			if err := d.onBreakpointHit(); err != nil {
+				return err
+			}
		default:
		...
		}
	}
}
```

ブレークポイントの挙動を確認する前に、再度メモリアドレスの位置を確認しておきます。今回は `4ae618` と `4ae61d` を使います。

```bash
tail -50 tmp

#00000000004ae5a0 <main.main>:
#  4ae5a0:       49 3b 66 10             cmp    rsp,QWORD PTR [r14+0x10]
#  4ae5a4:       76 7d                   jbe    4ae623 <main.main+0x83>
#  ...
#  4ae618:       e8 e3 ae ff ff          call   4a9500 <fmt.Println>
#  4ae61d:       48 83 c4 48             add    rsp,0x48
#  ...
#  4ae628:       e9 73 ff ff ff          jmp    4ae5a0 <main.main>
```

まずは fmt.Println のアドレス `4ae618` にブレークポイントを設定して continue を実行します。すると Hello, World 出力前に処理が停止しています。その後、再度 continue を実行すると Hello, World! が出力されていることがわかります。

```bash
go run . -path ./cmd/helloworld/

# go-debugger> b 4ae618

# go-debugger> c
# hit breakpoint at 0x4ae618

# go-debugger> c
# Hello, World!
# go-debugger gracefully shut down
```

さらに fmt.Println の次の命令のアドレス `4ae61d` にブレークポイントを設定して continue を実行します。すると Hello, World が出力されてから処理が停止しています。これらの挙動から、意図した通りにブレークポイントが動作していることが分かります。

```bash
go run . -path ./cmd/helloworld/

# go-debugger> b 4ae61d

# go-debugger> c
# Hello, World!
# hit breakpoint at 0x4ae61d

# go-debugger> c
# go-debugger gracefully shut down
```

# レジスタ情報の出力

最後に、デバッグ用に各レジスタの値を出力する関数を実装しておきます。今までと同様に syscall.PtraceRegs 構造体に値を読み取ってから、 reflect パッケージを利用して各フィールドの値を出力します。

```go:go-debugger/debugger/register.go
func dumpRegisters(pid int) error {
	regs := &syscall.PtraceRegs{}
	if err := syscall.PtraceGetRegs(pid, regs); err != nil {
		return fmt.Errorf("failed to get registers of pid %d: %w", pid, err)
	}

	v := reflect.ValueOf(regs).Elem()

	for i := 0; i < v.NumField(); i++ {
		field := v.Field(i)
		fmt.Printf("%s: 0x%x\n", v.Type().Field(i).Name, field.Uint())
	}

	return nil
}
```

Debugger 構造体で先ほど定義した dumpRegisters 関数を実行します。

```diff:go-debugger/debugger/debugger.go
func (d *Debugger) SetBreakpoint(addr uint64) error {
	...
}

+func (d *Debugger) DumpRegisters() error {
+	return dumpRegisters(d.pid)
+}
```

最後に、 command を定義します。

```diff:go-debugger/terminal/command.go
func NewCommands() *Commands {
	return &Commands{
		cmds: []command{
			...
			{
				aliases: []string{"break", "b"},
				cmdFn:   setBreakpoint,
			},
+			{
+				aliases: []string{"dump", "d"},
+				cmdFn:   dumpRegisters,
+			},
		},
	}
}

...

+func dumpRegisters(dbg *debugger.Debugger, args []string) error {
+	return dbg.DumpRegisters()
+}
```

実装が終わったら、動作確認しておきましょう。 dump を実行すると各レジスタの値が出力されます。デバッガ実装中にレジスタの値を確認したくなったら dump を実行すると便利です。

```bash
go run . -path ./cmd/helloworld/

# go-debugger> b 4ae61d

# go-debugger> c
# Hello, World!
# hit breakpoint at 0x4ae61d

# go-debugger> d
# ...
# Rbp: 0xc0000acf68
# ...
# Rip: 0x4ae61d
# ...
# Rsp: 0xc0000acf20
# ...

# go-debugger> q
# go-debugger gracefully shut down
```