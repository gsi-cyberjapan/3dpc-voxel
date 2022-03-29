# LASデータをボクセル状のベクトルタイルにする手順 その1

## 前提条件
- 本手順書では、LASデータが格納されたフォルダパスを`C:\input\01_featuremabiki`とし、Windows on Linux Subsystem 2 (WSL2)での Debian 11を使用した場合の例を示します。
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

### SpatiumGLを取得してビルド
~~~
sudo apt -y install --no-install-recommends \
  cmake \
  libglew-dev \
  mesa-utils \
  xorg-dev

git clone https://github.com/martijnkoopman/SpatiumGL
mkdir SpatiumGL/build
cd    SpatiumGL/build
cmake .. \
  -DBUILD_SHARED_LIBS=ON \
  -DSPATIUMGL_MODULE_IDX=ON \
  -DSPATIUMGL_MODULE_GFX3D=ON \
  -DSPATIUMGL_MODULE_GFX3D_OPENGL=ON \
  -DSPATIUMGL_MODULE_IO_LAS=ON \
  -DSPATIUMGL_APP_LASINFO=ON \
  -DSPATIUMGL_APP_LASGRID=ON \
  -DSPATIUMGL_APP_LASVIEWER=ON \
  -DSPATIUMGL_APP_LASOCTREEVIEWER=ON
sudo make install
sudo cp -r bin /usr/local
cd ../..
sudo rm -rf SpatiumGL
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

### フィルタースクリプトの記述
- このスクリプトはpdalが出力したCSVを標準入力から読み取り、点群のポイントから正方形ポリゴンを形成するNode.jsプログラムです。
- GeoJSON Seq 形式に整形して標準出力します。
- 属性: 
  - `height`: ポイントのZ(m)
  - `spacing`: ボクセル1辺の長さ(m)
  - `color`: カラー
- Tippecanoeオプション:
  - `asprs0`: ソースレイヤ名
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
        const x = parseFloat(atts[0]);
        const y = parseFloat(atts[1]);
        const z = parseFloat(atts[2]);
        const r = parseInt(atts[3]);
        const g = parseInt(atts[4]);
        const b = parseInt(atts[5]);

        // 正方形ポリゴンを形成
        const geometry = gdal.Geometry.fromGeoJson({"type":"Polygon", "coordinates":[[
              [y - CELL/2, x - CELL/2]
            , [y + CELL/2, x - CELL/2]
            , [y + CELL/2, x + CELL/2]
            , [y - CELL/2, x + CELL/2]
            , [y - CELL/2, x - CELL/2]
        ]]});
        // 座標系変換
        geometry.transform(transformation);

        // featureオブジェクトを組み立てる
        /* 属性
         * "height":*_feature.las のポイント標高(m)
         * "spacing":ボクセル1辺の長さ(m)
         * "color":カラー
         */
        const feature = {"type":"Feature", "tippecanoe":{"layer":"asprs0", "minzoom":ZOOM, "maxzoom":ZOOM}, "properties":{"height":z, "spacing":CELL, "color":`rgb(${r},${g},${b})`}, "geometry":geometry.toObject()};
        // GeoJSON Seqのレコードを出力。RFC8142に則りRSコードと改行コードでラップする。
        process.stdout.write("\x1e" + JSON.stringify(feature) + "\n");
    } catch (e) {}
});
EOS
~~~


### 座標系の変換
- メートル単位で間引きを行うために、平面直角座標系に戻します。
~~~
find /mnt/c/input/01_featuremabiki/ -name "*.las" | xargs -P $(nproc) -I{} sh -c 'pdal translate {} $(basename -s .las {}).6678.laz reprojection --filters.reprojection.in_srs=EPSG:6668 --filters.reprojection.out_srs=EPSG:6678'
~~~

### LASデータをグリッド状に間引き
- SpatiumGLのlasgridコマンドでLASデータをグリッド状に間引きます。
~~~
for space in 1 2 4 8 16 32; do
    find -name "*.6678.laz" | xargs -P $(nproc) -I{} sh -c 'lasgrid -s '${space}' -i {} -o $(basename -s .6678.laz {}).'${space}'.laz'
done

rm -f *.6678.laz
~~~

### LASデータからボクセル状のベクトルタイルを作成
~~~
find -name "*.1.laz" | xargs -P $(($(nproc)/2)) -I{} bash -c 'basezoom=17; mesh=$(basename -s .1.laz {}); for i in $(seq 0 $((${basezoom} - 13))); do cell=$((2**${i})); zoom=$((${basezoom} - ${i})) ; pdal translate ${mesh}.${cell}.laz STDOUT --writers.text.order="X,Y,Z:2,Red:0,Green:0,Blue:0" --writers.text.keep_unspecified=false --writers.text.write_header=false | node filter.js ${cell} ${zoom} >> ${mesh}.geojsons; done; tippecanoe -f -P -pk -pf -ah -Z 13 -z ${basezoom} -o ${mesh}.mbtiles ${mesh}.geojsons && rm -f ${mesh}.geojsons'
~~~

- pdal translate のオプションについて
  - 第一引数:入力ファイル（.las）
  - 第二引数:出力ファイル（ただしSTDOUTで標準出力）
  - `--writers.text.order="X,Y,Z:2,Red:0,Green:0,Blue:0"`: CSV項目（コロンに続く数字は小数点以下桁数）
  - `--writers.text.keep_unspecified=false`: 指定した項目以外を出力しない
  - `--writers.text.write_header=false`: CSVヘッダ行を付けない

### pbfファイルに展開
~~~
tile-join -f -pk -e 01_featuremabiki *.mbtiles
~~~

### 中間生成物を削除
~~~
rm -f *.laz *.mbtiles
~~~
