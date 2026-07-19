# CMDB 조인 분석 — 청구비용 × 조직정보 결합으로 조직·서비스 단위 비용 집계

> **한 문장 요약**: 청구서에는 "무엇에 얼마"는 있지만 "누구(어느 조직)의 비용인가"는 없음 →  
> 조직정보(CMDB)를 **태그를 join key로** 결합(enrich)해야 조직·서비스 단위로 비용을 집계·귀속 가능함.  
> **핵심 결론**: 조인 키 = **CostCenter + org 폴백**, 조인 도구 = **Power BI(Power Query 병합)**,  
> 릴리스 게이트 = **총액 무손실 + 커버리지% 표면화**.

---

## 0. 개요

### 0.1 왜 필요한가 (WHY)

- Azure 청구 데이터(Cost Management Export)는 **리소스·서비스·구독** 단위로 제공되며,  
  "이 비용이 **어느 팀·사업부(BU)·책임자**의 것인가"라는 **조직 축**은 청구서 자체에 없음.  
- 조직 축은 리소스에 붙인 **태그**(예: `CostCenter`, `org`)에 담기지만, 태그 값은 **코드**(예: `1234`, `SubACM`)라  
  사람이 읽는 조직명(마케팅본부)으로 바로 안 보이고, 조직 개편 이력·책임자도 태그엔 없음.  
- 그래서 **조직 마스터(CMDB)** 를 별도로 두고, **청구 데이터와 태그를 매개로 결합**해 각 청구 행에 조직 속성을  
  부착(enrich)함. 이것이 FinOps **Inform - Allocation(배분)** 역량의 핵심 실습임.

### 0.2 무엇을 하는가 (WHAT)

| 항목 | 내용 |
|---|---|
| 결합 대상 | **청구비용**(FOCUS export의 `BilledCost`) + **조직정보 CMDB**(코드→팀·BU·책임자 매핑표) |
| join key | **태그 `CostCenter`** 우선, 없으면 **태그 `org`** 로 **폴백** |
| 집계 축 | **조직 단위**(BU) · **조직 × 서비스**(BU × ServiceCategory) |
| 조인 도구 | **Power BI Desktop — Power Query 병합(Merge Queries)** |
| 릴리스 게이트 | **조인 전후 총액 무손실**(DIFF=0) + **커버리지%(배분율) 표면화** |

### 0.3 대상·전제

- **대상**: 조직·서비스 단위 비용을 스스로 집계·조회해야 하는 FinOps 실무자·엔지니어·재무 담당자.  
- **전제**: ① Cost Management **Export 데이터**(FOCUS/Actual) 확보 ② **Power BI Desktop** 설치  
  ③ 조직 마스터(CMDB) 매핑표 준비 가능.  
- **데이터 전제**: 당월 비용은 미확정(변동). 확정 수치는 마감 후 조회가 정확함.

### 0.4 실습 데이터·산출물 (팩트 기준)

- **실습 데이터**: `admin/dataset-examples/EA-Cost-FOCUS_1.0.csv` — FOCUS 1.0 스키마 EA export,  
  **135,272행**, 통화 전액 **USD**, 총 **BilledCost 35,445.86 USD**.  
- **산출물**: 본 가이드 + 실습용 샘플 CMDB(`work/cmdb-costcenter-master.csv` · `work/cmdb-org-fallback.csv`)  
  + 조인·총액 무손실 **검증 결과**(3.7).

### 0.5 4단계 파이프라인 한눈에

| 단계 | 무엇을 | 담당 | 절 |
|---|---|---|---|
| ① 준비 | 청구 데이터 확보 · 조인 키 이해 · CMDB 설계 · **정규화 선행** | 애리 | 2 |
| ② 조인 | Tags 파싱 → CostCenter/org 추출 → **병합(Merge)** 1·2차 | 피비 | 3.1 ~ 3.5 |
| ③ 집계 | 조직 단위(BU) · 조직 × 서비스 측정값 | 피비 | 3.6 |
| ④ 검증 | **커버리지% KPI + 총액 무손실**(릴리스 게이트) · 대시보드 | 피비 | 3.7 ~ 3.8 |

---

## 1. 개념 — CMDB 조인이란, 왜 포털만으로는 안 되는가

### 1.1 CMDB 조인의 원리

- **CMDB(Configuration Management Database)**: 조직의 구성·자산 마스터. 본 실습에서는 **"코드 → 조직"**  
  (CostCenter 코드 → 팀·BU·책임자) **매핑표** 역할로 좁혀 사용함.  
- **조인(Join)**: 청구 데이터(왼쪽)의 각 행에서 **태그 값**을 꺼내, 같은 값을 키로 하는 **CMDB 행**(오른쪽)의  
  조직 속성을 **부착(enrich)** 함. 결과적으로 "이 청구 비용 = 어느 BU·책임자" 가 각 행에 붙음.  
- 부착 후 **조직 축으로 집계**하면 조직 단위 Showback(현황 조회)·Chargeback(정산)이 가능해짐.

### 1.2 왜 Azure 포털 비용 분석만으로는 부족한가

포털 비용 분석은 **이미 리소스에 붙은 태그로 "그룹화"** 만 가능함. 아래는 **외부 조인이 필요한 이유**임.

