---
title: "アドレス指定によるブレークポイントの実装（後編。執筆中）"
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

前半部分は以下のようになります。 RegisterClient のメソッドを通じてレジスタの値を読み書きするようにします。

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

type RegisterClient struct {
	pid int
}

func NewRegisterClient(pid int) RegisterClient {
	return RegisterClient{pid: pid}
}
```

RegisterClient がレジスタの値を読み取る処理は以下のようになります。先ほど定義した Register 型の引数を1つとり、対応するレジスタの値を読み取ります。
処理の流れとしては、最初に syscall.PtraceGetRegs を使用して、 syscall.PtraceRegs 構造体に各レジスタの値を読み取ります。
その後、欲しいデータだけを返したいので

```go:go-debugger/debugger/register.go
func (c RegisterClient) GetRegisterValue(register Register) (uint64, error) {
	regs := &syscall.PtraceRegs{}
	if err := syscall.PtraceGetRegs(c.pid, regs); err != nil {
		return 0, fmt.Errorf("failed to get register values for %s and pid %d: %s", register, c.pid, err)
	}

	v := reflect.ValueOf(regs).Elem()
	field := v.FieldByName(string(register))
	if !field.IsValid() {
		return 0, fmt.Errorf("no '%s' field in syscall.PtraceRegs", register)
	}
	if field.Kind() != reflect.Uint64 {
		return 0, fmt.Errorf("field %s is not of type uint64", register)
	}

	return field.Uint(), nil
}
```

```go:go-debugger/debugger/register.go
func (c RegisterClient) SetRegisterValue(register Register, value uint64) error {
	regs := &syscall.PtraceRegs{}
	if err := syscall.PtraceGetRegs(c.pid, regs); err != nil {
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

	return syscall.PtraceSetRegs(c.pid, regs)
}
```

```go:go-debugger/debugger/register.go
func (c RegisterClient) DumpRegisters() error {
	regs := &syscall.PtraceRegs{}
	if err := syscall.PtraceGetRegs(c.pid, regs); err != nil {
		return fmt.Errorf("failed to get regs for pid %d: %s", c.pid, err)
	}

	v := reflect.ValueOf(regs).Elem()

	for i := 0; i < v.NumField(); i++ {
		field := v.Field(i)
		fmt.Printf("%s: 0x%x\n", v.Type().Field(i).Name, field.Uint())
	}

	return nil
}
```

Disable メソッドでは、 INT3 命令で上書きした
```go:go-debugger/debuger/breakpoint.go
// Disable updates the instruction at the address of the breakpoint to the original instruction
// before overwriting it with the INT3 instruction
func (bp *Breakpoint) Disable() error {
	_, err := syscall.PtracePokeData(bp.pid, bp.addr, bp.originalInstruction)
	if err != nil {
		return err
	}

	bp.isEnabled = false
	return nil
}
```
