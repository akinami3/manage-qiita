---
title: 【図解】PID制御【Python】
tags:
  - Python
  - アニメーション
  - PID
  - 制御工学
  - 図解
private: false
updated_at: '2023-12-02T23:00:35+09:00'
id: 29a851c7051b1968b396
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
お疲れ様です，秋並です．

本記事では、できるだけ図解しながらPythonでPID制御を実装します。なお、どういった流れで制御しているのかを知るために
- 制御関係のライブラリを使用せずに

実装します。そのため、後述する「定常偏差」「オーバーシュート」などは正確にシミュレーションできないので，ご留意ください．

MacBook Air(M1, 2020), macOS Monterey バージョン12.6 Python3.10.9 で動作確認しました．なお，本記事内の全てのコードはgoogle Colaboratory(2023/1/23 現在)でも動作します．

**また、本記事に出てくるコードはnotebook上で動作させる（「ipynb」）ことを前提としているため「py」ファイルでは一部のコードは動作しないのでご注意ください（主に描画関連）**

# PID制御とは
PID制御は制御方法の1つであり，シンプルな仕組みでありながら，多くの場合においてそこそこ上手く動作するため広く利用されています．

PID制御は
- P制御（比例制御）
- I制御（積分制御）
- D制御（微分制御）

を組み合わせた制御方法です．

<br>
<br>

順番に説明した方が理解しやすいので「P制御」→「PI制御」→「PID制御」の順番で説明します．

なお，PID制御は，様々な制御に用いることがてきますが，今回は下図に示すような1軸のロボットアームを$0^\circ\rightarrow 90^\circ$に制御するパターンを考えます（普通はradianで計算しますが，分かりやすさのため今回はdegreeにします）．
![1deg_robot_arm.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/ed32e3ca-cdc5-41d9-ec26-dc5bd0373529.png)


# P制御
P制御は比例制御とも呼ばれ、その名の通り現在角度$\theta_{\rm{current}}$ と目標角度$\theta_{\rm{goal}}$ の差に比例した操作量$m$ を出力します。式にすると，以下のようになります．
```math
m = K_{\rm{P}}e \\
```
:::note 
$K_{\rm{P}}$ : 比例ゲイン（P制御をどのくらい反映させるかを決定する定数） \
$e = \theta_{\rm{goal}} - \theta_{\rm{current}}$ : 偏差（現在角度$\theta_{\rm{current}}$ と目標角度 $\theta_{\rm{goal}}$ の差） \
$m$ : 操作量（出力トルク，出力電圧など）
:::

<br>

例えば、下図のように
1. 現在角度$\theta_{\rm{current}}$ と目標角度$\theta_{\rm{goal}}$ が**遠ければ**、偏差$e$ が大きくなるため，**おおきな操作量$m$ を出力**し，
1. 逆に、現在角度$\theta_{\rm{current}}$ と目標角度$\theta_{\rm{goal}}$ が**近くなれば**偏差$e$ が小さくなるので**小さな操作量$m$ を出力**

するように制御されます．
![pid_overview.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/af463ef4-fc38-0ed0-0870-29c8c32dfadd.png)


<br>

日常生活を例にすると，車を運転していて赤信号になった時，**少しずつ**ブレーキの踏み具合を大きくして，信号の直前（目標地点）で停止するのがP制御のイメージです（あくまでイメージです）．
![pid_example_car.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/acecf946-85f5-056f-8ea3-5986f3cbdb82.png)


<br>

それでは，PythonでP制御を記述してみましょう．なお，今回は簡易的なシミュレーションなので．操作量$m$ を現在角度 $\theta_{\rm{current}}$ にそのまま足しています．実際には，操作量 $m$ をもとにモータを動かします（$m$ = 電圧，トルクなど）．
![compare_sim_real.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/d5c13639-10ad-aceb-4be5-5b65fddae5a7.png)





