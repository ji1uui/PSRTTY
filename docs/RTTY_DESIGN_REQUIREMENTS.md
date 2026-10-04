# RTTY アプリ設計要件・基本アルゴリズム集

対象: 今後の RTTY（AFSK / ITA2）アプリケーション開発
出典: PSRTTY Ver1.08 正本ソース
作成日: 2026-10-04

本書は、PSRTTY の実装のうち **再利用価値の高い「良い部分」** を抽出し、
次期アプリでそのまま設計要件（REQ-*）と基本アルゴリズム（ALG-*）として
明示できる形に整理したものである。

- 「要件」= 満たすべき性質・振る舞い（WHAT / WHY）
- 「アルゴリズム」= その要件を実現する具体手順（HOW）

出典は各項目にファイル名と行番号を付す。行番号は Ver1.08 時点。

---

## 0. 適用範囲と評価の前提

### 0.1 適用範囲

- 本ドキュメントが対象とする機能: 受信復調、ビット同期、ITA2 符号化、
  周波数同調・追従、同調可視化、マルチチャネル受信、音声 I/O とスレッディング、
  送信波形生成、PTT/CAT 連携の安全制御。
- 対象外: コンテスト得点計算、Cabrillo/ADIF 出力、UI レイアウト等
  （これらは `CABRILLO_SPEC.md` 等を参照）。

### 0.2 評価の前提（重要）

- 本書の「良い部分」は **アルゴリズムとソースの精読に基づく評価** であり、
  MMTTY / fldigi との同一音声による誤り率（CER）実測比較ではない。
- PSRTTY の復調コアは開発者自身が「成熟した弱信号デコーダの代替ではない」
  と位置づけている（`psrtty/decoder.py` L11-16）。
- したがって本書は「**競技用デコーダの完成形**」ではなく、
  「**堅実に作り込まれた設計パターンの抽出**」として扱う。

### 0.3 出典ファイル一覧

| 領域 | ファイル |
|---|---|
| 受信復調コア | `psrtty/decoder.py` |
| ITA2 符号・送信波形 | `psrtty/rtty_codec.py` |
| 周波数オフセット推定 | `psrtty/tuning.py` |
| 同調可視化（XY） | `psrtty/ui/cross_scope.py` |
| スペクトラム表示 | `psrtty/ui/spectrum.py` |
| マルチチャネル受信 | `psrtty/sub_decode.py`, `psrtty/ui/sub_decode.py` |
| 音声 I/O・スレッディング | `psrtty/audio_engine.py` |
| 第二音声出力 | `psrtty/secondary_audio.py`, `psrtty/offline_audio.py` |
| 周波数同調 UI | `psrtty/ui/main_window.py` |
| CAT 制御 | `psrtty/civ.py`, `psrtty/yaesu.py` |

---

## 1. 受信復調（デコーダ）

### 1.1 設計要件

- **REQ-RX-01 トーン判定は相関（マッチトフィルタ）で行うこと。**
  単純な振幅比較ではなく、MARK/SPACE 各トーンの複素相関エネルギーを比較し、
  窓関数で帯域外雑音とスペクトル漏れを抑圧すること。
  - 根拠: `decoder.py` L72-90（`_rebuild_basis`, `_logic`）。
  - 理由: 非コヒーレント AFSK において、目的トーンへの整合相関は
    帯域内雑音に対する最適検波に近い。

- **REQ-RX-02 判定に確度（confidence）を持たせること。**
  `confidence = |em - es| / (em + es)` を計算し、後段の品質ゲートに使えること。
  - 根拠: `decoder.py` L86。

- **REQ-RX-03 トーン品質を「トーン集中度」で定義すること。**
  全入力エネルギーに対するトーンエネルギーの比を品質指標とし、
  0〜1 に正規化すること（dB の SNR ではなく、窓内でのトーンらしさ）。
  - 根拠: `decoder.py` L84。
  - 理由: 絶対レベルに依存せず、SQ 判定に流用しやすい。

