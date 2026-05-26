# opera-tms-ws

ROS 2 TMS for Construction の **開発用 meta-workspace**。Docker + vcstool で `ros2_tms_for_construction` 本体と全外部依存を 1 ファイルで管理し、ホスト側で全 src を編集しながらコンテナ内で colcon build できる構成。

- **ベース image**: `osrf/ros:humble-desktop-full` (Ubuntu 22.04 jammy)
- **GUI**: X11 forward（VNC なし）
- **MongoDB**: 別サービス (`mongo:6.0`)
- **外部依存**: `src.repos` (public) / `src.private.repos` (private) で commit を固定して vcstool 管理

## 上流 PR との関係

「最小限の Docker テンプレート」として `irvs/ros2_tms_for_construction` の `docker/` 配下にも別の Dockerfile セットが存在する:

| 役割 | 場所 |
|---|---|
| 誰でもまず動かせる reference | `irvs/ros2_tms_for_construction:docker/` |
| 全依存を src.repos で管理する開発・実験環境 | この meta-repo |

両方を並存させる。本 meta-repo の `Dockerfile` / `entrypoint.sh` / `restore-db.sh` / `launch/bringup.launch.yaml` は上流の `docker/` をベースにしているため、上流が更新されたら追従が必要。

## Quick Start

```bash
git clone https://github.com/shimz-robotics/opera-tms-ws.git
cd opera-tms-ws

mkdir -p src
vcs import src/ < src.repos
vcs import src/ < src.private.repos || true

xhost +local:
UID=$(id -u) GID=$(id -g) docker compose build      # 初回 20-30 分
docker compose up -d                                  # 初回起動時にコンテナ内 colcon build (10 分)
docker compose exec tms restore-db.sh                # DB seed 投入 (初回のみ)

./scripts/shell.sh                                    # コンテナ shell に入る
./scripts/exec.sh ros2 node list                     # ホストから 1 コマンド
```

## ドキュメント

- [docs/setup.md](docs/setup.md) — 前提・ワークスペース構造・起動手順（詳細）
- [docs/usage.md](docs/usage.md) — `task_id=4` 完走手順 / 停止 / 既知の制約・トラブルシュート
- [docs/development.md](docs/development.md) — src 編集 → 再ビルドの運用（Python / C++ / pin bump / 初回 build やり直し / Dockerfile 変更）

## 関連

- 上流: [irvs/ros2_tms_for_construction](https://github.com/irvs/ros2_tms_for_construction)
- Unity 側シミュレータ: [shimz-robotics/OperaSim-PhysX](https://github.com/shimz-robotics/OperaSim-PhysX) — zx200 link の DeadTime tuning で制御の振動挙動を緩和済み（`shimz-main` branch）。Unity fork 版 `zx200_ros2` は本 meta-repo では取り込まない
