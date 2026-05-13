# WSL2 + Podman rootless + CUDA によるローカル LLM エージェント環境の構築

## 概要

Windows 11 上の WSL2 (Ubuntu 24.04) に Podman rootless コンテナと NVIDIA CUDA を組み合わせ、ローカル LLM エージェント環境を構築する手順を記述する。

構成のポイントは以下の2点である。

1. **セキュリティ**: Hermes Agent のように LLM がコード実行ツールを持つ構成では、コンテナが侵害された際にホスト root へ昇格されるリスクがある。Podman の rootless モードはコンテナをユーザー権限で実行するため、このリスクを軽減する。
2. **GPU パススルー**: Ollama が GPU を認識できるよう、NVIDIA Container Toolkit と CDI (Container Device Interface) を設定する。

### エージェント構成

```
Claude Code (Director / Orchestrator)  ← WSL ネイティブ実行
    ├── Hermes Agent   ← Podman rootless (コード実行・ブラウザ制御あり)
    ├── OpenCrawl      ← Podman rootless (Web クローリング)
    ├── Ollama         ← Podman rootless (GPU パススルー)
    └── Open WebUI     ← Podman rootless (GUI)
```

Claude Code をディレクターとして配置し、配下のエージェントをコンテナで隔離することで、自律実行のリスクをホスト OS から切り離す。

---

## 前提環境

| 項目 | 値 |
|------|-----|
| OS | Windows 11 |
| WSL2 ディストロ | Ubuntu 24.04 |
| GPU | NVIDIA (RTX シリーズ推奨) |
| WSL2 内 nvidia-smi | 動作確認済みであること |

WSL2 内で以下が通ることを確認してから進める。

```bash
nvidia-smi
```

---

## 1. Podman のインストール

Podman は apt で提供されている。rootless がデフォルトのため、追加設定なしにユーザー権限でコンテナが起動する。

```bash
sudo apt update && sudo apt install -y podman podman-compose
podman --version
```

### docker エイリアスの設定 (任意)

既存の Docker 向けドキュメントや手順をそのまま使えるよう alias を設定する。

```bash
echo 'alias docker=podman' >> ~/.zshrc.local
```

---

## 2. NVIDIA Container Toolkit のインストール

コンテナはデフォルトでホストの GPU ドライバに触れない。NVIDIA Container Toolkit はカーネルレベルで GPU デバイスとコンテナを橋渡しする役割を担う。

WSL2 のカーネルドライバと密結合しているため、**apt で直接インストールする**。

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit
```

---

## 3. CDI の設定

CDI (Container Device Interface) は Podman が GPU デバイスを参照するための標準インタフェースである。

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
nvidia-ctk cdi list
```

`nvidia.com/gpu=0` が出力されれば正常に認識されている。

動作確認:

```bash
podman run --rm --device nvidia.com/gpu=all nvidia/cuda:12.0-base nvidia-smi
```

コンテナ内から GPU が見えれば準備完了である。

---

## 4. 起動

### WEBUI_SECRET_KEY の生成

Open WebUI のセッショントークン (JWT) 署名に使用するランダム文字列を生成する。外部サービスから取得するものではない。

**このキーが漏洩すると、攻撃者が任意のユーザーとして有効なセッショントークンを偽造できる。** リポジトリには絶対にコミットしないこと。

```bash
openssl rand -hex 32
```

`.env.example` をコピーして `.env` を作成し、生成した値を記入する。

```bash
cp .env.example .env
```

```
# .env
WEBUI_SECRET_KEY=<生成した文字列>
```

`.env` は `.gitignore` で除外済みのためコミットされない。

#### キーが漏洩した場合

新しいキーを生成して `.env` を更新し、コンテナを再起動する。キーが変わると既存の全セッションが無効化され、全ユーザーが再ログインを求められる。

```bash
openssl rand -hex 32   # 新しいキーを生成
# .env の WEBUI_SECRET_KEY を更新してから:
podman-compose restart open-webui
```

### コンテナの起動

```bash
podman-compose up -d
```

Ollama・Open WebUI・Caddy が一括で起動する。

### モデルのダウンロード

```bash
podman exec -it ollama ollama pull llama3.2
```

`http://localhost:3000` をブラウザで開いて初期設定を行う。

---

## 5. Caddy によるリバースプロキシ

`Caddyfile` のホスト名を環境に合わせて書き換える。

```
https://your-host.local:8080 {
  tls internal
  reverse_proxy open-webui:8080
}
```

Caddy が自動的に自己署名証明書を生成する。生成された root 証明書をブラウザにインポートするとセキュリティ警告を解消できる。

---

## セキュリティに関する考察

### rootless コンテナの意義

Docker の標準的な構成ではコンテナ内のプロセスが root として実行される。コンテナのブレイクアウトが発生した場合、攻撃者はホスト OS の root 権限を得る可能性がある。

Podman rootless では、コンテナ内の root はホスト OS の一般ユーザーにマップされる。侵害されてもホスト OS への影響が限定的になる。

この構成は、Hermes Agent のように LLM がローカルターミナルでのコード実行やブラウザ制御、cron による自動実行を行う場合に特に有効である。

### 段階的な移行

- **フェーズ 1**: Podman rootless + Ollama + Open WebUI を動作させる
- **フェーズ 2**: Hermes Agent / OpenCrawl 等のツール実行エージェントを追加する際に、コンテナの権限設定を見直す

---

## 参考

- [local-ai-infra リポジトリ](https://github.com/au-aii/local-ai-infra)
- [Open WebUI](https://github.com/open-webui/open-webui)
- [Hermes Agent](https://hermes-agent.org/ja/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/index.html)
- [Podman rootless チュートリアル](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md)
