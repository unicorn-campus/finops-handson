# 할인 대상 판별 (Reservations · Savings Plans)

> 실습 환경: **EA(Enterprise Agreement)** 기준  
> 스크린샷: 각 캡처 지점에 `📸 스크린샷` 자리표시 콜아웃 표기 — 실습 중 실제 화면 캡처 후 교체  
> 1차 출처: Microsoft Learn(문서 하단 '용어·출처'에 링크·갱신일 명시)  
> 📌 한 줄 요약(TL;DR): **상시·안정 부하의 SKU·리전을 먼저 식별 → 해당 리소스가 예약/절감형 대상인지 판별 →  
> Advisor·Cost Management 권장으로 예상 절감·권장 수량 확인** 순으로 진행함. 실제 구매 실행은 Optimize 단계로 이관.

---

## 0. 개요 — 왜 '할인 대상 판별'인가

- FinOps **Rate optimization(요율 최적화)** 활동임. 상시 부하를 온디맨드(PAYG) 대신 약정으로 전환해 요율을 낮춤.
- 예약(Reservations)은 정가(PAYG) 대비 **최대 72%** 절감 가능함(출처: 예약 개요).
- **순서 원칙(중요)**: 할인은 *요율*만 낮출 뿐 *낭비*는 없애지 못함 → **먼저 Right-size(유휴·과대 리소스 제거)
  후 약정 구매**가 원칙임(출처: RI vs SP 선택 가이드).

### 두 가지 약정 비교 (RI vs SP)

