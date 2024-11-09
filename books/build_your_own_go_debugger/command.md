---
title: "コマンド入力の実装"
---

# コマンドの入力
本章では、 continue などをユーザーが指示して実行できるように、コマンドを入力できるようにします。

## ファイルの追加
まず、以下のようにファイルを作成、削除します。

```diff
go-debugger/
  ├── cmd
  │   └── helloworld
  │     └── main.go
+ ├── debugger
+ │   └── debugger.go
+ ├── terminal
+ │   ├── command.go
+ │   └── terminal.go
  ├── build.go
- ├── execute.go
  ├── go.mod
  └── main.go
```

## debbugger.go の実装
いままで、 execute.go などで実行していた処理を debugger パッケージの中で実行するようにします。この debugger パッケージを terminal パッケージ側から利用するような構成になります。

まず、 debuggee（デバッグ対象のプログラム）をビルドした成果物のパスを Config で受け取り、 Debugger 構造体を作成します。その後、 Launch メソッドを実行して、子プロセスを生成してデバッグを開始します。

また、 Continue メソッドを実装して Continue を任意のタイミングで実行できるようにしておきます。 Continue メソッド内では、 Wait4 を実行した後に WaitStatus を確認して子プロセスが終了したかどうかをチェックしています。子プロセスが終了していた場合は、 `ErrDebuggeeFinished` を返すようにします。
Quit メソッドは単純に `ErrDebuggeeFinished` を返すだけになります。

```go:go-debugger/debugger/debugger.go
package debugger

import (
	"errors"
	"fmt"
	"os"
	"os/exec"
	"syscall"

	"golang.org/x/sys/unix"
)

// ErrDebuggeeFinished is used when debuggee processs is finished.
var ErrDebuggeeFinished = errors.New("debuggee process is finished")

type Config struct {
	DebuggeePath string
}

type Debugger struct {
	config *Config
	pid    int
}

func NewDebugger(config *Config) (*Debugger, error) {
	d := &Debugger{config: config}
	if err := d.Launch(); err != nil {
		return nil, err
	}

	return d, nil
}

func (d *Debugger) Launch() error {
	cmd := exec.Command(d.config.DebuggeePath)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	// trace debuggee program
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Ptrace: true,
	}

	if err := cmd.Start(); err != nil {
		return fmt.Errorf("failed to launch debuggee process: %w", err)
	}

	d.pid = cmd.Process.Pid

	if _, err := d.wait(); err != nil {
		return err
	}

	return nil
}

func (d *Debugger) Continue() error {
	if err := syscall.PtraceCont(d.pid, 0); err != nil {
		return fmt.Errorf("faield to execute ptrace cont: %w", err)
	}

	ws, err := d.wait()
	if err != nil {
		return err
	}

	// ws.Exited() will be true when child process is finished.
	if ws.Exited() {
		return ErrDebuggeeFinished
	}

	return nil
}

func (d *Debugger) Quit() error {
	return ErrDebuggeeFinished
}

func (d *Debugger) wait() (unix.WaitStatus, error) {
	var ws unix.WaitStatus
	_, err := unix.Wait4(d.pid, &ws, unix.WALL, nil)
	if err != nil {
		return 0, fmt.Errorf("failed to wait pid %d", d.pid)
	}

	return ws, nil
}
```

## command.go の実装
Commands 構造体は、CLI でデバッガを操作する時のコマンドの情報を持ちます。コマンドの名前とそれに対応する関数をフィールドにもつ command 構造体を定義しています。
ユーザーの入力が aliases のどれかに一致していたら対応する関数を実行するようになります。したがって、 continue の場合は `continue` または `c` を入力すると実行できます。

```go:go-debugger/terminal/command.go
package terminal

import (
	"github.com/ksrnnb/go-debugger/debugger"
)

type cmdfunc func(dbg *debugger.Debugger, args string) error

type command struct {
	aliases []string
	cmdFn   cmdfunc
}

type Commands struct {
	cmds []command
}

func NewCommands() *Commands {
	return &Commands{
		cmds: []command{
			{
				aliases: []string{"continue", "c"},
				cmdFn:   cont,
			},
			{
				aliases: []string{"quit", "q"},
				cmdFn:   quit,
			},
		},
	}
}

func cont(dbg *debugger.Debugger, args string) error {
	return dbg.Continue()
}

func quit(dbg *debugger.Debugger, args string) error {
	return dbg.Quit()
}
```

## terminal.go の実装
Terminal の Run メソッドを実行すると、デバッガが起動してインタラクティブに操作することができるようになります。
for 文で繰り返し入力を受け付けるようになっていますが、コマンド実行時に `ErrDebuggeeFinished` が返ってきた場合は break して処理を終了します。

