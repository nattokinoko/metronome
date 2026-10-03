<!-- STREAMING_CHUNK:Writing document header and purpose... -->
# 『ケイデンス メトロノーム』 Webアプリケーション 開発仕様書

**ドキュメントバージョン**: v1.3.0

**最終更新日**: 2026年10月3日

---

## 1. システム目的 (Purpose)

本システムは、ロードバイクおよびインドアトレーナーでのペダリングケイデンス（RPM: Revolutions Per Minute）の維持および段階的向上のためのトレーニング用Webアプリケーション（Single File Component）である。

ウォームアップから目標ケイデンスまでのスムーズな移行（ランプアップ/ランプダウン）をサポートするため、一定時間で連続的に発音間隔を変化させる「無段階直線補間アルゴリズム」と、Web Audio APIによる「事前生成PCMバッファ再生方式」を採用し、全環境（PC・スマートフォン）において音量ゆらぎ・位相干渉・タイミング遅延のない極めて正確なトレーニング音響環境を提供する。

---

## 2. 改訂履歴 (Version History)

| バージョン | 改訂日 | 改訂内容概要 |
| :--- | :--- | :--- |
| **v1.0.0** | 2026年9月10日 | 初期作成。固定BPM指定による電子音（ビープ音）連続発音機能およびスタート/停止トグル制御を実装。 |
| **v1.1.0** | 2026年9月28日 | 機能拡張。スタートケイデンス、目標ケイデンス、遷移時間の入力項目を追加。無段階直線補間（Linear Interpolation）によるリアルタイムケイデンス変化アルゴリズムを導入。 |
| **v1.2.0** | 2026年10月1日 | 精度向上。JavaScriptタイマーのジッター（遅延）を吸収するため、`AudioContext.currentTime` を基準としたルックアヘッド・スケジューリング（Look-ahead Scheduling）を採用。 |
| **v1.3.0** | 2026年10月3日 | 音響エンジン刷新。リアルタイムオシレーター・ゲイン描画処理による128サンプル境界の周期的な音量不均一および位相干渉を根絶するため、事前生成PCMバッファ方式（`AudioBuffer`）を全改修・導入。 |

---

## 3. 動作環境・技術スタック (System Architecture & Environment)

<!-- STREAMING_CHUNK:Defining architecture and technical stack... -->
### 3.1 動作環境

* **対象デバイス**: スマートフォン (iOS / Android), PC, タブレット
* **推奨ブラウザ**: Google Chrome, Apple Safari, Microsoft Edge, Mozilla Firefox（最新版）
* **動作形態**: スタンドアロン動作（単一HTMLファイル構成）、PWA（ホーム画面追加）対応
* **通信環境**: 完全オフライン動作可能

### 3.2 技術スタック

* **フロントエンド**: HTML5, CSS3 (CSS Variables, Flexbox/Grid)
* **スクリプト言語**: JavaScript (ES6+ Vanilla JS / Single File)
* **音響・タイミング制御**: Web Audio API (`AudioContext`, `AudioBuffer`, `AudioBufferSourceNode`)

---

## 4. 状態管理・内部データ仕様 (State Management & Data Model)

本アプリケーションは単一ファイル構成であり、実行時のメモリ内で以下の内部状態（State）をリアルタイム管理する。

| 変数・オブジェクト名 | 型 | 初期値 | 説明・目的 |
| :--- | :--- | :--- | :--- |
| `startCadence` | `Number` | `70` | スタート時の初期ケイデンス (RPM) |
| `targetCadence` | `Number` | `90` | 遷移時間経過後に到達・維持する目標ケイデンス (RPM) |
| `transitionDurationSec` | `Number` | `300` | ケイデンス変化の遷移時間（秒数換算） |
| `startTime` | `Number` | `0` | スタートボタン押下時の `AudioContext.currentTime` 基準秒 |
| `nextNoteTime` | `Number` | `0` | 次回発音予定の `AudioContext.currentTime` 基準秒 |
| `isRunning` | `Boolean` | `false` | メトロノームの稼働状態フラグ (`true`: 動作中, `false`: 停止中) |
| `audioCtx` | `AudioContext` | `null` | Web Audio API のマスターコンテキストオブジェクト |
| `clickBuffer` | `AudioBuffer` | `null` | 事前レンダリング済みの均一クリック音PCMデータバッファ |

---

## 5. 機能仕様詳細 (Functional Specifications)

<!-- STREAMING_CHUNK:Describing functional specifications... -->
### 5.1 パラメータ設定機能 (Input Controls)