- **REQ-RX-04 ビット同期は start/data/stop の状態機械で行い、
  スタートビットとストップビットを必ず検証してから文字を確定すること。**
  ストップビットが MARK でない、またはフレーム品質が基準未満の場合は破棄する。
  - 根拠: `decoder.py` L107-141。
  - 理由: ノイズによる偽文字（スパイク）を大幅に抑制する
    （CHANGELOG L226 の設計意図）。

- **REQ-RX-05 フレーム確定後に再トリガ抑制期間を設けること。**
  1 スタート + 5 データ + 1.5 ストップ = 7.5 ビット相当を考慮し、
  直後の誤検出を防ぐこと。
  - 根拠: `decoder.py` L141（`_ignore_until = start + 7.1 * bit_time`）。

- **REQ-RX-06 SQ（スケルチ）は単一パラメータで誤り率と感度の
  トレードオフを制御できること。**
  0=OFF、1〜10 のしきい値テーブルとし、フレーム平均品質がしきい値未満なら
  文字を破棄すること。
  - 根拠: `decoder.py` L108, L135。

- **REQ-RX-07 受信チャネルは独立ストリームで並列稼働させ、
  メイン受信を絶対に待たせないこと。**
  ワーカー間は有界メールボックス（溢れたら古いものを破棄）で受け渡すこと。
  - 根拠: `sub_decode.py` L12-69。

- **REQ-RX-08 設定変更時は世代カウンタで古い音声を無効化すること。**
  設定変更前に投入された音声が新設定で復調されないようにすること。
  - 根拠: `sub_decode.py` L46-56, L94-99。

### 1.2 基本アルゴリズム

#### ALG-RX-1 複素相関によるトーン判定

事前計算（設定変更時のみ）:

```
win        = hanning(N)                     # N = window_ms * fs / 1000
mark_basis = win * exp(-j*2*pi*mark  * n / fs)   # n = 0..N-1
space_basis= win * exp(-j*2*pi*space * n / fs)
window_sum = sum(win)
```

各セグメントに対する判定:

```
em = |dot(segment, mark_basis)|^2
es = |dot(segment, space_basis)|^2
energy   = mean(segment^2)
quality  = min(1, 2*max(em,es) / (window_sum^2 * energy + eps))   # eps=1e-20
total    = em + es + eps
confidence = |em - es| / total
logic    = (em >= es) ? MARK : SPACE      # invert 時は反転
```

- 実装: `decoder.py` L72-90。

#### ALG-RX-2 ビット同期の状態機械

```
threshold = SQ_TABLE[sq_level]   # 0=OFF,1..10
状態: idle / start / data / stop

idle:
  if t < ignore_until: return
  if logic==SPACE and last_logic==MARK:
      start_t = t
      next_sample_t = t + 0.4 * bit_time      # スタートビット中央狙い
      bits = []; frame_quality = []
      state = start

start:
  if t < next_sample_t: return
  if logic != SPACE: state = idle; return      # スタートビット不正 → 破棄
  state = data
  next_sample_t = start_t + 1.5 * bit_time

data:
  if t < next_sample_t: return
  frame_quality.append(quality)
  bits.append(logic)
  next_sample_t += bit_time
  if len(bits)==5: state = stop

stop:
  if t < next_sample_t: return
  frame_quality.append(quality)
  if logic==MARK and mean(frame_quality) >= threshold:
      code = sum((bit & 1) << i for i, bit in enumerate(bits))
      char = ITA2.decode_code(code)
      if char: emit(char)
  state = idle
  ignore_until = start_t + 7.1 * bit_time
```

- 実装: `decoder.py` L107-141。
- 注意: ビットサンプリング点はスタートビットで一度決めた後は
  `bit_time` 固定刻み（自由走行）。連続的なクロック追従は行わない
  （→ §6 の改善項目参照）。

#### ALG-RX-3 SQ しきい値テーブル

```
SQ_LEVEL      : 0    1     2     3     4     5     6     7     8     9     10
threshold     : 0.0 .015  .030  .055  .090  .140  .210  .300  .420  .560  .700
```

- 実装: `decoder.py` L108。既定 SQ=4（`config.py` L78）。

