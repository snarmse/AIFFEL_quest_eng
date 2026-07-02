# Wind Turbine Blade Defect Detection

> YOLOv8 기반 드론 영상 풍력 터빈 블레이드 결함 탐지 — 심각도 판정 + 에스컬레이션 알림 시스템

---

## 프로젝트 개요

드론으로 촬영한 풍력 터빈 블레이드 이미지에서 **Dirt(오염)** 와 **Damage(손상)** 를 탐지하고,
탐지 결과를 심각도 등급으로 판정해 관리자에게 차등 알림을 전달하는 예방 정비 시스템입니다.

```
드론 이미지 입력
  → YOLOv8s 탐지 (Dirt / Damage)
  → 심각도 점수 = 클래스가중치 × 크기가중치 × confidence
  → 등급 판정: 정상 / 관찰 / 주의 / 위험
  → 에스컬레이션: 로그 → 대시보드 → 이메일 → Slack 즉시 경보
```

---

## 최종 모델

```python
from ultralytics import YOLO

model = YOLO("runs/exp_full_small_ep50/weights/best.pt")

results = model.predict(
    source="./test_images/",
    conf=0.20,   # 소형 객체(Damage 92%가 64×64 이하) 대응
    save=True,
)
```

### 최종 성능 (val 2,694장)

| mAP50 | mAP50-95 | Precision | Recall | Damage Recall | Damage AP50 |
|---|---|---|---|---|---|
| 0.752 | 0.517 | 0.710 | **0.769** | 0.645 | **0.688** |

| 목표 | 목표값 | 달성값 |
|---|---|---|
| Recall | 0.70+ | **0.769 ✅** |
| Damage AP50 | 0.58+ | **0.688 ✅** |

---

## 데이터셋