```python
from matplotlib import pyplot as plt # 描画用ライブラリ

# 関数定義 --------------------
# P制御
def P(kp, theta_goal, theta_current):
    error = theta_goal - theta_current # 偏差（error）を計算
    m = kp*error # 操作量（Manipulative Variable）を計算（例：電圧，トルクなど）
    return m

# 係数や初期値，目標値を設定 ------
kp = 0.01 # 比例ゲイン
theta_start = 0.0 # 初期角度
theta_goal = 90.0 # 目標角度
time_length = 1000 # 計測時刻
theta_current = theta_start # 現在角度
time_list = [0] # 時刻のリスト（描画用）
theta_list = [theta_start] # 現在地のリスト（描画用）

# 制御 -------------------------
for time in range(time_length):
    m = P(kp, theta_goal, theta_current) # 操作量を計算
    theta_current += m # 現在角度に操作量を足す（実際は，この操作量をもとにモータを動かす）
    time_list.append(time) # 描画用
    theta_list.append(theta_current) # 描画用

# 描画 --------------------------
plt.hlines([theta_goal], 0, time_length, "red", linestyles='dashed') #ゴールを赤色の点線で表示
plt.plot(time_list, theta_list, label="P", color="blue") # P制御のグラフを描画
plt.xlabel(r'$t$') 
plt.ylabel(r'$\theta$') 
plt.legend(loc='lower right') # 凡例を表示
plt.show() # 表示
```
<br>

実行結果を見てみると，目標角度 $\theta_{\rm{goal}}= 90^\circ$になめらかな曲線を描いて到達していることが分かります．
![P_control.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/1b919dda-27a6-b468-9b65-95ded06a498d.png)


# PI制御
先ほどまでで，P制御により上手く目標角度 $\theta_{\rm{goal}}$ に制御することができました．  
しかし，これは簡易的なシミュレーションなため上手くいっただけで，実際に制御をしてみると
- 定常偏差

と呼ばれるものが発生します．定常偏差とは下図のように，目標角度 $\theta_{\rm{goal}}$ に到達する直前で収束してしまう現象です．
![teijyo_hensa_P.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/b4689b46-c868-0ac4-35bd-631915b84049.png)


この現象が起こる原因は様々ありますが，今回のロボットアームの場合を考えてみます[^1]．
1. ロボットアームを$0^\circ$から$90^\circ$に制御する場合，最初のうちは偏差$e = \theta_{\rm{goal}}-\theta_{\rm{current}}$ が大きいため，操作量$m$ も大きくなります．
1. ロボットアームが徐々に目標角度$\theta_{\rm{goal}}$ に近づいていくと $e$ が小さくなり，操作量$m$ も小さくなります．
1. 目標角度$\theta_{\rm{goal}}$ の直前になると，操作量$m$ が小さくなりすぎて，ロボットアームを動かすのに十分な出力を出すことができなくなります．
![P_disadvantage.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/190602da-19b9-ea96-b056-0229c8de590b.png)


このように，P制御の場合は定常偏差の問題が解決できません（$K_{\rm{p}}$ の値を$\infty$にすれば理論上は解決できますが，現実世界では無理なので事実上不可能です）．

<br>

そこで，I制御が登場します．I制御は積分制御とも呼ばれ「偏差を積分した結果を出力」とするような制御です．式にすると以下のようになります．
```math
m = K_{\rm{P}}e + K_{\rm{I}}\int e \: dt \\
```
:::note 
$K_{\rm{P}}$ : 比例ゲイン（P制御をどのくらい反映させるかを決定する定数） \
$K_{\rm{I}}$ : 積分ゲイン（I制御をどのくらい反映させるかを決定する定数）
$e = \theta_{\rm{goal}} - \theta_{\rm{current}}$ : 偏差（現在角度$\theta_{\rm{current}}$ と目標角度 $\theta_{\rm{goal}}$ の差） \
$m$ : 操作量（出力トルク，出力電圧など）
:::

<br>

- P制御：どれだけ時間が経っても定常偏差に収束した後は，偏差$e=\theta_{\rm{goal}}-\theta_{\rm{current}}$ の値は変わらないので操作量$m$ も変わりません．
- I制御：積分するため，定常偏差に収束した後も時間が経てば偏差が蓄積されることで操作量$m$ が大きくなります．
![compare_P_PI.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/6ca0b9cf-c4db-f4cf-5553-c6b49ed46773.png)


<br>