| 필요 기능 | 포털 태그 그룹화 | 외부 조인(Power BI) |
|---|---|---|
| 코드 → 조직명(팀·BU·책임자) 변환 | ❌ 코드 그대로 표시 | ✅ CMDB 조인으로 조직명 부착 |
| 조인 키 폴백(CostCenter 없으면 org) | ❌ 단일 태그 그룹화 | ✅ 다단계 병합 |
| 키 정규화(대소문자·공백 통일) | ❌ 값 그대로 | ✅ 변환 단계에서 표준화 |
| 커버리지%·미배분 KPI 자동화 | △ 수동 확인 | ✅ 측정값·시각화로 상시 |
| 조직 개편 시 매핑표만 수정 | ❌ 태그 재작업 | ✅ CMDB 표 교체 |

> **핵심**: 포털은 "지금 붙어 있는 태그"의 조망용임. **외부 조직 마스터와의 결합·enrich**는 포털 범위 밖 →  
> **Power BI(Power Query 병합)** 가 필요함.

### 1.3 태그의 구조적 한계 (조인 전에 반드시 인지)

- 태그는 **사람이 붙임** → 깜빡·오타로 **미태깅**이 생겨 **커버리지 100% 불가**임.  
- 태그가 못 잡는 3종: ① **미태깅 리소스** ② **과거(소급) 비용**(태그는 소급 적용 안 됨) ③ **예약·Savings Plan·  
  마켓플레이스 구매**(태그 미부착).  
- 따라서 조인 결과의 **미배분(Untagged) 버킷을 숨기지 말고 항상 표면에 표시**하고, 못 잡는 비용은 **배분 규칙**으로  
  따로 채워야 함. (상세: [`관리그룹_청구계층_태그 가이드.md`](../hands-on/references/관리그룹_청구계층_태그%20가이드.md))

---

## 2. 데이터 준비 (담당: 애리)

### 2.1 청구 데이터 확보 — 어떤 export를 쓰나

Cost Management의 Export 기능은 청구 데이터를 스토리지 계정으로 정기 반출하는 기능임.  
데이터 유형은 크게 3가지임.

| Export 유형 | 특징 | 이 실습 |
|---|---|---|
| FOCUS Cost | FinOps Foundation 표준 스키마(FOCUS 1.0), 계약 유형과 무관하게 컬럼명이 통일됨 | 사용 |
| Actual cost | 실제 청구서 기준 원가(상각 전) | 미사용 |
| Amortized cost | 약정 구매분을 사용월에 상각해 배분 | 미사용(FOCUS의 EffectiveCost로 대체 확인) |

이 실습은 EA 계약의 FOCUS 1.0 export를 사용함. 파일 규모는 총 135,272행이며, 통화는 전액 USD임.  
BilledCost 합계는 35,445.86 USD, EffectiveCost(상각) 합계는 36,040.60 USD임.

조인에 필요한 핵심 컬럼은 다음과 같음.

| 컬럼 | 의미 | 조인 관련성 |
|---|---|---|
| Tags | 리소스 태그를 JSON 문자열로 담음(예: `{"CostCenter":"1234","org":"trey",...}`) | CMDB 조인 키(CostCenter·org)가 이 안에 있음 |
| SubAccountId / SubAccountName | 구독 ID·이름 | 구독 단위 참고용 |
| ServiceCategory / ServiceName | 서비스 대분류·이름 | 서비스별 배분 참고용 |
| x_CostCenter | **청구 계층**의 비용센터(태그 CostCenter와 별개) | 조인 키 아님(2.2 참고) |
| x_ResourceGroupName | 리소스그룹명 | 미태깅 원인 추적용 |
| BilledCost / EffectiveCost | 청구 금액 / 상각 금액 | 조인 후 집계할 측정값 |

**BilledCost vs EffectiveCost**: BilledCost는 청구서에 찍히는 금액(약정 구매 시점 그대로)이고,  
EffectiveCost는 약정 구매분을 사용 기간에 걸쳐 상각한 금액임. 조직별 배분·조인 분석은 실제 소비  
시점을 반영하는 EffectiveCost를 기준으로 삼는 것이 일반적임. 다만 본 실습의 조인·검증 수치는  
재현·대조 편의상 **BilledCost 기준**으로 산출함 — 실무 조직 배분 적용 시에는 EffectiveCost 기준으로  
동일 절차를 재수행하는 것을 권장함(3.6의 `실질비용` 측정값 활용).

> 실무 팁: 포털 Cost analysis의 Tag 그룹화(`cost-dashboard-analyze.md` 2.6)는 태그 1개 값별  
> 집계까지만 지원함. CMDB 같은 **외부 매핑표와의 조인은 포털에서 불가**하며 별도 도구(Power BI)가  
> 필요함 — 이번 실습에서 데이터 준비(본 섹션)와 조인(Power BI, 피비 담당)을 분리한 이유임.

> 함정: FOCUS export는 계약 유형(EA/MCA)에 따라 공급자 확장 컬럼(`x_` 접두)의 존재 여부가 다를  
> 수 있음. 이 실습의 `x_CostCenter`·`x_ResourceGroupName`은 EA export 기준이며, MCA에서는  
> 컬럼 구성이 다를 수 있으므로 실제 export 스키마를 먼저 확인할 것.

> 📷 (스크린샷 위치: Cost Management + Billing > Exports 목록에서 FOCUS 1.0 export 설정 화면)

### 2.2 조인 키 이해 — CostCenter + org 폴백

