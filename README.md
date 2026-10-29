# Physical AI: LLMによるMCP経由のTurtleSim制御

## 1. 概要

本プロジェクトは、ローカルLLMとModel Context Protocol（MCP）を利用して、自然言語の指示からROS 2上のTurtleSimを操作する実験です。

Windows上のLM Studioで動作するGemma 4 E4Bから、MCPサーバーを介してROS 2のトピックおよびアクションを呼び出し、亀に正方形を描画させることを目標としました。

特に、以下の2点を検証しました。

- 操作手順を明示した場合、LLMが複数のMCPツールを連続実行できるか。
- 描画目標だけを与えた場合、LLMが自ら操作手順を計画して実行できるか。

## 2. システム構成

```text
Windows 11
    |
    +-- LM Studio
    |     +-- Gemma 4 E4B
    |
    +-- WSL2 (Ubuntu 22.04)
    |     +-- ros-mcp (MCP Server)
    |
    +-- Docker Desktop (WSL2 backend)
          +-- ROS 2 Humble
          +-- rosbridge_server
          +-- turtlesim
```

制御の流れは次のとおりです。

```text
自然言語の指示
    ↓
Gemma 4 E4B（LM Studio）
    ↓
MCPツール呼び出し
    ↓
ros-mcp
    ↓
rosbridge（WebSocket）
    ↓
ROS 2
    ↓
TurtleSim
```

## 3. 使用した主な技術

| 項目 | 使用技術 |
|---|---|
| OS | Windows 11 |
| Linux環境 | WSL2 / Ubuntu 22.04 |
| LLM実行環境 | LM Studio |
| LLM | Gemma 4 E4B |
| ツール連携 | Model Context Protocol（MCP） |
| MCPサーバー | ros-mcp |
| ロボットミドルウェア | ROS 2 Humble |
| ROS通信 | rosbridge_server |
| シミュレータ | TurtleSim |
| コンテナ | Docker Desktop |

## 4. 使用したROS 2インターフェース

### 前進

- MCPツール：`publish_for_durations`
- トピック：`/turtle1/cmd_vel`
- メッセージ型：`geometry_msgs/msg/Twist`

前進速度と実行時間を指定して、亀を直進させました。

### 回転

- MCPツール：`send_action_goal`
- アクション：`/turtle1/rotate_absolute`
- アクション型：`turtlesim/action/RotateAbsolute`

目標角度を指定して、亀の向きを変更しました。

### 初期位置への移動

- サービス：`/turtle1/teleport_absolute`
- サービス型：`turtlesim/srv/TeleportAbsolute`

## 5. 実験A：操作手順を明示した正方形描画

### 目的

自然言語で8つの操作を指定し、LLMがMCPツールを順番に呼び出して正方形を描画できるか確認しました。

### 方法

4回の直進と4回の回転を指示しました。

直進では速度0.5、実行時間2秒を指定しました。回転には絶対角度を使用しました。

使用したプロンプトは以下に保存しています。

`prompts/draw_square.md`

### 結果

Gemma 4 E4BはMCPツールを連続して呼び出し、TurtleSim上で正方形に近い軌跡を描画しました。

途中、回転ツールの引数名に関するエラーが発生しましたが、モデルが`action_name`を指定するよう修正し、処理を継続しました。

描画結果はTurtleSim画面で目視確認しました。

### 描画結果

![実験A：正方形の描画結果](images/square_result.png)

### 考察

操作手順が明示されている場合、LLMは複数のROS 2操作をMCP経由で実行できました。

また、ツール呼び出し時の引数エラーに対して、モデルが修正を試みて処理を継続する挙動が確認できました。

一方、操作手順と主要なパラメータは人間が与えており、この実験だけでLLMが自律的に経路計画できたとはいえません。

## 6. 実験B：操作計画をLLMに任せた正方形描画

### 目的

正方形を描くという目標を与え、前進速度、移動時間、回転角度、操作順序をLLM自身に考えさせました。

### 結果

Gemma 4 E4Bは操作計画を生成し、最初の直進操作を実行しました。

しかし、回転アクションの呼び出しで、目標メッセージに存在しない`radians`や`angle`を指定してエラーとなり、正方形の描画は完了しませんでした。

また、絶対角度を指定するアクションと相対回転の違いについても、計画上の課題が見られました。

### 描画結果（途中停止）

![実験B：途中停止した描画結果](images/square_failed.png)

### 考察

LLMは自然言語の目標から操作計画を生成できましたが、ROS 2アクションのメッセージ仕様を正しく扱えず、実行途中で停止しました。

実験Aと比較すると、ツールの仕様をプロンプトで明示することが、実行成功に大きく影響する可能性が示されました。

## 7. AIを使用した部分

本プロジェクトでは、LLMを単なる文章生成ではなく、ROS 2操作のためのツール呼び出し主体として使用しました。

実験Aでは、LLMが指示された操作をMCPツールに変換し、連続実行しました。

実験Bでは、LLM自身に操作計画の生成も担当させました。

この比較を通じて、LLMによるロボット制御の可能性と、ツール仕様の理解に関する限界を確認しました。

## 8. リポジトリ構成

```text
physical-ai-mcp-turtlesim/
├── README.md
├── scripts/
│   └── run_pai_container.sh
├── prompts/
│   └── draw_square.md
├── images/
└── docs/
```

`images`には描画結果、`docs`には実験記録などを追加する予定です。

## 9. 今後の改善

- MCPツールの引数仕様をモデルへ正確に伝える。
- 回転アクションの完了を確認してから次の操作を実行する。
- TurtleSimの位置・姿勢を取得して動作結果を検証する。
- エラー発生時に再計画できる仕組みを検討する。

## 10. 実験日

2026年10月29日
