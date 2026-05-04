# 到達圏分析

「ある地点（複数指定可）からT分以内に到達できるエリア」を 125m メッシュ（L6）で出力するツール。

---

## アルゴリズム

始点ごとに個別 Dijkstra を実行し、各始点の到達圏マップを個別に出力する。

逆向きグラフ不要。外部プログラム不使用。`scipy.sparse.csgraph.dijkstra` で実装。

**処理時間の目安（埼玉県・949,637 リンク）:**

| 始点数 | 処理時間 |
|---|---|
| 1 | 約 10 秒 |
| 10 | 約 100 秒 |
| 100 | 約 1,000 秒 |

---

## 制約・注意事項

> **一方通行は考慮されていない**
> 国土数値情報の道路データには一方通行フィールドが存在しない。このため、全道路を双方向リンクとして扱っており、実際の交通規制は反映されていない。

---

## ディレクトリ構成

```
（リポジトリルート）
├── README.md
├── CLAUDE.md
├── src/
│   ├── reachability_search.py    到達圏分析（メイン）
│   ├── ksj_to_network_csv.py     国土数値情報 → 道路リンク・ノード parquet
│   └── make_access_links.py      L6 アクセスリンク生成
├── data/
│   ├── prefecture.parquet        都道府県境界ポリゴン（--pref クリップ用）
│   └── city.parquet              市区町村境界ポリゴン（--city クリップ用）
├── input/                        国土数値情報 GeoJSON 配置場所（gitignored・各自用意）
├── network/                      ネットワークデータ（gitignored・再生成可能）
└── output/                       分析出力（gitignored・実行時に自動生成）
```

---

## 必要環境

- Python 3.9 以上
- 依存ライブラリ

```bash
pip install geopandas pyarrow scipy
```

---

## 使い方

### ステップ 1: 国土数値情報のダウンロード

国土交通省「国土数値情報ダウンロードサービス」から道路データ（N13）をダウンロードする。

- **ダウンロード先**: https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N13-v2_1.html
- **対象年度**: 2024 年度版（N13-24）を推奨
- **形式**: GeoJSON

ダウンロードした ZIP を展開し、GeoJSON ファイルを以下の形式で `input/` に配置する。

```
input/
├── N13-24_5338/
│   └── N13-24_5338.geojson
├── N13-24_5339/
│   └── N13-24_5339.geojson
└── ...
```

### ステップ 2: ネットワークデータ生成

#### 都道府県単位（例: 埼玉県）

```bash
python3 src/ksj_to_network_csv.py \
  --meshes 5338,5339,5438,5439 \
  --case saitama_pref \
  --pref 埼玉県

python3 src/make_access_links.py \
  --meshes 5338,5339,5438,5439 \
  --case saitama_pref \
  --level 6 \
  --pref 埼玉県
```

出力先: `network/saitama_pref/`

#### 市区町村単位（例: さいたま市）

```bash
python3 src/ksj_to_network_csv.py \
  --meshes 5338,5339,5438,5439 \
  --case saitama_city \
  --city さいたま市 \
  --pref 埼玉県

python3 src/make_access_links.py \
  --meshes 5338,5339,5438,5439 \
  --case saitama_city \
  --level 6 \
  --city さいたま市 \
  --pref 埼玉県
```

### ステップ 3: 到達圏分析を実行

```bash
# 単一始点
python3 src/reachability_search.py \
  --links network/saitama_pref/KSJ_N13-24_saitama_pref_道路リンク.parquet \
  --nodes network/saitama_pref/KSJ_N13-24_saitama_pref_道路ノード.parquet \
  --access network/saitama_pref/KSJ_N13-24_saitama_pref_アクセスリンク_L6.parquet \
  --orig 35.8578,139.6490,埼玉県庁

# 複数始点（最近傍始点からの到達時間を出力）
python3 src/reachability_search.py \
  --links network/saitama_pref/KSJ_N13-24_saitama_pref_道路リンク.parquet \
  --nodes network/saitama_pref/KSJ_N13-24_saitama_pref_道路ノード.parquet \
  --access network/saitama_pref/KSJ_N13-24_saitama_pref_アクセスリンク_L6.parquet \
  --orig 35.8578,139.6490,埼玉県庁 \
  --orig 36.0420,139.4006,東松山市役所

# CSV で始点を一括指定（lat,lon,name 列）
python3 src/reachability_search.py \
  --links network/saitama_pref/KSJ_N13-24_saitama_pref_道路リンク.parquet \
  --nodes network/saitama_pref/KSJ_N13-24_saitama_pref_道路ノード.parquet \
  --access network/saitama_pref/KSJ_N13-24_saitama_pref_アクセスリンク_L6.parquet \
  --orig-csv input/origins.csv
```

`input/origins.csv` のフォーマット（サンプル: `input/origins.csv` として同梱）:

```csv
lat,lon,name
35.8578,139.6490,埼玉県庁
36.0420,139.4006,東松山市役所
35.9063,139.6239,大宮駅
```

緯度・経度は Google マップで地点を右クリックするとコピーできる。

---

## reachability_search.py オプション一覧

