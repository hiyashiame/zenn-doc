---
title: "[Rust]”Tock embedded OS”のKernelをVSCodeでデバッグする" # 記事のタイトル
emoji: "🌱" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["組み込み", "rust", "RTOS"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

# 1. 試していること

Tock embedded OS を試してみています。
今回は、VSCodeを用いてKernelのデバッグをしてみます。
ホストPCはUbuntu 22です。Windowsではありません。


# 2. VSCodeのインストール

sudo apt-get install code
でOKです。


# 3. VSCodeの設定
VSCodeでC/C++を扱うため、またCortexのデバッグを行うため、拡張機能をインストールする。
拡張機能メニューを　Ctrl + Shift + X で表示し、「C/C++ Extension Pack」と「Cortex-Debug」を検索、インストールすればOK。


# 4. プロジェクト？を開く
VSCodeのFileメニューから、Open Folder...[Ctl-k Ctl-O] を選択すると、フォルダを選択するためのダイアログボックスが開く。
tockのルートフォルダを選択する。
そうすると、VSCodからファイルが見れるようになる。

tock_vscode1.png


# 5. launch.jsonを編集する。
tockのルートフォルダ直下に、隠しフォルダとして .vscodeがある。
その中に、launch.jsonがあるので、それを編集する。

home@home-CFSX4-1L:~/tock$ cd .vscode/
home@home-CFSX4-1L:~/tock/.vscode$ ls
launch.json  settings.json

編集といっても、以下をくっつけるだけ。

       {
            "name": "nucleo_f446re Tock os Debug",
            "type": "cortex-debug",
            "servertype": "openocd",
            "request": "launch",
            "executable": "${workspaceRoot}/target/thumbv7em-none-eabi/debug/nucleo_f446re-app.elf",
            "cwd": "${workspaceRoot}",
            "runToMain": true,
            "configFiles": [
                "${workspaceRoot}/boards/nucleo_f446re/openocd.cfg",
            ]
        }
        
ここで、
nucleo_f446re-app.elf
というのは、Kernelとアプリがくっついたバイナリのこと。
こうしておくと、アプリ動作下でのKernel動作がデバッグできる。

# 6 デバッグの設定
そうそう、デバッグするならGDBをインストールしなきゃ。
sudo apt-get install gdb-multiarch

それから、settings.jsonに以下を追記する。
launch.json同様、tockのルートフォルダ直下の隠しフォルダ.vscodeの中にある。

    "cortex-debug.armToolchainPath": "/usr/bin",
    "cortex-debug.gdbPath": "gdb-multiarch"

# 7 ビルド＆ダウンロード＆実行
ではいよいよデバッグだ。
実機をPCに接続する。
で、VSCodeで「Run and Debug」の三角ボタンを押し、VSCodeをデバッグモードにする。
いよいよデバッグ開始。すぐさまDebug Startボタンを押したいが焦らない。

実機のリセットを押したまま、Debug Startを押し、すぐに実機のリセットボタンを離す。
すると、バイナリのダウンロードが行われ、main関数先頭でブレークする。

＜tock_vscode2.png＞


、、、てな感じでカーネルはVSCode＆ST-Linkでデバッグができる。
ユーザランドはどうだろうか。
ちょっとやり方がわからないが、もしわかったらまたここに書きに来ます。




    
    
    