それでは，PI制御を実装してみましょう．
今回は，「制御した直後に一定角度だけ，$\theta_{\rm{current}}$ から引く」という方法で（強引ですが），定常偏差を再現することにします．
```python
from matplotlib import pyplot as plt

# ------------------------------- P制御関連 -------------------------------
# 関数の定義 -------------------
# P制御の定義
def P(kp, theta_goal, theta_current):
    error = theta_goal - theta_current# 偏差（error）を計算
    m = kp*error # 操作量（Manipulative Variable）を計算（例：モータのトルクなど）
    return m

# 係数等設定 --------------------
kp = 0.1 # 比例ゲイン
theta_start = 0.0 # 初期角度
theta_goal = 90.0 # 目標角度
time_length = 600 # 計測時間
offset = 1.0 # 追加：定常偏差
theta_current = theta_start # 現在角度
time_list = [0] # 時刻のリスト（描画用）
theta_list = [theta_start] # 現在地のリスト（描画用）

# 制御 --------------------------
for time in range(time_length):
    m = P(kp, theta_goal, theta_current) # 操作量を計算
    theta_current += m # 現在角度に操作量を足す（実際は，この操作量をもとにモータを動かす）
    theta_current -= offset # # 追加：定常偏差分だけ引く
    time_list.append(time) # 描画用
    theta_list.append(theta_current) # 描画用

# 描画 --------------------------
plt.hlines([theta_goal], 0, time_length, "red", linestyles='dashed') #ゴールを赤色の点線で表示
plt.plot(time_list, theta_list, label="P", color="black") # P制御のグラフ描画

# ------------------------------- PI制御関連 -------------------------------
# 関数の定義 -----------------------
# PI制御
def PI(kp, ki, theta_goal, theta_current, error_sum):
    error = theta_goal - theta_current# 偏差（error）を計算
    error_sum += error # P制御からの追加：偏差の総和（積分）を計算
    m = (kp * error) + (ki * error_sum) # 操作量を計算
    return m, error_sum

# 係数などの設定 ----------------------
ki = 0.0005 # 積分ゲイン
error_sum = 0.0 # 偏差の総和（積分）

# P制御時の数値を初期化
theta_start = 0.0; theta_current = theta_start; time_list = [0]; theta_list = [theta_start]

# 制御 --------------------------------
for time in range(time_length):
    m_val, error_sum = PI(kp, ki, theta_goal, theta_current, error_sum) # 操作量を計算
    theta_current = theta_current + m_val # 現在角度に操作量を足す（実際は，この操作量をもとにモータを動かす）
    theta_current -= offset # 定常偏差分だけ引く
    time_list.append(time) # 描画用
    theta_list.append(theta_current) # 描画用

# 描画 -------------------------------
plt.plot(time_list, theta_list, label="PI", color="blue") # PI制御の結果を描画
plt.xlabel(r'$t$') 
plt.ylabel(r'$\theta$') 
plt.legend(loc='lower right') # 凡例を表示
plt.show() # グラフ表示
```

<br>

実行結果を見ると，P制御では定常偏差が残り続けているのに対して，PI制御では偏差が蓄積することで定常偏差が最終的になくなっていることが分かります．
![P_PI_control.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/7a42589b-19f4-813f-0478-f2124ab0e9af.png)



# PID制御
ここまでで，全て上手く制御できているように見えますが，実際は
- オーバーシュート

と呼ばれる現象が起こります．これは，「操作量 $m$ が大きすぎて目標角度 $\theta_{\rm{goal}}$ をとおりすぎてしまう」ような現象をいい，上手く制御できていない場合は振動し続けて収束しません．
![overshoot.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/31360c88-71e3-6345-33f6-4a493417db49.png)

<br>

これを解決するためにD制御を導入します．D制御は微分制御とも呼ばれ「微分をすることで，極端な変化を抑制する」ような働きを持ちます．式にすると，以下のようになります．
```math
m = K_{\rm{P}}e + K_{\rm{I}}\int e \: dt + K_{\rm{D}}\frac{d}{dt}\:e \\
```
:::note 
$K_{\rm{P}}$ : 比例ゲイン（P制御をどのくらい反映させるかを決定する定数） \
$K_{\rm{I}}$ : 積分ゲイン（I制御をどのくらい反映させるかを決定する定数）
$K_{\rm{D}}$ : 微分ゲイン（D制御をどのくらい反映させるかを決定する定数）
$e = \theta_{\rm{goal}} - \theta_{\rm{current}}$ : 偏差（現在角度$\theta_{\rm{current}}$ と目標角度 $\theta_{\rm{goal}}$ の差） \
$m$ : 操作量（出力トルク，出力電圧など）
:::

