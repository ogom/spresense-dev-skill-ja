---
name: spresense-dev
description: Spresense 開発をターミナルで支援する。Use when creating an application root, importing SDK examples, creating C/C++ user apps, configuring NuttX, building, flashing to a Spresense board, opening the serial terminal, or updating Spresense app files such as Kconfig, Makefile, Make.defs, CMakeLists.txt, and *_main.c/cxx.
---

# Spresense 開発

## 基本方針

Spresense 開発では、まず作業環境を読み込み、SDK ルートとアプリケーションルートを確認してから変更する。

詳細なコマンドは必要になった時点で `references/command.md` を読む。
Kconfig、defconfig、SPI、アプリ自動起動などの設定を扱うときは `references/config.md` を読む。
サンプルの取り込み、`hello`、`led_blink`、新規アプリ、SPI/WS2812 の構成例が必要な時点で `references/examples.md` を読む。

## 作業フロー

1. `source ~/spresenseenv/setup` で Spresense ツールの PATH を読み込む。
2. `cd spresense/sdk` して、必要なら `source tools/build-env.sh` を実行する。
3. `spr-info` で `SPRESENSE_SDK`、`SPRESENSE_HOME`、シリアルポートを確認する。
4. アプリケーションルートが未設定なら `spr-create-approot ../../myproject "My Project"`、既存なら `spr-set-approot ../../myproject` を使う。
5. サンプルを使う場合は `spr-import-example <example>`、新規に作る場合は `spr-create-app <name>` または `spr-create-app <name> "Description" -x` を使う。
6. `spr-config <config>` で設定し、`spr-make` でビルドする。
7. ボードが必要な作業は、ポート確認後に `spr-flash`、`spr-terminal` で検証する。ユーザー承認が必要な環境では実行前に確認する。

## 実装時の注意

- ユーザーアプリは `SPRESENSE_HOME` 配下に置き、SDK 本体のサンプルは直接編集しない。
- `Kconfig`、`Makefile`、`Make.defs`、`*_main.c`、`*_main.cxx` の既存パターンに合わせる。`CMakeLists.txt` がある場合は同じ設定名へ揃える。
- `spr-create-app` で作るアプリ名は英数字と `_` のみにし、数字で始めない。
- C++ アプリは `spr-create-app <name> "Description" -x` を使う。
- 新規アプリをビルドしたらログの `Register: <name>`、`CXX: *_main.cxx` または `CC: *_main.c`、`out/*.spk` 生成を確認する。
- ビルドログはファイルに保存し、`tail` や `rg` で末尾とエラー行だけ確認する。`spr-make` の全ログを会話に流さない。
- ビルドエラーが出たら、最初に `SPRESENSE_HOME` と `spr-config` の対象、次に Kconfig のシンボル名、最後に include とリンク対象を確認する。
- `spr-distclean` や `spr-make distclean` は生成物を大きく消す可能性があるので、必要性を説明してから使う。

## 検証

ビルドだけで済む依頼は `spr-make` の成功と `out/*.spk` または `nuttx.spk` の生成を確認する。
実機検証を求められた場合は `spr-flash` 後に `spr-terminal` で NSH プロンプトを開き、対象コマンドを実行して出力や LED などの動作を確認する。
LED などの物理的な見た目を直接観察できない環境では、端末上の成功とユーザーが見る必要のある確認点を分けて報告する。
