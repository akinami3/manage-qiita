---
title: （M1）Limaを使ってDockerを爆速化する（VScodeのremote-containerとの連携も）
tags:
  - Mac
  - Docker
  - VSCode
  - M1
  - Lima
private: false
updated_at: '2022-02-21T07:27:47+09:00'
id: d38b9e7c7f37bd070f40
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
Docker Desktop for Mac よりも Docker + Lima の方が速いらしいので実装してみました．

Limaは仮想環境の一つです．windowsでWSLを使用したことがある人は似たようなものらしいです（詳しくは分からん）．

MacBook Air(M1, 2020), macOS Monterey バージョン12.1で動作確認しました．
# パッケージのインストール

最初に必要なパッケージをインストールします．

- wget
    
    `brew install wget`
    
- Lima
    
    `brew install lima`
    
- Docker, Docker Compose
    
    `brew install docker`
    
    `brew install docker-compose`

# Limaのyamlファイルを作成

Limaは「構築したい環境」についてyamlファイルに記述します．
![yaml2Lima.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/fafe0e36-e20d-a1a1-cd0a-25ed33de25c4.png)
今回は「Dockerの環境を記述したyamlファイル」を準備します．
一からyamlファイルを作成することもできますが，あらかじめ用意してあるサンプルが存在するので， `docker.yaml`をダウンロード、一部を編集して使用することにします．

1. 適当な場所に `docker.yaml` をダウンロードします．
    
    ```shell
    wget https://github.com/lima-vm/lima/raw/master/examples/docker.yaml
    ```
    
2.  ダウンロードした `docker.yaml` に下記を追記します．
    - リソースを変更します．皆さんの使用したい環境によって適切な値を設定してください

        ```yaml
        cpus: 2
        memory: "8GiB"
        disk: "20GiB"
        ```
<br>
    - sshによるエラーが起こる可能性があるらしいので `loadDotSSHPubKeys: false` を設定して回避します．また， `localPort` でポートの番号を指定します．
        
        ```yaml
        ssh:
          loadDotSSHPubKeys: false
          localPort: 60006
        ```
<br>
    - Limaの仮想環境内の「指定したディレクトリ配下にあるファイル」に書き込み権限を与えます．（**デフォルトでは読み込み権限しか与えられていないので，編集できません**）
        
        下の例では， `~` 以下全てのディレクトリ内のファイルに書き込み権限を与えています （ `~` をtrueにするのは、あまり推奨されていないので、気になる人は `location` に指定するディレクトリを細かく決めましょう）
        
        ```yaml
        mounts:
        	- location: "~"
              writable: true # ダウンロードした時点では，この行がないと思うので追記
        ```

<br>
最終的な `docker.yaml` は以下になります

```yaml
# Example to use Docker instead of containerd & nerdctl
# $ limactl start ./docker.yaml
# $ limactl shell docker docker run -it -v $HOME:$HOME --rm alpine

# To run `docker` on the host (assumes docker-cli is installed):
# $ export DOCKER_HOST=$(limactl list docker --format 'unix://{{.Dir}}/sock/docker.sock')
# $ docker ...

# This example requires Lima v0.8.0 or later
images:
# Hint: run `limactl prune` to invalidate the "current" cache
- location: "https://cloud-images.ubuntu.com/impish/current/impish-server-cloudimg-amd64.img"
  arch: "x86_64"
- location: "https://cloud-images.ubuntu.com/impish/current/impish-server-cloudimg-arm64.img"
  arch: "aarch64"
mounts:
- location: "~"
  writable: true
- location: "/tmp/lima"
  writable: true
containerd:
  system: false
  user: false
provision:
- mode: system
  script: |
    #!/bin/sh
    sed -i 's/host.lima.internal.*/host.lima.internal host.docker.internal/' /etc/hosts
- mode: system
  script: |
    #!/bin/bash
    set -eux -o pipefail
    command -v docker >/dev/null 2>&1 && exit 0
    export DEBIAN_FRONTEND=noninteractive
    curl -fsSL https://get.docker.com | sh
    # NOTE: you may remove the lines below, if you prefer to use rootful docker, not rootless
    systemctl disable --now docker
    apt-get install -y uidmap dbus-user-session
- mode: user
  script: |
    #!/bin/bash
    set -eux -o pipefail
    systemctl --user start dbus
    dockerd-rootless-setuptool.sh install
    docker context use rootless
probes:
- script: |
    #!/bin/bash
    set -eux -o pipefail
    if ! timeout 30s bash -c "until command -v docker >/dev/null 2>&1; do sleep 3; done"; then
      echo >&2 "docker is not installed yet"
      exit 1
    fi
    if ! timeout 30s bash -c "until pgrep rootlesskit; do sleep 3; done"; then
      echo >&2 "rootlesskit (used by rootless docker) is not running"
      exit 1
    fi
  hint: See "/var/log/cloud-init-output.log". in the guest
portForwards:
- guestSocket: "/run/user/{{.UID}}/docker.sock"
  hostSocket: "{{.Dir}}/sock/docker.sock"
message: |
  To run `docker` on the host (assumes docker-cli is installed), run the following commands:
  ------
  docker context create lima --docker "host=unix://{{.Dir}}/sock/docker.sock"
  docker context use lima
  docker run hello-world
  ------

ssh:
  loadDotSSHPubKeys: false
  localPort: 60006

cpus: 2
memory: "8GiB"
disk: "20GiB"
```

