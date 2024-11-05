---
title: "ptrace の解説"
---

# ptrace とは
[ptrace](https://man7.org/linux/man-pages/man2/ptrace.2.html) はシステムコールの一種で、あるプロセス（tracer）が他プロセス（tracee）で実行されているプログラムを追跡し、 tracee のメモリやレジスタの値を読み取ったり更新したりすることができます。ブレークポイントを利用したデバッグや、システムコールの追跡などで利用されます。

システムコールを実行する時は以下のように実行します。
```cpp
#include <sys/ptrace.h>

long ptrace(enum __ptrace_request op, pid_t pid,
            void *addr, void *data);
```

第一引数に ptrace の operation を渡します。 operation には `PTRACE_TRACEME` や `PTRACE_PEEKDATA` などがありますが、これに関しては後述します。
第二引数には、対応する Linux スレッドのスレッド ID になります。基本的には追跡したいプログラムのスレッド ID を指定することになります。
第三引数以降は、 operation に応じて異なりますが、メモリアドレスの値などが入ることがあります。

## 主要な operation
ptrace システムコールの第一引数に渡す operation の主要なものについて解説します。

### PTRACE_TRACEME
ptrace を実行するのは tracer がほとんどですが、 `PTRACE_TRACEME` オペレーションは tracee が実行します。
tracer が `fork(2)` を実行して、生成した子プロセス（tracee）が `PTRACE_TRACEME` を実行することで、tracer が tracee を追跡できるようになります。
その後、 tracee が `execve(2)` を実行すると一時停止して、 tracer が次の ptrace を実行するまで処理を停止します。また、カーネルからシグナルを受け取った時も tracee は停止するようになります。

### PTRACE_CONT
停止した tracee の処理を再開します。ブレークポイントを設定していない場合は最後まで tracee の処理を実行します。

### PTRACE_PEEKDATA/PTRACE_POKEDATA
`PTRACE_PEEKDATA` は `addr` で指定したメモリアドレスの値を読み取り、 `PTRACE_POKEDATA` は `addr` で指定したメモリアドレスの値を書き込みます。
これらの operation は主にブレークポイントを設定するときに使用します。


### PTRACE_GETREGS/PTRACE_SETREGS
`PTRACE_GETREGS` はレジスタの値を読み取り、 `PTRACE_GETREGS` はレジスタの値を書き込みます。
これらの operation はブレークポイントにヒットしたときや、プログラムカウンタを取得したいときなどで利用します。

# wait4
[wait4](https://man7.org/linux/man-pages/man2/wait4.2.html) は、 pid 引数で指定した子プロセス（tracee）の状態が変化するまで、呼び出し元のスレッド（tracer）の処理を一時停止します。 tracee の状態が変化すると、 tracer の処理が再開します。
ここで状態の変化は、 tracee が終了した場合、シグナルによって処理を停止した場合や、再開した場合のことを指します。


```cpp
#include <sys/wait.h>

pid_t wait4(pid_t pid, int *_Nullable wstatus, int options,
            struct rusage *_Nullable rusage);
```

# Go での使い方
Go では [syscall パッケージ](https://pkg.go.dev/syscall)や [sys パッケージ](https://pkg.go.dev/golang.org/x/sys)が用意されています。これらは OS によって使用できない関数もあります。
この本では Linux 環境を前提としていますが、それは `syscall.PtraceXxx` や `sys/unix` の `Wait4` などを利用するためになります。

Go での簡単な例を以下に示します。この例では、指定した pid のプロセス（tracee）に対して、 [syscall.PtraceCont](https://pkg.go.dev/syscall#PtraceCont) で tracee の処理を再開し、 [unix.Wait4](https://pkg.go.dev/golang.org/x/sys@v0.26.0/unix#Wait4) で tracee の状態が変化するまで待機しています。

[syscall.PtracePeekData](https://pkg.go.dev/syscall#PtracePeekData) など、他の ptrace オペレーションに対応するメソッドもあるので適宜ドキュメントを参照してください。

```go
package main

import (
	"fmt"
	"os"
	"syscall"

	"golang.org/x/sys/unix"
)

func main() {
	// 本来は exec.Cmd.Process.Pid などから pid を取得する
	pid := 123456

	if err := syscall.PtraceCont(pid, 0); err != nil {
		fmt.Fprintf(os.Stderr, "faield to execute ptrace cont: %s\n", err)
		return
	}

	var ws unix.WaitStatus
	_, err = unix.Wait4(pid, &ws, unix.WALL, nil)
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to wait pid %d\n", pid)
		return
	}
}
```
