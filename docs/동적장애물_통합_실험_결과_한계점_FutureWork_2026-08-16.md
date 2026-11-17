# 동적 장애물 통합 실험 결과 정리 (2026-08-16)

이 문서는 `slam_dynamic_frontier_super_manager.launch.py` 기반 통합 실험(동적 탐지 + SLAM + Frontier 탐색 + Dijkstra 주행) 세션의 결과·한계점·Future Work를 정리한다. 관련 기록: [[실행 커맨드_2대_동시.txt]] Section 17, `PROJECT_HANDOFF_KO.md`.

## 1. 결과

- **`/map_static` 발행 지연 문제 해결**: 지도 마스킹을 `dynamic_detector.py`에서 `dynamic_map_filter.py`(별도 프로세스)로 분리. 기존 12.247초 정지 현상의 원인이 지도 콜백 지연이었음을 확인하고 제거. WSL 반영·colcon build·pytest(14개) 전체 통과.
- **하드웨어 원인 규명**: "RViz 지도/로봇 모델이 빙글빙글 도는" 현상의 실제 원인이 코드가 아니라 jshim OpenCR IMU 자이로 부팅 보정 오류임을 `tf2_echo`로 확정 (translation 고정, yaw만 무작위 요동). 재부팅으로 해결.
- **Nav2 lifecycle 복구 로직 개선**: `nav2_lifecycle_recovery.py`의 `map` 신선도 선행조건 제거. 실제 lifecycle 활성화 실패("spin action server not available") 상황에서 정상적으로 개입해 `bt_navigator`→`waypoint_follower`→`velocity_smoother`를 순차 복구하는 것을 실측으로 검증.
- **코너 반사 아티팩트 대응**: 실제 장애물이 없는 코너에 occupied 셀이 찍히는 현상 확인(멀티패스 반사로 추정) → `obstacle_max_range`/`raytrace_max_range`를 3.0/2.5 → 2.2/1.8로 축소해 원거리 반사 리턴 컷오프.
- **주행/탐색 파라미터 튜닝**: `max_vel_x` 0.12→0.18 m/s(회전속도 0.15rad/s는 벽 오검출 안전판으로 유지), `planner_frequency` 1.0→0.33Hz(목표 flapping 억제), `movement_time_allowance` 30→15초(90도 회전까지 커버 확인).
- **디버깅 가시성 개선**: `GLOG_minloglevel=2`로 Cartographer 로그 노이즈 제거, `debug_trajectory_details: False`로 DWB 오버헤드 제거.
- **기본 파이프라인 동작 확인**: 자율 SLAM + Frontier 탐색 + Dijkstra 주행이 소규모 테스트 공간에서 목표 도달 및 탐색 완료까지 실제로 동작함을 확인.

## 2. 한계점

| 제목 | 설명 |
|---|---|
| 성능 병목 미해결 | `planner_server` 루프가 목표 20Hz 대비 1.18Hz로 저하되는 현상을 확인했으나 근본 원인(WSL/CPU 부하 추정)은 미해결. 지금까지 튜닝한 Nav2 파라미터들은 이 병목 위에서 조정된 것 |
| 코너 반사 아티팩트 완화 수준 | range 축소로 원거리 반사 리턴만 줄였고, teleop 재스캔으로 실제 지워지는지 등 근본 검증은 하지 않음 |
| 회전 시간 허용치의 구조적 한계 | `movement_time_allowance=15초`는 ~129도까지만 커버하며, 180도 완전 유턴은 여전히 progress_checker 오탐 위험이 있음 |
| 벽 오검출 억제 로직의 전제 불일치 | `detector.yaml`은 정지 로봇 실험 기준 값인데 완전 자율주행(이동+회전) 시나리오에 그대로 사용 중 — 명시적 재현은 없었으나 잠재 리스크로 남음 |
| 검증 규모 제한 | 테스트 공간이 약 1.5~4.5m 수준으로 작아, 더 크거나 복잡한 환경에서의 확장성은 검증되지 않음 |
| 맵 저장 정렬 경고 미해결 | `map_snapshot_saver`의 "Map origin not aligned" 경고 원인 미규명, 실주행 영향은 없는 것으로 추정 |
| IMU 미활용 | Cartographer가 `use_imu_data=false`로 스캔매칭+오도메트리만 사용 중이라 회전 안정성 개선 여지가 남아있음 |
| 원격 진단 도구 신뢰성 문제 | 이 환경에서 `ros2 topic echo`가 메시지 내용 출력에 일관되게 실패(원인 미규명), 실시간 디버깅 효율을 떨어뜨림 |
| dynamic_map_filter 종료 크래시 원인 미규명 | 예외 흡수 코드만 추가했고, 종료 레이스의 근본 원인은 규명하지 않음 |

## 3. Future Work

| 제목 | 설명 |
|---|---|
| 성능 병목 원인 규명 | CPU 프로파일링, RViz 분리 실행 등으로 원인을 파악한 뒤 Nav2 파라미터를 재검토한다 |
| IMU 융합 실험 | `use_imu_data=true`로 전환해 회전 안정성 개선 여부를 검증한다 |
| 코너 반사 재현 실험 | 다각도 재스캔 또는 신뢰도 기반 필터링으로 반사 아티팩트를 체계적으로 검증한다 |
| 대규모 환경 확장 검증 | 더 크고 복잡한 실제 공간에서 탐색·주행 성능을 검증한다 |
| 장시간 통합 안정성 테스트 | tb1 동적 장애물을 포함한 풀 시나리오를 장시간 돌려 안정성을 확인한다 |
| 회전 특화 progress_checker | 회전량 기준으로 별도 판정하는 커스텀 로직을 검토해 시간 기준의 구조적 한계를 보완한다 |
| 맵 저장 경고 원인 규명 | `map_snapshot_saver`의 누적 정렬 경고에 대한 정확한 원인을 파악한다 |
| topic echo 실패 원인 조사 | `ros2 topic echo` 실패의 근본 원인을 규명하고 표준 진단 절차를 문서화한다 |