---

## 2. ITA2 符号化と送信波形

### 2.1 設計要件

- **REQ-TX-01 送信波形は位相連続 FSK とすること。**
  トーン切替時に位相を連続させ、スプリアス（クリック）を抑圧すること。
  - 根拠: `rtty_codec.py` L87-97（`phase` の累積更新）。

- **REQ-TX-02 端数ビットのタイミング補正を行うこと。**
  45.45 baud は `fs/baud` が非整数のため、サンプル数の丸め誤差を累積補正し、
  長時間送信でもビットレートがドリフトしないようにすること。
  - 根拠: `rtty_codec.py` L85-92（`sample_error`）。

- **REQ-TX-03 フレームは 1 スタート / 5 データ / 1.5 ストップとすること。**
  PTT・音声の立ち上がりを吸収するため、先頭に MARK リードイン、
  末尾に MARK テールを付けること。
  - 根拠: `rtty_codec.py` L99-106。

- **REQ-TX-04 送信は PTT/ACK の確認が取れない限り開始しないこと。**
  PTT ON 確認 → 待機 → ストリーム開始 → ドレイン → PTT OFF 確認の順を守ること。
  - 根拠: `audio_engine.py` L214-279。

- **REQ-TX-05 送信ストリームは単一ワーカースレッドのみが操作すること。**
  他スレッドからの `sd.stop()` 等で競合させないこと。
  - 根拠: `audio_engine.py` L163-166, L214。

### 2.2 基本アルゴリズム

#### ALG-TX-1 位相連続 AFSK 生成

```
phase = 0.0; sample_error = 0.0
tone(logic_mark, bit_units=1.0):
    exact = fs / baud * bit_units + sample_error
    n     = round(exact)
    sample_error = exact - n                 # 端数を持ち越す
    freq  = (logic_mark XOR invert) ? mark : space
    phases = phase + 2*pi*freq/fs * [0..n-1]
    append(amplitude * sin(phases))
    phase = (phases[-1] + 2*pi*freq/fs) mod 2*pi   # 位相連続
```

- 実装: `rtty_codec.py` L85-97。
- ITA2 符号化は `encode_ita2_codes`（L47-69）:
  LTRS/FIGS を自動切替、FIGS_SHIFT=27、LTRS_SHIFT=31。

---

## 3. 周波数オフセット推定（AFC の材料）

### 3.1 設計要件

- **REQ-AFC-01 周波数オフセット推定は FFT ピークではなく
  ゼロ交差→瞬時周波数で行うこと。**
  FFT 最大値は FSK キーイングの過渡でバイアスするため使わないこと。
  - 根拠: `tuning.py` L7-36。

- **REQ-AFC-02 推定前に急峻だが平坦な通過域で帯域外雑音を除去すること。**
  8 次ガウス型応答 `exp(-((f-fc)/width)^8)` を用いること。
  - 根拠: `tuning.py` L16。

- **REQ-AFC-03 安定区間のみを使って推定すること。**
  8 サンプル移動窓の標準偏差が閾値未満の連続（安定した MARK/SPACE）だけを
  対象とすること。
  - 根拠: `tuning.py` L25-26。

- **REQ-AFC-04 MARK/SPACE 両方のオフセットを独立推定し、
  両者が一致する場合のみ有効とすること。**
  差が閾値（3 Hz）を超える場合は破棄。最終値は履歴平均で平滑化すること。
  - 根拠: `tuning.py` L30-35, L40-58。

- **REQ-AFC-05 推定値はまず「表示」として提供し、
  オペレータの同調判断を支援すること。**
  （復調ループへの自動フィードバックは §6 の改善項目）。
  - 根拠: `tuning.py` L61-69（`tuning_display`）、`ui/main_window.py`。

### 3.2 基本アルゴリズム

#### ALG-AFC-1 瞬時周波数からのセンターオフセット推定

