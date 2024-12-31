---
title: WSL2上にGPU対応のPyTorchをインストールするまで
tags:
  - PyTorch
private: true
updated_at: '2024-12-31T23:56:47+09:00'
id: 743381d516523ded5ff4
organization_url_name: null
slide: false
ignorePublish: false
---

手元のRTX 3060が入ったゲーミングノートPCでPyTorchを使えるようにしたかったので年末に作業しました．
そのときの操作ログです．

## 1. NVIDIAのドライバを入れる

[NVIDIAのサイト](https://www.nvidia.com/ja-jp/drivers/) から自分のPCに入ってるGPUに合ったドライバをダウンロードしてくる．
自分が使ってるGPUがよくわかんない場合は，[NVIDIA アプリ](https://www.nvidia.com/ja-jp/software/nvidia-app/) 経由でインストールするか，「デバイスマネージャ」を開いて「ディスプレイ アダプタ」を見て確認しましょう．

## 2. WSL2をインストール

省略．
[マイクロソフトのサイト](https://learn.microsoft.com/ja-jp/windows/wsl/install)
あたりを参照してください．

PowerShellを管理者権限で開いて

```PowerShell
wsl --install
```

でいけるはず．

### 動作確認

WSL2からGPUが認識されているかを確認

```bash
nvidia-smi
```

こんな感じでGPUの名前がリストに上がっていればOK.

```script
$ nvidia-smi
Sun Dec 29 09:58:02 2024
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 565.77.01              Driver Version: 566.36         CUDA Version: 12.7     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3060 ...    On  |   00000000:01:00.0  On |                  N/A |
| N/A   50C    P8             15W /  115W |     885MiB /   6144MiB |      6%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

## 3. CUDA ToolKitのインストール

WSL2のターミナル上で実行します．
[https://developer.nvidia.com/cuda-downloads](https://developer.nvidia.com/cuda-downloads)
から

- Operating System: Linux (Windowsじゃないです)
- Architecture: x86_64
- Distribution: WSL-Ubuntu
- Version:2.0
- Installer Type: deb(network) ※どれでもいいです

を選択すると，以下のような感じの"Installation Instructions"が表示されるので，WSL2のターミナル上で順番に入力して実行します．

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-6
```

## nvcc

ここまでの操作で`/usr/local/cuda/bin/nvcc`が使えるようになっているはず．

使わないかもしれないけど，`nvcc`だけで起動できるように環境変数を設定しておく．

[https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#environment-setup](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#environment-setup)参照．

以下の2行を`~/.bashrc`に追記．※12.6の部分はインストールしたバージョンに合わせて変更してください．

```bash
export PATH=/usr/local/cuda-12.6/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-12.6/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
```

以下のコマンドで設定を反映

```bash
source ~/.bashrc
```

これでnvccが使えるようになる．

```bash
nvcc --version
```

こんな感じの表示になればOK

```script
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2024 NVIDIA Corporation
Built on Tue_Oct_29_23:50:19_PDT_2024
Cuda compilation tools, release 12.6, V12.6.85
Build cuda_12.6.r12.6/compiler.35059454_0
```

## 4. python環境のインストール

anacondaは有料化されてしまったので，[miniforge](https://github.com/conda-forge/miniforge)を使います．

インストールは以下のコマンドでできます．

```bash
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

いろいろ質問されるけど、基本的にはyesと入力したり，ENTERキーを押していくだけでOK.最後の

```script
You can undo this by running `conda init --reverse $SHELL`? [yes|no]
[no] >>> yes
```

はyesと入力してENTER．

```bash
source ~/.bashrc
```

を実行して設定を反映．

### python動作確認

```bash
python --version
```

2024/12時点で`Python 3.12.8`です．とりあえずバージョン番号が表示されればOK．

## 5. 新しい環境作成

base環境のまま作業を進めてもいいのですが，なんとなく気持ち悪いので新しく環境を作りましょう．
以降は`myenv`という名前で新しく環境を作っていきます．（名前はお好きにどうぞ）

以下のコマンドでpythonのバージョンを指定して環境を作成します．

```bash
mamba create -n myenv python=3.12.8
```

以下のコマンドで環境をbase⇒myenvに切り替えます．

```bash
mamba activate myenv
```

切り替わっているかどうかは，プロンプトの左側に環境名が表示されているかで確認できます．
こんな感じ．

```script
(myenv) user@mynotebook:~$
```

### 5.1 いろいろパッケージをインストール

anacondaが使えないので，scikit-learnやpandas，numpy，matplotlibなどは自分でインストールしないと駄目です．
環境が`myenv`に切り替わっていることを確認してから以下のコマンドを実行してください．(何を入れるかはお好みで)

```bash
mamba install numpy pandas scikit-learn matplotlib
```

※`conda`コマンドでもいいのですが，パッケージ間の依存解決が糞遅いので`mamba`コマンドを使っています．

## 6. pytorchをインストール

PyTorch 2.5以降はcondaの正式サポートを終了しました．仕方ないのでconda-forgeチャンネルを使ってインストールします．

```bash
mamba install pytorch torchvision torchaudio
```

注：このときにインストールされるpytorch等のバージョンが表示されるので確認してください．こんな感じ

```script
  + pytorch                    2.5.1  cuda126_py312h1763f6d_306  conda-forge       37MB
  + torchvision               0.20.1  cuda126_py312_h17ccbaa_4   conda-forge        3MB
  + torchaudio                 2.5.1  cuda_126py312haeea92a_0    conda-forge        6MB
```

バージョン番号（2.5.1）あとにcudaのバージョン（cuda126）が表示されていることを確認してください．
これがcpu版の場合は`cpu`と表示されます．GPU版でないとGPUが宝の持ち腐れになります．

以降，別のパッケージを入れようとするとpytorchがCPU版になってしまうことがたまにあるので，
インストールされるときに表示されるバージョンを都度確認してください．

### PyTorchの動作確認

`torch.cuda.is_available()`がTrueになることを確認

```bash
python -c "import torch; print(torch.cuda.is_available())"
```

## 7. jupyter lab

いつもVSCodeでコーディングしているのでjupyter labの出番はほとんどないのですが，ついでなのでjupyter labまで入れてしまいます．

```bash
mamba install jupyterlab
```

ターミナルから`jupyter lab`と入力すると，

```script
    To access the server, open this file in a browser:
        file:///home/user/.local/share/jupyter/runtime/jpserver-50231-open.html
    Or copy and paste one of these URLs:
        http://localhost:8888/lab?token=f2**********************************************
        http://127.0.0.1:8888/lab?token=f2**********************************************
```

みたいな表示が出るので，そのURLにアクセスすればjupyter labが使えます．

以上．
