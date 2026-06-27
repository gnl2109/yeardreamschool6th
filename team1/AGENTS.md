# AGENTS.md

## 목적

이 저장소는 `dataset`의 서울 지하철 승하차 데이터와 역 위치 데이터를 이용해 `지하철_승하차_혼잡도분석`을 수행하는 작업 공간이다. 모든 작업은 재현 가능한 Python 코드, 명확한 근거, 중요 상태/에러 로그를 남기는 방식으로 진행한다.

## 핵심 원칙

- 짧고 명확하게 답변한다.
- 공식 문서 기반으로만 작성한다. 라이브러리는 해당 공식 문서, 프로젝트 분석 원칙은 `명저마스터_요청사항.md`와 `reference` 자료를 우선한다.
- 불확실하면 `확인 필요`라고 명시한다.
- 모든 중요 상태/에러를 로깅한다. 로그에는 입력 파일, 처리 단계, 행 수, 컬럼 수, 결측치, 예외 메시지를 포함한다.
- 원본 CSV는 수정하지 않는다.
- 한국어로 답변한다.

## 저장소 구조

```text
C:\yeardreamschool6th\team1
├── AGENTS.md
├── 명저마스터_요청사항.md
├── dataset
│   ├── Seoul_subway_data_20210705.csv
│   └── subway_location_data.csv
└── reference
    ├── SCQA, 피라미드 원칙.pdf
    ├── 기초통계이론.pdf
    └── 시각화교안.pdf
```

## 데이터 프로파일

### `dataset/Seoul_subway_data_20210705.csv`

- 인코딩: `cp949`
- 크기: 45,338행 x 52열
- 기간: `201501`부터 `202106`까지 78개월
- 노선 수: 26개
- 역명 고유값: 579개
- 결측치: 확인 결과 없음
- 주요 컬럼: `사용월`, `호선명`, `지하철역`, 시간대별 승차인원 24개, 시간대별 하차인원 24개, `작업일자`
- 전체 합계: 총 승차 16,304,736,857명, 총 하차 16,238,320,584명
- 전체 유동량 상위 시간대: `08시-09시`, `18시-19시`, `19시-20시`, `17시-18시`, `09시-10시`
- 전체 유동량 상위 역/노선: `2호선 강남`, `2호선 홍대입구`, `2호선 신림`, `2호선 구로디지털단지`, `2호선 신도림`

### `dataset/subway_location_data.csv`

- 인코딩: `utf-8-sig`
- 크기: 579행 x 4열
- 역명 고유값: 525개
- 중복 역명: 54건
- 결측치: 확인 결과 없음
- 주요 컬럼: `지하철역`, `주소`, `x좌표`, `y좌표`
- 좌표 의미: 샘플상 `x좌표`는 위도, `y좌표`는 경도로 보인다. 원천 데이터 문서가 없으므로 `확인 필요`.

## 참고 자료 사용

- `명저마스터_요청사항.md`: 통계 분석의 상위 지침이다. Python 코드만 제공, 6단계 분석 프로세스, 효과크기 보고, SCQA/피라미드 보고서 구조를 요구한다.
- `reference/SCQA, 피라미드 원칙.pdf`: 보고서 구조와 SCQA 작성 시 참고한다.
- `reference/기초통계이론.pdf`: 통계 개념, 가설검정, 기술통계, 추론통계 설명 시 참고한다.
- `reference/시각화교안.pdf`: 차트 선택, 시각화 설계, 그래프 해석 시 참고한다.
- PDF의 구체 페이지나 원문 근거가 필요하면 작업 시점에 직접 확인한다. 확인하지 못한 페이지는 추정하지 말고 `확인 필요`라고 쓴다.

## 혼잡도 정의

현재 데이터만으로는 실제 열차 내 혼잡률을 계산할 수 없다. 차량 정원, 운행 횟수, 배차 간격, 승강장 면적 데이터가 없기 때문이다.

기본 분석에서는 혼잡도를 다음 대리 지표로 정의한다.

- 시간대별 유동량 = 승차인원 + 하차인원
- 역별 월간 유동량 = 모든 시간대 승차인원 + 하차인원 합계
- 출근 피크 = `07시-08시`, `08시-09시`, `09시-10시`
- 퇴근 피크 = `17시-18시`, `18시-19시`, `19시-20시`
- 혼잡도 지수 = 특정 역/시간대 유동량을 전체 또는 노선 내 최대값으로 나눈 값

실제 혼잡률 또는 열차 내부 혼잡을 주장해야 하면 추가 데이터가 필요하므로 `확인 필요`라고 명시한다.

## 데이터 결합 주의사항

- 승하차 데이터의 역명은 대체로 `강남`, `서울역`처럼 저장되어 있다.
- 위치 데이터의 역명은 대체로 `강남역`, `서울역`처럼 저장되어 있다.
- 원본 역명으로 단순 조인하면 현재 기준 정확 일치 매칭이 0건이다.
- 조인 전 역명 정규화와 충돌 검사가 필수다.

권장 정규화:

```python
def normalize_station_name(name: str) -> str:
    return str(name).strip().replace(" ", "").removesuffix("역")
```

주의: `서울역`, `삼각지역`, `신답역`처럼 실제 역명 자체에 `역`이 포함될 수 있다. 정규화 후 중복/충돌을 확인하고, 문제가 있으면 수동 매핑 테이블을 만든다.

## 필수 작업 흐름

