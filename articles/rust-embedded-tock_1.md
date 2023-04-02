---
title: "Rustで書かれたRTOS ”Tock embedded OS”を試してみた" # 記事のタイトル
emoji: "😀" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["組み込み", "rust", "RTOS"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

# 1. はじめに

米国の３大学、Stanford University、University of MichiganとUC Berkeleyが開発を進めている、Rustで書かれたリアルタイムOSがあります。 
まだまだ実用には遠いのかと思いきや、自分が知らなかっただけで、2020年にはGoogle OpenSKのOSとして採用されているのだとか。 
https://www.tockos.org/blog/2020/hello-opensk/

実際どれくらい自分の近くに来ているのかと興味があり、サポートされているハードウェアを調べてみたところ、STMのNucleoシリーズやRISC-VのHiFive1ボード等、なかなか身近なところまで来ていました。 

最近組込み分野でもRustを検討し始める声が聞こえてきて、乗り遅れないようにしなきゃと焦りがありました。 
それでとりあえずはFreeRTOSの上でRustを動かしてみようかなと思ったのです、先に”Tock embedded OS”を味見してみたくなりました。 

https://www.tockos.org/



# 2. 動かして見るハードウェア
秋月電子の通販サイトからSTMのNucleo F446REが購入できます。（現時点では） 
![](/images/article1-tock1/tock1_1.JPG)
この実機はTockの対象ハードウェアに入っているため、これで味見をしてみることにします。 

味見するにあたり、ハードウェアの機能としてはどのくらい対応されているのかを調べました。 
https://github.com/tock/tock/tree/master/chips/stm32f4xx/src

見た限り、adc, can, gpio. i2c, spi, usart等、そこそこ揃っています。
全てが動作するのかはわかりませんが、そこそこ動作するのであればそれなりなIoTが作れそうです。 


# 3. ユーザランド
ユーザランド：いわゆるユーザアプリケーションのことです。 
Tockの特徴の一つとして、 

「カーネルとユーザランドが独立している」

が挙げられます。 
組込み用途の小メモリで動くリアルタイムＯＳではめずらしいなぁと思っています。 

そのユーザランドですが、Rustでも書けますし、C/C++でも書けます。 
今のところはC/C++で書くことが前提かもしれません。 
というのは、Rustで書かれたユーザランドのサンプルプログラムが少なすぎるからです。 
今後は充実してくのでしょう。 
自分としてはRustほぼ初心者なので、これは嬉しい配慮として素直に受け取ります。 


# 4. 開発環境構築に使用するホストPC
ホストPCはUbuntuかMacです。 
Windows11で使用できる、WSL2(Windows Subsystem for Linux2)の上で使用しようとすると、ICEのUSB認識でハマります。  （注：2023年4月時点）
なので、おとなしくUbuntuをVirtual Boxに構築するなりした方がいいです。 
自分は開発用にLinuxPCを持っているので、そちらで試してみます。 
ちなみに、LinuxPCといっても、秋葉原に行って格安で購入した古いWindowsPCをUbuntu化しただけです。 
ですが、これが非常に使いやすく、重宝しています。


# 5. セットアップ
では、いよいよセットアップしてみます。 

基本
https://github.com/tock/tock/blob/master/doc/Getting_Started.md
に、書かれています。

自分が試した時の、大まかな流れは以下です。 

① GNUツールチェインのインストール 
② Rustツールチェインのインストール（nightly-2022-10-22） 
③ Tock一式の取得 
④ ハードウェア制御用ICEのセットアップ 
⑤ Tockカーネルをビルド＆ハードウェアにダウンロード 
⑥ ユーザランドをビルド＆ハードウェアにダウンロード 

## 5.1 GNUツールチェインのインストール 
STMのNucleo F446REはARM CortexM4なので、普通にarm-none-eabi toolchainが使用できます。(version >= 5.2) 
\$ sudo apt install gcc-arm-none-eabi


## 5.2 Rustツールチェインのインストール
\$ curl https://sh.rustup.rs -sSf | sh


## 5.3 Tock一式の取得
Tockカーネル、C/C++ユーザランド、Rustユーザランド、Tockloaderを取得します。 

Tockカーネル：
\$ git clone https://github.com/tock/tock.git

C/C++ユーザランド：
\$ git clone https://github.com/tock/libtock-c.git

Rustユーザランド：
\$ git clone --recursive https://github.com/tock/libtock-rs

tockloader：
\$ pip3 install -U --user tockloader
\$ export PATH=\$HOME/Library/Python/3.9/bin/:\$PATH
\$ grep -q dialout <(groups \$(whoami)) || sudo usermod -a -G dialout \$(whoami) # Note, will need to reboot if prompted for password
\$ source \$HOME/.cargo/env
\$ export PATH=\$HOME/.local/bin:$PATH  #tockloader path

tockloaderはPythonで書かれています。
なお、後で気がついたのですが、STM系の実機を使う場合tockloaderは使用しないようです。


## 5.4 ハードウェア制御用ICEのセットアップ  

Nucleo F446REとの接続は、ボード上に存在するST-Linkとなります。
ボード上のUSBコネクタから出ているので、単純にUSBケーブルでPCと接続するだけです。
Ubuntu側で、OpenOCDとST-Link用のソフトウェアをインストールする必要があります。

\$ sudo apt-get install openocd

これだけでSTLinkもインストールされたように思いますが、定かではありません。:(
さらに、STLinkが動作できるためにudev ruleをつけないと行けないと公式に書いてあります。
https://github.com/tock/tock/blob/master/doc/Getting_Started.md
これをしなくても動いたので、使用中のPCにもともとSTLinkソフトウェアが入っていたのかもしれない。。。今となっては確かめようがなくて。。。


## 5.5 Tockカーネルをビルド＆ハードウェアにダウンロード  

ここからは、ボード個別にReadmeが存在します。
https://github.com/tock/tock/blob/master/boards/README.md
ST Nucleo F446REを選択して、以下のページに飛べました。
https://github.com/tock/tock/blob/master/boards/nucleo_f446re/README.md
このとおりに行っていけばよさそうです。

ビルドだけ行う場合：
\$ cd tock/boards/nucleo_f446re
make
ビルド＆ダウンロード両方：
\$ cd tock/boards/nucleo_f446re
\$ make flash

[注意]
 - make flashをする前に、当然ですがnucleo実機とPCをUSBで接続していなければなりません。
 - 初回、nucleo実機と接続できない旨のエラーが出たので、一旦リセットを押しっぱなしにしてmake flashしました。当然エラーになりますが気にせず、リセットボタンを指から話して再度make flashするとうまくいきました。


## 5.6 ユーザランドをビルド＆ハードウェアにダウンロード  

ビルド：
\$ cd libtock-c/examples/buttons
\$ make

ダウンロード：
Nucleo F446REの場合、Makefileに変更が必要です。
APP, KERNEL, KERNEL_WITH_APPの定義を追記する必要があります。
buttonsサンプルをダウンロードしたい場合、以下のように記述します。
```
.PHONY: program
APP=../../../libtock-c/examples/buttons/build/cortex-m4/cortex-m4.tbf
KERNEL=$(TOCK_ROOT_DIRECTORY)/target/$(TARGET)/debug/$(PLATFORM).elf
KERNEL_WITH_APP=$(TOCK_ROOT_DIRECTORY)/target/$(TARGET)/debug/$(PLATFORM)-app.elf
program: $(TOCK_ROOT_DIRECTORY)target/$(TARGET)/debug/$(PLATFORM).elf
	arm-none-eabi-objcopy --update-section .apps=$(APP) $(KERNEL) $(KERNEL_WITH_APP)
	$(OPENOCD) $(OPENOCD_OPTIONS) -c "init; reset halt; flash write_image erase $(KERNEL_WITH_APP); verify_image $(KERNEL_WITH_APP); reset; shutdown"
```
その上で、以下を実行すればOK。
ダウンロードでは、Tockカーネル側のフォルダに行かないといけないのが注意ポイント。

\$ cd tock/boards/nucleo_f446re
\$ make program


## 5.7 動いた
動きました。
実機上の青いボタンを押すとLEDが光ります。
もう一度押すと、LED消えます。
これを繰り返します。
![](/images/article1-tock1/tock1_2.JPG)


## 5.8 参考：ログ出力
参考になるかどうかわかりませんが、ほぼ自分用のメモのため、出力されたログを書いておきます。

make flashの場合のログ出力：

```
:~/tock/boards/nucleo_f446re$ make flash
    Finished release [optimized + debuginfo] target(s) in 0.06s
   text	   data	    bss	    dec	    hex	filename
  88068	      0	  17028	 105096	  19a88	/home/#####/tock/target/thumbv7em-none-eabi/release/nucleo_f446re
openocd -f openocd.cfg -c "init; reset halt; flash write_image erase /home/#####/tock/target/thumbv7em-none-eabi/release/nucleo_f446re.elf; verify_image /home/#####/tock/target/thumbv7em-none-eabi/release/nucleo_f446re.elf; reset; shutdown"
Open On-Chip Debugger 0.11.0
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
DEPRECATED! use 'adapter driver' not 'interface'
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
Info : clock speed 2000 kHz
Info : STLINK V2J33M25 (API v2) VID:PID 0483:374B
Info : Target voltage: 3.255010
Info : stm32f4x.cpu: hardware has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32f4x.cpu on 3333
Info : Listening on port 3333 for gdb connections
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
target halted due to debug-request, current mode: Thread 
xPSR: 0x01000000 pc: 0x0800e634 msp: 0x20002000
Info : device id = 0x10006421
Info : flash size = 512 kbytes
Info : Flash write discontinued at 0x08015800, next section at 0x08040000
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
shutdown command invoked
```


make programの場合のログ出力：
```
:~/tock/boards/nucleo_f446re$ make program
    Finished dev [optimized + debuginfo] target(s) in 0.06s
   text	   data	    bss	    dec	    hex	filename
 137220	      0	  17048	 154268	  25a9c	/home/#####/tock/target/thumbv7em-none-eabi/debug/nucleo_f446re
arm-none-eabi-objcopy --update-section .apps=../../../libtock-c/examples/buttons/build/cortex-m4/cortex-m4.tbf /home/#####/tock//target/thumbv7em-none-eabi/debug/nucleo_f446re.elf /home/#####/tock//target/thumbv7em-none-eabi/debug/nucleo_f446re-app.elf
openocd -f openocd.cfg -c "init; reset halt; flash write_image erase /home/#####/tock//target/thumbv7em-none-eabi/debug/nucleo_f446re-app.elf; verify_image /home/#####/tock//target/thumbv7em-none-eabi/debug/nucleo_f446re-app.elf; reset; shutdown"
Open On-Chip Debugger 0.11.0
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
DEPRECATED! use 'adapter driver' not 'interface'
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
Info : clock speed 2000 kHz
Info : STLINK V2J33M25 (API v2) VID:PID 0483:374B
Info : Target voltage: 3.255010
Info : stm32f4x.cpu: hardware has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32f4x.cpu on 3333
Info : Listening on port 3333 for gdb connections
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
target halted due to debug-request, current mode: Thread 
xPSR: 0x01000000 pc: 0x0800b678 msp: 0x20002000
Info : device id = 0x10006421
Info : flash size = 512 kbytes
Info : Padding image section 1 at 0x08021800 with 124928 bytes
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
shutdown command invoked
```

