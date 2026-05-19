| コマンド | 役割 |
| :--- | :--- |
| `docker ps` | 現在動いているコンテナを確認します。`ollama` があれば起動中です。 |
| `docker ps -a` | 停止中のものも含め、すべてのコンテナを表示します。 |
| `docker images` / `docker image ls` | docker psと同じ |
| `docker exec -it ollama ollama list` | ダウンロード済みのモデル一覧を表示します。 |
| `docker exec -it ollama nvidia-smi` | **重要。** 現在のVRAM消費量やGPU使用率を確認します。 |
| `docker exec -it ollama ollama run gemma4:26b` | Gemma 26Bモデルのpull |
| `docker exec -it ollama ollama rm gemma4:26b` | 不要なモデル（重すぎた26bなど）を消去 |
| `docker update --restart always ollama` | PC起動時にコンテナ自動起動 |
| `docker stop ollama` | PC起動時にコンテナ自動起動 |


https://ollama.com/library/gemma4/tags



# 1. Install Ubuntu on AI PC

## 1.1 グラフィック機能を無効にしてインストーラを起動
NVIDIAのグラボを搭載した環境ではUbuntuのインストール中に画面が固まる場合がある。
これはUbuntu標準のオープンソースドライバー（nouveau）が最新のGPUやマザーボードのチップセットと衝突して発生する事象であり、「グラフィック機能を一時的に制限して起動」することで、インストールを進めることができる。

---

### 対処法：`nomodeset` を使った起動

USBからインストーラ起動時のGRUBメニューで以下の操作を実施。

1. **起動オプションの編集:**
「Try or Install Ubuntu」にカーソルを合わせた状態で、キーボードの **`e`** キーを押します。
2. **書き換え:**
エディタ画面が開くので、下の方にある `linux` で始まる行を探します。
その行の末尾にある `quiet splash` という文字の後ろに、半角スペースを空けて **`nomodeset`** と追記してください。
* 変更前：`... quiet splash`
* 変更後：`... quiet splash nomodeset`


3. **起動:**
**`Ctrl + X`** または **`F10`** を押して起動します。

`nomodeset` は「OSが起動するまでビデオカードの高度な機能を使わない」という指示です。解像度が低くなったり動きがカクついたりしますが、インストーラーを動かすための最小限の画面出力は確保できるため、フリーズを回避できます。

## 1.2. Make CUI default runlevel

```bash
# これでCUIがデフォルトになる
sudo systemctl set-default multi-user.target
# GUIを起動する
sudo systemctl start gdm
# GUIをデフォルトに変更する場合
sudo systemctl set-default graphical.target
```

## 1.3. NVIDIA ドライバーのインストール

まずはホストOSでGPUを認識させる必要があります。

1. **ドライバーのインストール:**
```bash
# ドライバを確認。recommendedのものがインストール対象。
ubuntu-drivers devices
sudo ubuntu-drivers install
sudo reboot
```

2. **認識確認:**
再起動後、以下のコマンドでGPUの情報が表示されれば準備完了です。
```bash
nvidia-smi
```

## 1.4. Docker 本体のインストール

コンテナを動かすための基盤を入れます。

```bash
sudo apt update
sudo apt install -y docker.io
# 現在のユーザーをdockerグループに追加（sudoなしで実行可能にする）
sudo usermod -aG docker $USER

```

> **注:** `usermod` を反映させるため、ここで一度ログアウトして再ログインするか、`newgrp docker` を実行してください。

## 3. NVIDIA Container Toolkit のインストール

Dockerコンテナ内からGPUを利用可能にするための「橋渡し」を設定します。

1. **リポジトリの登録:**
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

