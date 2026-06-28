# spresense-dev-skill-ja

Spresense の開発を Codex から進めるための skill です。
環境確認、SDK 取得、ボード接続、ブートローダー書き込み、アプリ作成、ビルド、書き込み、シリアルでの動作確認までを扱います。

## インストール

リポジトリをクローンします。

```sh
git clone https://github.com/ogom/spresense-dev-skill-ja.git
```

クローンしたフォルダを Codex や VS Code で開きます。

## 使い方

Codex に次のように依頼します。

```text
$spresense-dev を使って、myproject に examples/led_blink を作成して LED を光らせてください。
```

主な流れは次の通りです。

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

echo でコマンドを送る例

```sh
echo -e "led_blink" > /dev/cu.SLAB_USBtoUART
```

スキル本体は `.codex/skills/spresense-dev/` にあります。
詳細なコマンドは `references/command.md`、サンプル手順は `references/examples.md` を参照してください。

## Spresense CLI

任意のターミナルで動作を確認する流れは次の通りです。

```sh
source ~/spresenseenv/setup
source spresense/sdk/tools/build-env.sh
spr-make && spr-flash && spr-terminal
```

シリアルターミナルは `Ctrl-]` で閉じます。