<br>

- P, I制御：偏差の変化を考慮することはできない
- D制御：偏差の変化を考慮して，操作量 $m$ を抑制する．
    - なお，P制御とI制御は単体でも動作しますが，D制御単体では動作しません．

D制御においてはグラフの傾きが大きい（=角速度が大きい）時に，より抑制するように働きます．
![D_control_explain.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/d8d02d38-4c2d-2e79-b13d-e6256f39bac4.png)



それでは，PID制御を実装してみましょう．
今回の簡易的なシミュレーションでは，積分ゲイン$K_{\rm{I}}$ を大きくすれば振動を起こすことができるので，その方法で試すことにします．

```python
from matplotlib import pyplot as plt

# ------------------------------- PI制御関連 -------------------------------
# 関数の定義 ------------------------
# PI制御
def PI(kp, ki, theta_goal, theta_current, error_sum):
    error = theta_goal - theta_current# 偏差（error）を計算
    error_sum += error # 偏差の総和（積分）を計算
    m = (kp * error) + (ki * error_sum) # 操作量を計算
    return m, error_sum

# 係数などの設定 --------------------
kp = 0.1 # 比例ゲイン
ki = 0.5 # 積分ゲインの値を大きくして，意図的に振動を発生させる
theta_start = 0.0 # 初期角度
theta_goal = 90.0 # 目標角度
time_length = 200 # 計測時間 
theta_current = theta_start # 目標角度
error_sum = 0.0 # 偏差の総和（積分）
time_list = [0] # 時刻のリスト（描画用）
theta_list = [theta_start] # 現在地のリスト（描画用）

# 制御 ------------------------
for time in range(time_length):
    m, error_sum = PI(kp, ki, theta_goal, theta_current, error_sum) # 操作量を計算
    theta_current += m # 現在角度に操作量を足す（実際は，この操作量をもとにモータを動かす）
    time_list.append(time) # 描画用
    theta_list.append(theta_current) # 描画用
    
# 描画 --------------------------
plt.hlines([theta_goal], 0, time_length, "red", linestyles='dashed') #ゴールを赤色の点線で表示
plt.plot(time_list, theta_list, label="PI", color="black") # PI制御のグラフを描画

# ------------------------------- PID制御関連 -------------------------------
# 関数の定義 ---------------------
# PID制御
def PID(kp, ki, kd, theta_goal, theta_current, error_sum, error_pre):
    error = theta_goal - theta_current# 偏差（error）を計算
    error_sum += error # 偏差の総和（積分）を計算
    error_diff = error-error_pre # PI制御からの追加：1時刻前の偏差と現在の偏差の差分（微分）を計算
    m = (kp * error) + (ki * error_sum) + (kd*error_diff) # 操作量を計算
    return m, error_sum, error

# 係数などの設定 ------------------
kd = 0.5 # 微分ゲイン：急激な変化を抑える
error_pre = 0.0 # 1時刻前の偏差
# PI制御の時の数値を初期化
theta_start = 0.0; theta_current = theta_start; error_sum = 0.0; time_list = [0]; theta_list = [theta_start] 

# PID制御 -----------------------
for time in range(time_length):
    m, error_sum, error = PID(kp, ki, kd, theta_goal, theta_current, error_sum, error_pre) # 操作量を計算
    theta_current += m # 現在角度に操作量を足す（実際は，この操作量をもとにモータを動かす）
    error_pre = error # 一時刻前の偏差として保存しておく（D制御用）
    time_list.append(time) # 描画用
    theta_list.append(theta_current) # 描画用

# 描画
plt.plot(time_list, theta_list, label="PID", color="blue") # PID制御のグラフを描画
plt.xlabel(r'$t$') 
plt.ylabel(r'$\theta$') 
plt.legend(loc='lower right') # 凡例を表示
plt.show() # グラフの表示
```
<br>

実行結果から分かるようにPI制御に比べてPID制御の方が速く収束していることが分かります．
![PI_PID_control.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/8b79e2fe-cefc-69c8-31b0-5aacb6eafb06.png)


::: note
今回の簡易的なシミュレーションでは，そこまで上手く再現できなかったので割愛していますが，D制御は「ノイズ」のような突発的な急激な変化にも対応でき，PI制御に比べて速く元の目標位置に戻ることができます．
:::

