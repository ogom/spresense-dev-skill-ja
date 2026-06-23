# Spresense サンプル

## hello

実演で使った最小サンプル。アプリケーションルートを作り、`hello` を取り込んで有効化する。

```sh
source ~/spresenseenv/setup
cd spresense/sdk
source tools/build-env.sh
spr-create-approot ../../myproject "My Project"
spr-import-example apps/examples/hello
spr-config userapp/hello
spr-make
spr-flash
spr-terminal
```

端末で実行する。

```text
nsh> hello
Hello, World!!
```

`hello_main.c` の基本形:

```c
#include <nuttx/config.h>
#include <stdio.h>

int main(int argc, FAR char *argv[])
{
  printf("Hello, World!!\n");
  return 0;
}
```

## led_blink

メインボードの LED を順番に点灯させるサンプル。既存の SDK サンプルを取り込むのが最短。

```sh
source ~/spresenseenv/setup
cd spresense/sdk
source tools/build-env.sh
spr-set-approot ../../myproject
spr-import-example examples/led_blink
spr-config userapp/led_blink
spr-make
spr-flash
spr-terminal
```

端末で実行する。

```text
nsh> led_blink
```

実行後は LED が順番に点滅し、アプリは前面で動き続ける。シリアル端末にプロンプトが戻らない場合でも正常な挙動なので、確認後に `Ctrl+]` で端末を閉じる。

ビルド成功時はアプリケーションルートに次のような成果物ができる。

```text
myproject/led_blink/led_blink_main.c
myproject/configs/userapp/led_blink/defconfig
myproject/out/myproject.nuttx.spk
```

LED 制御でよく使う include と関数:

```c
#include <arch/board/board.h>
#include <arch/chip/pin.h>

board_gpio_write(pin, value);
board_gpio_config(pin, 0, false, true, PIN_FLOAT);
```

`led_blink` は `PIN_I2S1_BCK`、`PIN_I2S1_LRCK`、`PIN_I2S1_DATA_IN`、`PIN_I2S1_DATA_OUT` を LED 用の GPIO として使う。

## 新規 C アプリ

```sh
source ~/spresenseenv/setup
cd spresense/sdk
source tools/build-env.sh
spr-set-approot ../../myproject
spr-create-app sensor_log "Sensor Logger"
spr-config userapp/sensor_log
spr-make
```

生成されたアプリディレクトリで確認するファイル:

- `sensor_log_main.c`: エントリポイント。
- `Makefile`: `PROGNAME`、`PRIORITY`、`STACKSIZE`、`MAINSRC`。
- `Make.defs`: `CONFIG_USERAPP_SENSOR_LOG` のとき `CONFIGURED_APPS` に追加される。
- `Kconfig`: `USERAPP_SENSOR_LOG` と関連設定。
- `CMakeLists.txt`: 存在する場合は CMake ビルド対象。

## 新規 C++ アプリ

```sh
source ~/spresenseenv/setup
cd spresense/sdk
source tools/build-env.sh
spr-set-approot ../../myproject
spr-create-app cpp_demo "C++ Demo" -x
spr-config userapp/cpp_demo
spr-make
```

C++ 実装では、生成された `*_main.cxx` の構成に合わせる。
クラスやヘルパーを追加する場合も、まず `Makefile` と、存在する場合は `CMakeLists.txt` が対象ソースを拾うか確認する。
macOS 標準 bash で `bad substitution` が出た場合でも、アプリディレクトリだけ作られていることがある。
`Kconfig`、`Makefile`、`Make.defs`、`configs/userapp/<name>/defconfig` を確認し、`MYPROJECT_` などのアプリルート接頭辞が残っていれば `USERAPP_` に揃える。

## led_ws2812

SPI4 の MOSI で WS2812B を 8 個光らせる C++ アプリ例。

```sh
source ~/spresenseenv/setup
cd spresense/sdk
source tools/build-env.sh
spr-set-approot ../../myproject
spr-create-app led_ws2812 "WS2812 LED Demo" -x
spr-config userapp/led_ws2812
spr-make
spr-flash
spr-terminal
```

`configs/userapp/led_ws2812/defconfig` には、アプリ本体に加えて SPI4 のキャラクタデバイスを使う設定を含める。

```text
+USERAPP_LED_WS2812=y
+CXD56_SPI4=y
+CXD56_SPI_DRIVER=y
+SPI_DRIVER=y
+SPI_EXCHANGE=y
+SYSTEM_SPITOOL=y
```

`SYSTEM_SPITOOL` は `spi` コマンドだけでなく、Spresense の bringup で `/dev/spi4` を登録する条件にもなる。

アプリ側の最小 API は次の形にする。

```cpp
SPI_WS2812 led(8);
led.set_rgb(0, 32, 0, 0);
led.fill(0, 32, 0);
led.show();
led.clear();
```

実装の要点:

- `/dev/spi4` を `open()` し、`SPIIOC_TRANSFER` で送信する。
- SPI は 8 MHz、8 bit、`SPIDEV_MODE3` を使う。
- WS2812B は GRB 順で送る。
- 8 MHz では 0 bit を `0x60`、1 bit を `0x7c` に展開する。
- reset/latch 用に先頭へ 60 バイト以上のゼロを入れる。80 バイト程度にしておくと余裕がある。
- WS2812B は明るくなりやすいので、デモの初期輝度は低めにする。`wheel()` の 0..255 出力をそのまま出さず、`value / 12` 程度から始める。
- レインボーなどのアニメーションは無限ループだけにせず、`rainbow [rounds]` のように周回数を指定できる形にする。デフォルトは 10 周程度にして、実機確認では短い回数も指定できるようにする。

端末で実行する。

```text
nsh> led_ws2812
nsh> led_ws2812 rainbow
nsh> led_ws2812 rainbow 2
nsh> led_ws2812 fill 32 0 0
nsh> led_ws2812 clear
```

## Kconfig とコマンド名の対応

`spr-create-app hello` や `spr-import-example apps/examples/hello` の場合、Kconfig シンボルと実行コマンドはおおむね次の対応になる。

```text
USERAPP_HELLO=y
CONFIG_USERAPP_HELLO_PROGNAME="hello"
nsh> hello
```

アプリ名を変えたら、`USERAPP_<APPNAME>`、`CONFIG_USERAPP_<APPNAME>_PROGNAME`、`PROGNAME` の対応を確認する。

## うまく動かないとき

- `spr-info` で `SPRESENSE_HOME` が意図した `myproject` を指しているか確認する。
- `spr-config userapp/<name>` の `<name>` が `SPRESENSE_HOME/configs/userapp/<name>` と一致するか確認する。
- `Kconfig` の `default n` を保ち、設定は `spr-config` で有効化する。
- NSH でコマンドが見つからない場合は、`CONFIG_NSH_BUILTIN_APPS=y` と `CONFIG_USERAPP_<APPNAME>=y` を確認して再ビルドする。
- 実機検証が必要なときは `spr-set-port`、`spr-flash`、`spr-terminal` の順に確認する。