1. **入力パラメータ項目**:
   * **スタートケイデンス ($C_{start}$)**: 範囲 `30` 〜 `250` RPM（初期値: `70`）
   * **目標ケイデンス ($C_{target}$)**: 範囲 `30` 〜 `250` RPM（初期値: `90`）
   * **遷移時間 ($T_{trans}$)**: 範囲 `0` 〜 `120` 分（初期値: `5` 分、入力ステップ: `0.5` 分刻み）

2. **バリデーション & フォールバック**:
   * 空白入力または不正値入力時は、デフォルト値（70 RPM / 90 RPM / 0分）に安全にフォールバック処理される。

---

### 5.2 制御アルゴリズム (Cadence Transition Algorithm)

1. **イニシャライズ（発音開始）**:
   * スタートボタン押下時点を $t = 0$ とし、$C_{start}$ に基づく初回の発音間隔 $I_{start} = \frac{60}{C_{start}}$ [秒] で第1音を発声する。

2. **ランプ（遷移）フェーズ ($0 \le t < T_{trans}$)**:
   * 開始後の経過時間 $t$ [秒] に応じて、現在の目標ケイデンス $C(t)$ を以下の無段階直線補間（一次関数）により計算する：
     $$C(t) = C_{start} + \left( \frac{C_{target} - C_{start}}{T_{trans}} \right) \cdot t$$
   * クリック音の発声毎に次回のクリック間隔 $I(t) = \frac{60}{C(t)}$ [秒] をリアルタイム再計算し、`nextNoteTime` を更新する。

3. **ホールド（維持）フェーズ ($t \ge T_{trans}$)**:
   * 経過時間 $t$ が遷移時間 $T_{trans}$ に達した後は、$C(t) = C_{target}$ に固定・ロックし、手動で停止ボタンが押されるまで一定間隔で発音を維持する。

---

### 5.3 音響エンジン & スケジューリング仕様 (Audio Engine & Look-ahead Scheduler)

1. **事前生成PCMバッファ方式 (Pre-rendered PCM Buffer Strategy)**:
   * オシレーターによるリアルタイム音量カーブ（エンベロープ）計算を行わず、初回動作時に `AudioContext` 内に50msの880Hz（A5音）正弦波＋3ms直線アタック＋指数関数ディケイ（減衰）処理を施した100%決定論的なPCM音源データ（`AudioBuffer`）を1回のみ生成・保持する。
   * これにより、Web Audio APIにおける128サンプルレンダリングブロック境界（約2.7ms周期）の位相干渉および音量落ち（波形相殺）を物理的に100%根絶する。

2. **ルックアヘッド・スケジューリング (Look-ahead Scheduling)**:
   * JavaScriptの `setTimeout`（25ms周期）でルックアヘッドループを回し、常に `audioCtx.currentTime + 0.2` 秒先（200msバッファエリア）までの発音予定をWeb Audio APIのタイムスタンプへ先行予約する。
   * スマホのバックグラウンドタイマー遅延やUIレンダリング負荷が生じた場合でも、音のタイミングに一切のジッター（揺らぎ）を発生させない。

---

### 5.4 表示 & 操作UI (User Interface & Controls)

1. **リアルタイムケイデンス表示**:
   * 各クリック音の発声タイミングと同期して、現在の計算値 $C(t)$ を四捨五入した整数値（RPM）として画面中央に巨大フォントでリアルタイム表示する。
   * 停止時は `--` と表示。

2. **トグルボタン (Start / Stop Control)**:
   * 停止時: 青色表示（テキスト: 「スタート」）
   * 動作時: 赤色表示（テキスト: 「停止」）
   * タップ時に即座にスケジューラを停止し、表示および内部タイマーを初期状態へ安全にリセットする。

---

## 6. 非機能要件・技術的特徴 (Non-functional Requirements)

<!-- STREAMING_CHUNK:Detailing non-functional requirements and UI responsiveness... -->
1. **完全な音量安定性 (100% Volume Consistency)**:
   * デスクトップ環境およびモバイルOS（iOS Safari / Android Chrome）環境におけるオーディオエンジンの量子化・サンプリング境界ノイズを排除し、完全同一音量での安定した打音再生を保証する。

2. **超軽量・無依存アーキテクチャ (Zero Dependency Single File)**:
   * 外部ライブラリ（React, Vue, jQuery等）や外部音声ファイル（MP3, WAV等）を一切使用せず、単一HTMLファイルのみで完結。ネットワーク接続を切断した状態でも完璧に動作する。

3. **レスポンシブ & 高コントラストUI**:
   * 屋外のサイクリングやインドアトレーナーでの乗車時にも視認性を確保するため、ダークモードを標準採用し、タッチ操作に適した十分なボタンタップ領域（Touch Target Size）を確保している。
```
eof