가장 흔한 실수는 **태그 CostCenter**와 **x_CostCenter**(청구 계층의 비용센터)를 같은 값으로  
착각하는 것임. 이 데이터셋에서 둘은 완전히 별개 체계임.

| 구분 | 무엇인가 | 예시 값 | 담당(태그 가이드 기준) |
|---|---|---|---|
| 태그 `CostCenter` | 사람이 리소스에 직접 붙인 값 | `SubACM`, `1234`, `ACM-PM` | 태그 — 세부 분류 담당 |
| `x_CostCenter` | 청구 계층에 이미 존재하는 비용센터 | `ACM9000` / `acm9000` 등 | 청구 계층 — 정산 담당 |

두 값의 알파벳 구성이 비슷해 보여도(`SubACM` vs `ACM9000`) 관리 주체가 다른 별개 코드 체계이므로  
혼용하면 안 됨. CMDB 조인은 **태그 CostCenter** 기준으로 수행함.

**왜 org 폴백이 필요한가**: 태그 CostCenter 보유 행은 전체의 31.45%(11,147.45 USD)뿐임 —  
SubACM 19.05% · 1234 12.39% · ACM-PM 0.01%로 구성됨. 나머지 68.55%는 CostCenter 태그가  
없는 경우로, Tags 자체가 완전히 비어있는 1,956행(20,966.08 USD, 59.15%)과 Tags는 있으나  
CostCenter 키가 없는 3,332.33 USD(9.40%)로 나뉨. 이대로면 CMDB 매핑 커버리지가 31.45%에  
그침. 태그 `org`(예: `trey`, 20.89%)를 2차 조인 키로 폴백하면 CostCenter가 비어도 조직 일부를  
복구할 수 있어 커버리지가 늘어남. 다만 org 폴백도 Tags가 완전히 비어있는 59.15%는 구조적으로  
채울 수 없음 — 이 몫은 태그가 못 잡는 비용(미태깅)으로 표면화해야 함  
(`관리그룹_청구계층_태그 가이드.md`의 "태그가 못 잡는 3종" 원칙 참고).

> 유의(20.89% ≠ 9.38%): org 태그 보유율 20.89%는 CostCenter와 **중복 보유분까지 포함한 전체  
> 비율**이고, 조인에서 org 폴백으로 **새로 배분되는 몫은 9.38%**(3.7)임. CostCenter를 이미 가진 행은  
> 1차 매칭으로 흡수되고, 폴백은 CostCenter 미보유 행 중 CMDB에 등록된 코드(`trey`)만 반영하기 때문임.

> 함정(포털 한계 재확인): 포털의 Tag 그룹화 축은 태그 1개 키만 그룹화 가능하고, "CostCenter가  
> 없으면 org" 같은 **폴백 로직 자체를 표현할 수 없음**. 폴백 로직은 Power Query 병합 단계  
> (피비 담당)에서 조건부 매핑으로 구현해야 함 — 본 섹션은 그 전 단계로 "왜 두 키가 필요한가"까지만  
> 정리함.

> 실무 팁: 조인 전 CostCenter·org 각각의 커버리지를 %로 미리 계산해두면, 조인 후 실제 매칭률이  
> 기대치와 맞는지 검증하는 기준선이 됨.

### 2.3 CMDB(조직정보 마스터) 설계

CMDB는 "코드 → 조직" 매핑표임 — `관리그룹_청구계층_태그 가이드.md`가 강조하는 "부서명을 태그에  
직접 쓰지 말고 코드 → 부서 매핑표로 관리" 원칙을 그대로 구현한 결과물임. 태그에는 코드만 남기고,  
코드가 실제 어느 팀·BU·담당자에 속하는지는 CMDB 표 한 장에서 관리함. 조직 개편이 있어도 태그를  
전부 재작업할 필요 없이 **CMDB 표만 갱신**하면 됨.

이 실습에서는 2개의 CMDB 표를 사용함(`work/` 폴더에 샘플로 생성됨).

**① `cmdb-costcenter-master.csv`** — CostCenter 코드 기준 마스터

| CostCenterCode | Team | BU | Owner | CompanyCode |
|---|---|---|---|---|
| 1234 | 디지털마케팅팀 | 마케팅본부 | 김민준 | TREY |
| SubACM | 구독관리팀 | IT인프라본부 | 이서연 | ACM |
| ACM-PM | PMO | 경영지원본부 | 박도윤 | ACM |

**② `cmdb-org-fallback.csv`** — org 태그 폴백용 보조 마스터

| OrgCode | BU_fallback | Owner_fallback |
|---|---|---|
| trey | Trey전사공통 | 최지우 |

**키 유일성이 핵심**: CostCenterCode·OrgCode는 표 안에서 반드시 값이 유일해야 함. 같은 코드가  
두 행에 중복되면 조인 시 원본 비용 1건이 CMDB 행 개수만큼 부풀려지는 **다대다(fan-out) 팬아웃  
오류**가 발생함(코드 하나가 CMDB에 2행 있으면 해당 비용이 조인 후 2배로 집계됨). 조인 전 CMDB  
표의 키 컬럼에 중복이 없는지 반드시 확인할 것.

> 실무 팁: CMDB 표는 HR·ITSM 등 별도 시스템에서 내려받는 것이 이상적이나, 이 실습처럼 소규모면  
> CSV로 시작해도 무방함. 다만 갱신 이력(버전·갱신일)을 표 밖에 남겨 "언제 기준 조직도인지"  
> 추적 가능하게 할 것.

