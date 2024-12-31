---
title: まっさらなUbuntu16.04LTSにOpenCV3.1とopencv_contribをインストール
tags:
  - Ubuntu
  - OpenCV
private: false
updated_at: '2017-05-24T15:00:34+09:00'
id: cfa9fde7ea2b825e4e6b
organization_url_name: null
slide: false
ignorePublish: false
---

※[2017/05/24] ソースコードの修正が不要になったので修正

GWで少し時間がとれたし，Ubuntu16.04LTSがリリースされたので手元のマシンにインストールしてみました．

## パッケージの更新
```
sudo apt-get update
sudo apt-get upgrade
```

## ソースのダウンロード
```
sudo apt-get install git
cd ; mkdir tmp ; cd tmp
git clone https://github.com/Itseez/opencv
git clone https://github.com/Itseez/opencv_contrib.git
```
ダウンロードに時間がかかるのでコーヒーでも飲みながら待つ．

## CUDAのインストール
NVIDIAのグラボを詰んでるマシンなので，CUDAもインストールしておきます．

## OpenCVのビルドに必要なパッケージのインストール
```
sudo apt-get build-dep opencv
```
でいけますが，まっさらなUbuntuだとソースが指定されていないぞと怒られるので，/etc/apt/sources.listの以下の行のコメントを外しておきます．
```deb-src http://jp.archive.ubuntu.com/ubuntu/ xenial universe```
ついでに
```sudo apt-get update```
してから，改めて
```sudo apt-get build-dep opencv```
します．
ここでnvidia-opencl-devが入っているとパッケージが削除されてocl-icd-opencl-devが代わりに入るので，nvidia-opencl-devを再度インストールしました（これでいいのかは自信なし）．
```sudo apt-get install nvidia-opencl-dev```

不要かもしれないけどVTKとGlogをインストール．
```sudo apt-get install libvtk6-qt-dev libgoogle-glog-dev```

## CMakeLists.txtの修正
このままmakeするとmemcpy関係でコンパイルエラーが出るので，~/tmp/opencv/CMakeLists.txtを編集して，先頭に以下を追加
```set(CMAKE_CXX_FLAGS "-D_FORCE_INLINES ${CMAKE_CXX_FLAGS}")```

## make
```
cd ~/tmp/opencv
mkdir build ; cd build
cmake -D WITH_TBB=ON -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules/ ..
＃CUDAインストールしてたので実際は，
cmake -D WITH_CUBLAS=ON -D WITH_TBB=ON -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules/ ..
make -j
sudo make install
```
makeには時間かかるのでおやつでも食べに行きましょう．

<!--
おやつから帰ってくると，
opencv_contrib/modules/tracking/include/opencv2/tracking/onlineMIL.hppの57行目でエラーが出て死んでます．
どうやらsignマクロが悪さをしているようなので，onlineMIL.hppの57行目の
``` #define  sign(s)  ((s > 0 ) ? 1 : ((s<0) ? -1 : 0))```
を
``` #define  sign2(s)  ((s > 0 ) ? 1 : ((s<0) ? -1 : 0))```
とでもしておいて，
opencv_contrib/modules/tracking/src/onlineMIL.cpp 310行目と339行目で呼び出されているsign関数を先ほど変更した関数名(sign2)にする．

改めてmake

```
make -j
sudo make install
```
-->

使用例はまたそのうちに．