```go:go-debugger/terminal/terminal.go
package terminal

import (
	"bufio"
	"errors"
	"fmt"
	"os"
	"slices"
	"strings"

	"github.com/ksrnnb/go-debugger/debugger"
)

const prompt = "go-debugger>"

type Terminal struct {
	debugger *debugger.Debugger
	cmds     *Commands
}

func NewTerminal(debugger *debugger.Debugger, cmds *Commands) *Terminal {
	return &Terminal{debugger, cmds}
}

func (t *Terminal) Run() error {
	sc := bufio.NewScanner(os.Stdin)

	fmt.Printf("%s ", prompt)

	for sc.Scan() {
		input := sc.Text()

		cmdFn, err := t.Find(input)
		if err != nil {
			fmt.Printf("failed to parse command: %s\n", err)
			fmt.Printf("%s ", prompt)
			continue
		}

		// input: "<command> <arg1> <arg2>"
		// -> args: [<arg1>, <arg2>]
		s := strings.SplitN(input, " ", 2)
		var args string
		if len(s) == 2 {
			args = s[1]
		}

		if err := cmdFn(t.debugger, args); err != nil {
			if errors.Is(err, debugger.ErrDebuggeeFinished) {
				break
			}
			return err
		}
		fmt.Printf("\n%s ", prompt)
	}

	return nil
}

func (t *Terminal) Find(commandWithArgs string) (cmdfunc, error) {
	s := strings.SplitN(commandWithArgs, " ", 2)
	command := s[0]
	for _, cmd := range t.cmds.cmds {
		if slices.Contains(cmd.aliases, command) {
			return cmd.cmdFn, nil
		}
	}

	return nil, fmt.Errorf("command %s is not found", command)
}
```

## main.go の更新
最後に、 main.go を以下のように更新してデバッガを CLI で起動します。

```go:go-debugger/main.go
package main

import (
	"flag"
	"fmt"
	"log"
	"os"

	"github.com/ksrnnb/go-debugger/debugger"
	"github.com/ksrnnb/go-debugger/terminal"
)

var debuggeePath string

func init() {
	flag.StringVar(&debuggeePath, "path", "", "path of debuggee program")
}

func main() {
	flag.Parse()

	if debuggeePath == "" {
		log.Fatalf("path of debuggee program must be given")
	}

	absDebuggeePath, cleanup, err := buildDebuggeeProgram(debuggeePath)
	if err != nil {
		log.Fatalf("failed to build debuggee program: %s", err)
	}
	defer cleanup()

	d, err := debugger.NewDebugger(&debugger.Config{
		DebuggeePath: absDebuggeePath,
	})
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to initialize debugger: %s", err)
		return
	}

	cmds := terminal.NewCommands()
	term := terminal.NewTerminal(d, cmds)

	if err := term.Run(); err != nil {
		fmt.Fprintf(os.Stderr, "failed to run terminal: %s", err)
		return
	}

	fmt.Printf("go-debugger gracefully shut down\n")
}
```

## デバッガの実行
ひととおり実装できたので、動作を確認してみます。まずは quit を試してみます。
```bash
go run . -path ./cmd/helloworld/

# go-debugger> quit
# go-debugger gracefully shut down
# Hello, World!
```

続いて、continue を何度か入力してみます。
```bash
go run . -path ./cmd/helloworld/

# go-debugger> continue

# go-debbuger> continue
# Hello, World!
# go-debugger gracefully shut down
```

それぞれ、想定と少し違った挙動をしているかと思います。これらを少しずつ修正していきます。
- quit したときに、子プロセスが終了せずに最後まで実行されている
- なぜか2回 continue しないと Hello, World! が出力されない

## quit の挙動修正
quit したときに子プロセスが最後まで処理が進んでしまっているので、 quit したときに子プロセスを終了するようにします。

まずは、 SysProcAttr の Setpgid を true に設定しておきます。

```diff:go-debugger/debugger/debugger.go
// trace debuggee program
cmd.SysProcAttr = &syscall.SysProcAttr{
	Ptrace:  true,
+	Setpgid: true,
}
```

そのうえで、 cleanup メソッドを作成し、子プロセスのグループに対して SIGTERM のシグナルを送信して、子プロセスを終了します。
```go:go-debugger/debugger/debugger.go
func (d *Debugger) cleanup() error {
	if err := syscall.Kill(-d.pid, syscall.SIGTERM); err != nil {
		return fmt.Errorf("failed to kill child process %d: %s", d.pid, err)
	}
	return nil
}
```

quit 実行時に、 cleanup メソッドを実行します。
```diff:go-debugger/debugger/debugger.go
func (d *Debugger) Quit() error {
+	if err := d.cleanup(); err != nil {
+		return fmt.Errorf("failed to cleanup: %s", err)
+	}

	return ErrDebuggeeFinished
}
```