| 구분 | Reservations(예약, RI) | Savings Plans(절감형 계획, SP) |
|---|---|---|
| 약정 방식 | 특정 인스턴스/**패밀리 + 특정 리전** 1·3년 | **시간당 $ 고정 지출** 1·3년 |
| 적용 범위 | 지정한 서비스+리전 조합에만 적용 | 참여 컴퓨팅 서비스·**전 리전** 자동 적용 |
| 할인폭 | 더 깊음(완전 활용 시 최대) | 유연하나 상대적으로 낮음 |
| 적합 워크로드 | 상시·안정·인스턴스/리전 변화 없음 | 변동·진화·리전 이동 가능 |
| 취소/환불 | 교환 가능 · 환불 12개월 롤링 **$50,000 한도** | **취소·환불 불가** |

> 판단 요령: "인스턴스·리전이 안 바뀌는 상시 부하" = RI, "여러 패밀리·리전으로 변동" = SP  
> (출처: RI vs SP 선택 가이드)

---

## 1단계: 사용 SKU·리전 확인 (Cost Analysis)

약정 후보 = **"매일 꾸준히 도는" 상시 부하**임. 무엇을 어느 리전에서 얼마나 쓰는지 먼저 식별해야 함.

- 비용 관리 + 청구 → **Cost Management** → **Cost analysis** 진입
  > 📸 **스크린샷**: Cost analysis 초기 화면

- 기간 설정: 상단에서 **최근 30일 이상(권장 60일)** 선택
  - 이유: 예약 권장 엔진이 **최근 7·30·60일** 사용량을 평가함(출처: 예약 권장). 기간이 짧으면 판단 왜곡됨.

- 그룹화(Group by)를 순서대로 바꿔 3가지를 확인함
  - **Service name**: 비용 상위·상시 서비스 식별(예: Virtual Machines, SQL Database)
    > 📸 **스크린샷**: Service name 그룹화 결과
  - **Location(리전)**: 해당 리소스가 실제 어느 리전에 있는지 확인 — **예약은 리전 종속이라 필수**
  - **Resource** 또는 **Meter**: 구체 SKU·미터(예: `Standard_D4s_v5`, `Korea Central`) 확인

- 차트를 **일자별 누적 열**로 보고 상시성 판단
  - 일별 사용량이 **편평·지속** → 약정 후보 ✅
  - **켜졌다 꺼졌다·간헐적** → 약정 부적합(권장이 아예 안 나옴). 리소스가 자주 종료되면 절감이 계산되지 않음  
    (출처: 예약 권장 — "resources shut down regularly → no recommendation")

---

## 2단계: 할인 대상 여부 판별

대상 여부는 ① 그 서비스에 약정 상품이 있는지, ② 비용이 **컴퓨팅/용량**인지(소프트웨어·네트워크·스토리지 트랜잭션 제외)로 갈림.

### 예약(Reservations) 대상 (컴퓨팅/용량 + 일부 소프트웨어 플랜 비용 커버)

| 범주 | 대표 서비스 |
|---|---|
| 컴퓨팅 | Virtual Machines(VM), Dedicated Host, App Service(스탬프 요금) |
| 데이터베이스 | SQL Database·SQL Managed Instance(컴퓨팅), Cosmos DB(처리량), MySQL·PostgreSQL(컴퓨팅), Cache for Redis |
| 분석 | Synapse Analytics(cDWU), Databricks(DBU), Data Explorer, Data Factory(데이터 흐름) |
| 스토리지(예약 용량) | Blob·Files·Disk(Premium SSD P30↑)·Backup·NetApp Files 예약 용량 |
| 소프트웨어 플랜 | SUSE Linux, Red Hat, Azure Red Hat OpenShift |

> (출처: 예약 개요 — "Charges covered by reservation")

### 절감형 계획(Savings Plans) 대상

- **Savings plan for compute**(1·3년): VM, App Service, Functions(프리미엄 플랜), Container Instances,
  Dedicated Host, Container Apps, Spring Apps for Enterprise
- **Savings plan for databases**(1년): SQL Database·Managed Instance·Hyperscale·serverless,
  PostgreSQL, MySQL, Cosmos DB 등
  > (출처: 절감형 계획 개요 — "What savings plans are available")

### 비대상·주의

- **소프트웨어(Windows·SQL 라이선스)·네트워킹·스토리지 대역폭/트랜잭션**은 예약/SP 미포함  
  → 라이선스 비용은 **Azure Hybrid Benefit(AHB)** 로 별도 절감(출처: 예약 개요·SP 개요)
- 스팟(Spot) VM, 이미 무료/버스트성 리소스, Classic(클래식) 리소스는 대상 아님

> ### ⚠️ 함정 ① — 리전·패밀리 종속(RI)
> 예약은 **특정 리전 + 인스턴스 패밀리**에 묶임. 1단계에서 확인한 리전과 **다른 리전**에 예약을 사면  
> **할인이 전혀 적용되지 않음**. 리전 이동·패밀리 변경이 잦다면 예약 대신 **SP(리전·패밀리 유연)** 를 검토함.  
> (출처: RI vs SP 선택 가이드)

### 판별 기준 — 무엇을 '대상'으로 볼 것인가

절차만 따라가지 말고, 아래 기준으로 **워크로드별 대상 여부를 판정**함(클릭 재현에 그치지 않도록).

| 판별 축 | 대상(약정 O) | 비대상(약정 X) |
|---|---|---|
| 상시성 | 24/7에 가깝게 **상시 가동**, 인스턴스/리전 변경 없음 | 켜졌다 꺼졌다·간헐·단기 실험 |
| 비용 성격 | 컴퓨팅/용량(또는 SUSE·RedHat 소프트웨어 플랜) | 일반 SW 라이선스·네트워크·스토리지 트랜잭션 |
| 권장 신호 | Advisor/Cost Management에 **권장 표시 + 예상 절감 > 0** | 권장 미표시(관측 부족·간헐) |

- **상시성 기준**: 예약은 "지속·안정 가동" 워크로드에 최대 효과(출처: RI vs SP 선택 가이드). 자주 종료되면  
  권장 자체가 나오지 않음(출처: 예약 권장).
- **손익분기 원칙(참고·일반 계산)**: 예약은 활용률이 낮으면 오히려 손해임. 개략 손익분기 활용률 ≈ **(1 − 할인율)**.  
  예) 할인 40%면 활용률 60% 이상일 때 온디맨드보다 이득 — 일반 손익분기 계산이며 MS 문서의 고정 수치가 아님.
- **결정 규칙**: 상시성 O + 컴퓨팅/용량 O + 권장·절감 O → **대상**.  
  하나라도 X면 재검토(SP 전환 / 관측 기간 연장 / 먼저 Right-size).

---

## 3단계: Advisor 권고로 예상 절감 확인

- **Advisor** → 왼쪽 **'비용'** 탭 진입 (또는 비용 관리 + 청구 → Advisor 권장 사항)
  > 📸 **스크린샷**: Advisor 비용 탭 권장 목록

- 표시되는 대표 권장(예)
  - `Consider {서비스} reserved instance to save...` — 예약 구매 권장(다수 서비스에 대해 표시)
  - `Consider purchasing a savings plan for compute...` — 절감형 계획 권장
  - `Configure automatic renewal for the expiring reservations` — 만료 임박/만료 예약 자동 갱신 권장

- 권장 클릭 → **예상 절감액·영향도(Impact: High)·상세** 확인
  > 📸 **스크린샷**: 개별 권장 상세(예상 절감·권장 수량)

- **Advisor 권장의 특성(반드시 인지)**
  - Advisor 예약 권장은 **단일 구독(single-subscription) 범위**임. **전체 청구 범위** 권장은  
    4단계(Cost Management → Reservations → Add)에서 확인해야 함(출처: 예약 권장).
  - Advisor 권장 수량·절감은 기본 **3년 기준**(3년 미판매 서비스는 **1년 가격(요율)** 으로 계산)(출처: 예약 권장).
  - **Classic 리소스**는 권장 엔진에서 명시적으로 제외됨. **종료 예정(deprecated) 서비스**는 자동 제외 대상은  
    아니나, 레거시 장기 약정 자체를 지양하도록 권고됨(출처: 예약 권장).

> ### ⚠️ 함정 ② — 최근 사용량(7·30·60일) 기반 과소 추천
> Advisor·권장 엔진은 **최근 7·30·60일 사용량**만 봄. 갓 배포한 **신규·변동 워크로드**는  
> 관측 데이터가 부족해 **과소 추천되거나 권장이 아예 안 나옴**. 신규 리소스는 며칠 관측 후 재확인함.  
> (참고: 예약 개요 문서에는 "Advisor는 VM 예약 권장" 표기 시점이 있으나, 현행 Advisor 비용 권고  
> 레퍼런스는 다수 서비스 예약 권장을 포함함 — 실제 화면에 표시되는 항목을 기준으로 판단.)

---

## 4단계: 예상 절감·권장 수량 상세 확인 (Cost Management)

전체 청구 범위 기준 권장 수량과 term·scope를 확정하는 단계임(판별까지가 목표, 구매 실행은 Optimize 단계).

- 비용 관리 + 청구 → **Reservations**(또는 **Reservations + Savings plans**) → **Add(추가)**  
  → 상품 유형 선택(예: Virtual machine)
  > 📸 **스크린샷**: Reservations → Add 상품 선택

- 확인 항목
  - **Recommended Quantity(권장 수량)**: Azure가 절감 최대가 되는 수량을 제시함
  - **Term(기간)**: 1년 vs 3년 — 3년이 할인폭 최대이나 고정 리스크↑
  - **Scope(범위)**: Shared(공유, 전 구독 유연) / Single subscription / Single resource group / Management group
  - **예상 절감액·활용률**: `See details`로 상세 확인 — 사용량이 들쭉날쭉하면 활용률이 100% 아닐 수 있음
    > 📸 **스크린샷**: See details 상세(절감·활용률 차트)

- 수량 조절 원리(출처: 예약 권장)
  - 권장보다 **많이** 사면 → 미사용분 발생 → 절감↓
  - 권장보다 **적게** 사면 → 초과분이 온디맨드로 청구 → 절감↓
  - 결론: **권장값에 최대한 가깝게** 구매

---

## 할인 대상 판별 체크리스트

- [ ] 대상 리소스가 최근 30~60일 **상시(편평)** 사용인가
- [ ] 사용 **리전·SKU(패밀리)** 를 확정했는가
- [ ] 해당 비용이 **컴퓨팅/용량**인가(소프트웨어·네트워크·트랜잭션 제외 확인)
- [ ] **Right-size**(유휴·과대 제거)를 먼저 했는가
- [ ] Advisor / Cost Management에서 **예상 절감·권장 수량**을 확인했는가
- [ ] 워크로드 성격에 맞는 **RI vs SP** 유형을 골랐는가
- [ ] **리전·패밀리 종속(RI)** 함정을 점검했는가

---

## 단계 구매 원칙 (실행은 Optimize 단계로 이관)

1. **Right-size 먼저** — 불필요·과대 리소스 제거. 할인은 요율만 낮춤(출처: RI vs SP 선택 가이드).
2. **최적화 시퀀스**: 저활용 예약 **교환** → 저활용 예약을 **SP로 전환** → **신규 예약** → **신규 SP** 순(출처: 동일).
3. 권장값 **전량 구매 지양**, 권장에 가깝게 단계적으로 구매.
4. **RI·SP를 동시에 사지 말 것** — 하나 구매 후 **최소 3일 대기**하여 권장 재계산 반영을 확인함  
   (출처: 예약 권장 — 공식 문서 기준 **3일**. 일부 사내 가이드의 '7일'과 상이하므로 **공식 기준 채택**).
5. **Shared 범위** 예약 구매 시 Advisor 권장이 사라지는 데 **최대 5일** 소요(출처: 예약 권장).

---

## 함정·주의 (※)

- ※ **리전·패밀리 불일치 → 예약 할인 미적용(RI)**. 1단계 리전 확인 필수.
- ※ **최근 7·30·60일 lookback** → 신규·간헐 워크로드는 과소추천/무추천.
- ※ 예약은 **컴퓨팅/용량만** 커버 — Windows/SQL 라이선스·네트워크·스토리지 트랜잭션 별도 → **AHB 병행**.
- ※ **Savings plan은 취소·환불 불가** → 커밋 전 신중. 예약은 교환·환불(12개월 $50,000 한도) 가능.
- ※ **EA는 특정 오퍼 유형**(MS-AZR-0017P / MS-AZR-0148P DevTest)만 SP 구매 가능(출처: SP 개요).
- ※ **Classic 리소스**는 권장에서 제외됨. **종료 예정 서비스**는 제외 대상은 아니나 레거시 장기 약정 지양 권고.

---

## 용어·출처

### 용어
- **RI(Reservation)**: 특정 서비스/패밀리·리전에 1·3년 약정하는 예약형 할인.
- **Savings Plan(절감형 계획)**: 시간당 $ 고정 지출 약정. 참여 컴퓨팅·전 리전에 자동 적용.
- **Term(기간)**: 약정 기간(1년/3년). 길수록 할인↑·고정 리스크↑.
- **Scope(범위)**: 할인이 적용되는 대상 범위 — Shared(공유)/단일 구독/리소스 그룹/관리 그룹.
- **Lookback(관측 기간)**: 권장 산정 기준 기간(7·30·60일).
- **Right-size**: 사용량에 맞게 리소스 크기·수량을 줄여 낭비 제거.
- **AHB(Azure Hybrid Benefit)**: 보유 Windows/SQL 라이선스로 소프트웨어 비용을 절감하는 혜택.
- **Recommended Quantity / Utilization**: 절감 최대 권장 수량 / 약정 실제 소진율.

### 출처 (Microsoft Learn · 괄호는 문서 갱신일)
- 예약 개요 — 최대 72%·커버 범위 (2026-07-17)  
  <https://learn.microsoft.com/azure/cost-management-billing/reservations/save-compute-costs-reservations>
- 절감형 계획 개요 — 대상 서비스·EA 오퍼·취소불가 (2026-03-18)  
  <https://learn.microsoft.com/azure/cost-management-billing/savings-plan/savings-plan-compute-overview>
- RI vs SP 선택·최적화 시퀀스 (2026-03-14)  
  <https://learn.microsoft.com/azure/cost-management-billing/savings-plan/decide-between-savings-plan-reservation>
- 예약 권장 — 7·30·60일·3일 대기·단일구독·3년 기준 (2026-07-17)  
  <https://learn.microsoft.com/azure/cost-management-billing/reservations/reserved-instance-purchase-recommendations>
- Advisor 비용 권고 — 예약·SP 권장 항목 (2026-02-10)  
  <https://learn.microsoft.com/azure/advisor/advisor-reference-cost-recommendations>
- FinOps Framework(Rate optimization)  
  <https://www.finops.org/framework/capabilities/rate-optimization/>
