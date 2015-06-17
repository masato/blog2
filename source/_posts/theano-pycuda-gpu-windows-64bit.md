title: "Windows7 64bitにPyCUDAとTheanoをインストールしてGPU計算する"
date: 2015-02-25 23:52:14
tags:
 - Windows7
 - MinGW-w64
 - Python
 - PyCUDA
 - Theano
 - GPU
 - CUDA
description: 前回構築したTDM-GCCとMinGW-w64で64bitのPython2.7環境にPyCUDAとTheanoをインストールしてGPUを使って計算してみます。Installing Theano with GPU on Windows 64-bitの続きを参考にして構築していきます。
---

[前回](/2015/02/24/tdm-gcc-mingw-w64-python27-windows7/)構築したTDM-GCCとMinGW-w64で64bitのPython2.7環境にPyCUDAとTheanoをインストールしてGPUを使って計算してみます。[Installing Theano with GPU on Windows 64-bit](http://pavel.surmenok.com/2014/05/31/installing-theano-with-gpu-on-windows-64-bit/)の続きを参考にして構築していきます。

<!-- more -->

## NumPyとSciPy

msys.batのショートカットをダブルクリックしてMSYSを起動します。まずMinGW-w64環境にpipをインストールします。

``` bash
$ wget --no-check-certificate -O-  https://bootstrap.pypa.io/get-pip.py | python
```

[NumPy](http://www.numpy.org/)と[SciPy](http://www.scipy.org/)は、[Unofficial Windows Binaries for Python Extension Packages](http://www.lfd.uci.edu/~gohlke/pythonlibs/)のパッケージを使います。


[NumPy](http://www.lfd.uci.edu/~gohlke/pythonlibs/#numpy)は`numpy‑1.8.2+mkl‑cp27‑none‑win_amd64.whl`をダウンロードしてpip installします。

``` bash
$ pip install /c/Users/masato/Downloads/numpy-1.8.2+mkl-cp27-none-win_amd64.whl
```

[SciPy](http://www.lfd.uci.edu/~gohlke/pythonlibs/#scipy)は`scipy-0.15.1-cp27-none-win_amd64.whl`をダウンロードします。

``` bash
$ pip install /c/Users/masato/Downloads/scipy-0.15.1-cp27-none-win_amd64.whl
```

## PyCUDA

[CUDA Toolkit 6.5](https://developer.nvidia.com/cuda-downloads)は[以前](/2015/02/19/visual-stuidio-community-2013-cuda-65/)に[Windows 7 64bit版のインストーラー](http://developer.download.nvidia.com/compute/cuda/6_5/rel/installers/cuda_6.5.14_windows_general_64.exe)をダウンロードしてインストールしてあります。C/C++のCUDA拡張でプログラムを書くのは大変です。Pythonを使ってコンパイル不要で実行できるのは便利です。[PuCUDA](http://www.lfd.uci.edu/~gohlke/pythonlibs/#pycuda)は`pycuda-2014.1+cuda6519-cp27-none-win_amd64.whl`をダウンロードします。

``` bash
$ pip install /c/Users/masato/Downloads/pycuda-2014.1+cuda6519-cp27-none-win_amd64.whl
```

### nvccがcl.exeを見つけられない

[Error compiling CUDA from Command Prompt](http://stackoverflow.com/questions/8125826/error-compiling-cuda-from-command-prompt)を参考にして、 `c:\mingw\msys\msys.bat`の最初にVisual StudioのPATHを設定します。

```bat c:\mingw\msys\msys.bat
rem ember value of GOTO: is used to know recursion has happened.

call "C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\bin\x86_amd64\vcvarsx86_amd64.bat"

if "%1" == "GOTO:" goto %2
```

### UnicodeDecodeErrorがでる

このままpycudaパッケージを使うと実行時にUnicodeDecodeErrorが出てしまします。[UnicodeDecodeError: 'utf8' codec can't decode byte #37](https://github.com/inducer/pycuda/issues/37)によると、nvccからUTF-8として有効でない文字が出力されているようです。compiler.pyの250行目辺りに以下のコードを追加します。

```python c:\Python27\lib\site-packages\pycuda\compiler.py
...
        self._check_arch(arch)

        if options is not None:
            options.extend(["-Xcompiler","/wd 4819"])
        else:
            options = ["-Xcompiler","/wd 4819"]
    
        cubin = compile(source, nvcc, options, keep, no_extern_c,
                arch, code, cache_dir, include_dirs)
...
```

### テスト

こちらの[Tutorial](http://documen.tician.de/pycuda/tutorial.html)を参考にしてサンプルコードを用意して実行してみます。

```python ~/pycuda-test.py
import pycuda.gpuarray as gpuarray
import pycuda.driver as cuda
import pycuda.autoinit
import numpy
a_gpu = gpuarray.to_gpu(numpy.random.randn(4,4).astype(numpy.float32))
a_doubled = (2*a_gpu).get()
print a_doubled
print a_gpu
```

簡単な配列計算の実行ですが、PyCUDAのインストールに成功したようです。

``` bash
$ python pycuda-test.py
[[-5.03633499 -1.2504214   0.87832445  0.86452949]
 [ 0.01358639  0.98460299  2.50277209  3.76451278]
 [ 0.14301641 -2.11651611  0.39508429 -0.88320005]
 [ 2.66914272  0.87537336 -1.17862082  1.94231701]]
[[-2.5181675  -0.6252107   0.43916222  0.43226475]
 [ 0.0067932   0.49230149  1.25138605  1.88225639]
 [ 0.07150821 -1.05825806  0.19754215 -0.44160002]
 [ 1.33457136  0.43768668 -0.58931041  0.9711585 ]]
```

## Theano

[Theano](http://deeplearning.net/software/theano/)は最近話題のDeep Learningで使うPythonの数値計算ライブラリです。実行時のC++コード生成と、CUDAでGPU計算もできるのが特徴です。[Theano](http://www.lfd.uci.edu/~gohlke/pythonlibs/#theano)は`Theano-0.6.0-py2-none-any.whl`をダウンロードします。

``` bash
$ pip install /c/Users/masato/Downloads/Theano-0.6.0-py2-none-any.whl
```

MSYSのホームディレクトリにTheanoの環境設定ファイルを作成します。

``` bash ~/.theanorc
[global]
device = gpu
floatX = float32
```

### inconsistent dll linkage

[サンプルコード](http://deeplearning.net/software/theano/tutorial/using_gpu.html)を実行すると、`'round' : dll リンクが一貫していません。`というエラーがでます。

``` bash
$ python theano-test.py
...
c:\python27\include\pymath.h(22) : warning C4273: 'round' : dll リンクが一貫して
いません。
        c:\program files\nvidia gpu computing toolkit\cuda\v6.5\include\math_fun
ctions.h(2455) : 'round' の前の定義を確認してください
```

["inconsistent dll linkage" when using Theano with CUDA 6.5 #2055](https://github.com/Theano/Theano/issues/2055)にissueがありました。リポジトリの[cuda_ndarray.cuh](https://github.com/Theano/Theano/blob/master/theano/sandbox/cuda/cuda_ndarray.cuh)はすでに修正済みです。[Unofficial Windows Binaries for Python Extension Packages](http://www.lfd.uci.edu/~gohlke/pythonlibs/)で配布しているパッケージは未対応なので`#include <algorithm>`を追加します。

```c: C:\Python27\Lib\site-packages\theano\sandbox\cuda\cuda_ndarray.cuh
#ifndef _CUDA_NDARRAY_H
#define _CUDA_NDARRAY_H

#include <algorithm>
```

### テスト

最初にtheanoパッケージが正常にインストールされているか確認します。

``` bash
$ pip install nose
$ python -c 'import theano; print theano.config' | less
...
```

[Testing Theano with GPU](http://deeplearning.net/software/theano/tutorial/using_gpu.html)あるサンプルコードを実行してみます。

``` python ~/theano-test.py
from theano import function, config, shared, sandbox
import theano.tensor as T
import numpy
import time

vlen = 10 * 30 * 768  # 10 x #cores x # threads per core
iters = 1000

rng = numpy.random.RandomState(22)
x = shared(numpy.asarray(rng.rand(vlen), config.floatX))
f = function([], T.exp(x))
print f.maker.fgraph.toposort()
t0 = time.time()
for i in xrange(iters):
    r = f()
t1 = time.time()
print 'Looping %d times took' % iters, t1 - t0, 'seconds'
print 'Result is', r
if numpy.any([isinstance(x.op, T.Elemwise) for x in f.maker.fgraph.toposort()]):
    print 'Used the cpu'
else:
    print 'Used the gpu'
```

まだ警告がでますが最後に`Used the gpu`と表示されるのでGPU計算ができているようです。

``` bash
$ python theano-test.py
...
c:\python27\include\pymath.h(22) : warning C4273: 'round' : dll リンクが一貫して
いません。
        c:\program files\nvidia gpu computing toolkit\cuda\v6.5\include\math_fun
ctions.h(2455) : 'round' の前の定義を確認してください
c:\python27\lib\site-packages\numpy\core\include\numpy\npy_1_7_deprecated_api.h(
12) : Warning Msg: Using deprecated NumPy API, disable it by #defining NPY_NO_DE
PRECATED_API NPY_1_7_API_VERSION
   ライブラリ C:/Users/masato/AppData/Local/Theano/compiledir_Windows-7-6.1.7601
-SP1-Intel64_Family_6_Model_58_Stepping_9_GenuineIntel-2.7.0-64/tmptykper/73dc8d
0fb3ff37614826ceae3280f478.lib とオブジェクト C:/Users/masato/AppData/Local/Thea
no/compiledir_Windows-7-6.1.7601-SP1-Intel64_Family_6_Model_58_Stepping_9_Genuin
eIntel-2.7.0-64/tmptykper/73dc8d0fb3ff37614826ceae3280f478.exp を作成中

[GpuElemwise{exp,no_inplace}(<CudaNdarrayType(float32, vector)>), HostFromGpu(Gp
uElemwise{exp,no_inplace}.0)]
Looping 1000 times took 1.23299980164 seconds
Result is [ 1.23178029  1.61879349  1.52278066 ...,  2.20771813  2.29967761
  1.62323296]
Used the gpu
```