| オプション | デフォルト | 説明 |
|---|---|---|
| `--links` | `network/saitama_pref/` 道路リンク | 道路リンク parquet パス |
| `--nodes` | `network/saitama_pref/` 道路ノード | 道路ノード parquet パス |
| `--access` | `network/saitama_pref/` アクセスリンク | L6 アクセスリンク parquet パス |
| `--orig lat,lon[,name]` | 埼玉県庁 | 始点（複数回指定可）。名前省略時は座標が名前になる |
| `--orig-csv path` | — | 始点 CSV（lat,lon,name 列） |
| `--out-dir` | `output/` | 出力ディレクトリ |

---

## ksj_to_network_csv.py 主なオプション

| オプション | 説明 |
|---|---|
| `--meshes 5338,5339,...` | 対象 1 次メッシュコード（カンマ区切り） |
| `--case {name}` | 出力ケース名（`network/{name}/` に出力される） |
| `--pref {都道府県名}` | 都道府県でクリップ（例: `埼玉県`） |
| `--city {市区町村名}` | 市区町村でクリップ（例: `横浜市`） |
| `--mode walk` | 徒歩モード（速度 3.6 km/h。省略時は vehicle モード） |
| `--filter` | 主要道路のみ（国道・都道府県道・高速 or 幅員 5.5m 以上）に絞り込み |
| `--nationwide` | 全国版（`input/` 以下を全件処理） |

## make_access_links.py 主なオプション

| オプション | 説明 |
|---|---|
| `--meshes 5338,5339,...` | 対象 1 次メッシュコード |
| `--case {name}` | ケース名（`ksj_to_network_csv.py` と同じ値を指定） |
| `--level 6` | アクセスリンクのメッシュレベル（5=250m, 6=125m） |
| `--pref {都道府県名}` | 都道府県でクリップ |
| `--city {市区町村名}` | 市区町村でクリップ |
| `--mode walk` | 徒歩モード |

---

## 出力ファイル

出力先: `output/`（`--out-dir` で変更可）

### `arrival_map_{始点名}.parquet` / `.qml`

始点からの到達時間マップ（全 L6 メッシュ）。

| カラム | 内容 |
|---|---|
| `mesh_code` | L6 メッシュコード（11 桁） |
| `dist_min` | 最近傍始点からの最短到達時間（分）。到達不能は NaN |
| `dist_rank` | 10 分刻みランク（0〜9）。到達不能は空文字 |
| `geometry` | メッシュポリゴン（EPSG:4326） |

複数始点を指定した場合、`dist_min` は最も近い始点からの到達時間となる。

### `origins_{始点名}.geojson` / `.qml`

始点ポイントデータ。QGIS で到達時間マップと重ねて表示できる。

| プロパティ | 内容 |
|---|---|
| `type` | `origin` |
| `name` | 地点名 |
| `lat` / `lon` | 入力した緯度・経度 |
| `snap_lat` / `snap_lon` | スナップした最寄道路ノードの緯度・経度 |
| `snap_m` | 入力座標から最寄ノードまでの距離（m） |

---

## QGIS での可視化

`.qml` ファイルをレイヤーに適用することで色分け表示できる。

### カラースキーム（arrival_map）

| ランク | 到達時間 | 色 |
|---|---|---|
| 0 | 0〜10 分 | 赤 |
| 1 | 10〜20 分 | 赤橙 |
| 2 | 20〜30 分 | 橙 |
| 3 | 30〜40 分 | 黄橙 |
| 4 | 40〜50 分 | 黄 |
| 5 | 50〜60 分 | 黄緑 |
| 6 | 60〜70 分 | 緑 |
| 7 | 70〜80 分 | 青緑 |
| 8 | 80〜90 分 | シアン |
| 9 | 90 分超 | 濃紫 |
| — | 到達不能 | グレー |

---

## 主要都道府県の 1 次メッシュコード表（参考）

| 都道府県 | 1 次メッシュコード |
|---|---|
| 北海道（札幌周辺） | 6441, 6442, 6443, 6444, 6541, 6542, 6543, 6544 |
| 宮城県 | 5640, 5641, 5740, 5741 |
| 東京都 | 5338, 5339, 5438, 5439 |
| 神奈川県 | 5238, 5239, 5338, 5339 |
| 埼玉県 | 5338, 5339, 5438, 5439 |
| 千葉県 | 5239, 5240, 5339, 5340, 5439, 5440 |
| 愛知県 | 5236, 5237, 5336, 5337, 5436, 5437 |
| 大阪府 | 5135, 5235 |
| 兵庫県 | 5134, 5135, 5234, 5235 |
| 広島県 | 5132, 5133, 5232, 5233 |
| 香川県 | 5133, 5134 |
| 福岡県 | 4930, 5030, 5031, 5032 |

---

## データについて

本ツールは国土交通省「国土数値情報」道路データ（N13）を使用する。

- **出典**: 国土数値情報 道路データ / 国土交通省
- **ダウンロード**: https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N13-v2_1.html
- **ライセンス**: 国土数値情報利用規約に準ずる
- **一方通行**: 国土数値情報には一方通行フィールドが存在しないため、全道路を双方向リンクとして扱っている

### 複製に関する承認

国土数値情報 道路データの原典資料は数値地図（国土基本情報）であり、測量法に基づく国土地理院長承認（複製）**R 6JHf503** を受けている。

> 本製品を複製する場合には、国土地理院の長の承認を得なければならない。
