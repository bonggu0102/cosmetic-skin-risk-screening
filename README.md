# 신제품 화장품의 피부유형별 부정 경험 위험 사전 선별

리뷰가 없는 신제품도 전성분·제품명·피부유형 정보만으로 부정 경험 위험을 미리 선별할 수 있는지 검토한 4인 팀 머신러닝 프로젝트입니다. 과거 제품의 리뷰로 학습용 라벨을 만들되, 신제품 예측에는 출시 전에 확인 가능한 정보만 사용했습니다.

![프로젝트 최종 포스터](reports/poster/poster_final.png)

## 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 데이터 | 올리브영 크림 231개 상품, 리뷰 47,487건 |
| 최종 분석 대상 | 206개 상품, 7개 피부유형 |
| 분석 단위 | 1,124개 `제품 × 피부유형` 조합 |
| 예측 목표 | 피부유형별 부정 경험 고위험군 선별 |
| 입력 변수 | 피부유형, 전성분, 성분 기능·조합, 제품명 테마 |
| 최종 모델 | V2 Feature Set + Random Forest |
| Test 성능 | PR-AUC 0.384, Recall 0.631 |

## 담당 영역

- **데이터 수집:** 올리브영 크림 상품의 상품명, 전성분, 피부타입·피부톤별 리뷰 수집
- **리뷰 감성분석:** Gemini API를 활용한 긍정·부정·애매 분류 및 배송·포장 관련 리뷰 제외
- **모델링:** V1·V2 Feature Set과 4개 분류 모델을 비교하고, 교차검증 결과를 바탕으로 최종 모델을 선정했습니다.

데이터 전처리와 Feature Engineering을 포함한 전체 분석 과정은 팀 협업으로 진행했습니다.

## 분석 흐름

```text
상품·리뷰 수집
  → 전성분 정규화
  → 리뷰 감성분석 및 고위험군 라벨 생성
  → Feature Engineering V1~V5
  → 모델 비교 및 교차검증
  → 독립 Test 평가
```

## 타깃 생성

1. 배송·포장 관련 리뷰를 제외하고 `제품 × 피부유형`별 긍정·부정 리뷰를 집계했습니다.
2. 리뷰가 10건 이상인 조합에 베이지안 평활화를 적용해 표본 수 차이의 영향을 줄였습니다.
3. 평활화된 부정 경험 비율이 상위 25%인 조합을 고위험군으로 정의했습니다.

여기서 고위험군은 임상적 부작용이 아니라, 수집된 리뷰에서 상대적으로 부정 경험 비율이 높은 조합을 의미합니다.

## Feature Engineering

| 구분 | 특징 | 생성 방식 | 특징 수 |
|---|---|---|---:|
| 공통 | 피부유형 | 7개 피부유형 원-핫 인코딩 | 7 |
| V1 | 전체 전성분 | 정규화된 모든 성분을 멀티핫 인코딩 | 1,292 |
| V2 | 빈도 필터 성분 | 8개 이상 상품에 등장하고 전체 상품의 80% 이하에 등장한 성분만 사용 | 248 |
| V3 | 성분 수·기능군 | 보습·진정·장벽 등 기능군별 성분 수 | 13 |
| V4 | 피부유형별 성분 조합 | 건성의 보습·장벽 구성, 민감성의 향료·산 성분 부담 등 문헌 기반 규칙을 점수화 | 32 |
| V5 | 제품명 테마 | 제품명에서 강조한 보습·진정·장벽 등의 주제 | 8 |

두 가지 최종 Feature Set을 구성해 비교했습니다.

- **V1 Feature Set (1,352개):** 피부유형 + V1 + V3 + V4 + V5
- **V2 Feature Set (308개):** 피부유형 + V2 + V3 + V4 + V5

V2는 지나치게 희귀하거나 대부분의 상품에 공통으로 등장하는 성분을 제외한 축소 버전입니다. 자세한 변수 생성 과정은 [Feature Engineering 문서](docs/feature_engineering.md)와 `notebooks/03_feature_engineering`에서 확인할 수 있습니다.

## 검증 방법

- 같은 성분 구성을 가진 데이터가 Train과 Test에 함께 들어가지 않도록 `formula_group`을 기준으로 분리했습니다.
- Train 내부에서 5-fold Group Cross-Validation으로 Feature Set과 모델을 비교했습니다.
- 분류 임계값은 Test 결과가 아니라 Train OOF 예측으로 정한 뒤, 독립 Test에서 최종 성능을 평가했습니다.

Train은 869건·140개 처방 그룹, Test는 255건·36개 처방 그룹으로 구성했습니다.

## 모델링 결과

### 교차검증 PR-AUC

| 모델 | V1 | V2 |
|---|---:|---:|
| Logistic Regression | 0.318 | 0.322 |
| Random Forest | 0.331 | **0.352** |
| XGBoost | 0.322 | 0.336 |
| CatBoost | 0.338 | 0.329 |

평균 CV PR-AUC가 가장 높은 `V2 + Random Forest`를 최종 모델로 선택했습니다.

### 독립 Test 성능