この状態で、デバッガを動かしてみましょう。 Hello, World! は出力されないようになり、 ps コマンドで子プロセスが残っていないことも確認できます。
```bash
go run . -path ./cmd/helloworld/

# go-debugger> quit
# go-debugger gracefully shut down

ps -a
#    PID TTY          TIME CMD
#  21787 pts/1    00:00:00 ps
```

### Setpgid
なぜこのような変更をしたのか、簡単に説明します。
まず、 SysProcAttr の Setpgid を true にしておくことで、コマンド実行時に [setpgid システムコールを実行するようになります](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/syscall/exec_linux.go;l=386-393)。

```go:exec_linux.go
// Fork succeeded, now in child.

...

// Set process group
if sys.Setpgid || sys.Foreground {
	// Place child in process group.
	_, _, err1 = RawSyscall(SYS_SETPGID, 0, uintptr(sys.Pgid), 0)
	if err1 != 0 {
		goto childerror
	}
}
```

[setpgid](https://www.man7.org/linux/man-pages/man2/setpgid.2.html) は、指定したプロセスのプロセスグループ ID（PGID）を設定します。上記のコードをみると、 setpgid の引数は SysProcAttr の Pgid と 0 になっています。今回は Pgid は未指定なので、 引数は全て 0 となります。
以下のような記載があるように、setpgid の引数の pid が 0 の場合は実行したプロセスのプロセス ID が使用され、 pgid が 0 の場合は、 pid と同じ値がプロセスグループ ID として設定されます。

> If pid is zero, then the process ID of the calling process is used. If pgid is zero, then the PGID of the process specified by pid is made the same as its process ID.

また、上記のシステムコールのコードは生成された子プロセスが実行することに注意してください。このことを考慮すると、 Setpgid を true にして cmd.Start を実行すると、子プロセスは setpgid システムコールを実行して、自身のプロセス ID をプロセスグループ ID として設定します。このような処理を経ない場合は、子プロセスは親プロセスと同じプロセスグループ ID となります。


### kill
[kill](https://man7.org/linux/man-pages/man2/kill.2.html) で引数に負の値を渡すと、プロセスグループのすべてのプロセスがシグナル送信の対象になります。
したがって、 Setpgid を true にしておくと、子プロセスのプロセス ID の値を負にして kill を実行したときに、子プロセスと、子プロセスが生成した全てのプロセス（同じプロセスグループ ID になる）に対してシグナルを送信することができます。
> If pid is less than -1, then sig is sent to every process in the process group whose ID is -pid.

## continue の挙動修正
continue を実行すると 2 回目で Hello, World! が出力される原因を調べるために Continue メソッドを以下のように更新します。 WaitStatus を詳しく調べることで、子プロセスがどのシグナルを受信して停止したのかを出力します。

```diff:go-debugger/debugger/debugger.go
func (d *Debugger) Continue() error {
	...

	if ws.Exited() {
		return ErrDebuggeeFinished
	}

+	if ws.Stopped() {
+		fmt.Printf("StopSignal: %s\n", ws.StopSignal())
+	}

	return nil
}
```

この状態で、continue を実行してみます。1回目の continue を実行したときに `urgent I/O condition` というシグナルを受信しています。これは[コード](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/syscall/zerrors_linux_amd64.go;l=1516) を調べると、シグナル番号23の [SIGURG](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/syscall/zerrors_linux_amd64.go;l=1349) シグナルに相当することが分かります。

```bash
go run . -path ./cmd/helloworld/

# go-debugger> c
# StopSignal: urgent I/O condition

# go-debugger> c
# Hello, World!
# go-debugger gracefully shut down
```

SIGURG シグナルを受信している理由は後述しますが、いったんはこれを無視するような処理にしておきます。

```diff:go-debugger/debugger/debugger.go
func (d *Debugger) Continue() error {
	...

	if ws.Exited() {
		return ErrDebuggeeFinished
	}

-	if ws.Stopped() {
-		fmt.Printf("StopSignal: %s\n", ws.StopSignal())
-	}

+	// ignore SIGURG signal because it is not expected signal
+	if ws.Stopped() && ws.StopSignal() == syscall.SIGURG {
+		return d.Continue()
+	}

	return nil
}
```

この状態で continue を実行してみると、1回の continue で最後まで処理が完了しており、意図した挙動になっているかと思います。

```bash
go run . -path ./cmd/helloworld/

# go-debugger> c
# Hello, World!
# go-debugger gracefully shut down
```

### SIGURG シグナル
Go プログラムをデバッグするとしばしば debuggee が SIGURG シグナルを受信することがあります。これは Go 1.14 以降でみられる挙動になります。 Go 1.14 以降はゴルーチンのプリエンプションに SIGURG シグナルを使っているため、プリエンプトされるたびに SIGURG シグナルを受信します。この挙動に関しては、[Non-cooperative goroutine preemption](https://go.googlesource.com/proposal/+/master/design/24543-non-cooperative-preemption.md) というプロポーザルに詳細が記載されています。