```
入力: x（>= 0.12s）, fs, mark, space
1. 平坦通過域:  width = max(450, |space-mark|*1.5)
   response = exp(-((freq - (mark+space)/2)/width)^8)
   y = irfft(rfft(x, size) * response, size)[:n]
2. 上昇ゼロ交差検出:
   crossings = {i : y[i] <= 0 < y[i+1]}, 両端 guard=10ms を除外
   if len(crossings) < 20: 無効
3. 線形補間で交差位置を精密化:
   pos = crossings - y[crossings]/(y[crossings+1]-y[crossings])
4. 瞬時周波数: instant = fs / diff(pos)
5. 安定区間抽出:
   runs = sliding_window(instant, 8)
   stable = std(runs) < 2.0 Hz
   centers = median(runs)
6. 各トーンで測定:
   measured = fs*8 / (pos[8:] - pos[:-8])       # 複数周期平均
   selected = stable & (|centers - tone| < min(65, |shift|*0.4))
   offset_tone = median(measured[selected]) - tone
7. 妥当性検証:
   if |offset_mark - offset_space| > 3: 無効
   value = mean(offset_mark, offset_space)
   return value if |value| < 25 else 無効
```

- 実装: `tuning.py` L7-36。
- 平滑化: `CenterTuning.update` で 0.5 秒の履歴平均（L40-58）。
- 表示色分け境界: 5/9/11/15/20/25 Hz（L67）。

---

## 4. 同調可視化

### 4.1 設計要件

- **REQ-VIS-01 XY 表示は「実測サンプルのみ」を描画すること。**
  合成楕円を描かないこと。
  - 根拠: `ui/cross_scope.py` L10-48（docstring, L43）。

- **REQ-VIS-02 同調判定を位相で行えるようにすること。**
  MARK/SPACE をオーバーラップする 1 次レゾネータで検波し、
  両軸の位相差を 90° に補償すること。同調時は細い楕円、
  離調時は楕円が傾く性質を利用すること。
  - 根拠: `ui/cross_scope.py` L10-42。

- **REQ-VIS-03 反対トーンの漏れはノッチで抑えるが、
  主共振器は狭めないこと。**
  共振器を狭めるとシンボル変化への応答速度が落ちるため。
  - 根拠: `ui/cross_scope.py` L35-42。

- **REQ-VIS-04 描画は安定区間のみとし、ギャップを繋がないこと。**
  変化率が閾値未満の区間のみを描画し、近傍ガードを設けること。
  - 根拠: `ui/cross_scope.py` L51-93。

- **REQ-VIS-05 スペクトラムは指数平均で表示を安定させること。**
  `新 = 0.65*旧 + 0.35*新` 等の平滑化を行うこと。
  - 根拠: `ui/spectrum.py` L30-34。

### 4.2 基本アルゴリズム

#### ALG-VIS-1 位相補償付き XY 生成

```
入力: samples, fs, mark, space, baud
shift = space - mark;  if |shift| < 1: 空
size  = 2*n
bandwidth = min(|shift|*0.4, max(|shift|*0.20, baud*0.8))
phase = sign(shift) * (pi/2 - atan(|shift|/bandwidth))   # 90° 補償
for tone, rotation in ((mark,0),(space,phase)):
    response = exp(j*rotation) / (1 + j*(freq - tone)/bandwidth)  # 1次レゾネータ
    opposite = (space if tone==mark else mark)
    distance = (freq - opposite) / max(|shift|*0.20, 1)
    notch    = 0.18 + 0.82 * distance^2 / (1 + distance^2)        # 反対トーン抑制
    channel  = irfft(rfft(samples,size) * response * notch, size)[n//4 : 3n//4]
xy = [channel_mark, channel_space];  正規化して返す
```

- 実装: `ui/cross_scope.py` L10-48。

#### ALG-VIS-2 安定トレース抽出

```
1. 解析エンベロープをヒルベルトで取得（表示ゲート専用）
2. change = |env - roll(env, lag)| / max(env, .02)   # lag ≈ 2ms
3. stable = change < 0.07,  両端 2*lag を除外
4. stable を ±1.5ms の窓で全数 True の近傍のみ残す
5. 各軸で最長の連続 run を採用、音声 3 周期分に制限して描画
```

