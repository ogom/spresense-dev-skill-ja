---
name: spresense-dev
description: Spresense の開発環境の準備、SDK の取得、ボード接続確認、ブートローダー書き込み、アプリルート作成、C/C++アプリやサンプルの追加、設定、ビルド、書き込み、シリアル動作確認を行う。Spresense の初期セットアップ、hello や led_blink の実行、SPI4を使うWS2812B制御、LEDの再点灯、ビルドやフラッシュの再実行を依頼されたときに使用する。
---

# Spresense 開発

Spresense のセットアップから実機確認までを、状態を確認しながらターミナルで進める。GUI のクリックや座標は再現せず、可能な限りコマンドを直接実行する。

## 基本方針

- 最初に現在のディレクトリ、既存の `mise.toml`、`spresense/`、アプリルート、生成物を確認し、完了済みの処理を重複させない。
- ユーザーの既存ファイルを上書きしない。特に `mise.toml` がある場合は内容を保ち、必要な差分だけを提案または編集する。
- SDK の保存先、アプリ名、サンプル、シリアルポートは固定値にせず、依頼内容と現在の環境から決める。
- USB デバイスが複数ある場合は候補を示し、ユーザーに選択を求める。
- ブートローダーやアプリの書き込みは実機の状態を変更する。対象ポートと実行内容を示し、直前にユーザーの確認を得る。
- 各段階で終了コードと期待する生成物を確認し、失敗時は先へ進まずに原因を診断する。

## 1. 前提を確認する

Spresense ボード、USB 接続、`mise`、Spresense CLI 環境を確認する。プロジェクトで Python の指定が必要な場合は、既存設定を保ちながら次の状態にする。

```toml
[tools]
python = "3"
```

設定後に信頼と Python を確認する。

```sh
mise trust
python -V
```

Spresense CLI 環境を読み込む。ファイルがない場合は推測して作らず、ユーザーに導入先を確認する。

```sh
source ~/spresenseenv/setup
```

`arm-none-eabi-gcc` が既に PATH にあっても、`kconfig-conf` などの補助ツールが不足する場合がある。`build-env.sh` 任せにせず、先に `~/spresenseenv/setup` を明示的に読み込む。

## 2. SDK を準備する

SDK がない場合だけ、サブモジュールを含めて取得する。ネットワークアクセスが必要であることを事前に伝える。

```sh
git clone --recursive https://github.com/sonydevworld/spresense.git
```

既存の `spresense/` がある場合は再 clone せず、リポジトリとサブモジュールの状態を確認する。SDK のシェル環境を現在のセッションへ読み込む。

```sh
source ~/spresenseenv/setup
source spresense/sdk/tools/build-env.sh
command -v arm-none-eabi-gcc
command -v kconfig-conf
```

以降の `spr-*` コマンドは、この環境を読み込んだ同じシェルで実行する。

zsh を使う場合は、`build-env.sh` が zsh から source されたファイル自身の位置を解決し、`.spresense_env` を SDK ルートから読み込めることを確認する。`SPRESENSE_SDK`、`SPRESENSE_HOME`、`SPRESENSE_PORT` の表示値が意図どおりでなければ先へ進まない。`= not found` や `compgen: command not found` が出た場合は bash 固有の構文が残っているため、互換性を修正するか対応するシェルへ切り替える。macOS標準の古いbashでは `${name^^}` も使えないため、シェルを変えれば必ず解決すると決めつけず、失敗地点と生成済みファイルを確認する。

## 3. ボードを確認する

`/dev/cu.*` を列挙して Spresense のシリアルポートを特定し、選択した値を設定する。記録時の例は `/dev/cu.SLAB_USBtoUART` だが、環境ごとに必ず確認する。

```sh
spr-set-port <serial-port>
spr-info
```

`spr-info` は SDK、アプリルート、ポートなどの環境情報だけを表示し、実機通信は行わない。接続確認が目的なら `spr-terminal` でポートを実際に開き、NuttShellのバージョンと `nsh>` の応答を確認して `Ctrl-]` で閉じる。

## 4. 必要な場合だけブートローダーを書き込む

初期セットアップまたは更新を依頼された場合だけ実行する。対象ポートを再確認し、ユーザーの明示的な了承を得てから進める。

```sh
spr-flash -B
```

対話プロンプトを読み、内容が想定どおりの場合だけ承認する。成功を確認するまで USB を抜かず、アプリ作成へ進まない。

## 5. アプリを作成してビルドする

アプリルートがない場合は、保存先と表示名を決めて作成する。

```sh
spr-create-approot <app-root-path> "<display-name>"
```

hello サンプルを使う場合は、取り込み、設定、ビルドを順に行う。

```sh
spr-import-example apps/examples/hello
spr-config userapp/hello
spr-make
```

LED 点滅サンプルを使う場合は、SDK 直下のサンプルと対応するユーザー設定を使う。

```sh
spr-import-example examples/led_blink
spr-config userapp/led_blink
spr-make
```

独自のC++アプリを作る場合は `-x` を指定する。

```sh
spr-create-app <app-name> "<description>" -x
```

