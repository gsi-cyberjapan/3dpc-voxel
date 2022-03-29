# LASデータをボクセル状のベクトルタイルにする手順 その2

## 前提条件
- 本手順書では、LASデータが格納されたフォルダパスを`C:\input\01_featuremabiki`とし、Windows on Linux Subsystem 2 (WSL2)での Debian 11を使用した場合の例を示します。また、内挿補間されたグラウンドデータのファイル名は`*_interpolated-ground.las`とします。
- LASデータは、平面直角座標系の座標値x, y, zのほか、色情報r, g, bを持つものとします。
- 他のLASをデータソースにする場合は、`01_featuremabiki`を適宜読み替えてください。
- 動作環境を保証するため手順中で初期化しますので、既にWSL2のDebianがインストール済みの場合はご注意ください。


## 環境構築

### Debianの初期化

- コマンドプロンプトで、Debian を初期化（削除してインストール）
~~~
wsl --unregister debian
wsl --install -d debian
~~~

- ここから Debian のコンソール

- Debian 9 から Debian 11 にアップグレード
~~~
sudo apt -y update
sudo apt -y install debian-archive-keyring

cat << EOS | sudo tee /etc/apt/sources.list
deb http://ftp.jp.debian.org/debian bullseye main
deb http://ftp.jp.debian.org/debian bullseye-updates main
deb http://security.debian.org/debian-security/ bullseye-security main
EOS

sudo apt -y update
sudo DEBIAN_FRONTEND=noninteractive apt -y full-upgrade
~~~

### Node.js及びyarnのサードパーティリポジトリの取込
~~~
sudo apt -y install curl
curl -fLsS https://deb.nodesource.com/setup_lts.x   | sudo -E bash -
curl -fLsS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo gpg --batch --yes --dearmor -o /usr/share/keyrings/yarnpkg.com.gpg
echo "deb [signed-by=/usr/share/keyrings/yarnpkg.com.gpg] https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarnpkg.com.list

sudo apt -y update
sudo apt -y install \
  gdal-bin \
  git \
  pdal \
  yarn
~~~

### Tippecanoeを取得してビルド
~~~
sudo apt -y install --no-install-recommends \
  build-essential \
  libsqlite3-dev \
  zlib1g-dev

git clone https://github.com/mapbox/tippecanoe
cd tippecanoe
make -j
sudo make install
cd ..
sudo rm -rf tippecanoe
~~~

### ビルドが済めば不要となるパッケージを削除
~~~
sudo apt -y purge --auto-remove \
  build-essential \
  cmake \
  libsqlite3-dev \
  libglew-dev \
  xorg-dev \
  zlib1g-dev
~~~

### node-gdal-async をカレントディレクトリにインストール
~~~
yarn add gdal-async
~~~

## 手順書本体

### 標高CSV作成スクリプトの記述
- ボクセルに各点の真下の地表の標高を付与するための、標高値を格納したCSVを作るシェルスクリプトを記述
~~~
cat << 'EOS' > base.sh
#!/usr/bin/env sh
mesh=$1
echo cxy,base > ${mesh}_base.csv
for cell in 1 2 4 8 16 32 64 128 256 512 1024; do
    pdal translate \
      /mnt/c/input/${mesh}_interpolated-ground.las \
      ${mesh}.${cell}.tiff \
      --writers.gdal.resolution=${cell} \
      --writers.gdal.output_type=idw \
      --writers.gdal.data_type=float
    gdal_fillnodata.py -q ${mesh}.${cell}.tiff
    gdal_translate -q -of xyz ${mesh}.${cell}.tiff /vsistdout/ | grep -v -e '-9999$' | awk -F " " -v cell=${cell} '{printf("%d %d %d,%.2f\n", cell, cell*(int($1/cell) - ($1 < 0)), cell*(int($2/cell) - ($2 < 0)), $3);}' >> ${mesh}_base.csv
    rm -f ${mesh}.${cell}.tiff
done
EOS
~~~

- pdal translate のオプションについて
  - 第一引数:入力ファイル（.las）
  - 第二引数:出力ファイル（.tiff）
  - `--writers.gdal.resolution=${cell}`: ラスタ1ピクセルの地理的長さ。データの座標系が平面直角座標系なので単位はメートル。
  - `--writers.gdal.output_type=idw`: 補間方式
  - `--writers.gdal.data_type=float`: 型（32bit浮動小数型）

  - `grep -v`でNO DATA行を除外
  - `awk`で座標値をボクセル1辺の長さの倍数に丸め、標高値を整数に切り捨て、CSVに整形

### 内挿補間されたグラウンドデータから標高値を格納したCSVを生成
~~~
chmod 775 base.sh
find /mnt/c/input/ -name "*_interpolated-ground.las" | xargs basename -s _interpolated-ground.las | xargs -P $(nproc) -n 1 ./base.sh
~~~

### フィルタースクリプトの記述
- このスクリプトは、pdalが出力したCSVを標準入力から読み取り、点群データの各ポイントから正方形ポリゴンを形成するNode.jsプログラムです。
- GeoJSON Seq 形式に整形して標準出力します。
- 属性:
  - `top`: ポイントのZ(m)
  - `cell`: ボクセル1辺の長さ(m)
  - `color`: カラー