1. 데이터 로드
   - `Seoul_subway_data_20210705.csv`: `encoding="cp949"`
   - `subway_location_data.csv`: `encoding="utf-8-sig"`
   - 파일 경로, 인코딩, 행 수, 컬럼 수를 로그로 남긴다.

2. 데이터 검증
   - 필수 컬럼 존재 여부를 확인한다.
   - 결측치, 중복, 음수 승하차 인원, 비정상 `사용월`을 확인한다.
   - 실패 시 분석을 중단하고 원인과 수정 방법을 명시한다.

3. 전처리
   - 시간대별 wide 컬럼을 long format으로 변환한다.
   - `사용월`을 월 단위 날짜형으로 변환한다.
   - 역명 정규화 컬럼을 만든다.
   - 위치 데이터 결합 시 매칭률과 미매칭 역 목록을 로그로 남긴다.

4. 탐색적 분석
   - 월별 총 승차/하차 추세
   - 노선별 총 유동량
   - 역별 총 유동량
   - 시간대별 유동량
   - 출근/퇴근 피크 비교
   - 위치 결합 후 좌표 기반 시각화

5. 통계 분석
   - 분석 질문을 한 문장으로 정의한다.
   - 귀무가설과 대립가설을 명시한다.
   - 변수 유형과 분석 방법을 명시한다.
   - 가정 검토를 수행한다.
   - p-value와 효과크기를 함께 보고한다.
   - 가정 위배 시 비모수 대안을 제시한다.

6. 보고서 작성
   - 결론을 먼저 제시한다.
   - `명저마스터_요청사항.md`의 SCQA/피라미드 템플릿을 따른다.
   - 통계량은 표로 정리한다.
   - 한계점과 `확인 필요` 항목을 분리한다.

## Python 구현 규칙

- Python만 사용한다.
- 주요 함수에는 타입힌트를 작성한다.
- 파일 경로는 `pathlib.Path`를 사용한다.
- 데이터 처리는 `pandas`, 수치 계산은 `numpy`, 통계 검정은 `scipy.stats` 또는 `statsmodels`를 우선한다.
- 머신러닝이 필요하면 `scikit-learn`을 사용한다.
- 시각화는 `matplotlib` 객체지향 API와 `seaborn`을 사용한다.
- `seaborn`은 `sns.set_theme()`과 `sns.set_context()`를 사용한다.
- 색상은 colorblind-friendly 팔레트를 우선한다.
- 새 라이브러리를 쓰기 전 설치 상태와 공식 문서를 확인한다.

기본 코드 골격:

```python
from __future__ import annotations

import logging
from pathlib import Path

import pandas as pd


LOGGER = logging.getLogger(__name__)


def setup_logging() -> None:
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
    )


def load_ridership(path: Path) -> pd.DataFrame:
    LOGGER.info("load_start file=%s encoding=cp949", path)
    df = pd.read_csv(path, encoding="cp949")
    LOGGER.info("load_success file=%s rows=%s cols=%s", path, len(df), len(df.columns))
    return df


def load_locations(path: Path) -> pd.DataFrame:
    LOGGER.info("load_start file=%s encoding=utf-8-sig", path)
    df = pd.read_csv(path, encoding="utf-8-sig")
    LOGGER.info("load_success file=%s rows=%s cols=%s", path, len(df), len(df.columns))
    return df
```

## 로깅 체크리스트

중요 상태는 `INFO` 이상으로 기록한다.

- 데이터 로드 시작/성공/실패
- 파일 경로와 인코딩
- 행 수와 컬럼 수
- 필수 컬럼 검증 결과
- 결측치 수
- 중복 수
- 역명 정규화 전후 매칭률
- 분석 대상 기간, 노선, 역, 시간대
- 생성한 결과 파일 경로

에러는 `ERROR` 또는 `LOGGER.exception(...)`으로 기록하고 예외 메시지를 포함한다.

```python
try:
    ridership = load_ridership(Path("dataset/Seoul_subway_data_20210705.csv"))
except Exception:
    LOGGER.exception("load_failed file=dataset/Seoul_subway_data_20210705.csv")
    raise
```

## 산출물 위치

산출물은 원본 데이터와 분리한다.

```text
outputs/
├── figures/
├── tables/
├── reports/
└── logs/
```

`outputs`가 없으면 필요한 하위 디렉터리를 명시적으로 생성한다.

## 금지 사항

- 근거 없는 수치 생성 금지
- 공식 문서나 제공 자료에 없는 내용을 확정적으로 말하기 금지
- `혼잡도`를 실제 열차 혼잡률로 단정 금지
- 역명 정규화 없이 위치 데이터 조인 금지
- p-value만으로 결론 내리기 금지
- 원본 CSV 수정 또는 덮어쓰기 금지
- 사용자 또는 다른 에이전트의 기존 변경사항 임의 되돌리기 금지

## 완료 전 체크리스트

- 원본 파일을 수정하지 않았는가?
- 파일 인코딩을 올바르게 지정했는가?
- 필수 컬럼, 결측치, 중복, 음수 값을 확인했는가?
- 역명 정규화 후 조인 매칭률을 확인했는가?
- 혼잡도 정의와 한계를 명시했는가?
- 가설, 변수 유형, 분석 방법을 명시했는가?
- 통계적 가정과 효과크기를 보고했는가?
- 시각화 축, 단위, 범례가 명확한가?
- 불확실한 항목에 `확인 필요`를 표시했는가?
- 주요 상태와 에러가 로그에 남는가?
