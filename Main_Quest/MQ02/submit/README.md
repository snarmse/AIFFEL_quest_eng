# 나의 터빈일지 — 풍력 터빈 블레이드 결함 탐지

> YOLOv8 기반 드론 영상 풍력 터빈 블레이드 결함 탐지 — 심각도 판정 + 에스컬레이션 알림 시스템

---

## 브랜치 구성

| 브랜치 | 내용 |
|--------|------|
| [`model`](https://github.com/Jinn62/Hackathon/tree/model) | 모델 실험 — EDA · 전처리 · YOLOv8s 학습 · conf 최적화 · 실험 문서 |
| [`demo`](https://github.com/Jinn62/Hackathon/tree/demo) | 서비스 데모 — Streamlit 대시보드 · 위험도 스코어링 · 알림 시스템 |

---

## 프로젝트 소개

드론으로 촬영한 풍력 터빈 블레이드 이미지에서 **Dirt(오염)** 와 **Damage(손상)** 를 자동 탐지하고,
심각도를 등급으로 판정해 관리자에게 차등 알림을 전달하는 예방 정비 시스템입니다.

```
드론 이미지 입력
  → YOLOv8s 탐지 (Dirt / Damage)
  → 심각도 점수 = 클래스가중치 × 크기가중치 × confidence
  → 등급 판정: 정상 / 관찰 / 주의 / 위험
  → 에스컬레이션: 로그 → 대시보드 → 이메일 → ntfy 즉시 푸시
```

## 최종 성능

| 지표 | 목표 | 달성 |
|------|------|------|
| Recall | ≥ 0.70 | **0.769** ✅ |
| Damage AP50 | ≥ 0.58 | **0.688** ✅ |

- 모델: YOLOv8s / conf: 0.20 / 학습: full dataset, 50 epoch

---

## 1. EDA

![EDA 개요](docs/figures/rubric/01_eda_overview.png)
![샘플 이미지](docs/figures/rubric/01_eda_sample_images.png)
![정상·결함 비율](docs/figures/rubric/01_eda_normal_defect_ratio.png)
![클래스 분포](docs/figures/rubric/01_eda_class_distribution.png)
![bbox 크기 분포](docs/figures/rubric/01_eda_bbox_size.png)
![클래스 불균형](docs/figures/rubric/01_eda_imbalance.png)

---

## 2. 모델 선정

![모델 비교](docs/figures/rubric/02_model_comparison.png)
![YOLOv8s 학습 곡선](docs/figures/rubric/02_model_v8s_results.png)
![YOLOv8s 혼동 행렬](docs/figures/rubric/02_model_v8s_confusion.png)

---

## 3. 성능 향상 실험

![데이터셋 비교 (그룹 막대)](docs/figures/rubric/03_experiment_grouped_bar.png)
![데이터셋 비교 (레이더)](docs/figures/rubric/03_experiment_radar.png)
![전체 비교](docs/figures/rubric/03_experiment_full_comparison.png)

---

## 4. 지표 설계

![지표 우선순위](docs/figures/rubric/04_metric_priority.png)

---

## 5. 최종 모델 결과

![학습 곡선](docs/figures/rubric/05_final_results.png)
![혼동 행렬](docs/figures/rubric/05_final_confusion.png)
![PR 곡선](docs/figures/rubric/05_final_pr_curve.png)
![F1 곡선](docs/figures/rubric/05_final_f1_curve.png)
![conf 최적화](docs/figures/rubric/05_final_conf_optimization.png)
![val 예측 결과](docs/figures/rubric/05_final_val_pred.jpg)
![베이스라인 예측](docs/figures/rubric/05_final_baseline_pred.png)

---

## 6. 배포 설계

서비스 코드(Streamlit 대시보드 · 위험도 스코어링 · 알림 시스템)는 [`demo` 브랜치](https://github.com/Jinn62/Hackathon/tree/demo)에서 확인할 수 있습니다.

### 심각도 스코어링

```python
# 객체 점수 = 클래스가중치 × 크기가중치 × confidence
# 클래스가중치: Damage=10, Dirt=2
# 크기가중치: 1 + (박스면적비율 × 5)
터빈_위험도 = Σ(각 탐지 객체 점수)
```

### 등급별 에스컬레이션

| 점수 | 등급 | 알림 채널 | 대상 |
|------|------|-----------|------|
| 0 | 🟢 정상 | 로그 기록 | — |
| 1~15 | 🟡 관찰 | 대시보드 | 현장 담당자 |
| 15~40 | 🟠 주의 | 대시보드 + 이메일 | 담당자 + 팀 |
| 40+ | 🔴 위험 | 대시보드 + ntfy 즉시 푸시 | 담당자 + 관리자 |

### conf 구간별 처리

| confidence | 처리 |
|------------|------|
| ≥ 0.7 | 자동 경보 |
| 0.4 ~ 0.7 | ⚠️ 재확인 필요 (Human-in-the-loop) |
| < 0.4 | 무시 |
