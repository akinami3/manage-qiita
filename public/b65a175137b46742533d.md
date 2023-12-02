---
title: 【図解】三角関数による順運動学の解法【Python】
tags:
  - Python
  - Robot
  - ロボット
  - robotics
private: false
updated_at: '2023-09-18T02:47:32+09:00'
id: b65a175137b46742533d
organization_url_name: null
slide: false
ignorePublish: false
---
:::note
こちらの記事は，[ロボティクスの辞書的な記事（にしたい記事）](https://qiita.com/akinami/items/eb0741b0d9c322e5d5ec)のコンテンツです．
:::

事前に必要な知識（見ておいた方が良い記事）：[順運動学【概要】](https://qiita.com/akinami/items/676c0cb77f26ba2b1064)



# 三角関数による順運動学の解法
（おさらい）
順運動学とは，あるロボットアームにおいて

:::note
「各関節の角度」から「エンドエフェクタの位置，姿勢」を求める
:::
問題です．

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/0e783f0e-a635-af25-85ad-3d6a69d427c9.png)

（おさらいここまで）

---

今回は，「三角関数」を用いた順運動学の解法を紹介します．

## 2軸の回転関節を持ったロボットアーム
2軸の回転関節を持ったロボットアームの順運動学を解いてみましょう．

一気に「エンドエフェクタの位置 $(x_{\rm{e}}, y_{\rm{e}})$」まで求めても良いですが，ここでは
1. **リンク1の根本（=原点）からみた時の**リンク1の先端位置 $(x_1, y_1)$
1. **リンク2の根本からみた時の**リンク2の先端位置（=エンドエフェクタの位置） $(x_2, y_2)$
1. **リンク1の根本（=原点）から見た時の**エンドエフェクタの位置$(x_{\rm{e}}, y_{\rm{e}})$

の順番で求めていくことにします．

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/531ef951-4611-eebf-81de-05a6560d3af7.png)


### 1. リンク1の根本（=原点）からみた時のリンク1の先端位置

「**リンク1の根本から見た時の**リンク1の先端位置$(x_1,y_1)$」は，
- リンク1の長さを$L_1$
- リンク1の関節角度を$\theta_1$

とした時
```math
\begin{align}
x_{\rm{1}} &= L_1\cos(\theta_1)\\
y_{\rm{1}} &= L_1\sin(\theta_1)
\end{align}
```
となります[^1]．
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/ef9ae0e2-694b-a8da-4f05-b2dd09f31b6f.png)


### 2. リンク2の根本からみた時のリンク2の先端位置（=エンドエフェクタの位置）

次に，
「**リンク2の根本から見た時の**リンク2の先端位置$(x_{2}, y_{2})$」は，
- リンク2の長さを$L_2$
- 各関節の角度を$\theta_1, \theta_2$

とした時
```math
\begin{align}
x_{2} &= L_2\cos(\theta_1+\theta_2)\\
y_{2} &= L_2\sin(\theta_1+\theta_2)
\end{align}
```
となります．

:::note warn
ここで， $(x_2, y_2)$ を求める際は，$\theta_1$も考慮する必要があることに注意してください．
:::

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/77dc418e-ad1b-d1ef-0187-d3f538878b6a.png)



### 3. リンク1の根本（=原点）から見た時のエンドエフェクタの位置
ここまでに求めた位置を足し合わせることで，「**リンク1の根本（=原点）から見た時の**エンドエフェクタの位置$(x_{\rm{e}}, y_{\rm{e}})$」を求めることができます．

```math
\begin{align}
x_{\rm{e}} &= x_1 + x_2\\
y_{\rm{e}} &= y_1 + y_2
\end{align}
```

:::note
多くのロボットアームにおいて，
- 「$i$番目のリンクの根本」と「$i+1$番目のリンクの先端」の位置

は一致しているため，上記の式のように単純に足し合わせることで**リンク1の根本（=原点）からみた時の**エンドエフェクタの位置を求めることができます．もしも，一致していない場合はそれを考慮する必要があるので注意してください．
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/d90605de-8082-3cdf-e4ed-d6e466e53cf6.png)

:::

## n軸の回転関節を持ったロボットアーム
$n$軸の関節があるロボットアームの場合も，これまでと同様に
```math
\begin{align}
x_{\rm{e}} &= x_1 + x_2 + ... x_n\\
y_{\rm{e}} &= y_1 + y_2 + ... y_n
\end{align}
```

といった手順で，ロボットアームの根本から順番に解いていけばOKです．

# Pythonで三角関数による順運動学を実装
ここまでで，三角関数を用いた順運動学の解法がわかったと思うので，ここからはPythonで実装してみましょう．

今回は，上記で示したような
- 平面2自由度ロボットアーム（回転関節）

の例を実装します．

下記コードの
- 各変数の設定（ここの値を変更すれば順運動学の結果が変わる）
 
の箇所を変更すれば順運動学の結果が変わります．
各変数の数値を色々変更して結果が変わることを確認してみてください．

