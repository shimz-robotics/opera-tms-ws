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

## RMW 切替 (Cyclone DDS / Zenoh)

デフォルトは Fast DDS (`rmw_fastrtps_cpp`)。3 RMW すべて image に同梱済 (`ros-humble-rmw-{fastrtps,cyclonedds,zenoh}-cpp`)。ホスト側に DDS / Zenoh を別途 install する必要はない (RMW プラグインは container 内 ROS 2 プロセスに linked-in)。

### Cyclone DDS

```bash
RMW_IMPLEMENTATION=rmw_cyclonedds_cpp docker compose up -d
```

profile を指定する場合 (例: 後述の Node-RED 並列運用):

```bash
RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
CYCLONEDDS_URI=file:///workspace/src/cyclonedds.conf \
  docker compose up -d
```

profile XML は host の `src/cyclonedds.conf` (gitignored) に置けば `./src/:/workspace/src/` の bind mount 経由で container 内に見える。

### Zenoh

```bash
RMW_IMPLEMENTATION=rmw_zenoh_cpp docker compose --profile zenoh up -d
```

`--profile zenoh` で `zenoh-router` (zenohd) も同時起動。tms 内のノードは `localhost:7447` の router を経由する (`network_mode: host` で localhost 共有)。

### env 注入の確認

```bash
docker inspect opera_tms_dev \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep -E "RMW|CYCLONE|ZENOH|DOMAIN"
```

`.env` の export 漏れで silent に Fast DDS に戻ることがある (compose の `${RMW_IMPLEMENTATION:-rmw_fastrtps_cpp}` のため)。起動後に必ず確認。

## Node-RED (nodered-ros2-opera) との並列運用

[`shimz-robotics/nodered-ros2-opera`](https://github.com/shimz-robotics/nodered-ros2-opera) は Cyclone DDS hardcoded (`rmw_cyclonedds_cpp` + `launch_content/cyclone_profile.xml` を `CYCLONEDDS_URI` で指す)。両者を同 host で起動するだけで DDS discovery が通る。

### profile を揃える

nodered 側 `launch_content/cyclone_profile.xml` と同等の内容を TMS 側にも置く。host の `src/cyclonedds.conf` (gitignored) に以下を保存:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<CycloneDDS>
 <Domain>
     <General>
         <Interfaces>
            <NetworkInterface address="127.0.0.1" multicast="true"/>
         </Interfaces>
         <DontRoute>true</DontRoute>
     </General>
 </Domain>
</CycloneDDS>
```

両 profile の `<DontRoute>` / `<NetworkInterface>` が一致していないと discovery が片方向になることがある。

### 起動

```bash
# TMS 側 (このリポ)
cd opera-tms-ws
RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
CYCLONEDDS_URI=file:///workspace/src/cyclonedds.conf \
  docker compose up -d

# Node-RED 側 (別リポ)
cd ../nodered-ros2-opera
docker compose up -d
```

両 container 共に `network_mode: host` + `ROS_DOMAIN_ID=0` (デフォルト) + Cyclone DDS の localhost multicast で discovery する。

### discovery 確認

```bash
# TMS 側から
./scripts/exec.sh ros2 node list
./scripts/exec.sh ros2 topic list

# Node-RED 側から
docker exec -it shimz-node-red \
  bash -lc 'source /opt/ros/humble/setup.bash && ros2 node list'
```

両方から相手側の node が見えれば接続成功。

### つまずきポイント

- 相手の node が見えない → 両 container の `RMW_IMPLEMENTATION` と `ROS_DOMAIN_ID` を `docker inspect` で確認 (上述 env 注入の確認を両 container に対して実行)
- 片方向のみ → profile の `<DontRoute>true</DontRoute>` が片方欠けていないか、両 container `network_mode: host` で立っているか
- Node-RED 側 flow が topic を出さない → http://localhost:1880 admin で flow を deploy 必要

## 既知の制約 / トラブルシュート

- ROS 2 humble の Tier 1 サポートは Ubuntu 22.04 のみ
- GPU アクセラレーションは未設定
- moveit / nav2 / tf 系は wildcard を避け、`src.repos` の `<depend>` を grep した実依存ベースで explicit list 化（image を 12 GB → 5-6 GB を目標）。colcon build で「未 install」と言われたら Dockerfile のコメント済み apt-get install ブロックにそのパッケージを足す
- **ポート衝突**: `network_mode: host` のため `27017`（MongoDB）が他で使われていると起動失敗。`sudo systemctl stop mongod` で native mongod を停止、または `docker ps --filter "publish=27017"` で別 container を確認して `docker stop <name>` で空ける
- **`sequence size exceeds remaining buffer` 警告が大量に出る**: 共有マシン上で他人の ROS 2 publish が混入している。`ROS_DOMAIN_ID` を被らない番号に変更（`ROS_DOMAIN_ID=42 docker compose up -d`）
- **`xhost +local:` は閉じ忘れに注意**: 作業後 `xhost -local:`
- **xrdp 等で `$DISPLAY` が変わる環境**: `scripts/exec.sh` / `scripts/shell.sh` は呼び出し時の `$DISPLAY` を毎回 container に渡すので、ホスト側 shell の DISPLAY が正しければ OK。compose で記録された値を強制更新したい場合は `docker compose down && up`
