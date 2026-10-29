# TurtleSim正方形描画プロンプト

## 実験A：操作手順を明示した正方形描画

使用モデル：Gemma 4 E4B（LM Studio）

MCPツールを使用して、TurtleSimの亀に正方形を描かせてください。

以下の8操作を順番に実行してください。

1. 前進
2. theta=1.5708 に回転
3. 前進
4. theta=3.14159 に回転
5. 前進
6. theta=-1.5708 に回転
7. 前進
8. theta=0.0 に回転

### 【前進】

ツール: publish_for_durations

topic: /turtle1/cmd_vel

msg_type: geometry_msgs/msg/Twist

messages:

```json
[{
  "linear": {"x": 0.5, "y": 0.0, "z": 0.0},
  "angular": {"x": 0.0, "y": 0.0, "z": 0.0}
}]
```

durations: [2.0]

rate_hz: 10

### 【回転】

ツール: send_action_goal

action: /turtle1/rotate_absolute

action_type: turtlesim/action/RotateAbsolute

goal: {"theta": 指定角度}

### 【重要】

- 必ず1操作ずつ実行する。
- 回転が完了してから次の前進を開始する。
- 前進時のTwistは6成分すべて指定する。
- 途中で停止した場合は、完了した操作番号を報告する。
- ツールの実行結果だけで描画成功と判断しない。
- 途中の説明は最小限にして、ツール実行を優先する。

## 実験結果（2026年10月29日）

Gemma 4 E4Bが8操作を連続実行し、TurtleSim上で正方形の描画に成功した。

途中、`send_action_goal`の呼び出しで`action_name`が指定されていないというエラーが発生したが、モデルが引数を修正して再実行し、処理を継続した。

最終的な描画結果は`images/square_result.png`に保存する。