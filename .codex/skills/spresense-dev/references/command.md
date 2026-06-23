# Spresense コマンド

## 環境の読み込み

ローカルの Spresense ツールを PATH に追加する。

```sh
source ~/spresenseenv/setup
```

SDK コマンドを使うときは SDK ディレクトリへ移動し、ビルド環境を読み込む。

```sh
cd spresense/sdk
source tools/build-env.sh
```

## zsh 互換の注意

zsh で `source tools/build-env.sh` がうまく動かない場合は、`tools/build-env.sh` を zsh でも source できるように修正する。

- `BASH_SOURCE[0]` だけに依存しない。zsh では `${(%):-%x}` で source されたファイル名を取る。
- `SCRIPT_NAME` と `SCRIPT_DIR` は空白を考えてクォートする。
- `.spresense_env` は `"${SPRESENSE_SDK}/.spresense_env"` のようにクォートして source する。
- bash completion は bash のときだけ読み込む。`BASH_VERSION` が空なら読み込まない。
- zsh の `[` では `==` や `"$#"` が問題になることがある。単一ブラケットでは `=` を使い、引数数は `$#` をクォートせず数値比較する。
- zsh では `FUNCNAME[0]` が空になることがある。ヘルプ表示の関数名が重要なら、bash と zsh の両方で使える関数名取得を用意する。

確認:

```sh
zsh -f -c 'source ~/spresenseenv/setup && cd spresense/sdk && source tools/build-env.sh >/tmp/spr-env-zsh.out && spr-info'
bash --noprofile --norc -c 'source ~/spresenseenv/setup && cd spresense/sdk && source tools/build-env.sh >/tmp/spr-env-bash.out && spr-info'
```

## 状態確認

```sh
spr-info
```

確認する値:

- `SPRESENSE_SDK`: SDK ルート。
- `SPRESENSE_HOME`: ユーザーアプリのアプリケーションルート。
- `SPRESENSE_PORT`: `spr-flash` と `spr-terminal` が使うシリアルポート。
- `SPRESENSE_BAUD`: `spr-flash` のボーレート。未設定なら既定値を使う。

## アプリケーションルート

新規作成して現在のアプリケーションルートに設定する。

```sh
spr-create-approot ../../myproject "My Project"
```

既存のアプリケーションルートを選ぶ。

```sh
spr-set-approot ../../myproject
```

現在のアプリケーションルート設定を外す。

```sh
spr-unset-approot
```

## ユーザーアプリ作成

C アプリを作成する。

```sh
spr-create-app my_app "My Application"
```

C++ アプリを作成する。

```sh
spr-create-app my_cpp_app "My C++ Application" -x
```

制約:

- アプリ名は `A-Z`、`a-z`、`0-9`、`_` のみ。
- アプリ名は数字で始めない。
- 説明文に二重引用符を含めない。

生成後に次を確認する。

- `Kconfig`、`Makefile`、`Make.defs` が同じ `USERAPP_<APPNAME>` シンボルを参照している。
- `CMakeLists.txt` がある場合は同じシンボルを参照している。
- `configs/userapp/<app>/defconfig` が存在し、`+USERAPP_<APPNAME>=y` を含む。
- `spr-make` ログに `Register: <app>` が出る。出ない場合は `Make.defs` の条件が古いシンボル名のままになっていないか確認する。

## サンプルの取り込み

SDK 側のサンプルをアプリケーションルートにコピーする。

```sh
spr-import-example apps/examples/hello
spr-import-example examples/led_blink
```

取り込み後は `SPRESENSE_HOME` 配下のコピーを編集する。

## 設定

取り込んだ、または作成したユーザーアプリを有効化する。

```sh
spr-config userapp/hello
spr-config userapp/led_blink
spr-config userapp/my_app
```

`spr-config` は SDK の `tools/config.py` を使って NuttX の `.config` を作る。設定を変更したい場合は `spr-make menuconfig`、保存したい場合は `spr-make savedefconfig` を使う。

## ビルド

```sh
spr-make
```

代表的な make ターゲット:

```sh
spr-make clean
spr-make distclean
spr-make menuconfig
spr-make savedefconfig
```

ビルド成功後は `SPRESENSE_HOME/out/*.spk` または SDK 側の `nuttx.spk` を確認する。

## 書き込みとシリアル端末

ポートを設定する。

```sh
spr-set-port /dev/tty.usbserial-XXXX
spr-set-baud 115200
```

ファームウェアを書き込む。

```sh
spr-flash
```

書き込み成功の目安:

```text
Package validation is OK.
Saving package to "nuttx"
Restarting the board ...
```

シリアル端末を開く。

```sh
spr-terminal
```

macOS や Codex のサンドボックス環境で `Operation not permitted`、`Cannot open port` が出た場合は、次を確認する。

- `spr-info` の `SPRESENSE_PORT` と `/dev/cu.*` の存在。
- 別の `spr-terminal` やシリアルアプリがポートを掴んでいないか。
- 実機端末を開く操作にユーザー承認が必要か。

シリアル端末は `tty` 付きで起動し、NSH へコマンドを送ったあとは `Ctrl+]` で閉じる。

NSH プロンプトが表示されたら、ビルドしたアプリ名を実行する。

```text
nsh> hello
nsh> led_blink
nsh> led_ws2812
```

`led_blink` やレインボー表示のように前面で動き続ける LED アプリは、点灯や起動メッセージ、エラー有無を確認したら miniterm の終了キー `Ctrl+]` で端末を閉じる。
ボード上の処理を止めたい場合はアプリ側の停止操作、`clear` コマンド、またはボードリセットを使う。
端末を閉じる操作は環境に依存する。実行中に戻れない場合は、起動した端末ツールの終了キーを確認する。
LED の見た目を直接観察できない場合は、端末出力とプロンプト復帰だけを実機実行結果として報告し、明るさや回転方向などの目視確認はユーザーに委ねる。

## クリーンアップ

SDK アプリ配下の生成物を掃除する。

```sh
spr-distclean
```

確認なしで掃除する。

```sh
spr-distclean -f
```

`spr-distclean` は `sdk/apps` の untracked/ignored ファイルを削除し、必要に応じて `spr-make distclean` も実行する。
ユーザーが作った作業を消さないよう、実行前に対象を確認する。

## 移動

```sh
spr-go-sdk
spr-go-approot
```

SDK ルート、アプリケーションルートへ移動するための補助コマンド。