```Python3
import numpy as np
import matplotlib.pyplot as plt
from sympy.geometry import *

#  -------- 各変数の設定（ここの値を変更すれば順運動学の結果が変わる）------------
link1_length = 2.0; theta1 = 30 # link1の長さ, 関節角度
link2_length = 1.0; theta2 = 45 # link2の長さ, 関節角度
#  ---------------------------------------------------------------------
# 各角度をdegree -> radianに変換
theta1, theta2 = np.radians(theta1), np.radians(theta2)

# ============= 三角関数による順運動学（メイン部分）=========================
# link1の根本から見た時の，link1の先端位置
x1 = link1_length*np.cos(theta1)
y1 = link1_length*np.sin(theta1)

# link2の根本から見た時の，link2の先端位置（=エンドエフェクタの位置）
x2 = link2_length*np.cos(theta1+theta2)
y2 = link2_length*np.sin(theta1+theta2)

# link1の根本（原点座標）から見た時の，エンドエフェクタの位置
xe = x1 + x2
ye = y1 + y2
# ======================================================================


# ----------- 描画（ここからは，順運動学と直接的に関係はなし） ----------------
fig = plt.figure(figsize=(9,3))
ax1 = fig.add_subplot(1, 1, 1)
ax1.set_aspect("equal")#画像の比率を同じにする
plt.axes(ax1)

# 描画範囲：リンクを伸ばし切った時の長さを描画範囲に設定
ax1.set_xlim(-(link1_length+link2_length), link1_length+link2_length)
ax1.set_ylim(-(link1_length+link2_length), link1_length+link2_length)

s0 = Segment(Point(0.0 ,0.0), Point(x1, y1)) # リンク1
s1 = Segment(Point(x1, y1), Point(xe, ye)) # リンク2 
segments = [s0, s1]

# 各リンクを描画
for i,s in enumerate(segments) :
    ax1.plot([s.p1.x, s.p2.x], [s.p1.y, s.p2.y], lw=5)

plt.show()
```
- 結果例
    ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/772d88f3-2307-38dc-a1c8-d34205d0b777.png)

# まとめと「同時変換行列による解法」について少し
今回は，三角関数を使用した順運動学の解法について説明しました．

三角関数による解法には
- 図で説明できるので，直感的に理解できる

といったメリットがあります．

<br>
<br>

一方で，
- ロボットアームの構成が複雑になると，数式も複雑になる

といったデメリットもあります．

今回は，2次元平面上のロボットアームを想定しましたが，実世界は「3次元空間」になるため，数式はもっと複雑になります．

このように，三角関数を用いた解法は直感的である一方，ロボットが複雑になると，数式を考えるのが難しくなってしまいます．

<br>
<br>

そこで，
- 同次変換行列

を用いた解法の登場です．

同次変換行列による解法は，三角関数による解法と比較すると直感的ではありませんが，機械的にエンドエフェクターの位置$(x_{\rm{e}}, y_{\rm{e}})$を求めることができるため，コンピュータで扱うときなども非常に便利です．

次回はこの「[同次変換行列を使用した順運動学の解法](https://qiita.com/akinami/items/9e65389929cedb1c9551)」について解説します．

<br>
<br>

以上で，「三角関数による順運動学」の説明は終わりになります．ロボティクスに興味のある方のお役に少しでも立てたら幸いです．

# 参考文献
- [【図解】運動学入門：順運動学と逆運動学について解説！](https://agirobots.com/introduction-to-kinematics/)
- [ロボット工学の基礎　順運動学と逆運動学を理解する](https://tajimarobotics.com/kinematics/)

[^1]:余談ですが，$\sin, \cos, \tan$はそれぞれの「頭文字の筆記体」をイメージすると，分子/分母の関係が覚えやすいです（参考:[三角関数の基礎知識。sinθ cosθ tanθ の覚え方・弧度法・三角比の表まとめ](https://atarimae.biz/archives/18037)）．



<!--
すなわち，順運動学では
- 「$i$番目のリンクの根本から先端までの相対位置」を求めてその総和をとる

ということをすれば，エンドエフェクタの位置 $(x_{\rm{e}}, y_{\rm{e}})$を求めることができます．


先ほどの「$n$軸の回転関節を持ったロボットアーム」の数式は

-  「原点位置$(x_0, y_0)$」　
→ 「リンク１の先端位置$(x_1, y_1)$」 
→ 「リンク2の先端位置$(x_2, y_2)$」 
→ ... 
→ 「リンクnの先端位置（=エンドエフェクタの位置）$(x_{\rm{e}}, y_{\rm{e}})$」

という順番で，求めていることになります．

ここで
- 「リンク$i$の先端位置」は「リンク$i+1$の根本位置」と等しい
  - ただし，$n$番目の最先端のリンクは除く

ので，それを考慮して書き直すと
-  「リンク１の根本位置$(x_0, y_0)$」　
→ 「リンク2の根本位置$(x_1, y_1)$」 
→ 「リンク3の根本位置$(x_2, y_2)$」 
→ ... 
→ 「エンドエフェクタの位置）$(x_{\rm{e}}, y_{\rm{e}})$」

という順番でエンドエフェクタの位置を求めていると言えます．





## 1軸の回転関節を持ったロボットアーム
最初に，下図のような1軸の回転関節を持ったロボットアームについて説明します．
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/36d08ba6-bcda-5b60-9e49-e107834dba1a.png)

「エンドエフェクタの位置 $(x_{\rm{e}},y_{\rm{e}})$」は，「リンクの長さを$L_1$」, 「関節の角度を$\theta_1$」とした時

```math
\begin{align}
x_{\rm{e}} &= L_1\cos(\theta_1)\\
y_{\rm{e}} &= L_1\sin(\theta_1)
\end{align}
```
で求めることができます[^1]．
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/7d2e19b6-bb7a-3ca6-871f-1d9371e8cb6a.png)

-->