> 함정: CMDB 표의 코드 표기가 실제 태그 값과 대소문자·공백까지 완전히 일치해야 매칭됨  
> (2.4 정규화로 보완). CMDB 설계 단계에서부터 코드 표기 규칙(예: 대문자 통일)을 정해두면 2.4의  
> 정규화 부담이 줄어듦.

### 2.4 정규화 선행 (조인 전 필수)

프로파일링 결과, 조인 전 정규화하지 않으면 매칭이 누락되는 실측 이슈가 3가지 확인됨.

| 이슈 | 실측 내용 | 조인 전 조치 |
|---|---|---|
| 태그 키 앞 공백 | 정상 키 `'org'`(105,744행)와 앞 공백 오염 키 `' org'`(35,238행)가 별개 키로 취급됨(27,151행은 두 키 동시 보유) | 태그 키 이름 트림(trim) 후 병합 |
| CostCenter 대소문자 중복 | `x_CostCenter`에 `'ACM9000'`(21.89%) vs `'acm9000'`(19.38%), `'Test123'` vs `'test123'` 공존 | 대소문자 통일(예: 대문자화) 후 비교 |
| env 값 표기 불일치 | `'prod'`(19.79%) · `'dev'`(0.78%) · `'Dev'`(0.06%) · `'Production'`(0.02%) · `'trey'`(오입력, 0.30%) | 값 표준화(동의어 사전 매핑) |

트림 처리의 효과가 실측으로 확인됨: `org` 태그 키를 트림해 병합하면 org 폴백 커버리지가  
8.89%에서 9.38%로(+0.5%p, +173.82 USD) 회복됨. 트림 없이 조인하면 이 173.82 USD는 org  
값이 있음에도 매칭 실패로 처리되어 미태깅으로 잘못 집계됨. 특히 `' org'`만 보유한 행(약 8,087행  
= 35,238 − 27,151)은 트림 없이는 `'org'` 조회에서 통째로 누락되는 몫임.

**정규화 체크리스트(조인 전 필수 수행)**

- [ ] 태그 키 이름 트림(앞뒤 공백 제거) — 특히 `org` 키  
- [ ] CostCenter·x_CostCenter 대소문자 통일  
- [ ] env 등 자유 입력 값의 표준화(오입력·동의어 정리)  
- [ ] CMDB 표 키 컬럼과 태그 값의 표기 규칙(대소문자·공백) 사전 합의

> 함정: 정규화를 건너뛰고 바로 조인하면 "매칭 실패=미태깅"으로 오판해 실제보다 미태깅 비중을  
> 과대평가하게 됨. 매칭 실패의 원인이 ①진짜 미태깅인지 ②표기 불일치인지 반드시 구분할 것.

> 실무 팁: 정규화 자체는 Power Query의 Trim·Uppercase 변환 단계(피비 담당)에서 처리되나,  
> 어떤 값들이 서로 불일치하는지 사전 프로파일링(이 섹션의 실측 표)은 데이터 준비 단계에서  
> 미리 끝내 두어야 조인 단계가 매끄러움.

---

## 3. Power BI 조인·집계·검증 (담당: 피비)

> 데이터 준비(청구 데이터 Export·CMDB 설계)는 애리 담당 산출물을 그대로 사용함 — 이 섹션은 Power BI(Power Query) 조인·집계·검증만 다룸.  
> **정직한 보고**: 이 실습 환경에서는 Power BI Desktop을 실제 실행하지 않았고, 아래 모든 수치는 Power Query 병합과 동일한 로직을  
> pandas로 재현해 검증한 값임. Power BI에서 동일 CSV로 아래 절차를 그대로 수행하면 같은 결과가 나와야 하나, 실제 Merge 실행 시  
> 결측값·데이터 형식(텍스트/숫자) 차이 등 환경 요인으로 미세한 차이가 날 수 있음 — 각자 화면에서 재현한 값을 이 문서의 값과  
> 반드시 대조할 것(스크린샷은 아래 자리표시자를 실제 캡처로 교체).

### 3.1 데이터 로드 — billing CSV + CMDB CSV 가져오기

**수행**: 홈 탭 → 데이터 가져오기(Get Data) → 텍스트/CSV(Text/CSV) → 아래 3개 파일을 각각 별도 쿼리로 로드함.

| 쿼리명 | 원본 파일 | 역할 |
|---|---|---|
| `billing` | 청구 데이터 CSV(FOCUS, EA export) | 조인 대상 팩트 테이블 |
| `CMDB_CC` | `cmdb-costcenter-master.csv` | CostCenter 기준 배분 마스터 |
| `CMDB_ORG` | `cmdb-org-fallback.csv` | org 폴백 배분 마스터 |

- 가져오기 대화상자의 **파일 원본(File Origin)** 드롭다운에서 **65001: 유니코드(UTF-8)** 를 명시 지정함.  
  CMDB 파일의 `Team`·`BU`·`Owner`(한글 값, 예 "디지털마케팅팀"·"마케팅본부")가 이 설정에 의존함.  
- 3개 쿼리 모두 로드 후 **Power Query 편집기(Transform data)** 로 진입해 이후 단계를 진행함.

**함정**: 파일 원본이 자동 감지되어 UTF-8이 아닌 다른 코드페이지로 잘못 인식되면 한글 컬럼값이 깨짐(모지바케). 미리보기에서  
한글이 정상 표시되는지 반드시 육안 확인 후 다음 단계로 진행할 것.