- 実装: `ui/cross_scope.py` L51-93。

---

## 5. マルチチャネル受信・負荷制御

### 5.1 設計要件

- **REQ-MC-01 サブデコーダは別スレッドで稼働させ、
  メイン受信とは有界メールボックス（上限 8）で受け渡すこと。**
  満杯時は最古を捨て、ドロップ数を計数すること。
  - 根拠: `sub_decode.py` L12-69。

- **REQ-MC-02 近傍の別 RTTY 局を自動発見できること。**
  ノイズフロア（35 パーセンタイル）+ マージン以上のピークから、
  シフト幅に近いペアを候補化し、メインセンター周辺は除外すること。
  - 根拠: `sub_decode.py` L105-134。

- **REQ-MC-03 同一局の重複受信を検出して自動スイープを止めること。**
  メイン本文と一定文字数一致＋同時刻なら重複と判定すること。
  - 根拠: `ui/sub_decode.py` L273-279。

- **REQ-MC-04 負荷に応じて更新間隔を適応的に伸ばすこと。**
  ドロップ増加やバックログ増加で更新間隔を段階的に延ばし、CPU を保護すること。
  - 根拠: `ui/sub_decode.py` L220-231。

- **REQ-MC-05 自動スイープは外側からメインへ段階移動し、
  空ペアを繰り返し試行しないこと。**
  - 根拠: `ui/sub_decode.py` L247-264。

### 5.2 基本アルゴリズム

#### ALG-MC-1 ペア候補ランキング

```
入力: freqs, power, center, shift, side, other=None, guard=15
floor = percentile(power, 35)
peaks = 局所最大 かつ power > floor + 9
for i in top-28 peaks:            # 強度降順
  for j in top-28 peaks:
     if |(f_j - f_i) - shift| > 25: continue
     candidate = (f_i + f_j)/2
     if サイド条件に反する: continue
     if min|候補トーン - メイントーン| < guard: continue
     if other のトーンとも guard 未満: continue
     strength = min(power_i, power_j) - floor
     候補に追加
外側（|candidate-center| 大）優先でソートして返す
```

- 実装: `sub_decode.py` L105-134。

#### ALG-MC-2 重複検出

```
sub  = 英数記号のみ抽出した受信文（直近）
main = 同上（メイン本文）
if メイン最終文字と当該サブの最終文字の時刻差 <= 3s
   and len(sub) >= 10 and len(main) >= 10
   and (sub[-10:] in main[-100:] or main[-10:] in sub[-100:]):
        duplicate = True   # スイープ停止
```

- 実装: `ui/sub_decode.py` L273-279。

---

## 6. 音声 I/O・スレッディング

### 6.1 設計要件

- **REQ-IO-01 入力コールバックは軽量に保つこと。**
  復調は低レイテンシのためコールバック内で行うが、
  スペクトラム/スコープ等の重い処理は最新値のみのメールボックス経由で
  GUI 側に渡すこと。
  - 根拠: `audio_engine.py` L135-161。

- **REQ-IO-02 ブロックサイズは固定（例 960 = 20ms@48kHz）とすること。**
  - 根拠: `audio_engine.py` L103。

- **REQ-IO-03 複数出力は各デバイスが独立ストリームを所有すること。**
  第二出力はコールバック方式で TX ループをブロックしないこと。
  - 根拠: `secondary_audio.py` L22-72, `offline_audio.py` L10-86。

- **REQ-IO-04 未接続時の送信許可時は、片方でも使える出力先で再生すること。**
  - 根拠: `offline_audio.py` L66-86。

- **REQ-IO-05 デバイス識別は OS 変更に強い安定 ID で行い、
  勝手な代替デバイスへの差し替えをしないこと。**
  - 根拠: `audio_devices.py`（WASAPI エンドポイント ID）。

### 6.2 実装メモ

- 入力ストリーム: `InputStream(channels=1, dtype=float32, blocksize=960,
  callback=...)`、Windows は `WasapiSettings(auto_convert=True)`
  （`audio_engine.py` L98-106）。
