# CLAUDE.md

このファイルは、このリポジトリで作業する際にClaude Code(claude.ai/code)が参照するガイドです。

## このリポジトリについて

**Seaside39**（Keyball39をSeeed XIAO BLE nRF52840マイコンで無線化したトラックボール付き分割キーボード）のZMKファームウェア設定リポジトリです。このリポジトリに含まれるのはファームウェアの*設定*のみ（keymap、Kconfig、devicetreeオーバーレイ、`west.yml`マニフェスト）で、実際のZMK/Zephyrのソースコード自体はCIビルド時に取得されるものであり、このリポジトリには含まれません。

- R（右）側 = ZMKスプリットの**central**。PMW3610トラックボールセンサーを搭載し、PC/iPad等のホストとBLE接続する側。
- L（左）側 = ZMKスプリットの**peripheral**。R側とのみ接続する側。

## ビルド/CI

このリポジトリにも開発環境にもローカルのZephyr/ARMツールチェーンは無く、**ビルドはすべてGitHub Actions経由**（`.github/workflows/build.yml`）で行われます。このワークフローは再利用可能な`zmkfirmware/zmk/.github/workflows/build-user-config.yml`を呼び出しています。board+shieldのビルドマトリクスは`build.yaml`で定義されており、`Seaside39_R rgbled_adapter`・`Seaside39_L rgbled_adapter`・`settings_reset`の3種のファームウェアが生成されます。

手順:
1. 設定ファイルを編集し、コミットして`main`に`git push`する。
2. `gh` CLIでビルド結果を確認する: `gh run list --limit 1`、失敗時は`gh run view <id>` / `gh run view <id> --log-failed`でログを確認する。
3. 成功したrunのArtifactsから`.uf2`をダウンロードする。
4. 書き込み: XIAOのリセットボタンを素早く2回押してブートローダーモードにする（`XIAO SENSE`というUSBドライブとして認識される）と、そこへ`.uf2`をコピーする。BLEのボンド/アイデンティティに関わる設定（例: `CONFIG_BT_PRIVACY`）を変更した場合のみ、先に`settings_reset-*.uf2`を書き込んでからshield本体のファームウェアを書き込む。それ以外は直接shield本体のファームウェアを書き込めばよい。

**重要な落とし穴（2025年10月〜2026年7月の約9ヶ月間、プロジェクトを悩ませた問題）:** かつて`config/west.yml`と`.github/workflows/build.yml`は`zmk`の`revision: main`および`build-user-config.yml@main`を追従する設定になっていた。ZMK本体のmainブランチ側の変更（Zephyr 4.1のboard qualifier移行）でこのボードのビルドが壊れ、**誰も気づかないままCIが全てのpushで失敗し続けていた**（ローカルビルドで検知する手段が無かったため）。**設定変更が実機に反映されたと思い込む前に、pushが実際にCIで成功しているか(`gh run list`)を必ず確認すること** — ビルドに一度も成功していない設定変更は、実機には一度も書き込まれていない。

この経緯から、`zmk`・`zmk-rgbled-widget`・ビルドワークフローの参照は現在`config/west.yml`/`build.yml`内で**`v0.3.0`に固定**されている。これらのバージョンを上げる場合は、`zmk`と`zmk-rgbled-widget`を**同じ**ZMKリリース/タグに揃えて固定すること — ワークフロー・`zmk`本体・コミュニティモジュール（`zmk-rgbled-widget`、`zmk-pmw3610-driver`）間でバージョンが食い違うと、Kconfigやdevicetreeエイリアス関連のエラーが発生し、無関係な設定変更のせいだと誤診断しやすい。`zmk-pmw3610-driver`は2024年4月以降コミットが無く（実質凍結状態）、`main`のままにしてある。

## リポジトリ構造

