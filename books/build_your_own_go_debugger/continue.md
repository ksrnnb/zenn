---
title: "continue の実装"
---

# Hello, World! プログラムの実行
最初に以下のようなディレクトリ階層となるように、 cmd/helloworld ディレクトリを作成し、 main.go ファイルを作成します。

```
go-debugger/
├── cmd
│   └── helloworld
│     └── main.go
└── go.mod
```

その後、 main.go で Hello, World! を出力するプログラムを作成します。

```go:go-debugger/cmd/helloworld/main.go 
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
```

念の為、動作を確認します。
```shell
go run ./cmd/helloworld/main.go 
# Hello, World!
```

# Hello, World! プログラムのビルド
簡単なプログラムを作成できたので、これをデバッグできるようにビルドするプログラムを作成します。
今度は以下のようにトップレベルに main.go と build.go を作成します。

```diff
go-debugger/
  ├── cmd
  │   └── helloworld
  │     └── main.go
+ ├── build.go
  ├── go.mod
+ └── main.go
```

ファイルを作成したら、 Hello, World! プログラムをビルドして、終了前にビルドしたファイルを削除するプログラムを作成します。

まず、 build.go の中身は以下のようになります。
`buildDebuggeeProgram` 関数はデバッグしたい Go プログラムをビルドして、ビルドしたファイルのパスとクリーンアップ関数を返す関数になります。ビルドするときのオプションの `gcflags` に関しては後述します。

```go:go-debugger/build.go 
package main

import (
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"time"
)

// buildDebuggeeProgram build go program to debug and returns cleanup function.
func buildDebuggeeProgram(path string) (absPath string, cleanup func() error, err error) {
    // ビルドするファイル名を __debug__1730159170 のような形式にする
	name := fmt.Sprintf("__debug__%d", time.Now().Unix())
	cmd := exec.Command("go", "build", "-o", name, "-gcflags", "all=-N -l", path)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		return "", nil, fmt.Errorf("failed to build go program: %w", err)
	}

    // ビルドしたファイルの絶対パスを取得
	absPath, err = filepath.Abs(name)
	if err != nil {
		return "", nil, fmt.Errorf("failed to get absolute path of debuggee program: %w", err)
	}

	return absPath, func() error {
		if err := os.Remove(absPath); err != nil {
			return fmt.Errorf("failed to remove debuggee program: %w", err)
		}

		return nil
	}, nil
}
```

main.go を以下のように実装します。 Go プログラムのパスを引数で受け取って、先ほどの `buildDebuggeeProgram` 関数を実行してビルドします。今回は特に何もしないので、処理が終わったらログを出力しておきます。

```go:go-debugger/main.go 
package main

import (
	"flag"
	"fmt"
	"log"
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

	fmt.Printf("built program path is %s\n", absDebuggeePath)
}
```

ここまでコードが書けたら、プログラムを実行しておきましょう。
以下のように、ビルドしたプログラムのパスが出力されていると成功です。

```shell
$ go run . -path ./cmd/helloworld
# built program path is /Users/<username>/lima/go-debugger/__debug__1730159855
```