- **출처**: [NordTank 586×371 — Kaggle](https://www.kaggle.com/datasets/ajifoster3/yolo-annotated-wind-turbines-586x371)
- **전체**: 13,470장 (결함 2,995장 / 정상 10,475장)
- **클래스**: 0=Dirt(581 boxes), 1=Damage(8,770 boxes) — 비율 1:15

### 외부 데이터 검토

| 데이터셋 | 결과 |
|---|---|
| Wind Turbine v1i (Roboflow) | NordTank와 100% 중복 → 미사용 |
| Wind Turbine v11i (Roboflow DTU) | NordTank와 100% 중복 → 미사용 |
| **Wind Turbine Blade v7i** | 완전 신규. crack(class 0) → Damage(class 1) 리맵핑 후 활용 |

---

## EDA 핵심 발견

1. **정상 이미지 과다**: 전체 77.8%가 정상 → 그대로 학습 시 배경 희석, Precision 붕괴
2. **클래스 불균형**: Dirt:Damage = 1:15 → Dirt 증강 전략 필요
3. **소형 객체**: Damage의 92%가 64×64픽셀 이하 → confidence threshold 하향 필요

---

## 데이터 전처리

| 처리 | 방법 | 근거 |
|---|---|---|
| 정상 이미지 활용 | 빈 라벨(.txt)로 background image 처리 | YOLO 빈 라벨 = FP 억제 학습에 활용 |
| Dirt 증강 | ×8 (flip, brightness, contrast, saturation) | 581 → 4,648개로 불균형 완화 |
| 외부 데이터 편입 | blade v7i — train 1,474장만 사용 | valid/test는 도메인 혼재 방지로 제외 |

---

## 모델 선정

4종 비교 실험 (dataset_yolo labeled only, 20 epoch):

| 모델 | mAP50 | Recall | Precision |
|---|---|---|---|
| YOLOv8n | 0.794 | 0.747 | 0.818 |
| **YOLOv8s ← 선정** | **0.818** | 0.750 | **0.854** |
| YOLO11n | 0.788 | 0.741 | 0.812 |
| YOLO11s | 0.799 | 0.746 | 0.851 |

**선정 이유**: mAP50 최고 + 드론 탑재 가능한 경량 (11M params, 22MB)

---

## 평가 지표 설계

| 순위 | 지표 | 근거 |
|---|---|---|
| **1** | **Recall** | FN(미탐) = 블레이드 파손 → 사고 직결. FP는 재확인 플래그로 보완 |
| **2** | **Damage AP50** | 서비스 심각도에서 Damage 가중치 10 (Dirt의 5배) |
| **3** | mAP50 | Dirt/Damage 동등 평균 → 서비스 가치와 괴리. 참고 지표 |

> **목표**: Recall 0.70+, Damage AP50 0.58+ 동시 달성

---

## 실험 결과

### Phase 1 — 데이터셋 비교 (YOLOv8s, 50 epoch, full val 2,694장)

| 실험 | Precision | Recall | mAP50 | mAP50-95 | Damage AP50 |
|---|---|---|---|---|---|
| **Baseline (full)** | 0.710 | **0.769** | **0.752** | 0.517 | **0.688** |
| + Dirt×8 | 0.674 | 0.751 | 0.718 | 0.515 | 0.684 |
| + Blade | 0.710 | 0.763 | 0.748 | 0.517 | 0.688 |
| + Dirt×8 + Blade | 0.645 | 0.735 | 0.726 | **0.525** | 0.682 |

> 4개 full 변형은 동일 val 2,694장 → 공정 비교. **베이스라인(full dataset)이 전 지표 최고 또는 동등**

### conf 파라미터 최적화

| conf | mAP50 | Recall | Damage AP50 |
|---|---|---|---|
| 0.25 | 0.7016 | 0.7691 | 0.6045 |
| **0.20 ← 선정** | **0.7137** | 0.7691 | **0.6287** |

> `model.val()`은 threshold 내부 sweep → Recall 불변. mAP 개선으로 conf 0.20~0.25 구간 TP 존재 확인

---

## 전체 실험 목록

### 모델 선정 (labeled-only, 20 epoch)

| 실험명 | 모델 | mAP50 | Recall | Precision |
|---|---|---|---|---|
| exp_v8n_ep20 | YOLOv8n | 0.794 | 0.747 | 0.818 |
| **exp_v8s_ep20 ← 선정** | **YOLOv8s** | **0.818** | 0.750 | **0.854** |
| exp_v11n_ep20 | YOLO11n | 0.788 | 0.741 | 0.812 |
| exp_v11s_ep20 | YOLO11s | 0.799 | 0.746 | 0.851 |

### 데이터셋 비교 (YOLOv8s, 50 epoch, full val 2,694장)

| 실험명 | 데이터셋 | mAP50 | Recall | Damage AP50 |
|---|---|---|---|---|
| **exp_full_small_ep50 ← 최종** | dataset_yolo_full | **0.752** | **0.769** | **0.688** |
| exp_full_aug_small_ep50 | dataset_yolo_full_aug | 0.718 | 0.751 | 0.684 |
| exp_full_blade_ep50 | dataset_yolo_full_blade | 0.748 | 0.763 | 0.688 |
| exp_full_aug_blade_ep50 | dataset_yolo_full_aug_blade | 0.726 | 0.735 | 0.682 |

모든 가중치 경로: `runs/<실험명>/weights/best.pt`

---

## 서비스 설계

### 심각도 스코어링

```python
# 객체 점수 = 클래스가중치 × 크기가중치 × confidence
# 터빈 위험도 = Σ(각 탐지 객체 점수)
class_weight = {"Damage": 10, "Dirt": 2}
size_weight = 1 + (bbox_area_ratio * 5)
```

### 등급 및 에스컬레이션

| 점수 | 등급 | 알림 채널 | 알림 대상 |
|---|---|---|---|
| 0 | 🟢 정상 | 로그 기록 | — |
| 1~15 | 🟡 관찰 | 대시보드 | 현장 담당자 |
| 15~40 | 🟠 주의 | 대시보드 + 이메일 | 담당자 + 팀 |
| 40+ | 🔴 위험 | 대시보드 + Slack 즉시 경보 | 담당자 + 관리자 |

### 오탐 관리 (Human-in-the-loop)

| confidence | 처리 |
|---|---|
| ≥ 0.7 | 자동 경보 |
| 0.4 ~ 0.7 | ⚠️ 재확인 필요 플래그 |
| < 0.4 | 무시 |

---

## 프로젝트 구조

```
Hackathon/
├── project.ipynb                      # 메인 노트북 (EDA → 전처리 → 학습 → 평가)
├── outputs/                           # EDA 시각화 + 예측 결과 이미지
├── runs/
│   ├── exp_v8n_ep20/                  # 모델 선정 비교 실험
│   ├── exp_v8s_ep20/
│   ├── exp_v11n_ep20/
│   ├── exp_v11s_ep20/
│   ├── exp_full_small_ep50/           # ★ 최종 모델
│   │   └── weights/best.pt
│   ├── exp_full_aug_small_ep50/       # 데이터셋 비교 실험
│   ├── exp_full_blade_ep50/
│   └── exp_full_aug_blade_ep50/
└── docs/
    ├── 00_인수인계.md
    ├── 01_실험요약.md
    ├── 02_의사결정기록.md
    ├── 03_루브릭평가자료.md
    ├── 04_서비스데모설계.md
    ├── 05_발표구성안.md
    ├── 06_발표용_수치정리.md
    ├── 07_프로젝트_리포트.md
    ├── 08_나의기여정리.md
    ├── 09_발표스크립트_채진현.md
    ├── 나의터빈일지_발표자료.pptx
    └── figures/
        ├── eda/                       # EDA 시각화
        ├── dataset_comparison/        # 데이터셋 4종 비교 차트
        ├── model_selection/           # 모델 4종 비교 (20 epoch)
        └── final_model/               # 최종 모델 결과 (50 epoch)
```

> `data/` 폴더(원본 데이터셋)와 `*.pt` 모델 가중치는 용량 문제로 제외.

---

## 환경

```
Python      3.12
PyTorch     2.12 + CUDA 13.0
Ultralytics 8.4.83
GPU         NVIDIA TITAN RTX 24GB
```

---

## 주요 기술 결정 사항

| 결정 | 내용 |
|---|---|
| 절대경로 사용 | ultralytics가 상대경로에 `runs/detect/` prefix 자동 추가 → `Path("./runs").resolve()` 사용 |
| conf=0.20 | Damage 92%가 소형 객체. 해상도 상향보다 conf 하향이 효과적 |
| val() vs predict() | `val()`은 threshold 내부 sweep → conf 효과는 `predict()`에서만 반영 |
| blade val/test 제외 | 도메인(384×384, 다른 터빈) 혼재 → 평가 일관성 훼손 방지 |
| 정상 이미지 전체 활용 | 빈 라벨 = YOLO FP 억제 학습 → Precision 안정화 |
