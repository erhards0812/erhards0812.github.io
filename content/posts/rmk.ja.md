+++
title = "Rust製キーボードファームウェア"
description = "分割キーボードにおけるKVMスイッチフリーズ問題の修正"
date = "2026-09-24"
tags = ["Rust", "embassy", "rmk", "rust mechanical keyboard", "qmk", "quantum mechanical keyboard"]

[extra]
image = "/images/posts/qmk_rmk.jpg"
+++

{{ img(path="images/posts/qmk_rmk.jpg", alt="qmk to rmk and embassy") }}

> AIの使用について：このポストにおける概念、経験、そして技術的なメタファー（比喩）はすべて筆者自身のものです。
> AIアシスタントは、全体の構成の整理や文章の推敲をサポートするために使用しました。

分割型自作キーボードをハードウェアKVMスイッチ経由で複数PC間を行き来しながら使っていると、イライラするようなエッジケースに遭遇することがあります。接続マシンを切り替えた瞬間にUSB接続が突然切断され、カスタムファームウェアがフリーズしたりUSB状態マシンがロックアップしてしまう現象です。

PCを切り替えるたびにUSB-Cケーブルを抜き差ししたりハードウェアリセットを強要されるのは、KVMがもたらすはずのシームレスなワークフローを完全に台無しにしてしまいます。ここでは、Dilemma Maxキーボードで発生していたUSB認識（enumeration）問題の調査をきっかけに、ボードのファームウェアをQMKからEmbassy駆動のRMK（Rust Mechanical Keyboard framework）へと移植した経緯を紹介します。

## 根本原因：突然のUSB切断とマイコンのループ処理

KVMがホストポートを切り替える際、必ずしも整然としたUSBサスペンドや正常な終了シグナル（teardown）が送られるわけではありません。切り替えスイッチの挙動によっては、VBUS電源の挙動が不安定なまま、USBデータ線が唐突に切断されたり再ルーティングされたりします。

QMK（LUFAやChibiOSといったC言語ベースのUSBスタック実装に依存）では、クリーンなバスリセットを伴わない突然のUSB状態の低下が発生すると、ホスト側の通信ドライバーや左右の分割通信タスクがブロッキングループに入ってロックアップするか、エンドポイントハンドラーがクラッシュしてしまうことがあります。

レガシーなC言語のマクロ抽象化の中で重箱の隅をつつくようなバグ修正を追いかけるくらいなら、Embassyの非同期組み込みエコシステム上に構築された最新のRust製キーボードフレームワーク「RMK」を試す絶好の機会だと考えました。

## EmbassyによるノンブロッキングなUSB列挙（Enumeration）

RMKの最大の強みは、その下層にあるランタイム「Embassy」にあります。従来のC言語ファームウェアは、期待されるUSBハンドシェイクシグナルが届かない場合、ブロッキングループや中断処理によってフリーズしてしまうことが多々ありました。

Embassyでは以下のように動作します：

1. **非同期USBスタック**: `embassy-usb`ドライバーは非同期（async）で動作します。KVMが切り替わってパケット送信の途中でUSBバスが切断されても、USBタスクはハードウェイトでスピンすることなく、実行権限を他に譲ります（yield）。

2. **スムーズな再列挙（Re-enumeration）**: KVMが目的のPCに接続されると、USB状態マシンがVBUS状態の変化を検知し、自動的にクリーンな再列挙シーケンスを開始します。

3. **分離されたタスク**: キーマトリクスのスキャンと分割通信用UART処理は独立した非同期タスクとして実行されるため、ホスト側のUSB接続状態に関わらず、手元の周辺機器としての応答性が維持されます。

## QMKソースを解読してハードウェアピン配置をマッピングする

自作分割キーボードを新しいファームウェアフレームワークに移植するのは一見難しそうに思えますが、既存のQMKソースファイルがあれば、そこにハードウェアの設計図がすべて揃っています。キーボードのQMK設定ファイル（`keyboard.json`、`config.h`、`halconf.h`）を参照することで、物理マトリクスマッピングやマイコンのピン割り当てを簡単に抽出できます。

