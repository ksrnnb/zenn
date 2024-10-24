---
title: "開発環境構築"
---

# 開発環境構築
今回は、 x86-64 アーキテクチャ上で動作する Linux 環境を用意します。

Linux ホストマシンを持っている方はそちらを使用するといいのですが、持っていない方は他の手法で Linux 環境を用意してください。

ここでは、 [Lima](https://github.com/lima-vm/lima) を用いた開発環境構築方法を解説します。 Lima は VM の1つで、 Apple Silicon などの場合でも開発できるようにするために利用します。

（[Multi-platform build](https://docs.docker.com/build/building/multi-platform/) を利用すると Docker 環境でも開発できる可能性はありますが、現在のところ動作確認ができていません。）

## Lima の開発環境を構築
ここでは Mac を想定してインストール作業を解説します。

他の環境の場合や、最新の情報は Lima の[ドキュメント](https://lima-vm.io/docs/installation/)を参照してください。

### Lima のインストール
まずはホストマシンに Lima をインストールします。

```shell
brew install lima

limactl --version
# limactl version 0.23.2
```

### Lima のインスタンスを作成
ubuntu のテンプレートを利用して、 Lima のインスタンスを作成します。

今回は `go-debugger` という名前にしていますが、こちらは任意の名前で大丈夫です。

```shell
limactl create --name=go-debugger --arch=x86_64 template://ubuntu
```

"Open an editor to..." を選択します。
```shell
? Creating an instance "go-debugger"  [Use arrows to move, type to filter]
  Proceed with the current configuration
> Open an editor to review or modify the current configuration
  Choose another template (docker, podman, archlinux, fedora, ...)
  Exit
```

テキストエディタが開くので、mount の一部を修正して、ソースコードを保存するディレクトリをマウントしておきます。

今回は `~/lima/go-debugger` というディレクトリにソースコードを保存しますが、他のディレクトリでも問題ありません。 

```diff yaml
# This template requires Lima v0.7.0 or later.
images:
  # Try to use release-yyyyMMdd image if available. Note that release-yyyyMMdd will be removed after several months.
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release-20240821/ubuntu-24.04-server-cloudimg-amd64.img"
    arch: "x86_64"
    digest: "sha256:0e25ca6ee9f08ec5d4f9910054b66ae7163c6152e81a3e67689d89bd6e4dfa69"
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release-20240821/ubuntu-24.04-server-cloudimg-arm64.img"
    arch: "aarch64"
    digest: "sha256:5ecac6447be66a164626744a87a27fd4e6c6606dc683e0a233870af63df4276a"
  # Fallback to the latest release image.
  # Hint: run `limactl prune` to invalidate the cache
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img"
    arch: "x86_64"
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-arm64.img"
    arch: "aarch64"
mounts:
- - location: "~"
+ - location: "~/lima/go-debugger"
+   writable: true
  - location: "/tmp/lima"
    writable: true
arch: x86_64
```

### Lima インスタンスの起動


```shell
limactl start go-debugger
```

### Lima 環境に入る

```shell
limactl shell go-debugger
```
## Go のインストール

Lima 環境に入ったら、 Go をインストールします。

[公式ドキュメント](https://go.dev/doc/install) の Download ボタンをクリックし、インストールしたいファイルのリンクをコピーしてダウンロードしておきます。

```shell
cd lima/go-debugger

wget https://go.dev/dl/go1.23.2.linux-amd64.tar.gz
```

あとはドキュメントの Linux のインストール方法に従ってコマンドを実行していきます。

```shell
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.23.2.linux-amd64.tar.gz

echo 'export PATH=$PATH:/usr/local/go/bin' >>  ~/.profile

source $HOME/.profile

go version
# go version go1.23.2 linux/amd64
```

## Go プロジェクトの作成
Go のインストールが終わったら、カレントディレクトリがマウントしているディレクトリ（今回は `/Users/<username>/lima/debugger`） となっていることを確認して、 `go mod init` を実行します。モジュールのパスは適宜変更してください。
```shell
go mod init example.com/go-debugger
```

## テキストエディタの準備
どうやって開発するかは、読者に委ねます。筆者がパッと思いつくのは以下の手法になりますが、ホストマシン上で開発するとおそらく Linux でしか扱えない関数などの補完が働かないので注意が必要です。

- VSCode などを利用してホストマシンからリモート接続
- Lima 環境内で vim 環境をつくる
- リモート接続せずに、ホストマシン上で開発

リモート接続して開発する場合は、以下のコマンドを実行すると ssh 接続の設定を追加できます。
```shell
cat /Users/<username>/.lima/go-debugger/ssh.config >> ~/.ssh/config
```