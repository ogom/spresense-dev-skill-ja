# Spresense 設定

## spr-config

取り込んだ、または作成したユーザーアプリを有効化する。

```sh
spr-config userapp/hello
spr-config userapp/led_blink
spr-config userapp/my_app
```

`spr-config` は SDK の `tools/config.py` を使って NuttX の `.config` を作る。設定を変更したい場合は `spr-make menuconfig`、保存したい場合は `spr-make savedefconfig` を使う。

確認する場所:

- `SPRESENSE_HOME/configs/userapp/<app>/defconfig`: `spr-config userapp/<app>` の入力。
- `spresense/nuttx/.config`: `spr-config` 後に生成される実設定。
- `SPRESENSE_HOME/sdk.config`: アプリケーションルート側に保存された設定スナップショット。

## Kconfig とコマンド名

`spr-create-app hello` や `spr-import-example apps/examples/hello` の場合、Kconfig シンボルと実行コマンドはおおむね次の対応になる。

```text
USERAPP_HELLO=y
CONFIG_USERAPP_HELLO_PROGNAME="hello"
nsh> hello
```

アプリ名を変えたら、`USERAPP_<APPNAME>`、`CONFIG_USERAPP_<APPNAME>_PROGNAME`、`PROGNAME` の対応を確認する。

生成されたアプリで確認する設定ファイル:

- `Makefile`: `PROGNAME`、`PRIORITY`、`STACKSIZE`、`MAINSRC`、`CXXSRCS`。
- `Make.defs`: `CONFIG_USERAPP_<APPNAME>` のとき `CONFIGURED_APPS` に追加する。
- `Kconfig`: `USERAPP_<APPNAME>` と関連設定。
- `configs/userapp/<app>/defconfig`: `+USERAPP_<APPNAME>=y` を含める。
- `CMakeLists.txt`: 存在する場合は同じ設定名へ揃える。

`Kconfig` のアプリ本体は `default n` を保ち、設定は `spr-config` の defconfig で有効化する。NSH でコマンドが見つからない場合は、`CONFIG_NSH_BUILTIN_APPS=y` と `CONFIG_USERAPP_<APPNAME>=y` を確認して再ビルドする。

## SPI4 キャラクタデバイス

SPI4 の `/dev/spi4` をアプリから使う場合、`configs/userapp/<app>/defconfig` に次の設定を含める。

```text
+CXD56_SPI4=y
+CXD56_SPI_DRIVER=y
+SPI_DRIVER=y
+SPI_EXCHANGE=y
+SYSTEM_SPITOOL=y
```

`SYSTEM_SPITOOL` は `spi` コマンドだけでなく、Spresense の bringup で `/dev/spi4` を登録する条件にもなる。

## 自動起動を ON にする

Spresense 起動時にアプリを自動起動するには、NSH の ROMFS startup script を有効化する。例として `poc_max30102` を起動する場合:

1. `configs/userapp/poc_max30102/defconfig` に startup script 設定を追加する。

```text
+USERAPP_POC_MAX30102=y
+NSH_CUSTOMROMFS=y
+NSH_CUSTOMROMFS_HEADER="../../system/startup_script/nsh_romfsimg.h"
+NSH_ROMFSETC=y
+SYSTEM_STARTUP_SCRIPT=y
```

2. `spresense/sdk/system/startup_script/rcS.template` に起動コマンドを追加する。

```sh
echo "Auto start poc_max30102."
poc_max30102 &
```

バックグラウンド起動にするため `&` を付ける。前面で無限ループするアプリを `&` なしで起動すると、NSH の後続処理やプロンプト復帰を妨げる。

3. 設定を作り直してビルドする。

```sh
spr-config userapp/poc_max30102
spr-make
```

確認:

```sh
rg -n "CONFIG_NSH_ROMFSETC|CONFIG_SYSTEM_STARTUP_SCRIPT|CONFIG_USERAPP_POC_MAX30102" spresense/nuttx/.config
```

期待値:

```text
CONFIG_NSH_ROMFSETC=y
CONFIG_SYSTEM_STARTUP_SCRIPT=y
CONFIG_USERAPP_POC_MAX30102=y
```

`spresense/sdk/system/startup_script/nsh_romfsimg.h` が生成され、`spr-make` が成功すれば自動起動入りの `*.spk` が作られる。

生成された `nsh_romfsimg.h` で `romfs_img` の重複定義が出た場合は、壊れた生成ヘッダを削除して再ビルドする。

```sh
rm spresense/sdk/system/startup_script/nsh_romfsimg.h
spr-make
```

並列ビルドで再発する場合は、`spresense/sdk/system/startup_script/Makefile` に `.NOTPARALLEL:` を追加して、ROMFS ヘッダ生成を同時実行させない。

## 自動起動を OFF にする

アプリの自動起動を止める場合:

1. `configs/userapp/<app>/defconfig` から startup script 設定の行を削除する。

```text
+NSH_CUSTOMROMFS=y
+NSH_CUSTOMROMFS_HEADER="../../system/startup_script/nsh_romfsimg.h"
+NSH_ROMFSETC=y
+SYSTEM_STARTUP_SCRIPT=y
```

2. `spresense/sdk/system/startup_script/rcS.template` から対象アプリの起動行を削除する。

```sh
# 削除する例
poc_max30102 &
```

3. 設定を作り直してビルドする。

```sh
spr-config userapp/poc_max30102
spr-make
```

確認:

```sh
rg -n "CONFIG_NSH_ROMFSETC|CONFIG_SYSTEM_STARTUP_SCRIPT|CONFIG_USERAPP_POC_MAX30102" spresense/nuttx/.config
```

期待値:

```text
# CONFIG_NSH_ROMFSETC is not set
CONFIG_USERAPP_POC_MAX30102=y
```

startup script 機能を他用途で残したい場合は、defconfig の `NSH_ROMFSETC` 系は残し、`rcS.template` から対象アプリの起動行だけ削除する。
