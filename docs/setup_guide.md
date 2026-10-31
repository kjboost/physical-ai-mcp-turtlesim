# 環境構築・起動マニュアル

## 1. プロジェクトの概要

本プロジェクトでは、松尾研Physical AI応用1講座で使用したROS 2環境を利用し、ローカルLLMからMCP（Model Context Protocol）を介してTurtleSimを操作するシステムを構成した。

### 1.1 講義で使用した環境

第4回講義では、ROS 2 HumbleのDocker環境でTurtleSimを使用し、以下の操作を学習した。

- `turtlesim_node` の起動
- `turtle_teleop_key` による遠隔操作
- `ros2 topic list` によるトピック一覧の確認
- `ros2 topic echo` による通信データの確認
- `ros2 topic type` によるメッセージ型の確認
- ROS 2アクションのフィードバック確認

第5回講義では、TurtleSimを用いてrosbagによる通信データの記録・再生を学習した。

本プロジェクトでは、この講義環境を最終課題の実行基盤として再利用した。

使用したDockerイメージ：

`airobotbook/ros2-desktop-ai-robot-book-humble:latest`

### 1.2 最終課題で追加・設定した構成

講義環境を基盤として、以下の構成を追加・設定した。

- Dockerコンテナへの `rosbridge_server` のインストール
- rosbridge WebSocketサーバの起動
- WSL2 Ubuntu 22.04上での `ros-mcp` の利用
- Windows上でのLM Studioの利用
- ローカルLLM「Gemma 4 E4B」の利用
- LM StudioからMCPサーバを起動する設定
- 自然言語からROS 2操作を実行するプロンプトの作成と改善

ROS 2やTurtleSimそのものを新規開発したものではなく、既存のROS 2環境にLLMとMCPを接続し、その動作を検証した。

## 2. システム構成

```text
Windows 11
│
├── LM Studio
│     └── Gemma 4 E4B
│           │
│           │ MCP（stdio）
│           ▼
├── WSL2 Ubuntu 22.04
│     └── ros-mcp
│           │
│           │ WebSocket
│           ▼
└── Docker Desktop
      └── pai_ros2
            ├── ROS 2 Humble
            ├── rosbridge_server（9090）
            └── turtlesim_node
```

使用環境：

| 項目 | 内容 |
|---|---|
| ホストOS | Windows 11 |
| Linux環境 | WSL2 Ubuntu 22.04 |
| コンテナ環境 | Docker Desktop |
| ROS 2 | Humble |
| Dockerイメージ | `airobotbook/ros2-desktop-ai-robot-book-humble:latest` |
| コンテナ名 | `pai_ros2` |
| LLM実行環境 | LM Studio |
| 使用モデル | Gemma 4 E4B |
| MCPサーバ | `ros-mcp` |
| ROS 2との接続 | rosbridge WebSocket |
| デスクトップ画面 | ポート6080 |
| rosbridge | ポート9090 |

## 3. Dockerコンテナの起動【WSL】

WindowsでDocker Desktopを起動する。

WSL2 Ubuntu 22.04で以下を実行する。

```bash
cd ~/physical-ai-mcp-turtlesim

docker ps -a --filter name=pai_ros2
docker start pai_ros2
```

`docker start` は既存のコンテナを再開するコマンドである。

初回のみ、コンテナが存在しないことを確認してから、以下を実行する。

```bash
bash scripts/run_pai_container.sh
```

このスクリプトでは、ROS 2 HumbleのDockerイメージを利用してコンテナを作成する。

**注意：** `docker run` は新しいコンテナを作成する操作であり、`docker start` とは異なる。

## 4. TurtleSimの起動【Docker・ターミナル1】

WSL側からDockerコンテナに入る。

```bash
docker exec -it pai_ros2 bash
```

コンテナ内で以下を実行する。

```bash
source /opt/ros/humble/setup.bash
export DISPLAY=:0

ros2 run turtlesim turtlesim_node
```

Windowsのブラウザで以下を開く。

`http://localhost:6080`

TurtleSimのウィンドウに亀が表示されることを確認する。

## 5. rosbridgeの起動【Docker・ターミナル2】

WSL側で別のターミナルを開き、Dockerコンテナに入る。

```bash
docker exec -it pai_ros2 bash
```

コンテナ内で以下を実行する。

```bash
source /opt/ros/humble/setup.bash

ros2 launch rosbridge_server rosbridge_websocket_launch.xml
```