- Tippecanoeオプション:
  - `layer`: ソースレイヤ名
  - `minzoom`: 最小ズーム
  - `maxzoom`: 最大ズーム
- 座標系変換: 平面直角座標系→WGS84（以下は平面直角X系（EPSG:6678）の例）

~~~
cat << 'EOS'> filter.js
#!/usr/bin/env node
const readline = require("readline");
const gdal     = require("gdal-async");

/** 第一引数: ボクセルの1辺の長さ(m) */
const CELL = parseInt(process.argv[2]);
/** 第二引数: ズームレベル */
const ZOOM = parseInt(process.argv[3]);

const transformation = new gdal.CoordinateTransformation(
      gdal.SpatialReference.fromEPSG(6678)
    , gdal.SpatialReference.fromEPSG(4326)
);

const rl = readline.createInterface({"input":process.stdin});
rl.on("line", line => {
    // 行ごとの処理
    try {
        const atts = line.toString().split(",");
        const x = CELL*Math.floor(parseFloat(atts[0])/CELL); // 座標値をCELLの倍数に丸める
        const y = CELL*Math.floor(parseFloat(atts[1])/CELL); // 座標値をCELLの倍数に丸める
        const z = parseFloat(atts[2]);
        const r = parseInt(atts[3]);
        const g = parseInt(atts[4]);
        const b = parseInt(atts[5]);
        
        // 正方形ポリゴンを形成
        const geometry = gdal.Geometry.fromGeoJson({"type":"Polygon", "coordinates":[[
              [y       , x       ]
            , [y + CELL, x       ]
            , [y + CELL, x + CELL]
            , [y       , x + CELL]
            , [y       , x       ]
        ]]});
        // 座標系変換
        geometry.transform(transformation);
        
        // featureオブジェクトを組み立てる
        /* 属性
         * "cxy":後でDEM標高をマージするためのキー
         * "top":*_feature.las のポイント標高(m)
         * "cell":ボクセル1辺の長さ(m)
         * "color":カラー
         */
        const feature = {"type":"Feature", "tippecanoe":{"layer":"feature", "minzoom":ZOOM, "maxzoom":ZOOM}, "properties":{"cxy":`${CELL} ${x} ${y}`, "top":z, "cell":CELL, "color":`rgb(${r},${g},${b})`}, "geometry":geometry.toObject()};
        // GeoJSON Seqのレコードを出力。RFC8142に則りRSコードと改行コードでラップする。
        process.stdout.write("\x1e" + JSON.stringify(feature) + "\n");
    } catch (e) {}
});
EOS
~~~


### オリジナルデータからボクセル状のベクトルタイルを作成
~~~
basezoom=20 # 1mボクセルを適用するズームレベル 値が大きいほど粗い

find /mnt/c/input/ -name "*_feature.las" | xargs -P $(($(nproc)/2)) -I{} bash -c 'basezoom='${basezoom}'; mesh=$(basename -s _feature.las {}); for i in $(seq 0 $((${basezoom} - 10))); do cell=$((2**${i})); zoom=$((${basezoom} - ${i})) ; pdal translate {} STDOUT --writers.text.order="X:0,Y:0,Z:2,Red:0,Green:0,Blue:0" --writers.text.keep_unspecified=false --writers.text.write_header=false voxeldownsize --filters.voxeldownsize.cell=${cell} | node filter.js ${cell} ${zoom} >> ${mesh}.geojsons; done'

find *.geojsons | xargs basename -s .geojsons | xargs -I{} tippecanoe -f -P -pk -pf -ah -Z 10 -z ${basezoom} -o {}.mbtiles {}.geojsons

rm -f *.geojsons
~~~

- pdal translate のオプションについて
  - 第一引数: 入力ファイル（.las）
  - 第二引数: 出力ファイル（ただしSTDOUTで標準出力）
  - `--writers.text.order="X,Y,Z:2,Red:0,Green:0,Blue:0"`: CSV項目（コロンに続く数字は小数点以下桁数）
  - `--writers.text.keep_unspecified=false`: 指定した項目以外を出力しない
  - `--writers.text.write_header=false`: CSVヘッダ行を付けない
  - 第三引数: フィルタ（voxeldownsize）
  - `--filters.voxeldownsize.cell`: ボクセル1辺の長さ（座標系が平面直角座標系なので単位はメートル）


# ボクセル状のベクトルタイルに標高値を付与
- ボクセル状のベクトルタイルと標高CSVを突き合わせて、標高値付きベクトルタイルを作成します。
~~~
find -name "*_base.csv" | xargs basename -s _base.csv | xargs -I{} sh -c 'tile-join -f -x cxy -i -pk -c {}_base.csv -o {}_basetop.mbtiles {}.mbtiles'
~~~

- tile-join のオプションについて
  - `-f`: 出力ファイルと同名の既存ファイルがあれば強制上書き
  - `-x cxy`: cxy属性をキーとしてマージ
  - `-i`: キーが突き合わなかったレコードはベクトルタイルから除外
  - `-pk`: 1タイルあたりの容量に制限をかけない
  - `-c CSVファイル名
  - `-o 出力ファイル.mbtiles
  - 引数（可変個）: 入力ファイル.mbtiles

### pbfファイルに展開
~~~
tile-join -f -pk -e basetop1 *_basetop.mbtiles
~~~

### 中間生成物を削除
~~~
rm -f *.csv *.mbtiles
~~~