- 送信順序: PTT ON 確認 → 0.12s 待機 → OutputStream → 各 960 サンプル書込 →
  `stop()`（ドレイン）→ 0.08s → PTT OFF 確認（L214-279）。

---

## 7. PTT / CAT 連携の安全制御

### 7.1 設計要件

- **REQ-CAT-01 コマンドは ACK を検証してから成功とみなすこと。**
  例: CI-V 応答 `raw[4] == 0xFB`、Yaesu は期待フレーム一致。
  - 根拠: `civ.py` L321-332, `yaesu.py` L248-267。

- **REQ-CAT-02 送信前に必ず「非送信中」を確認すること。**
  - 根拠: `civ.py` L195-211, `yaesu.py` L161-186。

- **REQ-CAT-03 CI-V のコマンドエコーを除去してから応答を解釈すること。**
  - 根拠: `civ.py` L70-101。

- **REQ-CAT-04 接続確認は周波数読取の妥当性で行い、
  範囲外や無応答は未接続とすること。**
  - 根拠: `civ.py` L126-148。

- **REQ-CAT-05 DATA モード自動切替を設定で選択可能とし、
  失敗時は接続を破棄すること。**
  - 根拠: `civ.py` L424-438。

- **REQ-CAT-06 受信性能に関わる CAT 項目（フィルタ幅 / NB / NR / Notch /
  ALC / Tuner）を取得・設定できること。取得不能時は操作しないこと。**
  - 根拠: `civ.py` L180-284, L309-319, `yaesu.py` L127-246。

---

## 8. 周波数同調 UI（AUTO TUNE / AUTO TRACK）

### 8.1 設計要件

- **REQ-TUNE-01 自動同調は強度とシフト誤差のバランスでペアを選択すること。**
  - 根拠: `ui/main_window.py` L1281-1303
    （`score = p_i + p_j - err*0.15`）。

- **REQ-TUNE-02 自動追従はヒステリシスを持たせること。**
  複数回（3 回）連続一致で確定し、1 回の移動量を制限（±12 Hz）すること。
  - 根拠: `ui/main_window.py` L1249-1279。

- **REQ-TUNE-03 無信号・弱信号で誤ロックしないようフロアゲートを設けること。**
  - 根拠: `ui/main_window.py` L1263（`floor(35%ile)+8dB`）。

- **REQ-TUNE-04 送信中・停止直後は追従を抑止すること。**
  - 根拠: `ui/main_window.py` L1250-1252（停止後 2 秒）。

- **REQ-TUNE-05 トーン位置は MARK/SPACE の極性を保ちつつ移動し、
  範囲（100〜3900 Hz、シフト 10〜2000 Hz）を検証すること。**
  - 根拠: `ui/main_window.py` L1310-1323。

---

## 9. 既知の限界（次期アプリでは改善対象とすること）

本書は「良い部分」の抽出だが、次期アプリで **再発させない / 改善すべき**
限界も明示しておく。これらは「やってはいけない設計」の要件として扱う。

- **LIM-01 ビットクロックの連続追従（PLL）がない。**
  スタートビット 1 点で決めた位相を自由走行させるため、
  baud 誤差・送信側ジッタ・周波数ドリフトでフレーム後半ほど誤る。
  → 次期: DPLL / 早期遅延ゲートによるクロックリカバリを要件化。

- **LIM-02 AFC が復調ループに入っていない。**
  オフセット推定は表示専用、AUTO TRACK は離散ステップ。
  → 次期: 推定オフセットを復調器のトーン周波数へ連続フィードバック。

- **LIM-03 判定がビット中央 1 点のハード判定。**
  → 次期: ビット全長積分 + 軟判定（confidence 活用）。

- **LIM-04 SQ が固定テーブル。**
  → 次期: SNR / ノイズフロア自動追尾による適応スケルチ。

- **LIM-05 サブデコは独立復調のみで合成しない。**
  → 次期: 同一局の複数デコーダ出力を最大比合成し実効 SNR を改善。

- **LIM-06 送信波形にパルス整形（ナイキスト/ガウス）がない。**
  → 次期: 帯域外漏れ低減のため整形をオプション化。

