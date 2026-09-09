# 2026_B_446

STM32F446 (Nucleo-F446RE 系ボード) を用いて、6基のロータリーエンコーダの回転速度を読み取り、CAN通信で他のマイコン（モータドライバ等）へ送信するファームウェアです。STM32CubeMX (HAL) ベースで生成され、`Core/Src/main.c` にユーザーロジックが実装されています。

## 概要

- TIM1 / TIM2 / TIM3 / TIM4 / TIM5 / TIM8 の6つのタイマをエンコーダモード（A相/B相入力）で使用し、6軸分のエンコーダカウント値を取得します。
- TIM12 を周期タイマ（割り込み）として使用し、一定周期ごとに各エンコーダのカウント値を取得します。
- メインループで前回値との差分から速度（差分カウント数）を計算し、PWM出力値相当のスケール（-255〜255）に変換した後、絶対値化して6byteのデータとしてCAN1経由で送信します。
- USART2はデバッグ用の `printf` 出力（`_write` 経由）に使用しています。

## 処理の流れ

1. `TIM12` の周期割り込み (`HAL_TIM_PeriodElapsedCallback`) が発生するたびに、各タイマのカウンタ値 (`cntA1`〜`cntA6`) を読み出し、`control_trigger` を立てる。
2. メインループ (`while(1)`) で `control_trigger` が立っていたら、
   - 今回値 (`cntA*`) と前回値 (`cntB*`) の差分を `PV1`〜`PV6` に格納（＝一定周期あたりの回転量＝速度）
   - `cntB*` を `cntA*` で更新（次回差分計算用に保存）
   - `map()` 関数で差分カウント値を `-max_pwm`〜`max_pwm`（-255〜255）の範囲にスケーリング
   - 負の値は絶対値に変換（符号情報は失われる点に注意）
   - `CAN_TX()` でCAN ID `0x001` 宛に送信
3. これを無限に繰り返す。

## 主な変数・関数

### グローバル変数

| 変数名 | 型 | 説明 |
|---|---|---|
| `PV1`〜`PV6` | `int16_t` | 各エンコーダ（軸1〜6）の速度（差分カウント値をPWMスケールに変換し絶対値化した値）。CAN送信データとして使用。PV = Process Value（制御量）の意味と思われる |
| `cntA1`〜`cntA6` | `volatile int16_t` | 各タイマ（TIM1,2,3,4,5,8）から読み取った現在のエンコーダカウンタ値。TIM12の割り込み内で更新 |
| `cntB1`〜`cntB6` | `int16_t` | 直前の周期でのエンコーダカウンタ値（差分計算用の保存値） |
| `control_trigger` | `volatile int` | TIM12割り込みが発生したことをメインループへ知らせるフラグ（1=処理待ち） |
| `max_pwm` | `int`（`main()`内ローカル） | PWM換算後の最大値。255固定 |

### エンコーダ軸とタイマの対応

| 変数添字 | 使用タイマ | カウント値取得元 |
|---|---|---|
| 1 | TIM1 | `cntA1 = __HAL_TIM_GET_COUNTER(&htim1)` |
| 2 | TIM2 | `cntA2 = __HAL_TIM_GET_COUNTER(&htim2)` |
| 3 | TIM3 | `cntA3 = __HAL_TIM_GET_COUNTER(&htim3)` |
| 4 | TIM4 | `cntA4 = __HAL_TIM_GET_COUNTER(&htim4)` |
| 5 | TIM5 | `cntA5 = __HAL_TIM_GET_COUNTER(&htim5)` |
| 6 | TIM8 | `cntA6 = __HAL_TIM_GET_COUNTER(&htim8)` |

※軸5・6（TIM5, TIM8）は他の軸よりも分解能が高いエンコーダを想定しており、`map()` でのスケーリング入力範囲が `-1580〜1580` と他軸（`-155〜155`）より広く設定されています。

### 主要関数

| 関数名 | 説明 |
|---|---|
| `long map(long x, long in_min, long in_max, long out_min, long out_max)` | 値`x`を`[in_min, in_max]`の範囲から`[out_min, out_max]`の範囲へ線形変換する（Arduinoの`map()`と同様）。範囲外の値は`in_min`/`in_max`にクランプされる |
| `void CAN_TX(uint32_t recipient)` | CAN1の送信メールボックスに空きがあれば、`PV1`〜`PV6`（各1byte）を含む8byteのCANメッセージを標準ID`recipient`宛に送信する。7byte目は固定値`3`（用途識別子等）、8byte目は`0`（未使用） |
| `void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)` | HALのタイマ割り込みコールバック。`htim`がTIM12の場合、6軸分のエンコーダカウンタを取得し`control_trigger`を立てる |
| `int _write(int file, char *ptr, int len)` | `printf`をUSART2経由で出力するためのHAL実装（デバッグ用） |

## 周辺機能設定

- **CAN1**: Prescaler=3, TimeSeg1=7TQ, TimeSeg2=2TQ, SJW=1TQ、自動再送有効。フィルタはIDリストモード（16bit×4）。
- **TIM1/2/3/4/5/8**: エンコーダモード（`TIM_ENCODERMODE_TI1`）、Period=65535（16bit最大）、Prescaler=0。
- **TIM12**: ベースタイマとして使用し、Prescaler=299, Period=1999で周期割り込みを生成（速度計算・CAN送信のトリガ周期）。
- **USART2**: 115200bps, 8N1（デバッグ出力用）。
- **GPIO**: `LD2`（ボード上LED）、`B1`（ユーザーボタン、立ち下がり割り込み）。

## 開発環境

- MCU: STM32F446xx
- 生成ツール: STM32CubeMX + STM32 HAL
- ビルド: CMake（`CMakeLists.txt`, `CMakePresets.json`）