```
QMK config.h / keyboard.json          RMK TOML Configuration
┌─────────────────────────┐           ┌─────────────────────────┐
│ "matrix_pins": {        │           │ [matrix]                │
│   "cols": ["D0", "D1"], │  ───────► │ input_pins = ["PD0"]    │
│   "rows": ["B0", "B1"]  │           │ output_pins = ["PB0"]   │
│ }                       │           └─────────────────────────┘
└─────────────────────────└

```

QMKでは行/列ピン、分割通信プロトコル（UART/シリアルなど）、マトリクスダイオードの向きがプレーンなCヘッダーやJSONスキーマで定義されているため、これらをRMKの設定形式に直接マッピングする作業はほんの数分で完了しました。

## keyboard.toml による宣言型ファームウェア

RMKは構造化されたTOMLフォーマットを使用しており、ボイラープレートなファームウェアロジックを書くことなく、ボードのメタデータ、GPIOピンマトリクス、キーマップレイヤー、分割トランスポートのオプションを宣言できます。以下はRMKにおける分割キーボード設定の実際の代表例です：

**keyboard.toml**

```toml

[keyboard]
name = "Ferris_Sweep_Bling_MX"
product_name = "Ferris Sweep Bling MX"
vendor_id = 0x4242
product_id = 0x0001
manufacturer = "RMK"
chip = "rp2040"

# [light]
# Status LED configuration for power/connection state
# [light.capslock]
# pin = "PIN_17"
# low_active = true    # Set to true because Pro Micro RX LEDs are pulled HIGH to 3.3V and sink LOW to light up

[host]
vial_enabled = true
unlock_keys = [[0, 0], [0, 1]]

[storage]
enabled = true

[split]
connection = "serial"

[split.central]
rows = 4
cols = 5
row_offset = 0
col_offset = 0
serial = [
    { instance = "PIO0", tx_pin = "PIN_1", rx_pin = "PIN_1" }
]

[split.central.matrix]
matrix_type = "direct_pin"
direct_pin_low_active = true
direct_pins = [
  ["PIN_7",  "PIN_26",  "PIN_27",  "PIN_28",  "PIN_29"],
  ["PIN_22", "PIN_20",  "PIN_23",  "PIN_21", "PIN_0"],
  ["PIN_2", "PIN_3", "PIN_4", "PIN_5", "PIN_6"],
  ["PIN_8", "PIN_9", "_", "_", "_"]
]

[[split.peripheral]]
rows = 4
cols = 5
row_offset = 0
col_offset = 5
serial = [
    { instance = "PIO0", tx_pin = "PIN_1", rx_pin = "PIN_1" }
]

[split.peripheral.matrix]
matrix_type = "direct_pin"
direct_pin_low_active = true
direct_pins = [
  ["PIN_29",  "PIN_28",  "PIN_27",  "PIN_26",  "PIN_7"],
  ["PIN_0",  "PIN_21",  "PIN_23",  "PIN_20", "PIN_22"],
  ["PIN_6", "PIN_5", "PIN_4", "PIN_3", "PIN_2"],
  ["PIN_9", "PIN_8", "_",       "_",      "_"]
]

[layout]
rows = 4
cols = 10
map = """
    (0,0) (0,1) (0,2) (0,3) (0,4) (0,5) (0,6) (0,7) (0,8) (0,9)
    (1,0) (1,1) (1,2) (1,3) (1,4) (1,5) (1,6) (1,7) (1,8) (1,9)
    (2,0) (2,1) (2,2) (2,3) (2,4) (2,5) (2,6) (2,7) (2,8) (2,9)
    (3,0) (3,1)                   (3,5) (3,6)
"""

[keymap]
[[keymap.layer]]
keys = """
    B      F      R     L      V        J      Y      O     U     SemiColon
    N      S      H     T      D        P      C      A     E     I
    Z      X      K     M      Q        G      W      Comma  Dot  Slash
                        Mo(1)  Ent      Space  Mo(2)
"""

[[keymap.layer]]
keys = """
    No  No  No  No    No    No  No    No    No   No
    No  No  No  No    No    No  No    No    No   No
    No  No  No  No    No    No  No    No    No   No
                Mo(1) Ent   Space Mo(2)
"""

[[keymap.layer]]
keys = """
    No  No  No  No    No    No  No    No    No   No
    No  No  No  No    No    No  No    No    No   No
    No  No  No  No    No    No  No    No    No   No
                Mo(1) Ent   Space Mo(2)
"""

[behavior.combo]
timeout = "35ms"
prior_idle_time = "50ms"  # optional, prevents accidental triggers during fast typing
combos = [
  # left side
  { actions = ["f", "r"], output = "Tab"},
  { actions = ["r", "l"], output = "LGui"},
  { actions = ["h", "t"], output = "Escape"},
  { actions = ["k", "m"], output = "SHIFTED(SemiColon)"},
  # right side
  { actions = [",", "."], output = "Backspace"},
  { actions = ["o", "u"], output = "LGui"},
  { actions = ["w", ","], output = "SemiColon"},
]

```