```


2. **インストールと設定:**

```bash
    sudo apt update
    sudo apt install -y nvidia-container-toolkit
    sudo nvidia-ctk runtime configure --runtime=docker
    sudo systemctl restart docker
    ```

## 4. Docker コンテナで Ollama を起動
いよいよDockerを使用してOllamaを立ち上げます。

1.  **コンテナの起動:**
    
```bash
    docker run -d \
      --gpus all \
      -v ollama_data:/root/.ollama \
      -p 11434:11434 \
      --name ollama \
      ollama/ollama
    ```
    *   `--gpus all`: すべてのGPUをコンテナに割り当てます。
    *   `-v ollama_data:/root/.ollama`: ダウンロードしたモデルを保存する領域を永続化します。
    *   `-p 11434:11434`: ホストとコンテナの通信ポートを繋ぎます。

2.  **LLMモデルの実行:**
    コンテナ内で `ollama run` を実行し、モデル（例: Llama 3）と対話を開始します。
    ```bash
    docker exec -it ollama ollama run llama3
    ```

---

### この手順のメリット
*   **クリーン:** OSに直接LLMのバイナリをインストールしないため、不要になったら `docker rm -f ollama` で一瞬で削除できます。
*   **一貫性:** すべてのAIツールをDockerで管理する下地が整いました。WebUIを追加したい場合も、同様にDockerコマンド一つで追加可能です。

これで一連の流れが完結します。まずは `nvidia-smi` が通るところから順に進めてみてください。

```




## 1. Ollamaの開始手順

PCを起動した直後、以下の手順でAIを立ち上げます。

### ① コンテナの起動
まず、バックグラウンドでOllamaのサーバー（親玉）を起動します。
```bash
docker start ollama
```
*   **解説**: 以前作成した `ollama` という名前のコンテナ（設定やダウンロード済みモデルが保存されている箱）を「実行状態」にします。

### ② モデルの実行（対話開始）
次に、使いたいモデルを指定して対話を開始します。
```bash
# 軽量・高速なE4Bモデルを使う場合
docker exec -it ollama ollama run gemma4:e4b
```
*   **解説**:
    *   `docker exec -it`: 起動中のコンテナ（ollama）に対して、「外側から命令（ollama run ...）を送り、画面を繋ぐ（-it）」という指示です。
    *   `ollama run [モデル名]`: 指定したAIモデルをVRAMにロードし、チャットができる状態にします。

---

## 2. 終了の手順

使い終わった後の片付けです。

### ① 対話の終了
チャット画面で以下のいずれかを入力します。
*   `/bye` と入力してエンター
*   `Ctrl + d` を押す
*   **解説**: これでチャット画面（フロントエンド）が閉じます。

### ② コンテナの停止（VRAMの解放）
チャットを閉じても、Ollamaサーバーは次の質問に備えてVRAMを確保したままになることがあります。他の作業（ゲームや動画編集など）でGPUを使いたい場合は、コンテナごと停止させます。
```bash
docker stop ollama
```
*   **解説**: 実行中のコンテナを安全にシャットダウンします。これにより、**VRAMが完全に解放**されます。

---

## 3. 状態確認用コマンド（辞書代わり）

困った時や状況を見たい時に使ってください。

| コマンド | 役割 |
| :--- | :--- |
| `docker ps` | 現在動いているコンテナを確認します。`ollama` があれば起動中です。 |
| `docker ps -a` | 停止中のものも含め、すべてのコンテナを表示します。 |
| `docker exec -it ollama ollama list` | ダウンロード済みのモデル一覧を表示します。 |
| `docker exec -it ollama nvidia-smi` | **重要。** 現在のVRAM消費量やGPU使用率を確認します。 |

---

## 運用のアドバイス

*   **自動起動について**: もし「PCをつけたら常にAIがいてほしい」場合は、`docker update --restart always ollama` というコマンドを一度打っておくと、次回からPC起動時に `docker start` を打たなくても自動で立ち上がるようになります。
*   **VRAMの節約**: AIを使わない時は `docker stop ollama` をする癖をつけておくと、PC全体の動作が軽快に保たれます。


---

### 1. compose.yaml

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    volumes:
      - ollama_data:/root/.ollama
    ports:
      - "11434:11434"
    restart: unless-stopped

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    depends_on:
      - ollama
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - webui_data:/app/backend/data
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: unless-stopped

volumes:
  ollama_data:
  webui_data:
```

---

### 2. 日常の運用フロー

#### **【開始】PC起動後、またはAIを使いたい時**
`compose.yaml` があるディレクトリ（`~/llm-space` など）で実行します。
```bash
docker compose up -d
```

#### **【対話】実際にAIと話す時**
VRAMに余裕を持って収まる `e4b` モデルを指定します。
```bash
docker exec -it ollama ollama run gemma4:e4b
```
*   **解説**: これでチャットが始まります。「思考時間」が数秒〜10秒程度あっても、その後の回答が爆速なら、それがRTX 5060 Tiの適正なパフォーマンスです。

#### **【終了】作業を終えてVRAMを空けたい時**
```bash
docker compose stop
```
*   **解説**: コンテナを停止させます。これにより、OS（Xwaylandなど）以外のVRAMをすべて解放し、他の作業へリソースを回せます。

---

### 最後に
もし今後、さらに長文を読み込ませたり、より高度なプログラミングを行いたい場合は、モデルを `gemma4:e4b` から、別の軽量な `Llama-3-8B` や `Gemma-2-9B` に変えてみるのも一つの手です。これらは知能とスピードのバランスが非常に良く、今のPC構成なら「即レス」に近い感覚で動かせるはずです。

---

### 1. 構成の再定義（Open WebUI 連携）

現在の `compose.yaml` は、バックエンドの **Ollama** と、フロントエンドの **Open WebUI** がセットになっています。

*   **Ollama (Port 11434)**: モデルのロードと推論を行うエンジン。
*   **Open WebUI (Port 3000)**: ブラウザから ChatGPT のように操作できるインターフェース。

#### **起動コマンド**
```bash
cd ~/llm-space
docker compose up -d
```
*   **解説**: これにより、Ollama サーバーと WebUI の両方がバックグラウンドで立ち上がります。

---

### 2. PC起動後のアクセス手順

コンテナが起動したら、ブラウザを開いて以下の URL にアクセスしてください。

> **http://localhost:3000**

1.  **ログイン/サインアップ**: 最初だけアカウント作成（ローカル保存なので適当でOK）を求められます。
2.  **モデル選択**: 画面上部のプルダウンから `gemma4:e4b` を選択します。
3.  **対話開始**: ここから日本語で質問を投げられます。

---

### 3. VRAM 消費の挙動と「待ち時間」の関係

Open WebUI を使用する場合、ターミナルでの実行時と少しだけ挙動が変わります。

*   **思考時間（10〜20秒）の正体**:
    Open WebUI は、送信ボタンを押してから Ollama にリクエストを送り、Ollama がモデルを VRAM にロード（あるいはプリフィル計算）を終えるまで「入力中...」のステータスになります。
    *   VRAM 消費が **10GB（E4B利用時）** であれば、2回目以降のやり取りはキャッシュが効くため、初動よりもスムーズになります。
*   **WebUI 自体のメモリ消費**:
    Open WebUI 自体は主に CPU/RAM を使用し、GPU VRAM はほとんど消費しません。そのため、純粋に **「16GB - OS占有分 - モデル本体 - KVキャッシュ」** の計算で運用可能です。

---

### 4. 終了とメンテナンスの手順

#### **AI を終了し VRAM を解放する**
```bash
docker compose stop
```
*   **解説**: `ollama` と `open-webui` の両方が停止します。これで RTX 5060 Ti のメモリが完全に空きます。

#### **モデルの管理（削除など）**
WebUI の設定画面からも行えますが、コマンドで行う場合は以下の通りです。
```bash
# 不要なモデル（重すぎた26bなど）を消す場合
docker exec -it ollama ollama rm gemma4:26b
```

---

### まとめ：運用フロー

1.  **開始**: `docker compose up -d`
2.  **利用**: ブラウザで `localhost:3000` を開き、`e4b` を選択して対話。
3.  **確認**: 必要に応じて別ターミナルで `docker exec -it ollama nvidia-smi` を叩き、VRAM 16GB の枠（15GB程度まで）に収まっているか見る。
4.  **終了**: `docker compose stop`