作成後は、`Makefile`、`Make.defs`、`Kconfig`、`<app-name>_main.cxx`、`configs/userapp/<app-name>/defconfig` を確認する。コマンドが途中で失敗してもディレクトリだけ作成済みの場合があるため、同じコマンドを重ねず、次を確認して不足分だけ直す。

- 設定名を `USERAPP_<APP_NAME>` に統一する。
- `Kconfig` の既定値を `n` にする。
- C++の追加実装を `CXXSRCS` へ登録する。
- `configs/userapp/<app-name>/defconfig` に `+USERAPP_<APP_NAME>=y` を置く。
- ユーザーがコード表記を具体的に指定した場合は、等価な定数置換で済ませず、最終ソースが指定どおりか文字列検索で照合する。

別のサンプルや既存アプリを使う依頼では、ソースパスと設定名を対応するターゲットへ置き換える。ビルド後は、アプリルートの `out/` に `.spk` が生成されたことと、`sdk.config` で対象の `CONFIG_USERAPP_*` が有効化されていることを確認する。

設定またはビルドが途中で失敗して `Try 'make distclean' first` と表示された場合は、ツールの PATH を直してから次の順序で再実行する。

```sh
spr-make distclean
spr-config <config>
spr-make
```

## 6. SPI4でWS2812Bを制御する

SPI4をユーザーアプリから `/dev/spi4` として使う場合は、アプリ設定に少なくとも次を含める。

```text
+USERAPP_LED_WS2812=y
+SYSTEM_SPITOOL=y
+SPITOOL_MAXBUS=5
+CXD56_SPI4=y
+SPI_EXCHANGE=y
```

`SYSTEM_SPITOOL` は Spresense のボード初期化で `/dev/spi4` を登録するためにも必要になる。ビルド後は `sdk.config` で `CONFIG_USERAPP_LED_WS2812`、`CONFIG_SYSTEM_SPITOOL`、`CONFIG_CXD56_SPI4`、`CONFIG_SPI_EXCHANGE` を確認する。

WS2812Bの800kHz相当波形をSPIで生成する場合は、SPI4を2.4MHz、モード0、8ビット幅に設定し、データ1ビットを3ビットへ符号化する。

- `0` を `100` に符号化する。
- `1` を `110` に符号化する。
- 色データをWS2812BのGRB順で送る。
- 送信の前後に80µs以上のLow期間を設ける。
- 転送後にも80µs程度待ってラッチを確実にする。

8個のLEDを扱う依頼では、ユーザー指定どおり `SPI_WS2812 led(8)` を作り、個別色の `set_rgb`、一括転送の `show`、全LEDを同色にする `fill`、全消灯する `clear` を実装する。レインボーの連続実行に加え、実機確認しやすいコマンドも用意する。

```text
led_ws2812
led_ws2812 fill 255 0 0
led_ws2812 clear
```

起動メッセージとSPI転送エラーがないことはソフトウェア経路の確認であり、実際の発光を直接証明するものではない。発光しない場合は、SPI4 MOSIへのDIN接続、LEDの5V電源、GND共通化、信号電圧レベル、LEDの向きを確認する。

## 7. アプリを書き込んで動作確認する

対象ポートとビルド生成物を示し、ユーザーの了承を得てから書き込む。

```sh
spr-flash
```

成功後にシリアルターミナルを開く。

```sh
spr-terminal
```

端末が開いたら、NSHプロンプトでアプリ名を実行する。
`hello` サンプルでは NSH プロンプトに次を入力、期待する応答を確認する。

```text
hello
```

終了には `Ctrl-]` を使う。確認結果、使用ポート、生成した `.spk` の場所を簡潔に報告する。

`led_blink` サンプルでは NSH プロンプトに次を入力する。

```text
led_blink
```

このサンプルは 4 個の LED を 100ms 間隔で順送りする無限ループであり、通常は文字出力も NSH プロンプトへの復帰もない。
LED の点滅を動作確認とし、`Ctrl-]` でシリアル接続だけを閉じる。ボード上の処理はリセットまたは電源を切るまで継続する。

ポートファイルが存在するのに `Cannot open port` が出た場合は、ボード不在と即断しない。
別プロセスによる占有と実行環境のデバイス権限を確認し、必要ならユーザーの承認を得て、デバイスアクセス権限付きで同じ書き込みまたはターミナル操作を再実行する。

## 再実行時の分岐

- 環境準備済み: `source ~/spresenseenv/setup` と `source tools/build-env.sh` から再開する。
- SDK 取得済み: clone を省略し、サブモジュールと作業ツリーを確認する。
- アプリ作成済み: `spr-create-approot` とサンプル取り込みを省略し、設定またはビルドから再開する。
- ビルド済み: ソースの変更と `.spk` の時刻を確認し、必要な場合だけ `spr-make` を行う。
- 書き込みだけの依頼: ボード、ポート、対象 `.spk` を確認して `spr-flash` へ進む。
- 同じ LED ファームウェアをもう一度動かす依頼: 再ビルドと再書き込みを省略し、`spr-terminal` で接続して `led_blink` を再実行する。
- 同じ WS2812B ファームウェアをもう一度動かす依頼: 再ビルドと再書き込みを省略し、`spr-terminal` で接続して `led_ws2812`、`fill`、`clear` の必要なコマンドだけを実行する。
