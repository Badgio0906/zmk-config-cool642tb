# cool642tb ZMK Config

## 1. リポジトリ概要

| 項目 | 内容 |
|---|---|
| キーボード名 | cool642tb |
| マイコン | Seeeduino XIAO BLE (nRF52840) |
| 分割構成 | 左手 (cool642tb_L) ＋ 右手 (cool642tb_R) |
| 右手機能 | PMW3610-alt トラックボール（中央側）、ZMK Studio 対応 |
| 左手機能 | EC11 ロータリーエンコーダー（ペリフェラル側） |
| トラックボールドライバ | [badjeff/zmk-pmw3610-driver](https://github.com/badjeff/zmk-pmw3610-driver) |
| ビルド環境 | Docker（`ghcr.io/skubmdi/docker-zmk-builder:main`） |

### ビルド対象（build.yaml）

| シールド | ボード | 備考 |
|---|---|---|
| cool642tb_R | xiao_ble//zmk | ZMK Studio 用 USB UART スニペット付き |
| cool642tb_L | xiao_ble//zmk | ペリフェラル |
| settings_reset | xiao_ble//zmk | BLE ペアリングリセット用 |

---

## 2. 初回セットアップ手順

### 前提
- Docker Desktop（または Docker Engine + Compose plugin）がインストール済みであること
- `git` がインストール済みであること

### リポジトリのクローン

```bash
git clone https://github.com/Badgio0906/zmk-config-cool642tb
cd zmk-config-cool642tb
```

### GitHub 認証設定

`west update` が GitHub から依存リポジトリを取得するため、認証設定が必要。

```bash
git config --global credential.helper store
# 初回 git push 時などにユーザー名・PAT を入力すると ~/.git-credentials に保存される
```

Personal Access Token (PAT) は GitHub → Settings → Developer settings → Personal access tokens で発行する（`repo` スコープで十分）。

### Docker イメージのビルド

```bash
docker compose build
```

`west init` → `west update`（依存リポジトリのダウンロード）→ `west zephyr-export` がこの時点で実行される。
初回は数分かかる。

---

## 3. ファームウェアのビルド手順

### ビルド実行

```bash
docker compose up
```

`build.yaml` の全ターゲットが並列ビルドされ、`output/` に成果物が出力される。

### 出力ファイルの確認

```
output/
├── cool642tb_R-xiao_ble__zmk-zmk.uf2      ← 右手書き込みファイル
├── cool642tb_L-xiao_ble__zmk-zmk.uf2      ← 左手書き込みファイル
├── settings_reset-xiao_ble__zmk-zmk.uf2   ← BLEリセット用
├── cool642tb_R-xiao_ble__zmk-zmk.log      ← ビルドログ
└── ...
```

### エラー時のログ確認

```bash
# ビルドエラーの詳細を確認
cat output/cool642tb_R-xiao_ble__zmk-zmk.log

# エラー行だけ抽出
grep -E "error:|warning:" output/cool642tb_R-xiao_ble__zmk-zmk.log
```

---

## 4. ファームウェア書き込み手順

### ブートローダーモードへの移行

1. XIAO BLE の **リセットボタンを素早く2回押す**（ダブルタップ）
2. PC 上に `XIAO-SENSE` という USB ドライブとして認識される

> キーボードが起動中の場合は一度 USB を抜き、接続し直してからダブルタップすると確実。

### uf2 ファイルの書き込み

uf2 ファイルを認識された USB ドライブにコピーするだけで書き込みが完了する。

```bash
# Linux / WSL の場合（マウントパスは環境に応じて変更）
cp output/cool642tb_R-xiao_ble__zmk-zmk.uf2 /mnt/d/XIAO-SENSE/

# Windows の場合（エクスプローラーでコピーも可）
```

書き込み後、デバイスが自動的に再起動する。

### 左右の書き込み順序

左右どちらから書き込んでも問題ないが、**右手（central）から書き込む**と BLE ペアリングがスムーズに進む傾向がある。

---

## 5. Claude Code での開発フロー

### キーマップ変更からビルド確認まで

```bash
# 1. keymapを編集
code config/cool642tb.keymap

# 2. ビルド実行
docker compose up

# 3. 成功確認
ls -lh output/*.uf2
```

### エラー発生時の対処フロー

```
ビルドエラー発生
    │
    ├─ DTS/keymap 構文エラー → output/*.log の "error:" 行を確認
    │       → bindings の数が 43 個（レイヤーごと）か確認
    │       → タップダンス名のタイポがないか確認
    │
    ├─ west update 失敗 → GitHub 認証（credential.helper store）を確認
    │
    └─ ドライバ関連エラー → west.yml の revision が正しいか確認
           → docker compose build --no-cache で再ビルド
```

### git commit と push

```bash
# 変更をステージング
git add config/cool642tb.keymap

# コミット（GitHub Actions が自動でビルドを開始する）
git commit -m "keymap: レイヤー配置を調整"

# push（GitHub Actions ビルドが走る）
git push origin main
```

push 後は `.github/workflows/build.yml` により GitHub Actions でもビルドが実行される。Actions の成果物（Artifacts）からも uf2 をダウンロード可能。

---

## 6. 注意事項

### west.yml を変更した場合

`west.yml` は `docker compose build` 時に Docker イメージに組み込まれる。変更後はキャッシュを無効にして再ビルドが必要。

```bash
# west.yml を変更した場合は必ずこちらを実行
docker compose build --no-cache
```

### Zephyr 4.x 対応

ZMK は現在 Zephyr 4.x に対応中。ベースイメージ `ghcr.io/skubmdi/docker-zmk-builder:main` が Zephyr バージョンを管理しているため、イメージ更新後は `docker compose build --no-cache` で再取得すること。

`west.yml` で `zmk` の `revision` を `main` にピン留めしているため、ZMK 本体のブレーキングチェンジに追従するには適宜 `docker compose build --no-cache` を実行する。

### キーマップのバインディング数

cool642tb のマトリクスは **1 レイヤーあたり 43 キー**（10+12+12+9）。各レイヤーのバインディング数が合わないとコンパイルエラーになる。

### BLE ペアリングリセット

左右のペアリングがおかしくなった場合は `settings_reset` ファームウェアを両方に書き込んでから再ペアリングする。

```bash
# 左右両方に settings_reset を書き込む
cp output/settings_reset-xiao_ble__zmk-zmk.uf2 /mnt/d/XIAO-SENSE/
```