# おまけ1：アニメーション
ここまでで，PID制御の説明は終わりですが，アニメーションにした方が理解しやすいところもあると思うのでコードだけですが載せておきます．
![pid.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/e5717aa2-a34b-f6cd-dd6e-b2583eb846e7.gif)


```python
import math
from matplotlib.animation import FuncAnimation
from IPython.display import HTML
from matplotlib import pyplot as plt
%matplotlib inline

# ------------------------------- PID制御関連 -------------------------------
# 関数の定義 ---------------------
# PID制御
def PID(kp, ki, kd, theta_goal, theta_current, error_sum, error_pre):
    error = theta_goal - theta_current# 偏差（error）を計算
    error_sum += error # 偏差の総和（積分）を計算
    error_diff = error-error_pre # PI制御からの追加：1時刻前の偏差と現在の偏差の差分（微分）を計算
    m = (kp * error) + (ki * error_sum) + (kd*error_diff) # 操作量を計算
    return m, error_sum, error

# 係数などの設定 --------------------
# 以下の変数の数値を変えると結果が変わる
kp = 0.1 # 比例ゲイン
ki = 0.01 #0.5 # 積分ゲインの値を大きくして，意図的に振動を発生させる
kd = 0.5 #0.5 # 微分ゲイン：急激な変化を抑える
theta_start = 0.0 # 初期角度
theta_goal = 90.0 # 目標角度
time_length = 150 #150 # 計測時間 
offset = 1.0 # 定常偏差

# 以下の変数は変えなくて良い
theta_current = theta_start # 目標角度
error_sum = 0.0 # 偏差の総和（積分）
error_pre = 0.0 # 1時刻前の偏差
time_list = [0] # 時刻のリスト（描画用）
theta_list = [theta_start] # 現在地のリスト（描画用）
animation_time_list = [time_list.copy()]
animation_theta_list = [theta_list.copy()]

# PID制御 -----------------------
for time in range(time_length):
    m, error_sum, error = PID(kp, ki, kd, theta_goal, theta_current, error_sum, error_pre) # 操作量を計算
    theta_current += m # 現在角度に操作量を足す（実際は，この操作量をもとにモータを動かす）
    theta_current -= offset
    error_pre = error # 一時刻前の偏差として保存しておく（D制御用）
    time_list.append(time) # 描画用
    theta_list.append(theta_current) # 描画用
    animation_time_list.append(time_list.copy())
    animation_theta_list.append(theta_list.copy())

# ------------------------- アニメーション関連 -------------------------------------
# PID制御のグラフ描画
def plot_pid_graph(ax, time, theta_goal, animation_time_list, animation_theta_list):
    ax.hlines([theta_goal], 0, max(animation_time_list[-1]), "red", linestyles='dashed') #ゴールを赤色の点線で表示
    ax.plot(animation_time_list[time], animation_theta_list[time], label="PID", color="blue") # PID制御のグラフを描画
    ax.set_xlim(-1, max(animation_time_list[-1])) # min=0の場合，グラフの左端が切れるので，min=-1に設定
    if max(animation_theta_list[-1]) < theta_goal: # 定常偏差によりtheta_goalよりもグラフが下回ってしまったらtheta_goalの赤い点線が見えるように範囲を設定
        ax.set_ylim(0, theta_goal+1) # 赤い点線が見えるように +1 している
    else:
        ax.set_ylim(0, max(animation_theta_list[-1])+1)# 赤い点線が見えるように +1 している
    ax.set_xlabel(r'$t$') 
    ax.set_ylabel(r'$\theta$') 
    ax.legend(loc='lower right') # 凡例を表示

# ロボットアームの描画
def plot_robot_arm(ax, time, animation_theta_list):
    ax.plot([0,0], [0,-1], color="black") # 固定リンク
    rad = math.radians(animation_theta_list[time][-1]-90) # 真下方向を0°とする
    x = math.cos(rad) # 順運動学
    y = math.sin(rad) # 順運動学
    ax.plot([0,x], [0,y], color="orange") # 稼働リンク
    ax.set_xlim(-1, 1)
    ax.set_ylim(-1, 1)
    
# figureを作成
fig = plt.figure()
ax_pid = fig.add_subplot(1, 2, 1)
ax_arm = fig.add_subplot(1, 2, 2)
ax_pid.set_aspect("equal")#画像の比率を同じにする
ax_arm.set_aspect("equal")#画像の比率を同じにする
ax_arm.tick_params(labelbottom=False, labelleft=False, labelright=False, labeltop=False) # 軸のメモリを消す（ロボットアーム側はメモリの情報は不要なので）

# 各フレーム毎の描画処理
def update(time):
    ax_pid.cla()
    ax_arm.cla()
    plot_pid_graph(ax_pid, time, theta_goal, animation_time_list, animation_theta_list)
    plot_robot_arm(ax_arm, time, animation_theta_list)
    plt.suptitle('kp={}, ki={}, kd={}, offset={} \n \n t={}, theta={:.3g}'.format(kp, ki, kd, offset, animation_time_list[time][-1], animation_theta_list[time][-1]), x=0.5, y=0.90) # タイトルに時間と角度を表示
 
# アニメーション化
ani = FuncAnimation(fig, update, interval=50, frames=len(time_list))
HTML(ani.to_jshtml()) # HTMLに
#ani.save('pid.mp4', writer="ffmpeg") # mp4で保存．これを実行すると処理時間が増加します
```


