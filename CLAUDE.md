# 到達圏分析 — Claude Code 向けプロジェクト仕様

## プロジェクト概要

複数の始点（任意数）から各 L6 メッシュ（125m）への最短到達時間を Multi-source Dijkstra で算出し、GeoParquet 形式で出力するツール。

## ディレクトリ構成

```
src/
  reachability_search.py  メイン分析スクリプト（Multi-source Dijkstra）
  ksj_to_network_csv.py   国土数値情報 GeoJSON → 道路リンク/ノード parquet 変換
  make_access_links.py    L6 アクセスリンク（メッシュ重心→最寄道路ノード）生成

data/
  prefecture.parquet      都道府県境界（--pref クリップ用）
  city.parquet            市区町村境界（--city クリップ用）

input/                    国土数値情報 GeoJSON 配置場所（gitignored）
network/                  生成ネットワークデータ（gitignored）
output/                   分析出力（gitignored）
```

## 主要スクリプトの仕様

### reachability_search.py

- **入力**: 道路リンク / 道路ノード / L6 アクセスリンク（各 parquet）
- **アルゴリズム**:
  - 始点ごとに個別 Dijkstra（G, indices=orig_idx）を実行し、個別の到達圏マップを出力
  - 複数始点は `multiprocessing.Pool` で並列実行（`--workers` オプション、デフォルト: `os.cpu_count()`）
  - fork でグローバル変数（G, valid, safe_idxs 等）を子プロセスに引き継ぐ（pickleコストなし）
  - 1始点あたり約10秒（埼玉県・949,637リンク）。16並列で100始点 ≈ 63秒
- **出力**: `output/arrival_map_{名前}.parquet/.qml`、`output/origins_{名前}.geojson/.qml`
- **パス定数**: `REPO_ROOT = Path(__file__).parent.parent`（`src/` の 1 つ上）
- **デフォルトデータ**: `network/saitama_pref/` の埼玉県サンプル（事前生成が必要）

### ksj_to_network_csv.py

- **入力**: `input/N13-24_{mesh}/N13-24_{mesh}.geojson`
- **出力**: `network/{case}/KSJ_N13-24_{case}_道路リンク.parquet` など
- **パス定数**: `BASE_DIR = Path(__file__).parent.parent`
- **クリップ**: `data/prefecture.parquet`（--pref）、`data/city.parquet`（--city）
- **重要**: 全道路を双方向リンクとして生成（国土数値情報に一方通行フィールドなし）

### make_access_links.py

- **入力**: `network/{case}/KSJ_N13-24_{case}_道路ノード.csv`
- **出力**: `network/{case}/KSJ_N13-24_{case}_アクセスリンク_L6.parquet`
- **パス定数**: `BASE_DIR = Path(__file__).parent.parent`

## 出力ファイル仕様

**arrival_map_*.parquet** カラム:
- `mesh_code`: L6 メッシュコード（11 桁）
- `dist_min`: 最近傍始点からの最短到達時間（分）。到達不能は NaN
- `dist_rank`: 10 分刻みランク（0〜9 の文字列）。到達不能は空文字
- `geometry`: メッシュポリゴン（EPSG:4326）

## 新しいエリアで分析する手順

1. 国土数値情報（https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N13-v2_1.html）から対象都道府県の N13-24 GeoJSON をダウンロードし `input/` に配置
2. `python3 src/ksj_to_network_csv.py --meshes {meshes} --case {name} --pref {pref}` でネットワーク生成
3. `python3 src/make_access_links.py --meshes {meshes} --case {name} --level 6 --pref {pref}` でアクセスリンク生成
4. `python3 src/reachability_search.py --links network/{name}/... --nodes ... --access ... --orig lat,lon,name` で分析

## 制約

- **一方通行未考慮**: 国土数値情報に一方通行フィールドが存在しないため全道路双方向
- **メッシュレベル**: L6（125m）固定。変更する場合は `--level` オプションとアクセスリンクパスを合わせて変更する
- **速度モデル**: vehicle=道路種別・幅員による速度テーブル、walk=3.6 km/h 一律
