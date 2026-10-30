# TurtleSim操作マニュアル

## 1. 基本
ROS 2 CLIコマンドはDockerコンテナ `pai_ros2` 内で実行します。事前に `source /opt/ros/humble/setup.bash` を実行してください。

座標は右が+x、上が+y。角度はラジアンで、0が右、π/2が上、πが左、−π/2が下です。正の角度は反時計回りです。

## 2. 状態確認【Docker】
```bash
ros2 node list
ros2 topic list
ros2 action list
ros2 topic echo /turtle1/pose --once
```

## 3. 初期化【Docker】
```bash
ros2 service call /reset std_srvs/srv/Empty '{}'
```
今回の初期位置は約 `(5.5444, 5.5444, 0)` です。

## 4. 直進【Docker】
```bash
ros2 topic pub --once /turtle1/cmd_vel geometry_msgs/msg/Twist '{linear: {x: 0.5}, angular: {z: 0.0}}'
```
`--once` は速度指令の1回送信であり、距離指定ではありません。今回の環境では約0.504移動しましたが、再現性を保証する値ではありません。

停止指令：
```bash
ros2 topic pub --once /turtle1/cmd_vel geometry_msgs/msg/Twist '{linear: {x: 0.0}, angular: {z: 0.0}}'
```

## 5. 絶対角度への回転【Docker】
```bash
ros2 action send_goal /turtle1/rotate_absolute turtlesim/action/RotateAbsolute '{theta: 1.5708}'
```
**絶対角度**なので、同じthetaを繰り返し送っても90度ずつ回転しません。

## 6. LM StudioからのMCP操作
直進は `publish_once`、トピック `/turtle1/cmd_vel`、型 `geometry_msgs/msg/Twist`、メッセージ本体：
```json
{"linear":{"x":0.5},"angular":{"z":0.0}}
```
回転は `send_action_goal`、アクション `/turtle1/rotate_absolute`、型 `turtlesim/action/RotateAbsolute`、ゴール本体：
```json
{"theta":1.5708}
```
ツール全体の引数名はLM Studioに表示されるMCPスキーマに従ってください。ゴールのキーは `theta` であり、`angle` や `radians` ではありません。

## 7. 正方形を描く7操作
1. 直進（速度0.5）
2. 絶対回転（π/2）
3. 直進（速度0.5）
4. 絶対回転（π）
5. 直進（速度0.5）
6. 絶対回転（−π/2）
7. 直進（速度0.5）

始点付近に戻ります。**初期の向きまで戻す**なら、8番目に絶対回転（0）を追加します。

実験AおよびB-5では最終位置約 `(5.5496974, 5.5471921)`、始点との距離約 `0.00593` を観測しました。単一試行の結果であり、継続的な位置フィードバック制御を実証したものではありません。

## 8. 失敗時の確認
- Gemmaの「成功」という文章だけで判断せず、**実際のMCPツール履歴**とTurtleSimの画面・poseを照合する。
- `rotate_absolute` は相対回転ではない。
- `publish_for_durations` は送信終了後の停止を保証しない。今回、予想より長く移動する現象を観測した。
- MCPで生成されたJSONに余分なエスケープがないか確認する。

[環境立ち上げマニュアル](setup_guide.md)
