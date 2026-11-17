# tb3-dynamic-slam-frontier

TurtleBot3 Burger(ROS 2 Humble)에서 **LiDAR의 움직이는 물체를 SLAM에 넣기 전에 걸러내고**, Cartographer · Frontier 탐색 · Nav2를 함께 돌려 사람이 몰지 않아도 지도를 만들며 이동하게 하는 ROS 2 패키지(`tb3_dynamic_detector`)입니다.

개인 프로젝트이며 모빌리티 AI 취업역량강화 과정(2026.07~08) 중 진행했습니다. 실물 TurtleBot3 Burger와 WSL(Ubuntu 22.04)에서 실험했습니다.

## 만든 이유

기존 방식은 사람이 로봇을 몰아 지도를 먼저 만든 뒤 RViz로 목적지를 찍는 것이었습니다. 로봇이 스스로 미탐색 영역을 찾아가며 지도를 만들려고 하자, 앞을 지나간 다른 로봇의 점이 지도에 벽으로 남아 Frontier 탐색이 그 앞을 막힌 곳으로 판단했습니다. 그래서 움직이는 물체를 SLAM 입력 단계에서 먼저 걸러냈습니다.

## 데이터 흐름

```text
/scan ─► sensor_freshness_filter ─► dynamic_detector ─► /scan_static ─► Cartographer ─► /map
          (오래되거나 순서가 뒤바뀐              (군집화, 정적 배경, 시간 점유격자,         │
           메시지 폐기)                          Kalman 추적, 5프레임 중 3회 확정)         ▼
                                                                          dynamic_map_filter ─► /map_static ─► Frontier 탐색(explore_lite) · Nav2
```

- 동적 판정은 거리 기반 군집화 → 정적 배경 이력(30프레임) → 0.05 m 시간 점유격자(과거 8회 이상 비어 있던 칸에 새로 생긴 점) → 등속 Kalman 추적의 순서로 합니다. 최근 5프레임 중 3회 이상 동적일 때만 확정합니다. 값은 `config/detector.yaml`에 있습니다.
- 스캔 시각의 TF가 0.30초 안에 없으면 최신 자세로 대신 계산하지 않고 그 스캔을 버립니다(과거 스캔에 현재 자세를 붙이면 멈춘 벽이 움직인 것처럼 보이기 때문).
- `/map_static` 필터는 10 Hz LiDAR 콜백이 지연되지 않도록 **별도 프로세스**(`dynamic_map_filter`)로 분리했습니다.

## 막혔던 문제와 원인

| 증상 | 원인이 있던 층 | 조치와 결과 |
|---|---|---|
| LiDAR scan·odom이 시간당 2~3회 끊기고 재부팅마다 평균 약 5분 | 통신(DDS) | discovery 설정·QoS·RMW 혼용을 나눠 확인 → RMW 통일, 역할별 DDS 설정 XML 분리, systemd 상시 구동. 이후 실험 동안 다시 끊기지 않음(관찰, 로그 집계 아님). [상세](docs/네트워크_FastDDS_트러블슈팅_2026-08-12.md) |
| 지도 갱신이 12.247초 멈춤 | 프로세스 구조 | 지도 마스킹이 탐지 노드 안에서 지도 콜백을 늦추고 있었음 → `dynamic_map_filter`를 별도 프로세스로 분리. `/map_static`이 입력과 같은 1.000 Hz, 최악 콜백 0.019초(약 74.7분 로그). [상세](docs/동적장애물_통합_실험_결과_한계점_FutureWork_2026-08-16.md) |
| RViz의 지도와 로봇 모델이 도는 현상 | 하드웨어 | `tf2_echo`로 translation은 고정, yaw만 흔들림을 확인 → OpenCR IMU 자이로 부팅 보정 오류. 재부팅으로 해결 |

## 한계 (확인된 것만)

- `planner_server` 루프가 목표 20 Hz 대비 1.18 Hz로 떨어지는 문제는 **원인을 규명하지 못했습니다**.
- 위 1.000 Hz 결과는 움직이는 물체가 있던 구간이 로그 477줄 중 18줄뿐이어서, 동적 물체가 많은 상황의 검증은 충분하지 않습니다.
- 검증 공간이 약 4.5 m 규모로 작아 더 크거나 복잡한 환경의 확장성은 확인하지 않았습니다.
- `detector.yaml`은 정지 로봇 실험 기준 값을 그대로 완전 자율주행에 쓰고 있습니다.
- 12.247초 정지는 `ros2 topic hz`로 관찰한 값이며 ROS 로그에는 남아 있지 않습니다.
- 전체 목록: [docs/동적장애물_통합_실험_결과_한계점_FutureWork_2026-08-16.md](docs/동적장애물_통합_실험_결과_한계점_FutureWork_2026-08-16.md)

## 개발 방식과 출처

- 노드 구현과 단위 테스트 작성에 **AI 코딩 도구를 사용했습니다**. 무엇을 측정하고 어떤 조건을 넣을지, 실물 로봇에서의 확인은 직접 했습니다. 테스트(`test/`)는 AI가 작성했습니다.
- Frontier 탐색은 외부 패키지 [`explore_lite`](https://github.com/robo-friends/m-explore-ros2)를 사용했고, 이 저장소는 그 입력(`/map_static`)과 launch 구성을 맡습니다. 직접 구현한 탐색 알고리즘이 아닙니다.
- Cartographer, Nav2, TurtleBot3 공식 패키지에 의존합니다.

## 설치 (WSL Ubuntu 22.04, ROS 2 Humble)

이 저장소 자체가 colcon 워크스페이스입니다.

```bash
git clone <이 저장소 URL> ~/tb3_dynamic_ws
cd ~/tb3_dynamic_ws
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

지도 저장 경로는 환경변수 `TB3_MAP_DIR`(기본 `~/tb3_maps`)로 바꿀 수 있습니다.

## 실행

TurtleBot3 bringup(필요 시 Discovery Server 포함)을 먼저 실행한 뒤:

```bash
# SLAM + Frontier 탐색 + 동적 물체 제거 + Nav2 통합
ros2 launch tb3_dynamic_detector slam_dynamic_frontier_super_manager.launch.py

# 동적 후보 박스만 RViz로 확인 (SLAM 입력은 건드리지 않음)
ros2 launch tb3_dynamic_detector dynamic_boxes.launch.py
```

박스·토픽 사용법은 [docs/dynamic_boxes_usage.md](docs/dynamic_boxes_usage.md)를 참고하세요.

## 테스트

```bash
colcon test --packages-select tb3_dynamic_detector
```

## 결과 예시

![frontier 탐색으로 만든 지도](assets/tb3_dynamic_frontier_map_20260812_111935.png)

## 라이선스

Apache-2.0 ([LICENSE](LICENSE))
