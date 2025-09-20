# Spectral Mixing Overview

## 背景理論
本リポジトリの`spectral.js`および`shader/spectral.glsl`は、顔料混色をモデル化するために**Kubelka–Munk理論**を採用しています。この理論は、拡散反射を行う薄層における光の吸収係数\(K\)と散乱係数\(S\)を扱い、反射率\(R\)と\(K/S\)比の間に以下の関係を与えます。

\[
K/S = \frac{(1 - R)^2}{2R}
\]

実装ではこの式を`KS(R)`として直接利用し、混色時には複数顔料の\(K/S\)比を濃度に応じて加重平均した後、Kubelka–Munkの逆変換

\[
R = 1 + \frac{K}{S} - \sqrt{\left(\frac{K}{S}\right)^2 + 2\frac{K}{S}}
\]

を`KM(KS)`として用いて再び反射率スペクトルに戻します。この近似は、顔料が完全拡散体として振る舞うという標準的なKubelka–Munk仮定に基づいており、塗料やインクの混色挙動を再現するための経験則です。

## サンプリングと定数
両ファイルはスペクトルを**38バンド**で表現します。これはおおよそ 380–780 nm を 10 nm 間隔でサンプリングした離散反射率ベクトルで、定数 `SIZE` (`SPECTRAL_SIZE`) に格納されています。sRGBとの相互変換では IEC 61966-2-1 に準拠したガンマ値 **2.4** (`GAMMA`, `SPECTRAL_GAMMA`) を用いたコンパンディング／アンコンパンディング式を採用しています。数値的不安定性を防ぐため、反射率には `Number.EPSILON` および GLSL 版の `SPECTRAL_EPSILON` を下限として適用しています。

## 既知の基底スペクトル
`spectral.js`内の`BASE_SPECTRA`とシェーダー内の対応する係数テーブルは、以下の7つの基底スペクトルを提供します。

- **W**: 理想的な白のスペクトル
- **C/M/Y**: シアン／マゼンタ／イエローの減法混色基底
- **R/G/B**: sRGB原色の高彩度スペクトル

線形RGBをスペクトルに変換する際は、線形RGB成分をホワイト、二色成分（C/M/Y）、一次成分（R/G/B）のウェイトに分解し、それぞれのスペクトルを線形結合して反射率ベクトル\(R_i\)を構築します。これにより、RGB色空間で与えられた入力でもスペクトル混色を行えるようになります。

## CIE 表色系への変換
スペクトル反射率は、D65標準光源で重み付けされた**CIE 1931 2° 標準等色関数** (`CIE.CMF`) を用いて XYZ に変換されます。

\[
\begin{bmatrix} X \\ Y \\ Z \end{bmatrix} = \mathrm{CMF} \cdot R
\]

得られた XYZ は行列 `CONVERSION.XYZ_RGB` を通じて線形RGBに変換され、さらにガンマ補正を施して8-bit sRGBを得ます。逆方向では `CONVERSION.RGB_XYZ` と OKLab 変換行列 (`XYZ_LMS`, `LMS_LAB` など) を利用し、OKLab/OKLCh 空間でのガムートマッピングや距離計算が行われます。

## Kubelka–Munk 混色アルゴリズム
### 1. 準備
各 `Color` インスタンスまたはシェーダー内の色ベクトルは、まず `spectral_srgb_to_linear` で線形RGBに変換されます。次に `lRGB_to_R`/`spectral_linear_to_reflectance` が基底スペクトルを組み合わせて反射率ベクトルを生成します。

### 2. 濃度とティンティング強度
`spectral.js` の `mix` およびシェーダーの `spectral_mix` は、各色に対して次の「濃度」を計算します。

\[
C = f^2 \cdot t^2 \cdot Y
\]

ここで \(f\) はユーザが与える混合比、\(t\) は `tintingStrength`、\(Y\) はスペクトルから計算した輝度（CIE Y成分）です。二乗を用いることで濃度パラメータの寄与を増幅し、小さな比率の色でも影響が現れるように調整しています。

### 3. KS の合成
各波長サンプルについて、前述の濃度を重みとした加重平均を行います。

\[
(K/S)_\text{mix} = \frac{\sum_i (K/S)_i \cdot C_i}{\sum_i C_i}
\]

### 4. 反射率への逆変換
得られた平均\(K/S\)を`KM`で反射率に戻し、新たなスペクトル反射率列`R`を得ます。最後に `spectral_reflectance_to_xyz` および `spectral_xyz_to_srgb` で可視色として出力します。

## JavaScript 実装 (`spectral.js`)
- **Color クラス**: CSSカラー文字列やsRGB配列からスペクトル表現までを双方向に変換し、OKLab/OKLCh 表色系、ガムート判定、ティンティング強度などを提供します。
- **ガムートマッピング**: `gamutMap`はOKLChの彩度に対して二分探索を行い、`deltaEOK`でOKLab距離を監視しながらガムート外の色を安全にクリップします。
- **ユーティリティ**: `utils` にはコンパンディング式、行列演算、線形補間などの数学的支援関数が集約されています。
- **スペクトルデータ**: `BASE_SPECTRA` と `CIE` は immutable な定数として提供され、 Kubelka–Munk 計算と表色系変換の両方に再利用されます。

## GLSL 実装 (`shader/spectral.glsl`)
GLSL 版は JavaScript 実装と同じ計算を GPU 上で行えるよう、以下の主要関数を提供します。

- `spectral_uncompand` / `spectral_compand`: sRGB と線形RGBの相互変換。
- `spectral_linear_to_reflectance`: 基底スペクトルを用いた反射率生成。
- `KS` / `KM`: Kubelka–Munk の正逆変換。
- `spectral_reflectance_to_xyz`: CIE CMF によるXYZ積分。
- `spectral_mix`: 2〜4色の混合に対応し、CPU版と同じ濃度計算・KS平均を実施。

GLSL 内の配列定数は JavaScript 版と一致しており、CPU/ GPU 間で視覚的一貫性を保ちます。`SPECTRAL_EPSILON` を通じてゼロ除算を防ぎ、`clamp` による sRGB 出力の範囲保証を行っています。

## まとめ
両実装は Kubelka–Munk 理論に基づく物理指向の混色を採用し、スペクトルデータを介して sRGB 表示と OKLab/OKLCh ガムート制御を行います。豊富な定数と行列は CIE 標準および sRGB 変換規格に由来し、JavaScript と GLSL の両環境で同等の結果を得られるよう調整されています。