# 環境構築

`docker.yaml` に定義されたLimaの仮想環境を構築します．
![lima_start.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/1cc83ff5-63db-3026-2ef9-fb54a4a8d181.png)

```bash
limactl start ./docker.yaml
```

:::note 
<strong>yamlファイルを編集する時の注意点</strong>

一度構築した仮想環境は，yamlファイル（今回の場合 <code>docker.yaml</code>）を修正しただけでは反映されません．<strong>yamlファイルを変更した場合は，仮想環境を再作成する必要があります．</strong>  
具体的には， <code>limactl stop docker</code> で仮想環境を停止し、<code>limactl rm docker</code> で仮想環境を削除した後に再作成してください．  
:::


## **DOCKER_HOST 環境変数の上書き**

ここまでで，おおよその設定は完了しました．
しかし，毎回「 `docker.yaml`のLimaの仮想環境」に入ってから、dockerのデーモンを起動するのは面倒です．

そこで、 `docker` コマンドを使用した時に、「Docker Desktop for MacのDockerデーモン（下図左）」ではなく、「Limaの仮想環境内のDockerデーモン（下図右）」にアクセスされるようにします．
※ この辺りは詳しくないので，解釈が間違っているかも知れません．詳しい構成については[yoichiwo7さんの記事](https://qiita.com/yoichiwo7/items/44aff38674134ad87da3)に詳しくまとめてあります．
![lima.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/89a75eb7-fdd2-6c64-5c2e-5c3391f33ac1.png)

具体的には， `DOCKER_HOST`のURLを変更するとことで「Limaの仮想環境内のDockerデーモン（上図右）」にアクセスされるようになります．

また，毎回 `DOCKER_HOST` を書き換えるのは面倒なので`.zshrc` に追記しておくといいと思います．
今回の場合は，`.zshrc` に下記を追記すればOKです．

```bash
export DOCKER_HOST=unix:///${HOME}/.lima/docker/sock/docker.sock
```
# シェル起動時に仮想環境を起動

パソコンの再起動を行うと仮想環境はstopの状態になります．そのため，パソコンを再起動するごとに，仮想環境をstartする必要がありこれまた面倒です．

そこで，先程と同様に`.zshrc`に仮想環境を立ち上げるように追記することで、自動起動するようにします．
具体的には，`.zshrc` に下記を追記してください．

```bash
limactl start docker
```

ここまでで，Lima + Docker の環境構築は完了です． `docker build` や `docker-compose build` などのコマンドが使用できるようになっているはずです．
# VScodeでRemote-Containers + Lima を実現する方法

VScodeで，Remote-Containersを使うとコンテナ内での作業が捗ります．
これを Lima でも実現したいので Remote-Containers + Lima を実現する手順を説明します．

::: note
Remote-Containers そのものの使い方は特段触れません．「VScode Remote-Containers」などで調べれば使い方が出てくると思います．
:::
1. VScodeの拡張機能である「Docker」「Remote - Containers」をインストールします
<img alt="a" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/c5545261-759c-f93f-d749-3fa8ca622fc5.png">
<img  alt="a" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2030125/dad144f7-ffb1-98b2-5ba6-37ba36fc78ee.png">
  
2. VScodeの `setting.json` に下記を追記して， `DOCKER_HOST` のURLを変更します．

  ```json
  {
   "docker.host": "unix:///Users/＜ユーザー名＞/.lima/docker/sock/docker.sock"
  }
  ```
    
::: note warn    
ここでは，<code>DOCKER_HOST=unix:///${HOME}/.lima/docker/sock/docker.sock</code> のような環境変数を含んだ記述は使えず，ベタうちする必要があるので注意してください（もしかしたら他に方法があるかも）．
:::  

上記の手順を実施することで Remote-Containersも使用することができ，Lima+Dockerを使用した開発が更に快適になりました．

# まとめ

Lima + Docker により高速なDocker環境を実現することができました．

Docker Desktop for Mac も有料化（個人用途なら問題なし）するらしいので，気になった人はやってみてください．
# 参考サイト
- Docker Desktop for Macの実用的な代替手段: lima + Docker  
  [https://qiita.com/yoichiwo7/items/44aff38674134ad87da3]
(https://qiita.com/yoichiwo7/items/44aff38674134ad87da3)
- LimaでDocker  
  [https://qiita.com/mm_sys/items/28a0217256b56918fee4](https://qiita.com/mm_sys/items/28a0217256b56918fee4)
- Docker Desktop 無しで Docker を使う with lima on Mac  
  [https://korosuke613.hatenablog.com/entry/2021/09/18/docker-on-lima]
(https://korosuke613.hatenablog.com/entry/2021/09/18/docker-on-lima)

