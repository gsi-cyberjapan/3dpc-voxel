# 3dpc-voxel
3次元点群データをボクセル状にしたベクトルタイル及びその閲覧サイトの試作

## データについて
- 公開されている任意の3次元点群データからボクセル状のベクトルタイルを作成する方法については、`howtomake_asprs0.md`又は`howtomake_feature.md`をご覧ください。

### 作成方法その1（howtomake_asprs0.md）
- ソースレイヤ名: asprs0
- ズームレベル範囲: ZL13～17
- 間引きに利用するライブラリ: SpatiumGL（ズームレベルごとの間引き）
- 密度: ZL17では 1辺1mボクセル、ズームレベルが下がるごとに1辺の長さ2倍
- 属性:
  - height: 点の標高（メートル）
  - color: カラー情報
  - spacing: ボクセル1辺の長さ
- 備考:
  - Mapbox Vector Tile形式の背景地図を標高タイルにより3次元表示した場合、ボクセル状のベクトルタイルは実際の位置の標高を2倍した高さの位置に表示されます。背景地図の3次元表示を考えている場合は「作成方法 その2」の方法で作成してください。

### 作成方法その2（howtomake_feature.md）
- ソースレイヤ名: feature
- ズームレベル範囲: ZL10～20（密度による。MAXZOOMで1辺1mボクセル）
- 間引きに利用するライブラリ: PDAL（ズームレベルごとの間引き）
- 密度: 調整可能
- 属性:
  - top: 点の標高（メートル）
  - base: 点がある地点の標高（メートル）
  - cell: ボクセル1辺の長さ
- 備考:
  - 標高タイルによる3次元表示したMapbox Vector Tile形式の背景地図と重ねて使えるよう、内挿補間したグラウンドデータにより、ボクセル個々に属性としてその地点の標高値が付与されます。
  - ボクセルの座標がボクセル1辺の長さの倍数格子にスナップされ、グリッド状に整列されます。

## スタイルファイルについて
- `style_vectortiles_demo.json`（v2の場合）又は`style_vectortiles_demo_for_v1.json`（v1の場合）の
  `sources`の各`tiles`に記載のURLを編集し、3次元点群データからMapbox Vector Tile形式に変換したデータを参照するようにしてください。
- デフォルトの例では、作成方法その1で作成したベクトルタイルを「01_feature」、作成方法その2で作成したベクトルタイルを「02_basetop1」（末尾の数字はグリッドの間隔）などとしています。

## 閲覧サイトについて
- `mapbox1_demo.html`はMapbox GL JS v1とdeck.glを用いたサイトです。
- `mapbox2_demo.html`はMapbox GL JS v2を用いたサイトです。63行目付近にアクセストークンを入力してください。

## 標高タイルについて
- `elevationTiles`フォルダに標高タイル（Mapbox GL JSに対応した方式でRGB値が格納されている、Webp形式又はPNG形式のファイル）を格納するか、又はhtml内の標高タイルのパスを適宜に書き換えてください。
- なお、WebP標高タイルの作成方法については、以下のサイトが参考になります。
  - [webp 標高タイルを作った話](https://qiita.com/hfu/items/915d7fb961d05670cab2)
