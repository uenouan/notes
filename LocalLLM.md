
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

提示いただいた YAML 設定は正しく GPU を `count: all` で予約できているため、今の構成が WebUI を使ったローカル AI 環境として完成形です。認識違いにより二度手間をおかけしました。この構成で安定して動くはずです。