### 3.2 Tags JSON 파싱 → CostCenter·org 컬럼 추출

`billing` 쿼리의 `Tags` 컬럼은 JSON 문자열임(예: `{"CostCenter":"1234","org":"trey","env":"prod"}`).  
실제 원본 데이터에는 정상 키 `"org"` 외에 앞에 공백이 붙은 오염 키 `" org"`가 별도로 섞여 있어, 두 키를 함께 흡수해야 함.

**수행**: 반드시 **Power Query 편집기** 안에서 진행함 — 홈 탭 → **데이터 변환(Transform data)** 진입 →  
`billing` 쿼리 선택 → **열 추가(Add Column) 탭 → 사용자 지정 열(Custom Column)** 로 아래 3개 컬럼을 순서대로 추가함.  
대화상자의 **새 열 이름** 칸에 "열 이름"을, **수식** 칸에 해당 M 수식을 입력함  
(수식 칸 왼쪽에 이미 `=`가 붙어 있으므로 `열 =` 같은 접두어는 넣지 않음).

| 순서 | 새 열 이름 | 사용자 지정 열 수식 (Power Query M) |
|---|---|---|
| ① | `TagsRecord` | `try Json.Document([Tags]) otherwise null` |
| ② | `CostCenter_raw` | `Record.FieldOrDefault([TagsRecord], "CostCenter", null)` |
| ③ | `org_raw` | `Record.FieldOrDefault([TagsRecord], "org", null) ?? Record.FieldOrDefault([TagsRecord], " org", null)` |

> ⚠️ **이 수식은 DAX가 아니라 Power Query M임 — 실행 위치를 반드시 구분할 것.**  
> `Json.Document`·`try…otherwise`·`Record.FieldOrDefault`·`??`는 모두 M 함수·연산자로 **Power Query 편집기의 사용자 지정 열**에서만 동작함.  
> 보고서/데이터 보기의 **DAX 새 열**(수식이 `열 = …`로 시작)에 입력하면 `'Json'의 구문이 잘못되었습니다(DAX…)` 오류가 발생함 —  
> 이 경우 잘못 만든 DAX 열을 삭제하고 위 경로(데이터 변환 → 사용자 지정 열)로 다시 추가할 것.

