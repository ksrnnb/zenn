---
title: "ptrace"
---

# ptrace とは
[ptrace](https://man7.org/linux/man-pages/man2/ptrace.2.html) はシステムコールの一種で、他プロセスで実行されているプログラムを追跡します。ブレークポイントを利用したデバッグや、システムコールの追跡などで利用されます。

システムコールを実行する時は以下のように実行します。
```c
ptrace(PTRACE_foo, pid, ...)
```

第一引数に ptrace の operation を渡します。 operation には `PTRACE_TRACEME` や `PTRACE_PEEKDATA` などがありますが、これに関しては後述します。
第二引数には、対応する Linux スレッドのスレッド ID になります。基本的には追跡したいプログラムのスレッド ID を指定することになります。
第三引数以降は、 operation に応じて異なりますが、メモリアドレスの値などが入ることがあります。

## 主要な operation
ptrace システムコールの第一引数に渡す operation の主要なものについて解説します。

### PTRACE_TRACEME

### PTRACE_ATTACH

### PTRACE_PEEKDATA

### PTRACE_POKEDATA

### PTRACE_GETREGS

### PTRACE_SETREGS

### PTRACE_CONT

# Go での使い方