> 注意: LIM-* は「PSRTTY が競技用デコーダとして未成熟」であることの根拠でもある。
> 次期アプリで MMTTY/fldigi 相当以上を狙う場合、LIM-01〜LIM-04 の実装が必須。

---

## 10. 受け入れ基準・検証方法

良い部分を要件として採用したかを検証するための基準。

- **T-01 復調ラウンドトリップ**: 生成→復調で文字一致。
  - 既存: `tests/test_core.py::test_audio_decoder_roundtrip_clean_signal`。
- **T-02 ビット同期の頑健性**: スタートビット欠落・ストップビット破壊を
  注入し、偽文字を出さないこと。
- **T-03 SQ 特性**: 各 SQ レベルで誤り率と無信号ノイズ文字数の
  トレードオフ曲線が単調であること。
- **T-04 オフセット推定精度**: 既知オフセット（例 ±10 Hz）を付与した信号で
  推定誤差が ±1〜2 Hz 以内であること。
- **T-05 重複検出**: メインと同一内容のサブ入力で duplicate が立つこと。
- **T-06 スレッディング**: サブ負荷を最大にしてもメイン RX の
  ドロップが 0 であること（有界メールボックスの検証）。
- **T-07 送信安全**: PTT/ACK 失敗時に音声が送出されないこと。
- **T-08 弱信号 A/B（次期必須）**: 同一録音 WAV を PSRTTY/MMTTY/fldigi に
  通し、CER を計測。弱信号・QSB・ドリフト条件を含めること。

---

## 11. 要件・アルゴリズム対応表

| 要件 | アルゴリズム | 主な出典 |
|---|---|---|
| REQ-RX-01〜03 | ALG-RX-1 | decoder.py L72-90 |
| REQ-RX-04〜06 | ALG-RX-2, ALG-RX-3 | decoder.py L107-141 |
| REQ-RX-07,08 | ALG-MC-1, ALG-MC-2 | sub_decode.py |
| REQ-TX-01〜03 | ALG-TX-1 | rtty_codec.py L85-109 |
| REQ-TX-04,05 | — | audio_engine.py L163-279 |
| REQ-AFC-01〜05 | ALG-AFC-1 | tuning.py L7-69 |
| REQ-VIS-01〜04 | ALG-VIS-1, ALG-VIS-2 | ui/cross_scope.py |
| REQ-VIS-05 | — | ui/spectrum.py L30-34 |
| REQ-MC-01〜05 | ALG-MC-1, ALG-MC-2 | sub_decode.py, ui/sub_decode.py |
| REQ-IO-01〜05 | — | audio_engine.py, secondary_audio.py |
| REQ-CAT-01〜06 | — | civ.py, yaesu.py |
| REQ-TUNE-01〜05 | — | ui/main_window.py L1249-1323 |

---

## 付録 A. 用語

- **MARK / SPACE**: RTTY の 2 トーン。本実装の既定は 2125 / 2295 Hz（shift 170 Hz）。
- **ITA2 (Baudot)**: 5 ビット符号。LTRS/FIGS の 2 シフトを持つ。
- **SQ (スケルチ)**: 品質しきい値による文字出力のゲート。
- **C同調**: 設定センターと実受信トーンのずれ（Hz）。本実装では表示専用。
- **CER / WER**: 文字誤り率 / 単語誤り率。

## 付録 B. 主要パラメータ既定値

| 項目 | 既定値 | 出典 |
|---|---|---|
| baud | 45.45 | config.py L66 |
| shift | 170 Hz | config.py L67 |
| mark / space | 2125 / 2295 Hz | config.py L68-69 |
| invert | False | config.py L70 |
| SQ | 4 | config.py L78 |
| sample_rate | 48000 | config.py L62 |
| blocksize | 960 | audio_engine.py L103 |
| 解析窓 | 10 ms / hop 2.5 ms | decoder.py L36-38 |
| スペクトラム | 4096 点 / 100 ms / 0-4000 Hz | audio_engine.py L143-159 |