- `Json.Document`는 JSON 텍스트를 Power Query 레코드로 변환하는 M 함수임  
  ([Microsoft Learn](https://learn.microsoft.com/en-us/powerquery-m/json-document)).  
- `Record.FieldOrDefault(record, field, default)`는 필드가 없으면 지정한 기본값(여기서는 `null`)을 반환함  
  ([Microsoft Learn](https://learn.microsoft.com/en-us/powerquery-m/record-fieldordefault)).  
- `??`는 좌변이 null이면 우변을 반환하는 코얼레스 연산자임  
  ([Microsoft Learn: M Language Operators](https://learn.microsoft.com/en-us/powerquery-m/m-spec-operators)).  
- 이후 병합 키의 대소문자·공백 차이를 제거하기 위해 `CostCenter_key = Text.Lower(Text.Trim([CostCenter_raw]))`,  
  `org_key = Text.Lower(Text.Trim([org_raw]))` 컬럼도 함께 추가함. Power Query 병합은 기본적으로 **대소문자를 구분**하므로  
  ([Community 사례](https://community.fabric.microsoft.com/t5/Desktop/How-to-make-a-Merged-Query-non-case-sensitive/td-p/1267731)),  
  이 정규화 컬럼으로 매칭해야 함.  
- **실측 효과**: `" org"` 오염 키를 트림해 흡수한 결과, org 폴백 매칭이 8.89% → 9.38%(+173.82 USD)로 회복됨(pandas 검증치).

**함정**: `Tags`가 빈 문자열이거나 유효한 JSON이 아닌 행이 있으면 `Json.Document`가 오류를 던짐. `try ... otherwise null`로  
감싸지 않으면 해당 행에서 쿼리 전체가 오류로 중단될 수 있음.

### 3.3 병합 1차 — billing ⟕ CMDB(CostCenter)

**메뉴 경로**: `billing` 쿼리 선택 → 홈 탭 → 결합(Combine) 그룹 → **쿼리 병합(Merge Queries)**  
([Microsoft Learn: Merge Queries Overview](https://learn.microsoft.com/en-us/power-query/merge-queries-overview)).

**병합 대화상자 설정**

| 항목 | 값 |
|---|---|
| 첫 번째 테이블 | `billing` |
| 두 번째 테이블 | `CMDB_CC` |
| 매칭 컬럼 | 양쪽 모두 `CostCenter_key`(정규화 컬럼) |
| 조인 종류(Join Kind) | **왼쪽 외부(Left Outer)** — billing 행 전부 보존, CMDB_CC 매칭분만 결합 |

- 병합 결과로 생긴 신규 컬럼(레코드)을 **확장(Expand)** 하여 `Team`·`BU`(→ `BU_cc`로 이름 변경)·`Owner`(→ `Owner_cc`)만 선택함.  
- 확장 시 원치 않는 컬럼(예: `CompanyCode`)은 체크 해제해 불필요한 폭 증가를 막음.

**함정(다대다 조인 중복집계 — 반드시 확인)**: `CMDB_CC`의 `CostCenterCode`가 **유일(unique)하지 않으면**, 왼쪽 외부 조인이어도  
billing 각 행이 CMDB_CC의 중복 행 수만큼 팬아웃(fan-out)되어 **비용이 실제보다 부풀려 중복 집계**됨. 병합 실행 전 CMDB_CC를  
`CostCenterCode` 기준으로 그룹화(Group By)해 행 수와 원본 CostCenterCode 종류 수가 일치하는지(=키 유일성) 반드시 검증할 것.  
실측 검증에서는 조인 전후 행 수가 135,272 = 135,272로 동일해 팬아웃이 없음을 확인함(3.7 게이트 참조).

### 3.4 병합 2차 — 미매칭 org 폴백

`CMDB_CC`와 매칭되지 않은 행(1차 병합 후 `BU_cc`가 null인 행)에 대해 `org_key`로 `CMDB_ORG`를 폴백 매칭함.

**수행 방식**: 3.3에서 확장까지 마친 `billing` 테이블 **전체**를 대상으로, `CMDB_ORG`와 `org_key` 기준 왼쪽 외부 병합을  
한 번 더 수행함(미매칭 행만 별도로 필터링 → 병합 → 재결합(Append)하는 대신, 전체 테이블에 2차 병합을 걸고  
3.5에서 우선순위 결합(coalesce)으로 정리하는 방식이 실습 단계 수가 적음). CMDB 양쪽 소스의 키가 유일하다면  
두 방식의 결과는 동일함.

**메뉴 경로**: 동일(홈 → 결합 → 쿼리 병합) · 매칭 컬럼 = 양쪽 `org_key` · 조인 종류 = 왼쪽 외부 · 확장 컬럼 =  
`BU_fallback`, `Owner_fallback`.

**함정**: 이 2차 병합 역시 `CMDB_ORG`의 `OrgCode` 유일성이 전제임. 실측 데이터는 `trey` 1건만 존재해 유일성이 자명하지만,  
조직 확장 시 `OrgCode` 중복 여부를 3.3과 동일한 방법으로 재검증해야 함.

### 3.5 배분 BU 컬럼 생성 + 미배분 버킷 표면화

**수행**: 열 추가 탭 → 사용자 지정 열로 `배분BU`를 아래 코얼레스 논리로 생성함. CostCenter 매칭(`BU_cc`)을 최우선,  
org 폴백(`BU_fallback`)을 차선, 둘 다 없으면 **"미배분(Untagged)"** 문자열로 명시 고정함.

```
// 열 추가 > 사용자 지정 열: 배분BU
[BU_cc] ?? [BU_fallback] ?? "미배분(Untagged)"
```

- 동일 패턴으로 `배분Owner = [Owner_cc] ?? [Owner_fallback] ?? "담당자 미상"`도 선택적으로 추가 가능함.  
- **"미배분(Untagged)"은 문자열 카테고리로 반드시 남겨야 함** — null로 두거나 필터로 제외하면 안 됨. 이는  
  `hands-on/references/cost-dashboard-analyze.md`의 미태깅 표면화 원칙(3.1 미태깅 범인 식별)과 동일한 사고임: 배분 안 된  
  비용도 "보이는 카테고리"로 남겨야 사각지대가 드러남.

**함정**: 미배분 버킷을 시각화에서 숨기거나(필터 제외) 행을 삭제하면 총액이 조인 전 총액(35,445.86 USD)과 어긋나  
3.7의 무손실 게이트를 위반하게 됨. "미배분"은 숨김이 아니라 표면화 대상임을 재차 확인할 것.

### 3.6 집계 측정값 — 조직 단위(BU)·조직×서비스

Power Query 편집기에서 **닫기 및 적용(Close & Apply)** 후, 데이터 모델에서 아래 DAX 측정값을 생성함.

```dax
청구비용 = SUM(billing[BilledCost])
실질비용 = SUM(billing[EffectiveCost])
총청구비용 = CALCULATE([청구비용], ALL(billing))
비중 = DIVIDE([청구비용], [총청구비용])
```

**BU별 배분(청구비용, 실측 pandas 검증치)**

| 배분BU | 청구비용(USD) | 비중(전체 대비) |
|---|---|---|
| 미배분(Untagged) | 20,974.98 | 59.17% |
| IT인프라본부(SubACM) | 6,753.78 | 19.05% |
| 마케팅본부(1234) | 4,390.50 | 12.39% |
| Trey전사공통(org폴백) | 3,323.43 | 9.38% |
| 경영지원본부(ACM-PM) | 3.17 | 0.01% |
| **합계** | **35,445.86** | **100.00%** |

**조직×서비스(배분BU × ServiceCategory) 상위 조합** — 아래는 상위 7개 조합이며 비중 합(87.56%)이 100%에  
못 미치는 나머지(12.44%)는 롱테일로 남아 있음을 명시함.

| 배분BU | ServiceCategory | 청구비용(USD) | 비중(전체 대비) |
|---|---|---|---|
| 미배분(Untagged) | Analytics | 20,551.68 | 57.98% |
| IT인프라본부 | Databases | 2,890.29 | 8.15% |
| 마케팅본부 | Compute | 2,037.49 | 5.75% |
| Trey전사공통 | Web | 1,782.91 | 5.03% |
| IT인프라본부 | Compute | 1,563.89 | 4.41% |
| IT인프라본부 | Other | 1,179.88 | 3.33% |
| IT인프라본부 | Integration | 1,033.10 | 2.91% |

**함정**: 위 표의 %는 모두 **전체 총액(35,445.86 USD) 대비**임. 매트릭스 시각화에서 "행 합계 대비 %"·"열 합계 대비 %" 옵션을  
켜면 분모가 바뀌어 위 수치와 달라 보일 수 있음 — 어느 분모를 쓰는지 시각화 설정에서 항상 확인할 것.

### 3.7 커버리지 KPI + 총액 무손실 검증 (릴리스 게이트)

**커버리지율과 무손실 검증은 서로 다른 지표임 — 혼동하지 말 것.** 무손실 검증(행 수·금액 보존)은 조인 로직 자체가  
올바른지를 보는 **필수 릴리스 게이트**이고, 커버리지율(매칭 비율)은 배분 완성도를 보여주는 **참고 KPI**임. 커버리지율이  
낮아도(예: 40.83%) 무손실 게이트만 통과하면 조인 로직 자체는 유효함 — 커버리지 개선은 별도 태깅 거버넌스 과제임.

```dax
매칭청구비용 = CALCULATE([청구비용], billing[배분BU] <> "미배분(Untagged)")
커버리지율 = DIVIDE([매칭청구비용], [총청구비용])
DIFF검증 = [총청구비용] - SUMX(VALUES(billing[배분BU]), [청구비용])
```

`DIFF검증`은 BU별로 재집계한 합계와 전체 총액을 비교하는 측정값으로, 값이 0이 아니면 조인 과정에서 금액이 새거나  
중복된 것이므로 즉시 릴리스를 중단해야 함. (참고: DAX에는 `COALESCE(<expr>, <expr>...)` 함수도 존재하나  
([Microsoft Learn](https://learn.microsoft.com/en-us/dax/coalesce-function-dax)), 이 실습에서는 조인 자체가 Power Query 단계에서  
이미 끝나 있으므로 배분 로직은 3.5의 M `??` 연산자로 처리하고, DAX 단계에서는 집계·검증 측정값만 둠.)

**릴리스 게이트 판정표(실측 pandas 검증치)**

| 게이트 | 기준 | 실측값 | 판정 |
|---|---|---|---|
| 행 수 보존 | 조인 전 행 수 = 조인 후 행 수 | 135,272 = 135,272 | 통과 |
| 총액 무손실 | BilledCost 합계 DIFF = 0 | 35,445.86 = 35,445.86 (DIFF 0.000000) | 통과 |
| 미배분 표면화 | "미배분(Untagged)" 카테고리 존재 확인 | 20,974.98 USD(59.17%) 노출됨 | 통과 |

**커버리지 KPI(참고치, 게이트 아님)**

| 지표 | 값 |
|---|---|
| CostCenter 직접 매칭 | 31.45% |
| org 폴백 매칭 | 9.38% |
| 매칭 합계(커버리지율) | 40.83% |
| 미배분 | 59.17% |

**Power Query에서의 수동 대조 방법**: 병합 전 `billing` 쿼리에서 `BilledCost` 컬럼 선택 후 하단 상태 표시줄(또는 열 통계)의  
합계를 기록하고, 3.6의 `배분BU`별 합계를 모두 더한 값과 일치하는지 대조함. `Table.RowCount(billing)` 값도 병합 전후로  
비교해 행 수 게이트를 재현할 수 있음.

**함정(다대다 조인 재확인)**: 만약 이 게이트에서 조인 후 행 수가 135,272보다 **커지면**, 십중팔구 3.3 또는 3.4에서 CMDB  
키 유일성이 깨져 팬아웃이 발생한 것임 — 즉시 CMDB 소스의 키 중복부터 재점검할 것(비중·합계 재계산 전에 원인 제거 필수).

### 3.8 대시보드 구성

| 시각화 | 축/필드 · 측정값 | 읽는 법 |
|---|---|---|
| BU별 막대(가로 막대형) | 축=`배분BU`, 값=`청구비용` | "미배분(Untagged)"이 최상위 막대로 노출되는지 확인(숨기지 않음) |
| BU × 서비스 매트릭스 | 행=`배분BU`, 열=`ServiceCategory`, 값=`청구비용` | 미배분×Analytics 셀이 전체 대비 몇 %인지 우선 확인(3.6 표 대조) |
| 커버리지 도넛/카드 | 값=`커버리지율`, `매칭청구비용`, `총청구비용` | 매칭(40.83%) vs 미배분(59.17%) 두 조각으로 배분 완성도 파악 |
| 무손실 검증 카드 | 값=`DIFF검증`, 행 수(조인 전/후) | 0/동일이 아니면 새로고침 즉시 릴리스 중단 신호로 사용 |

> 📷 (스크린샷 위치: BU별 청구비용 가로 막대 차트 — 실제 Power BI 캡처로 교체)  
> 📷 (스크린샷 위치: 배분BU × ServiceCategory 매트릭스 — 실제 Power BI 캡처로 교체)  
> 📷 (스크린샷 위치: 커버리지 도넛 + 무손실 검증 카드 — 실제 Power BI 캡처로 교체)

**함정**: 무손실 검증 카드에 조건부 서식(`DIFF검증`≠0이면 강조색)을 걸어 두면 새로고침마다 게이트 위반을 육안으로 즉시  
확인 가능함 — 수동 대조에만 의존하면 정기 새로고침 시 위반을 놓칠 위험이 있음.

---

## 4. 종합

### 4.1 표준 시퀀스 (재현용)

1. **준비**: 청구 export 확보 → 조인 키(CostCenter+org 폴백) 확정 → CMDB 매핑표 설계 → **정규화 선행**.  
2. **조인**: Tags 파싱 → CostCenter/org 추출 → billing ⟕ CMDB(CostCenter) → 미매칭 org 폴백 → 배분 BU 컬럼.  
3. **집계**: 조직 단위(BU) · 조직 × 서비스(BU × ServiceCategory) 측정값 산출.  
4. **검증**: 커버리지%(배분율) + **총액 무손실·행 수 보존** 릴리스 게이트 통과 → 대시보드 공유.

### 4.2 릴리스 게이트 체크리스트

- [ ] **총액 무손실**: 조인·집계 후 총 청구비용이 원본과 일치하는가(DIFF ≈ 0)?  
- [ ] **행 수 보존**: 병합 후 행 수가 원본과 같은가(다대다 팬아웃으로 늘지 않았는가)?  
- [ ] **키 유일성**: CMDB의 조인 키(CostCenterCode)에 **중복이 없는가**(중복 시 비용 중복집계)?  
- [ ] **정규화 선행**: 키 트림(`' org'`)·대소문자 통일(`acm9000`/`ACM9000`)을 조인 **전에** 했는가?  
- [ ] **커버리지% 표면화**: 배분율(matched)·**미배분(Untagged)** 비중을 KPI로 노출했는가?  
- [ ] **미배분 은폐 금지**: "미배분" 버킷을 삭제·숨김 없이 표에 남겼는가?  
- [ ] **폴백 검증**: CostCenter 미보유분에 org 폴백이 실제로 적용됐는가?

### 4.3 함정 (Pitfalls)

| 함정 | 왜 위험한가 | 대응 |
|---|---|---|
| **다대다 조인 팬아웃** | CMDB 키 중복이면 청구 행이 복제돼 **비용 중복집계** | 조인 전 CMDB 키 유일성 확인 + 행 수 보존 검증 |
| **미배분 은폐** | Untagged를 숨기면 배분율이 부풀려져 의사결정 왜곡 | 미배분을 **항상 표면에** 표시 |
| **정규화 누락** | `' org'`·`ACM9000/acm9000` 불일치로 매칭 누락 | 키 트림·대소문자 통일을 조인 전 수행 |
| **"태그 합계 = 전체" 착각** | 미태깅분을 빼면 총액 불일치 | 총액 무손실 게이트로 강제 검증 |
| **태그 CostCenter ↔ x_CostCenter 혼동** | 태그값(`1234`)과 청구계층 비용센터(`ACM9000`)는 별개 | 조인 키는 **태그 CostCenter**임을 명시 |
| **소급 태깅 기대** | 태그는 붙인 이후만 적용, 과거분 미배분 잔존 | 과거분은 별도 소급 배분 규칙 |

### 4.4 Showback → Chargeback 브릿지 (커버리지 올리기)

- **미배분 축소**: Azure Policy로 태그 **필수화 + 상속**(리소스그룹→리소스) → 신규 리소스 미태깅 차단.  
- **폴백 키 확대**: CostCenter 없으면 org, org도 없으면 구독→기본 BU 매핑 등 **다단계 폴백**으로 커버리지 향상.  
- **공유비용 배분**: 미배분 대형 버킷(공유 인프라·플랫폼)은 사용량 기반 **배분 규칙**으로 분배(단일 태그 귀속 불가).  
- **정산 전환**: 커버리지가 안정 궤도에 오르면 Showback(현황 조회) → **Chargeback(실제 정산)** 으로 이관.

---

## 5. 참조

**내부 문서**  
- [`관리그룹_청구계층_태그 가이드.md`](../hands-on/references/관리그룹_청구계층_태그%20가이드.md) — 태그·청구계층·매핑표 원칙, 미태깅 표면화  
- [`cost-dashboard-analyze.md`](../hands-on/references/cost-dashboard-analyze.md) — 포털 비용 분석·Tag 그룹화·배분 개념  
- 샘플 CMDB: [`cmdb-costcenter-master.csv`](cmdb-costcenter-master.csv) · [`cmdb-org-fallback.csv`](cmdb-org-fallback.csv)

**1차 출처**  
- [FinOps Framework — Allocation](https://www.finops.org/framework/capabilities/allocation/)  
- [FOCUS — FinOps Open Cost & Usage Specification](https://focus.finops.org/)  
- [Microsoft Learn — Power Query 쿼리 병합 개요](https://learn.microsoft.com/power-query/merge-queries-overview)  
- [Microsoft Learn — 비용 할당(Cost allocation)](https://learn.microsoft.com/azure/cost-management-billing/costs/allocate-costs)  
- [Microsoft Learn — 리소스 태그 지정](https://learn.microsoft.com/azure/azure-resource-manager/management/tag-resources)

---

> 본 가이드의 모든 수치는 실습 데이터셋(`EA-Cost-FOCUS_1.0.csv`)을 **pandas로 프로파일링·조인 검증**한 실측값임.  
> 조인 로직·총액 무손실은 Power Query 병합과 동일 로직으로 재현·검증됨(3.7). Power BI Desktop에서 동일 절차  
> 수행 시 같은 수치가 재현되어야 함.
