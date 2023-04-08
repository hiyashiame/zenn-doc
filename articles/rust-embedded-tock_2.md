---
title: "”Tock embedded OS”の中をちょっと覗いてみた" # 記事のタイトル
emoji: "😀" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["組み込み", "rust", "RTOS"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

# 1. 前回までの試み

Tock embedded OS を試してみています。 
前回、STMのNucleo F446REボードを使って、LEDチカまで試してみることができましたが、スタートアップのところや割り込み系についてどのように動いているのか謎です。

公式GitHubにあるドキュメントを参照したり、
https://github.com/tock/tock/tree/master/doc
公式GitHubのソースコードを見たりして、
https://github.com/tock/tock
自分なりに理解した結果を書こうと思います。

なにかの参考になれば幸いです。

間違いがあるかもしれません。気がついた方ぜひ指摘してください。



# 2. 全体構成
Tockの構成について、公式GitHubに図がありますので、引用します。
![](https://github.com/tock/tock/blob/master/doc/tock-stack.png)

OS部：
Kernel部があって、そこにI/Oドライバも含まれます。内蔵I/Oアクセス部のみunsafe部。これは仕方ないですね。全体としてはtrustedです。
Cupsel部があって、そこには各種センサ制御部等がカプセル化されています。ここだけuntrustedだそうです。

ユーザランド部：
Syscall Intarfaceを介し、ユーザランドのプロセスが走ります。OS部とのやり取りは基本的にサービスコールとの事。 

SVC割り込みを発生させればサービスコールを呼べる、という仕組みなので、ユーザランド部がC言語であろうがRustであろうが気にしないという事。



# 3. Startup
どのように動作しているのか、気になります。
スタートアップについて読み解いていきます。

リセットベクタ等のベクタ定義：
　tock/chips/stm32f4xx/src/lib.rs 
　pub static BASE_VECTORS:でベクタ定義している。
　リセットベクタには「initialize_ram_jump_to_main」が割り当てられているので、リセット直後にまずこの関数に分岐する。
　ちなみにこのファイルには、割り込みベクタ処理も書かれている。

tock/arch/cortex-m/src/lib.rs 
　pub unsafe extern "C" fn initialize_ram_jump_to_main()本体が書かれている。
　見るからにリセットベクタっぽい。
　メモリのイニシャライズが終わると、bl mainでmain()に分岐する。

tock/boards/nucleo_f446re/src/main.rs
　pub unsafe fn main()が書かれている。
  tock/doc/Startup.md　に書かれている後半部分をつかさどる。
　PeripheralとCapsuleの初期化、Applicaton Startupを行い、最後にKernelのメイン処理となるスケジューラを起動する。

ちなみに、ドライバコールの際のDRIVER_NUMはここで定義している様子。（違う？）
```
/// Mapping of integer syscalls to objects that implement syscalls.
impl SyscallDriverLookup for NucleoF446RE {
    fn with_driver<F, R>(&self, driver_num: usize, f: F) -> R
    where
        F: FnOnce(Option<&dyn kernel::syscall::SyscallDriver>) -> R,
    {
        match driver_num {
            capsules::console::DRIVER_NUM => f(Some(self.console)),
            capsules::led::DRIVER_NUM => f(Some(self.led)),
            capsules::button::DRIVER_NUM => f(Some(self.button)),
            capsules::adc::DRIVER_NUM => f(Some(self.adc)),
            capsules::alarm::DRIVER_NUM => f(Some(self.alarm)),
            capsules::temperature::DRIVER_NUM => f(Some(self.temperature)),
            capsules::gpio::DRIVER_NUM => f(Some(self.gpio)),
            kernel::ipc::DRIVER_NUM => f(Some(&self.ipc)),
            _ => f(None),
        }
    }
}
```
# 4. システムコール
次は、ユーザランドからカーネルへのシステムコールのやり取りの方法について読み解いていきます。

## 4.1 ドライバコール
## 4.1.1 ユーザランド(非特権)からカーネル(特権)へ
libtock-c/examples/blink/main.cでLED点滅制御に使用している、led_on(i);を追いかけていきます。

libtock-c/libtock/led.c
　led_on()が記述されている。
　syscall_return_t rval = command(DRIVER_NUM_LEDS, 1, led_num, 0);をさらに追う。
　
libtock-c/libtock/tock.c
　command()が定義されている。
　svc命令をアセンブラで書き、SVCall例外を発生させ特権モードに切り替え、Kernelを起動し、内蔵I/Oをアクセスしてもらう。

```
syscall_return_t command(uint32_t driver, uint32_t command,
                         int arg1, int arg2) {
  register uint32_t r0 __asm__ ("r0") = driver;
  register uint32_t r1 __asm__ ("r1") = command;
  register uint32_t r2 __asm__ ("r2") = arg1;
  register uint32_t r3 __asm__ ("r3") = arg2;
  register uint32_t rtype __asm__ ("r0");
  register uint32_t rv1 __asm__ ("r1");
  register uint32_t rv2 __asm__ ("r2");
  register uint32_t rv3 __asm__ ("r3");
  __asm__ volatile (
    "svc 2"
    : "=r" (rtype), "=r" (rv1), "=r" (rv2), "=r" (rv3)
    : "r" (r0), "r" (r1), "r" (r2), "r" (r3)
    : "memory"
    );
  syscall_return_t rval = {rtype, {rv1, rv2, rv3}};
  return rval;
}
```
 ちなみに当然ながら、RISC-V用コードも存在し、IFDEFで切り替えています。
```
x syscall_return_t command(uint32_t driver, uint32_t command,
                         int arg1, int arg2) {
  register uint32_t a0  __asm__ ("a0") = driver;
  register uint32_t a1  __asm__ ("a1") = command;
  register uint32_t a2  __asm__ ("a2") = arg1;
  register uint32_t a3  __asm__ ("a3") = arg2;
  register uint32_t a4  __asm__ ("a4") = 2;
  register int rtype __asm__ ("a0");
  register int rv1 __asm__ ("a1");
  register int rv2 __asm__ ("a2");
  register int rv3 __asm__ ("a3");
  __asm__ volatile (
    "ecall\n"
    : "=r" (rtype), "=r" (rv1), "=r" (rv2), "=r" (rv3)
    : "r" (a0), "r" (a1), "r" (a2), "r" (a3), "r" (a4)
    : "memory");
  syscall_return_t rval = {rtype, {rv1, rv2, rv3}};
  return rval;
}
```
## 4.1.2 SVCall例外発生時の処理
SVCall例外ベクタでの処理を追っていきます。

tock/arch/cortex-m/src/lib.rs
SVCall例外ハンドラが記述されている。
pub unsafe extern "C" fn svc_handler_arm_v7m() 
ここでは、リンクレジスタの値が0xfffffff9であればカーネルからユーザへの復帰と判断。[注１]
でなければユーザからのカーネルコールと判断。
今回はユーザからのカーネルコールである。

次に、SYSCALL_FIREDフラグに1をたて、リンクレジスタに0xfffffff9と書いてbx命令を発行。
こうすることで、メインスタックに積まれた復帰アドレスにジャンプできる。
カーネルの復帰点、入口と理解すればよい。

※ソースコード内で具体的にカーネルの復帰点がどこかについては見つけられなかった。。。

[注１]の補足：
次回SVCall例外が発生すると、リンクレジスタにはここで設定した0xfffffff9が残ったままこのルーチンに入る。
次回のSVCall例外、というのは、カーネルからユーザへの復帰の時になる。
すなわち、リンクレジスタ値＝0xfffffff9であればユーザへの復帰と判断し、リンクレジスタ値＝0xfffffffdに変更して bx lr命令を実行する。そうするとユーザスタックに積まれた復帰アドレスにジャンプできる。

## 4.1.3 ドライバコール処理
カーネルに復帰し、カーネルループ処理が動作しますが、それからどのようにドライバがコールされるのかを追ってみました。

tock/kernel/src/kernel.rs
カーネルループ処理: kernel_loop_operation()の中のdo_process()に入る。
context_switch_reasonを取得するために switch_to()関数に入る。

tock/kernel/src/process_standard.rs 
switch_to()関数からswitch_to_process関数に入る。

tock/arch/cortex-m/src/syscall.rs
switch_to_process()に遷移し、SYSCALL_FIREDフラグが立っていればシステムコールを行うための準備をする。
（SVC番号によってシステムコール種別を取得する、指定されたArgument値を取得する、等）

そしてdo_process()に戻り、handle_syscall()をコールする。
handle_syscall()内でそれぞれのシステムコールに相当する処理が動く。

該当するコード：
```
　Some(d) => d.command(subdriver_number, arg0, arg1, process.processid()),
```
libtock-c/libtock/led.cの
command(DRIVER_NUM_LEDS, 1, led_num, 0);
でコールされるドライバの本体は、TOCKのcapsulesにある。
tock/capsules/src/led.rs

## 4.2 コールバック
## 4.2.1 ユーザランド(非特権)からカーネル(特権)へ
割り込みが発生した場合にユーザランドに通知するコールバックはどのように実装されているのかを読み解いてみます。

具体的には、
libtock-c/examples/buttons/main.cで、
err = button_subscribe(button_callback, NULL);
と設定し、ボタン押下によって
```
static void button_callback(int                            btn_num,
                            int                            val,
                            __attribute__ ((unused)) int   arg2,
                            __attribute__ ((unused)) void *ud) {
```

がコールされるために、どのように動いているかについて調べていきます。

## 4.2.1 SVCall例外発生時の処理

libtock-c/libtock/button.c
　button_subscribe()が記述されている。
　 subscribe_return_t res = subscribe(DRIVER_NUM_BUTTON, 0, callback, ud);をさらに追う。


libtock-c/libtock/tock.c
　subscribe()が定義されている。
　svc命令をアセンブラで書き、SVCall例外を発生させ特権モードに切り替え、Kernelを起動し、ハンドラを設定してもらう。
```
subscribe_return_t subscribe(uint32_t driver, uint32_t subscribe,
                             subscribe_upcall cb, void* userdata) {
  register uint32_t r0 __asm__ ("r0") = driver;
  register uint32_t r1 __asm__ ("r1") = subscribe;
  register void*    r2 __asm__ ("r2") = cb;
  register void*    r3 __asm__ ("r3") = userdata;
  register int rtype __asm__ ("r0");
  register int rv1 __asm__ ("r1");
  register int rv2 __asm__ ("r2");
  register int rv3 __asm__ ("r3");
  __asm__ volatile (
    "svc 1"
    : "=r" (rtype), "=r" (rv1), "=r" (rv2), "=r" (rv3)
    : "r" (r0), "r" (r1), "r" (r2), "r" (r3)
    : "memory");

  if (rtype == TOCK_SYSCALL_SUCCESS_U32_U32) {
    subscribe_return_t rval = {true, (subscribe_upcall*)rv1, (void*)rv2, 0};
    return rval;
  } else if (rtype == TOCK_SYSCALL_FAILURE_U32_U32) {
    subscribe_return_t rval = {false, (subscribe_upcall*)rv2, (void*)rv3, (statuscode_t)rv1};
    return rval;
  } else {
    exit(1);
  }
}
```

## 4.2.2 SVCall例外発生時の処理
SVCall例外ベクタでの処理は、4.1.2に記述した通りなので省略。

## 4.2.3 subscribe処理
4.1.3 ドライバコール処理と途中まで同様。
最後のhandle_syscall()内でそれぞれのシステムコールに相当する処理が動く箇所、ここでコールバック関数情報が登録される。

該当するコード：
```
　let upcall = Upcall::new(process.processid(), upcall_id, appdata, ptr);
```
ここで、Upcallを行う情報を登録する。

## 4.2.4 実際に割り込みが入った場合
実際にボタンが押下され、IRQ割り込みが発生した場合、
ベクタに飛んでくる。
```
pub unsafe extern "C" fn generic_isr_arm_v7m() 
```

このベクタ関数内で、リンクレジスタに0xfffffff9を設定している。
```
// This is a special address to return Thread mode with Main stack
movw LR, #0xFFF9
movt LR, #0xFFFF
```
こうすることでbx命令実行でメインスタックに積まれた復帰アドレスにジャンプする。
すなわち、カーネルに復帰点に遷移し、4.1.3と同様、カーネルループ処理が動作する。
ちなみにbx命令実行する前に、割り込み番号など必要な情報をレジスタに保存している。

カーネルループ処理の途中、
tock/arch/cortex-m/src/syscall.rsのswitch_to_process()で、
コンテキストスイッチの理由として割り込み発生と設定する。

```
kernel::syscall::ContextSwitchReason::Interrupted
```
そしてdo_process()に戻り、スケジューラにすぐに実行してもらいたい割り込みタスクがあるということを連絡し、スケジューラに、割り込み要因に紐付けされたUpcall関数を起動してもらう。
（この辺あやふや。詳しい方訂正＆補足あればぜひ連絡ください。）


# 5 最後に
カーネルスケジューラの辺り、詳しくつっこんでいくうちに気力が続かなくなりましたが、概ね割り込み系でどう動いているかはわかったような気がします。
また新たな力が湧いてきたら、再チャレンジするかもしれません。
今は、どっちかというと、このOSを使ってみたい方に興味が動いてしまいました。
（あぁなんて飽きっぽいのだろうか。。。）

いろいろ使ってみよう！と思っているので、なにか進展したらまたZennにアップして、せっかく始めたアウトプット活動は絶やさないようにしたいです。