- `config/west.yml` — `zmk`（本体）、`zmk-pmw3610-driver`（トラックボール）、`zmk-rgbled-widget`（ステータスLED）、`prospector-zmk-module`（現在無効化中、下記参照）を取り込むマニフェスト。
- `config/Seaside39.keymap` — keymap本体（詳細は下記）。
- `config/boards/shields/Seaside39/`:
  - `Seaside39.dtsi` — L/R共通のdevicetree（物理キー配置、マトリクス変換、kscan）。
  - `Seaside39_R.overlay` / `Seaside39_L.overlay` — 各側固有のdevicetree（列のGPIO、R側のみSPI+PMW3610トラックボールノード）。
  - `Seaside39_R.conf` / `Seaside39_L.conf` — 各側固有のKconfig。BLE/トラックボール/LED関連の調整はほぼ全て、host側であるR側の`Seaside39_R.conf`に集中している。
  - `Kconfig.defconfig` — `ZMK_SPLIT`、`ZMK_SPLIT_ROLE_CENTRAL`（R側のみ）、各側のキーボード名を設定。
- `build.yaml` — GitHub Actionsのビルドマトリクス。
- `docs/buildguide.md` — エンドユーザー向けのハードウェア組み立て・ファームウェア書き込みガイド（日本語）。

## keymapの構造（`config/Seaside39.keymap`）

レイヤー: `default_Win`（0）、`default_iOS`（1）、`MOUSE`（4）、`Function`（2）、`Control`（3。ファイル内では最後に定義されているが`&lt 3`/レイヤー番号3である点に注意 — レイヤー番号をファイル中の記述順で判断せず、冒頭の`#define`（`MOUSE`/`SCROLL`/`NUM`）を確認すること）。

- ベースレイヤーが2つあるのは、WindowsとiOS/macOSで修飾キーの慣習（Ctrl vs Cmd等）が異なるため。`default_iOS`は`LEFT_GUI`/`LC(SPACE)`を使用し、`default_Win`は`LCTRL`/`GRAVE`を使用する。
- BLEプロファイル切り替えは`Control`レイヤーにある:
  - 1行目: 素の`&bt BT_SEL 0/1/2` — BLEプロファイルの切り替えのみ行う。
  - 3行目: `&bt0`/`&bt1`/`&bt2`マクロ — BLEプロファイル切り替えに加え、`default_iOS`レイヤーのON/OFFも同時に行う（`bt2`はiPad向けプロファイルという想定で、`default_iOS`をONにする。`bt0`/`bt1`はOFFにする）。切り替え先ホストの修飾キー配置がWindowsと異なる場合は、素の1行目よりこちらを優先して使うこと。
- トラックボールの挙動（`automouse-layer`、`scroll-layers`）はkeymapファイルではなく、`Seaside39_R.overlay`内の`trackball@0`ノードで設定されている。

## 扱いに注意が必要なKconfigオプション（安易に再有効化しない）

以下は、ビルド不能または機能不全の原因と特定された結果、現在`Seaside39_R.conf`内でコメントアウトされている。再有効化する場合は、実際にGitHub Actionsでビルドが成功することを確認してから信用すること:

- `CONFIG_BT_PRIVACY=y` — このZMKバージョンでは、`ZMK_SPLIT_ROLE_CENTRAL`が自動選択する`BT_SCAN_WITH_IDENTITY`と矛盾し、Kconfigの致命的なビルドエラーを引き起こす。
- `CONFIG_BT_SECURITY_LESC=n` / `CONFIG_BT_SECURITY_FORCE_LESC=n` — レガシー（非LESC）ペアリングを強制すると、iOSとのペアリングが失敗すると推定されている。
- `CONFIG_BT_MAX_CONN=2` — 既にボンド済みの2つのホストプロファイル(PC/iPad)間の切り替えが失敗する原因になった。central側はスプリット接続で常時1コネクション枠を消費するため、実質的な予備枠が1つしか残らず、プロファイルの切り替え時（旧接続の切断と新接続の確立が重なるタイミング）の余裕が不足していた。

`CONFIG_ZMK_STATUS_ADVERTISEMENT` / `CONFIG_BT=y` / `CONFIG_BT_PERIPHERAL=y`（Prospector scanner対応）も同様にコメントアウトされている — 有効化すると以前iOSとのペアリングが完全に失敗する原因になった。