## gcflags について
今回、 go build の引数として `-gcflags=all=-N -l` を渡しているのでここで簡単に解説します。
go build に渡す引数に関しては、 cmd/go の[Compile packages and dependencies](https://pkg.go.dev/cmd/go#hdr-Compile_packages_and_dependencies) に記載があります。

> -gcflags '[pattern=]arg list'
	arguments to pass on each go tool compile invocation.

go build の内部では、 go tool compile を実行するのですが、そのときに渡す引数を gcflags で指定できます。
pattern は今回は -all としていますが、ビルドしたいパッケージと、それが依存するパッケージ全てのパッケージが対象となります。pattern の詳細は `go help packages` で確認することができます。

引数のリストに関しては、 [cmd/compile](https://pkg.go.dev/cmd/compile) に記載があります。
-N はコンパイル時の最適化を無効にし、 -l はインライン化を無効にします。それぞれ、デバッグを簡単にするために必要なので指定しておきます。

>-N
	Disable optimizations.
-l
	Disable inlining.


Go コンパイラに関してさらに詳しく知りたい方は cmd/compile の [README](https://go.dev/src/cmd/compile/README) を読むとさらに理解が進むと思います。

# ptrace でプログラムを追跡可能にする
まず、プログラムを実行するための関数を記述するための execute.go ファイルを作成します。

```diff
go-debugger/
  ├── cmd
  │   └── helloworld
  │     └── main.go
  ├── build.go
+ ├── execute.go
  ├── go.mod
  └── main.go
```

ビルドしたプログラムを実行するときに、 [ptrace](https://man7.org/linux/man-pages/man2/ptrace.2.html) を使用するために、 `cmd.SysProcAttr` の `Ptrace` フィールドを true に設定しています。この設定によって、プログラムを追跡可能になります。 
その後、 `cmd.Start` によってプログラムを子プロセスで実行します。

```go:go-debugger/execute.go
package main

import (
	"log"
	"os"
	"os/exec"
	"syscall"
)

func executeDebuggeeProcess(debuggeePath string) (pid int, err error) {
	cmd := exec.Command(debuggeePath)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	// trace debuggee program
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Ptrace: true,
	}

	if err := cmd.Start(); err != nil {
		log.Fatalf("failed to start command: %s", err)
	}

	return cmd.Process.Pid, nil
}
```

## cmd.Start
`cmd.Start` をもう少し詳しく見てみると、以下のようなコードに辿り着きます。 [fork して子プロセスを生成](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/syscall/exec_linux.go;l=336)して、 [execve で実行](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/syscall/exec_linux.go;l=649-652)していることが分かります。
`cmd.Start` を実行した後は、生成した子プロセスのプロセス ID が `cmd.Process.ID` で取得できるようになります。

```go:exec_linux.go
// Time to exec.
pid, err1 = rawVforkSyscall(SYS_CLONE, flags, 0, uintptr(unsafe.Pointer(&pidfd)))

...

_, _, err1 = RawSyscall(SYS_EXECVE,
	uintptr(unsafe.Pointer(argv0)),
	uintptr(unsafe.Pointer(&argv[0])),
	uintptr(unsafe.Pointer(&envv[0])))
```


## ptrace
`syscall.SysProcAttr` の `Ptrace` フィールドがどのように使われているのかを調べてみると、以下のように `Ptrace` フィールドを使っている[コード](https://cs.opensource.google/go/go/+/refs/tags/go1.23.2:src/syscall/exec_linux.go;l=641-646)がみつかります。
fork によって生成された子プロセス（tracee）で ptrace システムコールを実行しており、 `PTRACE_TRACEME` を渡しています。 これを実行すると tracee が親プロセス（tracer）に追跡されている状態になります。さらに tracee が execve を実行すると一時停止して、 tracer が次の ptrace を実行するまで処理を停止します。
したがって、 `cmd.SysProcAttr`　の `Ptrace` フィールドを true にして、 `cmd.Start` を実行すると、プログラムは実行されずに停止します。

```go:exec_linux.go
// Fork succeeded, now in child.

...

if sys.Ptrace {
	_, _, err1 = RawSyscall(SYS_PTRACE, uintptr(PTRACE_TRACEME), 0, 0)
	if err1 != 0 {
		goto childerror
	}
}
```

`executeDebuggeeProcess` が実装できたら、 main.go で実行します。

```diff:go-debugger/main.go
func main() {
	...

	defer cleanup()

+	pid, err := executeDebuggeeProcess(absDebuggeePath)
+	if err != nil {
+		log.Fatalf("failed to execute debugee program: %s", err)
+	}

+	fmt.Printf("pid of debuggee program is %d\n", pid)
-	fmt.Printf("built program path is %s\n", absDebuggeePath)
}
```

ここまで実装できたら実行してみましょう。
ここで Hello, World! が出力されないことを確認してください。
```shell
go run . -path ./cmd/helloworld/
# pid of debuggee program is 9742
```

# continue で処理を再開する
まず、 `syscall.Wait4` で子プロセスが停止するまで待機します。これは内部的に [wait4 システムコール](https://man7.org/linux/man-pages/man2/wait4.2.html)を実行しています。

その後プログラムの処理を再開するために、`syscall.PtraceCont` を実行します。これは内部的には `PTRACE_CONT` を渡して ptrace システムコールを実行しています。これは追跡中のプログラムが停止している場合、処理を再開するためのものです。これに関しても詳細は次章で解説します。

continue の後も、 `syscall.Wait4` で子プロセスの処理が再開するまで待機するようにしておきます。

```diff:go-debugger/main.go
func main() {
	...

	fmt.Printf("pid of debuggee program is %d\n", pid)

+	var ws syscall.WaitStatus
+	_, err = syscall.Wait4(pid, &ws, syscall.WALL, nil)
+	if err != nil {
+		fmt.Fprintf(os.Stderr, "failed to wait pid %d\n", pid)
+		return
+	}

+	if err := syscall.PtraceCont(pid, 0); err != nil {
+		fmt.Fprintf(os.Stderr, "faield to execute ptrace cont: %s\n", err)
+		return
+	}

+	_, err = syscall.Wait4(pid, &ws, syscall.WALL, nil)
+	if err != nil {
+		fmt.Fprintf(os.Stderr, "failed to wait pid %d\n", pid)
+		return
+	}
}
```

それではプログラムを実行してみましょう。以下のように Hello, World! が出力されていたら成功です。プロンプトの後に出力されることもありますが、あまり気にしなくて大丈夫です。

```shell
go run . -path ./cmd/helloworld/
# pid of debuggee program is 16846
# Hello, World!
```
