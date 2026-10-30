# 環境立ち上げマニュアル

## 構成
Windows 11（LM Studio / Gemma 4 E4B）→ WSL2 Ubuntu 22.04（ros-mcp）→ rosbridge :9090 → Docker（ROS 2 Humble / TurtleSim）。

## 1. Docker起動【WSL】
```bash
cd ~/physical-ai-mcp-turtlesim
docker ps -a --filter name=pai_ros2
docker start pai_ros2
```
初回のみ、コンテナが存在しないことを確認して `bash scripts/run_pai_container.sh` を実行します。`docker run` は新規作成、`docker start` は既存コンテナの再開です。

## 2. TurtleSim起動【Docker・ターミナル1】
WSLから `docker exec -it pai_ros2 bash` で入ります。
```bash
source /opt/ros/humble/setup.bash
export DISPLAY=:0
ros2 run turtlesim turtlesim_node
```
Windowsのブラウザで http://localhost:6080 を開きます。

## 3. rosbridge起動【Docker・別ターミナル】
WSLから別の `docker exec -it pai_ros2 bash` を実行します。
```bash
source /opt/ros/humble/setup.bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml
```
起動したターミナルは開いたままにします。

## 4. 動作確認【Docker・3つ目のターミナル】
```bash
source /opt/ros/humble/setup.bash
ros2 node list
ros2 topic echo /turtle1/pose --once
ros2 action list
```

## 5. LM StudioのMCP設定【Windows】
今回の動作確認済み設定例：
```json
{"mcpServers":{"ros-mcp":{"command":"wsl.exe","args":["-d","Ubuntu-22.04","--","/root/.local/bin/uvx","ros-mcp","--transport","stdio"]}}}
```
このパスは今回のrootユーザー固有です。LM StudioでGemma 4 E4Bを読み込み、MCPを有効にします。`connect_to_robot` に `{"ip":"127.0.0.1","port":"9090"}` を渡し、接続結果を確認します。

## 6. 終了・再起動
TurtleSimとrosbridgeのターミナルで `Ctrl+C`。コンテナ停止はWSLで `docker stop pai_ros2`。再開後はROSプロセスを必要に応じて再起動します。

## 7. 注意
- `ps` をWSL側で実行しても、Docker内部のROSプロセスは通常表示されません。
- ポート6080はブラウザ画面、9090はrosbridge。9876はスクリプトで公開されていますが用途は未確認です。
- 起動スクリプトの `$(pwd)` は実行時の作業ディレクトリです。

[操作マニュアル](turtlesim_guide.md)
