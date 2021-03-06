rza1hを使ったボードで、dipswのOff/ONを、ユーザランド側で検知できるようにしたい。  
そのための流れ。

* 全体の流れ
	* GPIOの特定
	* 兼用機能、ポートモード
	* ピン設定
	* EDGE,LEVEL
	* 割り込みまでの道筋

* 実際の修正
	* gpio-keysドライバについて
	* デバイスツリーの編集


## 使用するGPIOの特定
SoCからは、外部とやりとりするための信号線が伸びている。
その末端はピンと呼ばれ、ユーザが自由に使えるピンをGPIOと呼ぶ。

SoCによるが、各GPIOには、それぞれ数種類の機能が割り振られていて、ユーザは、その中から使いたいものを選ぶことができる。
例えば、以下はRZA1Hマニュアルの、ポート1_0に関する記載。
![rza1h-gpios](https://github.com/mozomozo101/tech_memo/blob/master/images/rza1-gpios.png)

RIIC0SCL: RIIC0シリアルクロック入出力端子  
IRQ0: IRQ0端子  

RZA1の場合、GPIOポートは２通りのモードで使える。
* 兼用モード
  * 第1〜8までの機能があり、その中から選んで使用する
  * 第1機能を選ぶと、RIIC0シリアルクロック入出力端子へ入出力されるようになる
  * 第4機能では、そのピンへの入力は、IRQ0としてCPUに通知されるようになる
* ポートモード
  * 入出力の内容を、ユーザが自由に決められる

ここで、手元のボードでどのGPIOを使うか考える。
ボードのマニュアルによると、dipsw1(4)は、GPIO0_5に繋がっている。
マニュアルでの記載は以下の通り。
![rza1h-port0](https://github.com/mozomozo101/tech_memo/blob/master/images/rza1-port0.png)

GPIO0_5には、IRQ機能が割り当てられていない。
そのため、ポートモードとして設定して、そこへ入力があったら、割り込みとして通知するようにする。

## 割り込みまでの流れ
じゃあ、どうやって割り込みを上げるんだろう？
rza1のマニュアルを見ると、このように書いている。

```
端子割り込みは、TINT170 ～ TINT0 端子による割り込みです。
TINT170～TINT0端子は、汎用入出力ポートのモード/機能に関わらず、ポートからの入力信号を割り込みとして伝えます。
そのため、ポートモードで端子割り込みを使用する際は、ポート入力に設定してください。
```
実は、RZA1では、割り込みは4種類ある（詳しくはマニュアル参照）。

* NMI割り込み
* IRQ割り込み
* 内蔵周辺モジュール割り込み
* 端子割り込み

IRQ割り込みは、IRQ0〜7端子からの入力による割り込みで、GPIOの兼用モードに用意されるIRQnがこれに相当する。
で、端子割り込みは、GPIOからの割り込み。
「端子割り込みでは、ポートからの入力信号を割り込みとして伝える」とのことなので、

* ポートモードに設定
* ポート入力に設定

という設定しておけば、勝手に割り込みが上がるってわけ。
この設定は、デバイスツリーに書くか、ボードの初期化関数内で行う。

## 割り込み番号
割り込みが発生すると、その割り込みIDがCPUに通知される。
この割り込みのIDは、SoC依存。
rza1の場合、マニュアル7.5章に書かれており、423番とのこと。

## 割り込みハンドラ
今までで、dipswの変化により割り込みが発生し、CPUには、割り込みIDとして423番が通知されることが分かった。
ここで、request_irq()などを使い、割り込みベクタの423番に割り込みハンドラを登録するようなカーネルモジュール（ドライバ）を作ってロードしておく。
そうすれば、dipswが変化した時に、対応する割り込みハンドラが呼ばれるようになる。

割り込みハンドラのボトムハーフの実装は、カーネルスレッドとしてworkqueueを発行するのが一般的。
詳しくは、[こちら](https://github.com/mozomozo101/tech_memo/blob/master/kernel/%E5%89%B2%E3%82%8A%E8%BE%BC%E3%81%BF%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6.md)。

# 既存のドライバを使う

## Linuxソースツリーのドライバ
作成する割り込みハンドラがhello worldを表示する程度であれば、ドライバはいらない。
しかし実践面では、デバイスドライバを登録しておき、割り込みを上げたデバイスに関する情報をもとにゴニョゴニョすることが多い。
そのようなデバイスドライバを１から自前で書くのは大変。
そのため、Linuxカーネルソースツリーには、様々なデバイスドライバが、[driver/](https://elixir.bootlin.com/linux/latest/source/drivers) に用意されている。
これらのデバイスドライバは、それぞれにデバイスツリーの書き方が決まっており、それは全て[ドキュメント化](https://www.kernel.org/doc/Documentation/devicetree/bindings/)されている。

今回は、dipswのOFF/ONの変化を、ユーザランドで検知してみたい。
その目的に沿ったドライバとして、[gpio_keys](https://elixir.bootlin.com/linux/latest/source/drivers/input/keyboard/gpio_keys.c) を使っていこうと思う。

## 割り込みの通知
dipswの変化による割り込みをgpio_keysドライバに通知するようにしたい。
そのため、割り込み番号423番とgpio_keysドライバを対応させる。
これはデバイスツリーから行う。
gpio_keysデバイス用のデバイスツリーの書き方は、[ここ](https://www.kernel.org/doc/Documentation/devicetree/bindings/input/gpio-keys.txt)に記載されている。

gpio-keysノード配下に、今回の目的であるdipsw用のノード dipsw を作成した。

```
       gpio-keys {
               compatible = "gpio-keys";
               status = "okay";
               dipsw {
                       label = "WPS";
                       linux,code = <0x211>;
                       interrupt-parent = <&gic>;
                       interrupts = <GIC_SPI (423-32) IRQ_TYPE_EDGE_RISING>;
               };
       };
```

* interrupts  
interruptsは、割り込みに関する情報で、各セルに何を書くかは割り込みコントローラ依存。  
上記は、[rza1hで使用している割り込みコントローラのドキュメント](https://www.kernel.org/doc/Documentation/devicetree/bindings/interrupt-controller/arm%2Cgic.txt)を元に作成した。

* linux,code  
gpio-keysは、キーボード等のデバイスとしての仕様を想定している。
linux,codeは、何のキーを押されたかを表すもので、そのコードは、[ここ](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/input-event-codes.h)で定義されている。
（0x211は、WPSボタンが押されたことを示すコード。）





 