# おまけ2：インタラクティブにグラフを動かす
PID制御ではゲイン $K_{\rm{P}}, K_{\rm{I}}, K_{\rm{D}}$ を調整することで，制御の具合が変わるので，ゲイン $K_{\rm{P}}, K_{\rm{I}}, K_{\rm{D}}$ の値を色々いじって遊んでみたい人もいると思います．しかし，毎回「コード修正」→「実行」を繰り返すのはめんどくさいですよね．

そこで，下記コードでは下動画のようにインタラクティブにゲイン $K_{\rm{P}}, K_{\rm{I}}, K_{\rm{D}}$と擬似的な定常偏差の数値をいじれます．色々いじって遊んでみてください．

※ google Colaboratoryで実行する場合は，ラグが生じるため下動画のように滑らかには動かないので注意してください．

![pid_intractive.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/cc445d70-68bd-d2c5-e21d-84ccae6337f6.gif)


```python
import matplotlib.pyplot as plt
import ipywidgets

# ------------------------------- ウィジェット，インタラクティブの設定関連 ------------------------------
# text widgetsを作成 -> Vboxに格納して縦に並べる
def generate_vbox_text_widget():
    text_widgets = []
    text_widgets.append(ipywidgets.FloatText(min=0.0, max=359.0)) # theta_start
    text_widgets.append(ipywidgets.FloatText(min=0.0, max=359.0)) # theta_goal
    text_widgets.append(ipywidgets.FloatText(min=0.0, max=100.0)) # offset
    text_widgets.append(ipywidgets.IntText(min=-360, max=360)) # time_length
    text_widgets.append(ipywidgets.FloatText(min=0.00, max=1.50)) # kp
    text_widgets.append(ipywidgets.FloatText(min=0.00, max=1.50)) # ki
    text_widgets.append(ipywidgets.FloatText(min=0.00, max=1.50)) # kd
    vox_text_widgets = ipywidgets.VBox(text_widgets)
    return vox_text_widgets

# slider widgetsを7個作成 -> Vboxに格納して縦に並べる．
def generate_vbox_slider_widget():
    slider_widgets = []
    slider_widgets.append(ipywidgets.FloatSlider(value=0.0, min=0.0, max=359.0, description = "theta_start", disabled=False))
    slider_widgets.append(ipywidgets.FloatSlider(value=90.0, min=0.0, max=359.0, description = "theta_goal", disabled=False))
    slider_widgets.append(ipywidgets.FloatSlider(value=0.0, min=0.0, max=100.0, step=0.01, description = "offset", disabled=False))
    slider_widgets.append(ipywidgets.IntSlider(value=150, min=0, max=2000, description = "time_length", disabled=False))
    slider_widgets.append(ipywidgets.FloatSlider(value=0.10, min=0.00, max=1.50, step=0.001, description = "kp", disabled=False))
    slider_widgets.append(ipywidgets.FloatSlider(value=0.50, min=0.00, max=1.50, step=0.001, description = "ki", disabled=False))
    slider_widgets.append(ipywidgets.FloatSlider(value=0.50, min=0.00, max=1.50, step=0.001, description = "kd", disabled=False))
    vox_slider_widgets = ipywidgets.VBox(slider_widgets)
    return vox_slider_widgets


# Box内の複数のwidetを連携させる（二つのbox内のwidgetの数が同じである必要あり）
def link_slider_and_text(box1, box2):
    for i in range(7):
      ipywidgets.link((box1.children[i], 'value'), (box2.children[i], 'value'))

# 結果を表示
def draw_interactive():
    # slider widgetを作成
    sliders = generate_vbox_slider_widget()
    # text widgetを作成
    texts = generate_vbox_text_widget()

    # slider widget と　posture widget を横に並べる
    slider_and_text = ipywidgets.Box([sliders, texts])

    # slider wiget と text widget を連携
    link_slider_and_text(sliders, texts)

    # main文にslider widgetsの値を渡す
    params = {}
    for i in range(7):
        params[str(i)] = sliders.children[i]
    final_widgets = ipywidgets.interactive_output(main, params)
    
    display(slider_and_text, final_widgets)

# -------------------------------------- PID制御関連 ----------------------------------------
def PID(kp, ki, kd, theta_goal, theta_current, error_sum, error_pre):
    error = theta_goal - theta_current# 偏差（error）を計算
    error_sum += error # 偏差の総和（積分）を計算
    error_diff = error-error_pre # PI制御からの追加：1時刻前の偏差と現在の偏差の差分（微分）を計算
    m = (kp * error) + (ki * error_sum) + (kd*error_diff) # 操作量を計算
    return m, error_sum, error

def main(*args, **kwargs):
    # 係数などの設定 --------------------
    # スライダーやテキストボックスから得られた数値を代入
    params = kwargs
    theta_start = params["0"]
    theta_goal = params["1"]
    offset = params["2"]
    time_length = params["3"]
    kp = params["4"]
    ki = params["5"]
    kd = params["6"]
    # その他初期設定
    error_sum = 0.0
    error_pre = 0.0
    theta_current = theta_start
    time_list = [0]
    theta_list = [theta_start]

    # PID制御 -----------------------
    for time in range(1, time_length):
        m, error_sum, error = PID(kp, ki, kd, theta_goal, theta_current, error_sum, error_pre) # 操作量を計算
        theta_current += m # 現在角度に操作量を足す（実際は，この操作量をもとにモータを動かす）
        theta_current -= offset
        error_pre = error # 一時刻前の偏差として保存しておく（D制御用）
        time_list.append(time) # 描画用
        theta_list.append(theta_current) # 描画用

    # 描画
    plt.hlines([theta_goal], 0, time_length, "red", linestyles='dashed') #ゴールを赤色の点線で表示
    plt.plot(time_list, theta_list, label="PID", color="blue") # PID制御のグラフを描画
    plt.xlabel(r'$t$') 
    plt.ylabel(r'$\theta$') 
    plt.ylim(theta_start-20, theta_goal+60) # 大体の場合において，グラフが収まる範囲に設定（適当）
    plt.legend(loc='lower right') # 凡例を表示
    plt.title(r'final $\theta$={:.3g}'.format(theta_list[-1])) # タイトルに時間と角度を表示
    plt.show() # グラフの表示

# インタラクティブ描画実行
draw_interactive()
```

# 最後に
本記事ではPID制御について説明しました．
何度も言うようですが，これは簡易的なシミュレーションなので，これでPID制御のことがなんとなくわかった人は `python-control` などのライブラリを使って本格的なシミュレーションに挑戦してみると良いと思います．
# 参考サイトなど
- [1.6.(1) PID制御の演算式 - 宮崎技術研究所](http://www.miyazaki-gijutsu.com/series/control116.html)
- [PID制御について宇宙一わかりやすく解説してみる](https://www.yukisako.xyz/entry/pid_control)
- [P制御だと定常偏差が残ってしまう理由を「最終値の定理」を使わずに解説](https://okasho-engineer.com/p-control-error-reason/)
- [PID制御とは？仕組みと動作イメージを分かりやすく解説！](https://controlabo.com/pid-control-introduction/)

<!-- ここから注釈 ----->
[^1]: 定常偏差は，「最終値定理」を用いることで計算できますが，今回は簡易的なシミュレーションなので，適当な値に設定しています．