rosbridgeは、ROS 2の通信機能をWebSocket経由で利用できるようにする仕組みである。

今回のシステムでは、`ros-mcp` がrosbridgeを経由してTurtleSimを操作する。

このターミナルは起動したままにする。

## 6. ROS 2の動作確認【Docker・ターミナル3】

WSL側で3つ目のターミナルを開き、Dockerコンテナに入る。

```bash
docker exec -it pai_ros2 bash
```

コンテナ内で以下を実行する。

```bash
source /opt/ros/humble/setup.bash

ros2 node list
ros2 topic list
ros2 topic echo /turtle1/pose --once
ros2 action list
```

主な確認対象：

| ROS 2インターフェース | 用途 |
|---|---|
| `/turtle1/cmd_vel` | 亀の直進・回転速度の指令 |
| `/turtle1/pose` | 亀の位置と向きの取得 |
| `/turtle1/rotate_absolute` | 絶対角度への回転 |
| `/reset` | TurtleSimの初期化 |

## 7. LM StudioのMCP設定【Windows】

WindowsでLM Studioを起動し、Gemma 4 E4Bを読み込む。

今回、動作確認したMCP設定は以下のとおりである。

```json
{
  "mcpServers": {
    "ros-mcp": {
      "command": "wsl.exe",
      "args": [
        "-d",
        "Ubuntu-22.04",
        "--",
        "/root/.local/bin/uvx",
        "ros-mcp",
        "--transport",
        "stdio"
      ]
    }
  }
}
```

`/root/.local/bin/uvx` は、今回のWSL環境における実行ファイルのパスである。

他の環境で再現する場合は、実際のインストール先に合わせて変更する必要がある。

LM StudioでMCPを有効にし、`ros-mcp` のツールが利用可能であることを確認する。

### 7.1 ROS 2への接続

MCPツール `connect_to_robot` を使用する。

接続パラメータ：

```json
{
  "ip": "127.0.0.1",
  "port": "9090"
}
```

接続に成功したら、TurtleSimのトピックやアクションにアクセスできることを確認する。

## 8. 終了・再起動

### 8.1 終了

TurtleSimとrosbridgeを起動したターミナルで、それぞれ `Ctrl+C` を実行する。

WSL側でDockerコンテナを停止する。

```bash
docker stop pai_ros2
```

必要に応じてLM StudioとDocker Desktopを終了する。

### 8.2 再起動

WindowsでDocker Desktopを起動した後、WSL側で以下を実行する。

```bash
docker start pai_ros2
```

続いて、TurtleSimとrosbridgeを再起動する。

LM StudioでGemma 4 E4Bを読み込み、MCP接続を確認する。

## 9. 環境の再現に関する注意事項

### 9.1 Dockerコンテナの保持

現在の `pai_ros2` コンテナには、講義で使用したROS 2環境に加え、最終課題のためにインストールした `rosbridge_server` が含まれている。

既存コンテナを停止・再起動するだけであれば、通常はインストール済みパッケージが保持される。

一方、コンテナを削除して元のDockerイメージから再作成した場合、コンテナ内で追加したパッケージは引き継がれない。

### 9.2 rosbridge_serverのインストール

新規コンテナに `rosbridge_server` が存在しない場合は、コンテナ内で以下を実行する。

```bash
apt-get update
apt-get install -y ros-humble-rosbridge-server
```

今回の環境では、インストール時にAPTの署名鍵に関する問題が発生し、鍵情報の更新後に導入した。

このため、上記のコマンドだけで必ず再現できることまでは確認していない。

### 9.3 Dockerイメージのバージョン

使用したイメージのタグは `latest` である。

このタグが指すイメージは将来変更される可能性があるため、厳密な再現にはイメージのダイジェストなどの記録が望ましい。

### 9.4 その他の注意事項

- WSL側の `ps` では、Dockerコンテナ内のROS 2プロセスが通常表示されない。
- ポート6080はブラウザ画面、9090はrosbridgeに使用する。
- ポート9876も起動スクリプトで公開しているが、今回の実験における用途は未確認である。
- 起動スクリプト内の `$(pwd)` は、スクリプト実行時の作業ディレクトリを表す。
- 実験の再現には、Docker、WSL2、LM StudioおよびMCPサーバが正しく動作する必要がある。

## 10. 関連資料

- [TurtleSim操作マニュアル](turtlesim_guide.md)
- [プロジェクトREADME](../README.md)
- [実験B-5のプロンプト](../prompts/autonomous_square_b5.md)