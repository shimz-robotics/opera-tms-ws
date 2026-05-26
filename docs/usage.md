# Usage

ホストから `scripts/exec.sh` 1 本で container 内のコマンドを叩く運用を基本とする。インタラクティブに探索したいときは `scripts/shell.sh` で対話 shell に入る。

## 動作確認: `task_id=4` 完走

### Terminal 1: Unity ↔ ROS 2 ブリッジ

```bash
./scripts/exec.sh ros2 launch ros_tcp_endpoint endpoint.py
```

`Starting server on 0.0.0.0:10000` が出たら待機状態。ここで Unity 側を再生し、`Connection from <Unity-IP>` のログを確認。

### Terminal 2: bringup（zx200 / tms_if / tms_ts を 3 段連鎖起動）

Terminal 1 起動 → Unity 再生 → 接続確認後に Terminal 2 を起動:

```bash
./scripts/exec.sh ros2 launch /workspace/src/ros2_tms_for_construction/docker/launch/bringup.launch.yaml
```

詳細手順（Unity 設定、RViz の初期姿勢回避、緑ボタン押下）は上流 README を参照: <https://github.com/irvs/ros2_tms_for_construction/tree/main/docker>

### 対話 shell で操作したい場合

```bash
./scripts/shell.sh
# プロンプトが ros@<container> になる、ROS env も source 済
# あとは普通に ros2 launch / ros2 node list 等を実行
```

## 停止

```bash
docker compose down         # コンテナ停止（named volume は保持）
docker compose down -v      # named volume ごと削除（DB・build キャッシュも消える）
```

`src.repos` / `Dockerfile` を更新したときは `down -v` してから rebuild する（`down` だけでは old artifact が再利用される）。詳細は [development.md](development.md) 参照。

## 既知の制約 / トラブルシュート

- ROS 2 humble の Tier 1 サポートは Ubuntu 22.04 のみ
- GPU アクセラレーションは未設定
- moveit / nav2 / tf 系は wildcard を避け、`src.repos` の `<depend>` を grep した実依存ベースで explicit list 化（image を 12 GB → 5-6 GB を目標）。colcon build で「未 install」と言われたら Dockerfile のコメント済み apt-get install ブロックにそのパッケージを足す
- **ポート衝突**: `network_mode: host` のため `27017`（MongoDB）が他で使われていると起動失敗。`sudo systemctl stop mongod` で native mongod を停止、または `docker ps --filter "publish=27017"` で別 container を確認して `docker stop <name>` で空ける
- **`sequence size exceeds remaining buffer` 警告が大量に出る**: 共有マシン上で他人の ROS 2 publish が混入している。`ROS_DOMAIN_ID` を被らない番号に変更（`ROS_DOMAIN_ID=42 docker compose up -d`）
- **`xhost +local:` は閉じ忘れに注意**: 作業後 `xhost -local:`
- **xrdp 等で `$DISPLAY` が変わる環境**: `scripts/exec.sh` / `scripts/shell.sh` は呼び出し時の `$DISPLAY` を毎回 container に渡すので、ホスト側 shell の DISPLAY が正しければ OK。compose で記録された値を強制更新したい場合は `docker compose down && up`
