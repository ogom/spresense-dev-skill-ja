# spresense-dev-skill-ja

Spresense の開発を Codex から進めるための skill です。
環境確認、SDK 取得、ボード接続、ブートローダー書き込み、アプリ作成、ビルド、書き込み、シリアルでの動作確認までを扱います。

## 内容

- `.codex/skills/spresense-dev/SKILL.md`: Spresense 開発手順と注意点
- `.codex/skills/spresense-dev/agents/openai.yaml`: Codex 上の表示名と初期プロンプト

## インストール

このリポジトリの `.codex` ディレクトリを Codex の設定ディレクトリへ配置します。

```sh
cp -R .codex ~/.codex
```

既存の `~/.codex` がある場合は、必要な skill だけをコピーしてください。

```sh
cp -R .codex/skills/spresense-dev ~/.codex/skills/
```

## 使い方

Codex で次のように依頼します。

```text
$spresense-dev を使って、myproject に hello サンプルを追加し、ビルドしてください。
```

## LED 点滅の例

```text
$spresense-dev を使って、myproject に examples/led_blink を作成し、ビルドして実機へ書き込み、LED を点滅させてください。
```

内部では主に次の処理が行われます。

```sh
source ~/spresenseenv/setup
source spresense/sdk/tools/build-env.sh
spr-import-example examples/led_blink
spr-config userapp/led_blink
spr-make
spr-flash
spr-terminal
```

NSH で `led_blink` を実行すると、4 個の LED が 100ms 間隔で順番に点滅します。
シリアルターミナルを `Ctrl-]` で閉じても、ボードをリセットまたは電源を切るまで処理は継続します。

同じファームウェアをもう一度動かす場合は、短い依頼で済みます。

```text
$spresense-dev を使って、LED をもう一度光らせてください。
```

## 注意

spresense は bash のスクリプトなので、zsh でも動作するように調整します。

```text
build-env.sh を zsh でも .spresense_env を読み込めるように変更してください。
```
