# SigRe: Signal Regenerator

## Usage

```
$ sigre <nTrain> <nRegenBlockSize> <Regenerator Type> [Regenerator Parameters...] @ <Basis Function Builders...>
```

## Build on Docker

`dffdd`ディレクトリで以下を実行します．

```sh
$ docker build -t sigre -f apps/sigre/Dockerfile .
```

そうすると次のように実行できます．

```sh
$ docker run --rm sigre <args...>
```

## Examples

```
$ sigre 1000 80 PHammLS 32 @ OFDMUpSampler 64 16 8 % BFList X[1,0] L[0,1] X[3,0] M[0,1,*1] % OFDMDownSampler 64 16 8 % Orthonormalizer
```

## Basis Function Builder

以下のように`%`でブロック定義をつなげていくことで，複雑な多次元基底信号を作成できます．
このとき，BlockA -> BlockB -> BlockCという順番で作用していきます．

```
<BlockA> % <BlockB> % <BlockC>
```

### BFList

与えられた1次元の入力信号を，各基底関数に作用させます．

```
BFList <基底関数のリスト>
```

基底関数のリストには以下のような三つのものを使えます．

```
X[p,q] := x^p conj(x[n])^q
L[p,q] := 平均電力1で正規化された正規直交2次元ラゲール多項式
M[a,*b,c,d,*e,...] := x[n-a] * conj(x[n-b]) * x[n-c] * x[n-d] * conj(x[n-e]) * ...
```

注意として，この表記にスペース等を入れて`X[p, q]`だったり，`X [ p, q ]`のようにしてはいけません．
スペースを含まない形式で入力する必要があります．

次のように基底を組み合わせることができ，この場合は(x, Laguerre2D(2,0), x[n-1] * x[n-2]^4 * conj(x[n-3]^2) * x[n-3])の三つの基底になります．

```
BFList X[1,0] L[2,0] M[1,2,2,2,2,3,*3,*3,3]
```

以下のように多次元の信号を入力に与えることはできません．

```
# 以下の例では，後者の`BFList X[2,0]`の入力には，`BFList X[1,0] X[1,2]`からの2次元の信号が与えられるのでエラーになる
BFList X[1,0] X[1,2] % BFList X[2,0]
```


### Orthonormalizer

与えられた多次元の入力信号を正規直交化します．

```
# 3次元の信号を正規直交化する
BFList X[3,0] L[1,0] M[1,2,3] % Orthonormalizer
```

1次元の入力信号が与えられた場合は，単純に平均電力を1に正規化するだけです．

```
# 平均電力を1に正規化してから基底関数に入れる
Orthonormalizer % BFList L[3,0] L[1,0] L[2,3]
```


### OFDMUpSampler / OFDMDownSampler

入力信号の各次元がFFTサイズ`nFFT`，CP長`nCP`のOFDM信号だと仮定して，各次元でそれぞれアップサンプリング及びダウンサンプリングします．

```
OFDMUpSampler <nFFT> <nCP> <nUpScale>
OFDMDownSampler <nFFT> <nCP> <nDownScale>
```

`OFDMUpSampler`をかけた後に同一パラメータの`OFDMDownSampler`をかけると元に戻ります．

```
# 以下は無意味
OFDMUpSampler 32 4 8 % OFDMDownSampler 32 4 8
```

`OFDMDownSampler`には`0,1,-1,...`のような形式で整数値を引数に追加できます．
たとえば`0,1`のときには，OFDMDownSamplerの出力にはエイリアシングを考慮して，隣接する上側のバンドも出力されます．
`0,1,-1`であれば，主帯域と隣接する上側と下側の両方のバンドも出力されます．
この機能は，以下のように非線形変換によって生じるエイリアシングを再現するために利用可能です．
この機能を用いると，入力信号の次元数をdとすると，出力信号の次元数は`0,1`であればd*2次元，`0,1,-1`であればd*3のように，リストの個数倍の次元の信号が出力されます．

```
OFDMUpSampler 32 4 8 % BFList X[1,0] X[2,1] X[3,2] X[4,3] % OFDMDownSampler 32 4 8 0,1,-1,2,-2 % RemoveZeroBasis 1e-10
```


### RemoveZeroBasis

入力信号の中で，平均電力は指定値以下のものを除きます．
たとえば，OFDMDownSamplerの出力に入れることを想定しています．

引数として，信号の平均電力がゼロかどうかを判断するしきい値を指定できます．
デフォルト値は1e-10です．
