# cool642tb

ビルド・開発の共通手順は [`../CLAUDE.md`](../CLAUDE.md) を参照。

---

## 1. キーボード概要

| 項目 | 内容 |
|---|---|
| キーボード名 | cool642tb |
| マイコン | Seeeduino XIAO BLE (nRF52840) |
| 分割構成 | 左手 (cool642tb_L) ＋ 右手 (cool642tb_R) |
| 右手機能 | PMW3610-alt トラックボール（中央側）、ZMK Studio 対応 |
| 左手機能 | EC11 ロータリーエンコーダー（ペリフェラル側） |
| トラックボールドライバ | [badjeff/zmk-pmw3610-driver](https://github.com/badjeff/zmk-pmw3610-driver) |

### ビルド対象（build.yaml）

| シールド | ボード | 備考 |
|---|---|---|
| cool642tb_R | xiao_ble//zmk | ZMK Studio 用 USB UART スニペット付き |
| cool642tb_L | xiao_ble//zmk | ペリフェラル |
| settings_reset | xiao_ble//zmk | BLE ペアリングリセット用 |

---

## 2. output フォルダと uf2 ファイル

### output フォルダのパス

| 環境 | パス |
|---|---|
| Ubuntu (WSL2) | `/home/romcr/github/zmk/zmk-config-cool642tb/output` |
| Windows | `\\wsl.localhost\Ubuntu\home\romcr\github\zmk\zmk-config-cool642tb\output` |

Windows パスはエクスプローラーのアドレスバーにそのまま貼り付けて開ける。

### 書き込みに使う uf2 ファイル

uf2 以外のファイル（.bin / .elf / .hex / .log 等）は書き込みに不要。

| ファイル名 | 書き込み先 | タイミング |
|---|---|---|
| `cool642tb_R-xiao_ble__zmk-zmk.uf2` | **右手**（トラックボール側） | 通常の更新時 |
| `cool642tb_L-xiao_ble__zmk-zmk.uf2` | **左手**（エンコーダ側） | 通常の更新時 |
| `settings_reset-xiao_ble__zmk-zmk.uf2` | 左右**両方** | BLE 接続がおかしいときだけ |

---

## 3. ファームウェア書き込み手順

### ブートローダーモードへの移行

1. XIAO BLE の **リセットボタンを素早く2回押す**（ダブルタップ）
2. PC 上に `XIAO-SENSE` という USB ドライブとして認識される

> キーボードが起動中の場合は一度 USB を抜き、接続し直してからダブルタップすると確実。

### uf2 ファイルの書き込み

uf2 ファイルを認識された USB ドライブにコピーするだけで書き込みが完了する。

```bash
# WSL からコピーする場合（ドライブレターは環境に応じて変更）
cp output/cool642tb_R-xiao_ble__zmk-zmk.uf2 /mnt/d/XIAO-SENSE/
```

Windows の場合はエクスプローラーでコピーしてもよい。書き込み後、デバイスが自動的に再起動する。

### 左右の書き込み順序

左右どちらから書き込んでも問題ないが、**右手（central）から書き込む**と BLE ペアリングがスムーズに進む傾向がある。

---

## 4. キーボード固有の注意事項

### キーマップのバインディング数

cool642tb のマトリクスは **1 レイヤーあたり 43 キー**（行ごとに 10 + 12 + 12 + 9）。各レイヤーのバインディング数が合わないとコンパイルエラーになる。

### BLE ペアリングリセット

左右のペアリングがおかしくなった場合は `settings_reset` ファームウェアを両方に書き込んでから再ペアリングする。

```bash
# 左手・右手それぞれブートローダーモードにして書き込む
cp output/settings_reset-xiao_ble__zmk-zmk.uf2 /mnt/d/XIAO-SENSE/
```
