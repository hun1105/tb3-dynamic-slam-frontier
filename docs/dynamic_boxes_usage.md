# 동적 후보 바운딩 박스 사용법 (dynamic_boxes)

이 문서는 초기 단계(점 삭제 전 검증용 박스 표시) 기준입니다. 현재 통합 launch는 동적 점을 제거한 `/scan_static`을 사용합니다.

- `/scan`과 TF를 이용해 점을 `odom` 좌표로 보정합니다.
- 인접 LiDAR 점을 클러스터링합니다.
- 클러스터 중심을 추적해 속도를 계산합니다.
- 시간 일관성이 낮고 속도가 빠른 클러스터를 동적으로 판정합니다.
- RViz용 박스, 속도 화살표, 추적 ID를 발행합니다.
- 원본 SLAM 입력은 변경하지 않습니다.


## 실행

Discovery Server와 TurtleBot3 bringup을 먼저 실행합니다.

기존 SLAM을 그대로 두고 박스만 추가:

```bash
source ~/.bashrc
source ~/tb3_dynamic_ws/install/setup.bash
ros2 launch tb3_dynamic_detector dynamic_boxes.launch.py
```

SLAM과 박스를 동시에 실행:

```bash
source ~/.bashrc
source ~/tb3_dynamic_ws/install/setup.bash
ros2 launch tb3_dynamic_detector slam_with_dynamic_boxes.launch.py
```

이미 Cartographer를 실행했다면 첫 번째 명령만 사용합니다.

## RViz

RViz에서 다음 Display를 추가합니다.

1. `MarkerArray`
   - Topic: `/dynamic_boxes`
2. `PointCloud2`
   - Topic: `/dynamic_clusters`
3. Fixed Frame
   - `odom` 또는 `map`

표시 의미:

- 하늘색 `OBS`: 일반 관측 클러스터
- 빨간색 `DYN`: 동적 후보
- 화살표: 추정 속도 방향과 크기
- `H`: 최근 프레임 시간 일관성

## 토픽

```text
/dynamic_boxes          visualization_msgs/MarkerArray
/dynamic_clusters       sensor_msgs/PointCloud2
/dynamic_object_count   std_msgs/Int32
```

## 주요 파라미터

`src/tb3_dynamic_detector/config/detector.yaml`

```yaml
dynamic_velocity_threshold: 0.15
static_support_threshold: 0.50
association_distance: 0.35
dynamic_confirmations: 3
confirmation_window: 4
box_padding: 0.08
```

- 벽이 빨간색으로 잡히면 속도 기준을 `0.20`으로 올립니다.
- 사람이 안 잡히면 속도 기준을 `0.10~0.12`로 내립니다.
- 박스 ID가 자주 바뀌면 연결 거리를 `0.45`로 올립니다.
- 박스가 서로 합쳐지면 클러스터 최대 간격을 `0.12~0.15`로 내립니다.

## 현재 제한

- 정지한 사람과 새 가구를 구분하지 못합니다.
- 벽과 사람이 붙어 있으면 하나의 클러스터가 될 수 있습니다.
- 박스는 `odom` 축에 정렬된 2D 박스입니다.
- `dynamic_boxes.launch.py` 단독 실행은 SLAM 점을 삭제하지 않습니다.