## TOMLから卒業し、純粋なRustとEmbassyの世界へ

元使用してたキーボード（Dilemma Max）には、WS2812B RGBアンダーグローLEDが搭載されています。RMKのTOML設定スキーマは現時点でカスタムのアンダーグロー周辺機器を初期状態のままでサポートしていないため、LEDを駆動させるには宣言型設定ファイルを脱却し、純粋なRustでカスタムファームウェアを記述する必要がありました。

```Rust

// Embassy上でWS2812B LEDを手動制御するためのバックグラウンドタスクを生成
#[embassy_executor::task]
async fn run_underglow(mut ws2812: Ws2812Spi<...>) {
  loop {
    // マトリクススキャンをブロックすることなく、独立してRGB状態を駆動
    ws2812.write(render_lighting_frame()).await.ok();
    Timer::after_millis(30).await;
  }
}

```

コードベース全体をRustへ移行したことで、Embassyのドライバーを使ってハードウェア周辺機器を直接初期化できるようになりました：

1. **周辺機器の直接所有（Direct Peripheral Ownership）**: マイコン用のEmbassy HALを利用し、SPIやビットバングPWM周辺機器を手動で初期化して、WS2812B LEDが要求する厳密なタイミングプロトコルを制御可能にしました。

2. **並行非同期タスク（Concurrent Async Tasks）**: Embassyの `#[embassy_executor::task]` を通じて独立したバックグラウンドタスクを生成することで、キースキャンやUSBポーリングタスクを停止させることなく、アンダーグローのライティングアニメーションを並行実行できます。

3. **高度なカスタマイズ性（Deep Customization）**: 高レベルな抽象化をあえてバイパスすることで、省電力制御、カスタムレイヤー状態のインジケーター表示、独自のハードウェア状態ハンドリングなどを直接制御できるようになりました。

## 成果とまとめ

1. **ケーブル抜き差しからの解放**: ワークステーション間のKVM切り替えが極めてシームレスになりました。切り替え先のPCを選択すると、わずか数ミリ秒でキーボードが再列挙されます。

2. **保守性の高い設定**: C言語のマクロやヘッダー定義から、Rustの型で保護された抽象化やTOML設定へと移行したことで、将来のキーマップ変更やハードウェア調整が簡単かつ安全になりました。

3. **現代的な組み込みスタック**: 組み込みコンテキストで非同期Rustを活用することは、単にメモリ安全性（Memory Safety）を得るためだけではありません。USB、分割シリアル通信、各種ハードウェア周辺機器にまたがる複雑な状態管理をシンプルにしてくれる構造化並行性と非同期パラダイム（Structured Concurrency & Async Paradigms）こそが、C言語でありがちな並行処理トラブルや状態マシンのデッドロックバグを根絶してくれます。