| 지표 | 결과 |
|---|---:|
| Accuracy | 0.596 |
| Precision | 0.342 |
| Recall | **0.631** |
| F1-score | 0.443 |
| ROC-AUC | 0.623 |
| PR-AUC | **0.384** |

실제 고위험 조합 65개 중 41개를 탐지했습니다. Test의 고위험군 비율 25.5%보다 PR-AUC가 높았으며, 전체 정확도보다는 고위험 후보를 놓치지 않는 데 초점을 둔 결과입니다.

제품·피부유형별 예측값은 [Test 결과](reports/metrics/테스트_예측결과.csv)에서 확인할 수 있습니다.

## 향후 활용 방안

- **신제품 사전 검토:** 출시 예정 제품의 전성분과 제품명을 입력해 피부유형별 부정 경험 위험도를 비교하고, 우선적으로 검토할 피부유형과 제품 후보를 선별하는 데 활용할 수 있습니다.
- **출시 후 검증 및 모델 개선:** 제품 출시 후 실제 리뷰를 수집해 예측 결과와 비교하고, 축적된 데이터를 활용해 모델을 주기적으로 재학습할 수 있습니다.
- **적용 범위 확대:** 크림에 한정된 현재 분석을 토너·세럼·선케어 등 다른 제품군으로 확장하고, 제품군별 특성을 반영한 위험 예측 모델로 발전시킬 수 있습니다.
- **대시보드 구현:** 전성분과 제품명을 입력하면 피부유형별 예측 위험도와 검토 우선순위를 보여주는 제품 기획·연구개발용 도구로 구현할 수 있습니다.

장기적으로는 출시 전 위험 예측과 출시 후 리뷰 검증을 연결해, 신제품 기획 단계의 검토를 지원하는 시스템으로 발전시키는 것을 목표로 합니다.

## 한계

- 올리브영 크림 카테고리와 255건의 Test 데이터에 기반한 결과이므로 다른 플랫폼·제품군에는 추가 검증이 필요합니다.
- 고위험군은 임상 부작용이 아니라 Gemini 리뷰 감성과 부정 리뷰 비율로 만든 상대적 라벨입니다.
- Recall을 우선한 모델로 Precision은 34.2%이며, 자동 판정보다는 검토 우선순위를 정하는 보조 도구에 적합합니다.

## 저장소 구조

```text
├── notebooks/
│   ├── 01_data_collection/       # 상품·리뷰 수집
│   ├── 02_preprocessing/         # 전성분 정규화
│   ├── 03_feature_engineering/   # V1~V5 및 최종 결합
│   ├── 04_sentiment/             # Gemini 감성분석
│   └── 05_modeling/              # 최종 모델링
├── data/
│   ├── raw/                      # 크롤링 원본 데이터
│   ├── interim/                  # 전처리·Feature Engineering 결과
│   └── processed/                # 최종 모델 입력 및 감성분석 결과
├── reports/                      # Test 결과 및 프로젝트 포스터
├── experiments/                  # 팀원의 전면 재설계 모델링 실험
└── docs/                         # 데이터·Feature Engineering 설명
```

## 실행 방법

### Jupyter Notebook

```bash
git clone https://github.com/bonggu0102/cosmetic-skin-risk-screening.git
cd cosmetic-skin-risk-screening
pip install -r requirements.txt
jupyter notebook
```

### Google Colab

```python
!git clone https://github.com/bonggu0102/cosmetic-skin-risk-screening.git
%cd cosmetic-skin-risk-screening
!pip install -r requirements.txt
```

모든 노트북은 저장소 루트를 기준으로 한 상대경로를 사용합니다. Feature Engineering과 모델링은 저장된 데이터로 실행할 수 있어 크롤링과 Gemini 감성분석을 다시 수행할 필요가 없습니다.

크롤링·외부 전처리·Gemini API 재실행은 기본적으로 꺼져 있습니다. 크롤링은 `RUN_LIVE_CRAWLING=True`, Gemini 재분석은 `RUN_GEMINI_API=True`로 변경해 실행하며, Gemini API 키와 사용 비용은 실행자에게 발생합니다.

## 노트북 실행 순서

1. `notebooks/01_data_collection/01_crawl_oliveyoung.ipynb`
2. `notebooks/02_preprocessing/02_normalize_ingredients.ipynb`
3. `notebooks/03_feature_engineering/03_fe_v1_all_ingredients.ipynb`
4. `notebooks/03_feature_engineering/04_fe_v2_selected_ingredients.ipynb`
5. `notebooks/03_feature_engineering/05_fe_v3_function_counts.ipynb`
6. `notebooks/03_feature_engineering/06_fe_v4_literature_rules.ipynb`
7. `notebooks/03_feature_engineering/07_fe_v5_product_themes.ipynb`
8. `notebooks/03_feature_engineering/08_merge_features.ipynb`
9. `notebooks/04_sentiment/09_gemini_sentiment_analysis.ipynb`
10. `notebooks/05_modeling/10_train_and_evaluate.ipynb`

저장된 데이터로 핵심 분석만 재현하려면 3번부터 실행하면 됩니다. 팀원의 전면 재설계 실험은 `experiments/modeling_alternatives/high_risk_prediction_redesign.ipynb`에 별도로 정리했습니다.
