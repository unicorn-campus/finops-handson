# 단위경제 KPI & 협업 실무 운영 플레이북

> **목적**: FinOps팀·PO·BA가 단위경제 KPI를 직접 정의하고 비용 리뷰를 운영하는 데 바로 쓰는 실행 절차·표·템플릿 제공  
> **독자**: FinOps Practitioner, Product Owner(PO), Business Analyst(BA), 비용 리뷰 참여 엔지니어·재무  
> **구성**: S1 단위경제 KPI 수립 / S2 Capability 구성요소 / S3 Persona별 역할 / S4 PO·BA의 FinOps 범위 / S5 타사 실행 사례  
> **주 범위(프레임워크)**: Domain = **Quantify Business Value**(Unit Economics·KPIs & Benchmarking) + **Manage the FinOps Practice**(FinOps Practice Operations)  
> **원본**: 교재 [`textbook/m6.단위경제KPI협업.md`](../textbook/m6.단위경제KPI협업.md)의 20분 압축 모듈을 실무 플레이북으로 확장  
> 📖 **1차 출처(FinOps Foundation, 2026-07-19 대조)**:
> [Domains](https://www.finops.org/framework/domains/) ·
> [Capabilities](https://www.finops.org/framework/capabilities/) ·
> [Principles](https://www.finops.org/framework/principles/) ·
> [Personas](https://www.finops.org/framework/personas/)

> ⚠️ **수치 라벨 규칙**: 본 문서의 목표 수치(95% · 5% · 70% · 10% 등)는 **교육·운영 예시(공식 아님)** 임.  
> FinOps Foundation 공식 수치가 아니며, 조직 성숙도·계약 형태에 맞게 반드시 조정함.

---

## S1 단위경제 KPI 수립

### WHY - 단위경제를 먼저 보는 이유
"총 비용 증가 ≠ FinOps 실패"임. 서비스가 성장하면 총 클라우드 비용은 당연히 늘어남.  
핵심 질문은 **"트래픽 1건(비즈니스 단위)을 처리하는 비용이 줄고 있는가?"** 임.  
이 관점은 FinOps 원칙 **"Business value drives technology decisions"** 와 정렬됨.  
프레임워크 위치: Domain = Quantify Business Value / Capability = Unit Economics + KPIs & Benchmarking.

### (a) "우리 서비스의 1건" 정의 워크숍 절차
단위경제의 출발점은 비즈니스 단위(트래픽/주문/MAU/추론 건 등)를 조직이 합의해 정의하는 것임.  
아래 절차를 90분 워크숍 1회로 운영함.

| 항목 | 내용 |
|------|------|
| 입력 | 서비스 비즈니스 모델, 매출·사용 지표(주문/MAU/API 호출 등), 아키텍처·비용 구조, 최근 3개월 비용 데이터 |
| 참여자 | PO(주관), FinOps Practitioner(퍼실리테이션), Engineering(측정 가능성 검토), Finance(가치·매출 연동 검토), BA(데이터 정의 지원) |
| 절차 | ① 후보 단위 브레인스토밍 → ② 판단기준 4개로 평가 → ③ 대표 단위 1 ~ 2개 선정 → ④ 측정 방법·데이터 소스 확정 → ⑤ 단위 정의서 승인 |
| 산출 | 단위 정의서(단위명 / 측정 방법 / 데이터 소스 / 계산 주기 / 승인자) |
| 판단기준 | ① 비즈니스 가치와 직결됨 ② 측정 가능함 ③ 클라우드 비용에 연동됨 ④ 이해관계자가 공감함(4개 모두 충족 시 채택) |

재사용 템플릿(단위 정의서):
```
[비즈니스 단위 정의서]
- 서비스명:
- 대표 단위(1건의 정의):        예) 결제 API 성공 트랜잭션 1건
- 측정 방법:                     예) APM의 결제성공 카운트(일 단위)
- 데이터 소스:                   예) APM 대시보드 / DB 집계
- 계산 주기:                     예) 월간(주간 트렌드 병행)
- 승인자 / 승인일:
```

### (b) 단위당 비용 계산 공식·데이터 소스·계산 주기
- 단위당 비용 = **총 클라우드 비용 ÷ 비즈니스 단위 수**(트래픽/주문/MAU/추론 건)  
- 자원당 비용(예) = **총 컴퓨트 비용 ÷ 총 vCPU 시간**  
- 해석 예시: 총비용 100만원 → 380만원(↑)이어도 단위당 비용 1.00 → 0.61원/건(↓)이면 효율 개선(건강한 성장)임.

| 계산 요소 | 데이터 소스 | 수집·계산 주기 |
|-----------|-------------|----------------|
| 총 클라우드 비용 | Cost Management, 비용 Export(스토리지) | 일 단위 수집 / 월 단위 확정 |
| 총 컴퓨트·vCPU 시간 | Cost Management(리소스별), 리소스 메타데이터 | 월간 |
| 비즈니스 단위 수 | 서비스 지표(APM·DB·애널리틱스) | 월간(주간 트렌드 병행) |
| 단위당 비용(산출) | 위 값의 나눗셈, 리포팅 대시보드 | 월간 확정, 주간 모니터링 |

### (c) 대표 KPI 6종 운영표

> **교육용 예시 수치(공식 아님).** 아래 목표값(95% / 5% / 70% / 10% 등)은 본 플레이북의 학습용 자체 기준이며  
> FinOps Foundation 공식 수치가 아님. 조직별 성숙도·계약 형태에 맞게 반드시 조정함.

| KPI | 정의 | 계산식 | 데이터 소스 | 목표 설정법 | 수집주기 | 오너 |
|-----|------|--------|-------------|-------------|----------|------|
| 태깅 커버리지 | 필수 태그가 부여된 리소스·비용 비율 | 태그 완비 비용 ÷ 총 비용 | Cost Management, 태그 메타데이터 | 현 수준 기준선 → 단계적 상향(예시 95%↑) | 주간 | FinOps Practitioner (Engineering 실행) |
| 미할당 비용 비율 | 팀·서비스에 귀속 못 한 비용 비율 | 미할당 비용 ÷ 총 비용 | Cost Management(할당 규칙) | 기준선 → 하향 목표(예시 5%↓) | 주간 | FinOps Practitioner |
| RI·SP 커버리지 | 약정으로 충당된 사용량 비율 | 약정 적용 사용량 ÷ 대상 사용량 | Cost Management(약정 리포트) | 워크로드 안정성 따라 설정(예시 70%↑) | 월간 | FinOps Practitioner (Finance 승인) |
| 단위당 비용 추세 | 비즈니스 단위 1건당 비용의 시계열 방향 | 총비용 ÷ 비즈니스 단위 수 | (b) 계산 결과 | "지속 하락" 방향 목표(수치 아닌 추세) | 월간 | Product(PO) + FinOps Practitioner |
| 예측 정확도(MAPE) | 비용 예측과 실적의 평균 절대 오차율 | Σ\|예측-실적\|/실적 ÷ n | Forecasting 산출, Cost Management 실적 | 오차 상한 목표(예시 10% 이내) | 월간 | Finance + FinOps Practitioner |
| 회수된 낭비(월) | 최적화로 절감·회수한 월 비용 | Σ(적용된 최적화 절감액) | 최적화 액션 로그, Cost Management | 절대 목표보다 추적·누적 관리 | 월간 | Engineering |

### (d) 조직별 목표 조정 가이드
1. **기준선 먼저** - 목표값을 바로 정하지 말고 최근 3개월 실적으로 현 수준(baseline)을 측정함.  
2. **방향과 스텝** - baseline 대비 분기별 개선 스텝을 설정함(예: 태깅 70% → 80% → 90%).  
3. **성숙도 반영** - Crawl(가시화 시작) 단계는 커버리지·정확도 KPI 우선, Run 단계는 단위당 비용·예측 정확도 우선.  
4. **계약·환경 반영** - 약정 커버리지 목표는 워크로드 안정성·계약 형태에 따라 달리 설정함(변동 워크로드는 낮게).  
5. **추세형 KPI 주의** - 단위당 비용·회수된 낭비는 절대 목표보다 추세·누적으로 관리함.

---

## S2 Capability 구성요소

### 4 Domain × Capability 전체 지도
FinOps Framework는 4개 Domain과 그 하위 Capability로 구성됨. 아래는 현행 프레임워크 전체 목록과 요약 설명임.  
(각 Capability의 권위 있는 정의는 [Capabilities](https://www.finops.org/framework/capabilities/) 참조)

| Domain | Capability | 한 줄 설명(요약) |
|--------|-----------|------------------|
| **Understand Usage & Cost** (보이기) | Data Ingestion | 비용·사용 데이터를 수집·정규화해 분석 기반 마련 |
| | Allocation | 태그·계정 기준으로 비용을 팀·서비스에 귀속 |
| | Reporting & Analytics | 비용을 대시보드·리포트로 가시화·분석 |
| | Anomaly Management | 이상 비용을 탐지·알림·대응 |
| **Quantify Business Value** (가치화) ★ | Planning & Estimating | 워크로드 비용을 사전 계획·추정 |
| | Forecasting | 미래 비용을 예측 |
| | Budgeting | 예산을 수립·집행·관리 |
| | **KPIs & Benchmarking** ★ | 핵심 지표·벤치마크로 성과 측정 |
| | **Unit Economics** ★ | 비즈니스 단위당 비용으로 효율 측정 |
| **Optimize Usage & Cost** (줄이기) | Architecting for Cloud & Workload Placement | 아키텍처·배치 최적화 |
| | Rate Optimization | 약정·요율로 단가 절감 |
| | Usage Optimization | 유휴·과다 자원 축소 |
| | Cloud Sustainability | 지속가능성(탄소·효율) 관점 최적화 |
| | Licensing & SaaS | 라이선스·SaaS 비용 최적화 |
| **Manage the FinOps Practice** (체계화) ★ | **FinOps Practice Operations** ★ | FinOps 운영·리뷰·협업 체계 운영 |
| | Cloud Policy & Governance | 정책·거버넌스 수립·집행 |
| | FinOps Assessment | 성숙도·역량 자가진단·평가 |
| | FinOps Tools & Services | 도구·서비스 선정·운영 |
| | FinOps Education & Enablement | 교육·역량 강화 |
| | Invoicing & Chargeback | 청구·차지백 |
| | Intersecting Disciplines | 인접 영역(ITAM·ITFM·보안 등) 연계 |

★ = 본 플레이북의 주 범위: **Quantify Business Value**(Unit Economics·KPIs & Benchmarking)와  
**Manage the FinOps Practice**(FinOps Practice Operations, 정렬 원칙 "Teams need to collaborate")임.  
Understand·Optimize 영역은 KPI의 데이터 소스·개선 액션으로 본 플레이북과 연결됨.

### 자가진단 → KPI 우선순위 연결 절차
FinOps Assessment 관점의 간이 스코어링으로 우리 조직의 강·약 Capability를 파악하고 KPI 우선순위로 연결함.

**1) 스코어링 방법** - 각 Capability를 0 ~ 3점으로 자가평가함.  
- 0점: 활동 없음 / 1점: 수작업·비정기 / 2점: 부분 자동·정기 / 3점: 자동화·성숙  

**2) 도메인별 평균** - Capability 점수를 Domain별로 평균해 강·약 영역을 식별함.  

**3) 약한 Capability → 우선 KPI 연결표**

| 약한 Capability(저점수) | 우선 착수 KPI |
|-------------------------|----------------|
| Allocation / Data Ingestion | 태깅 커버리지, 미할당 비용 비율 |
| Reporting & Analytics | 단위당 비용 추세(가시화 우선) |
| Forecasting / Budgeting | 예측 정확도(MAPE) |
| Rate Optimization | RI·SP 커버리지 |
| Usage Optimization | 회수된 낭비(월) |
| Unit Economics / KPIs & Benchmarking | 단위당 비용 정의·측정 착수 |

**4) 재사용 템플릿(자가진단 시트)**
```
Capability            | 점수(0-3) | 근거/현황     | 우선 착수 KPI
----------------------|-----------|--------------|----------------
Allocation            |           |              |
Reporting & Analytics |           |              |
Forecasting           |           |              |
Rate Optimization     |           |              |
Unit Economics        |           |              |
...
=> Domain 평균이 낮은 2개 Domain의 저점수 Capability부터 KPI 착수
```

---

## S3 Persona별 역할

### 페르소나 구성(공식)
FinOps Foundation은 페르소나를 Core와 Allied로 구분함. (출처: [Personas](https://www.finops.org/framework/personas/))  
- **Core(6)**: FinOps Practitioner / Engineering / Finance / Product / Procurement / Leadership  
- **Allied(5)**: ITAM / ITFM / Sustainability / ITSM·ITIL / Security  

본 플레이북의 협업 정렬 원칙은 **"Teams need to collaborate"**, 운영 Capability는 **FinOps Practice Operations**임.  
운영 모델은 **"정책은 중앙, 실행은 각 팀"**(연합 거버넌스)이며, 이는 원칙 **"FinOps should be enabled centrally"** 와  
Capability **Cloud Policy & Governance** 로 뒷받침됨.

> 참고: m6 원본의 "Product Owner"는 공식 **Product** 페르소나에 대응함(상세는 S4).

### KPI 수립·비용 리뷰 운영 RACI 매트릭스
범례: **R**=실행(Responsible) / **A**=최종책임(Accountable, 활동당 1명) / **C**=자문(Consulted) / **I**=통보(Informed)  
열은 Core 6 페르소나임. 관련 Allied 페르소나는 표 아래 별도 표기함.

| 활동 | FinOps Practitioner | Engineering | Finance | Product(PO) | Procurement | Leadership |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| 비즈니스 단위 정의 | R | C | C | **A** | - | I |
| 단위경제 KPI 정의·목표 설정 | R | C | C | **A** | - | I |
| 태깅·할당 정책 수립 | **A**/R | C | I | I | - | I |
| 비용 데이터 수집·정확성 보장 | **A** | R | I | - | - | - |
| 주간 이상비용·최적화 리뷰 | **A** | R | - | I | - | - |
| 월간 예산 대비 실적·KPI 리뷰 | R | C | **A** | C | - | I |
| 분기 전략 정렬·투자 의사결정 | R | I | C | C | I | **A** |
| 최적화 액션 실행 | C | **A**/R | I | I | - | I |
| 약정(RI·SP) 구매 의사결정 | R/C | C | **A** | I | R | I |

Allied 페르소나 연계(자문 C):  
- 태깅·할당 정책, 약정 구매 → **ITAM**(자산·라이선스), **ITFM**(재무 연계)  
- 최적화·아키텍처 → **Sustainability**(탄소·효율), **Security**(보안 제약)  
- 운영 리뷰·프로세스 → **ITSM·ITIL**(변경·서비스 관리)

### 비용 리뷰 주기 운영표
FinOps Practice Operations의 핵심 루틴임. 주기가 올라갈수록 참석 범위·의사결정 권한이 확장됨.

| 주기 | 참석 | 주요 안건 | 산출물 | 의사결정 범위 |
|------|------|-----------|--------|----------------|
| **주간** | FinOps팀 + 엔지니어 | 이상비용 점검, 즉시 가능한 최적화 액션 | 액션 아이템(오너·기한), 이상비용 로그 | 실행 가능한 최적화 즉시 승인 |
| **월간** | + 재무 + 사업(Product) | 예산 대비 실적, KPI, 비용 트렌드 | KPI 대시보드, 예산 편차 리포트 | 예산 재배분, KPI 목표 조정 |
| **분기** | + 경영진(Leadership) | 전략 정렬, 투자 우선순위, 성숙도 | 성숙도 평가, 투자 로드맵, 약정 방향 | 전략 투자·약정 방향 결정 |

재사용 템플릿(리뷰 어젠다):
```
[○○ 리뷰 - YYYY-MM-DD]
1) 지표 리뷰: 태깅/미할당/단위당 비용/MAPE/회수 낭비
2) 이슈: 이상비용 · 예산 편차 · 목표 미달 KPI
3) 액션: {내용 / 오너 / 기한 / 관련 Capability}
4) 의사결정: {안건 / 결정 / 책임자(A)}
5) 다음 리뷰까지 팔로업
```

---

## S4 PO·BA의 FinOps 범위

### 페르소나 매핑 - 정직한 구분
- **PO(Product Owner)** → FinOps Foundation **공식 Product 페르소나**에 대응함.  
  Product 페르소나는 비즈니스 가치 기반 의사결정을 담당함(원칙 "Business value drives technology decisions").  
- **BA(Business Analyst)** → **FinOps Foundation 공식 페르소나가 아님**.  
  공식 페르소나 목록(Core 6 + Allied 5, [Personas](https://www.finops.org/framework/personas/))에 BA는 포함되지 않음.  
  BA는 **조직 역할**로서 Product·FinOps Practitioner의 Capability(특히 Unit Economics·데이터 정의)를 실무에서 지원하는 위치임.  
  본 문서에서 BA에게 부여하는 책임은 조직 실무 관행이며, 프레임워크 공식 정의가 아님을 명확히 함.

### 역할별 FinOps 책임
| 역할 | 페르소나 지위 | 핵심 FinOps 책임 |
|------|----------------|-------------------|
| PO | 공식 Product 페르소나 | ① 단위경제 KPI 정의('우리 서비스의 1건'은?) ② 비용을 백로그 우선순위에 반영 |
| BA | 비공식(조직 역할) | ① 요구사항에 비용 영향 분석 포함(Shift-Left) ② 단위경제 데이터 정의·검증 |

### Shift-Left FinOps 실행 절차
비용 고려를 개발 후반이 아니라 요구사항·설계 단계로 앞당기는(Shift-Left) 절차임.

1. **요구사항 정의** - BA가 각 요구사항에 "비용 영향" 항목을 필수 추가함(신규 리소스·트래픽·저장량 증가 여부).  
2. **설계 리뷰** - 예상 단위당 비용 영향을 추정해 설계안에 첨부함(Engineering·FinOps Practitioner 자문).  
3. **백로그 우선순위** - PO가 아래 판단 템플릿으로 비용/가치를 반영해 우선순위를 조정함.  
4. **릴리스 후 검증** - 실제 단위당 비용을 측정해 추정과 비교하고, 차이를 다음 정의에 반영(BA가 데이터 검증).

### 백로그 우선순위 판단 체크리스트
각 기능/스토리를 백로그에 넣기 전 아래를 점검함(하나라도 "확인 필요"면 설계·데이터 보완 후 재검토).
- [ ] 이 기능이 창출하는 **비즈니스 가치**가 명확한가?  
- [ ] 예상되는 **단위당 비용 영향**이 산정되었는가?(증가 / 중립 / 감소)  
- [ ] **자원당(vCPU당) 가치**가 정당한가? - "이 기능, vCPU당 가치 있나?"  
- [ ] 비용 데이터(단위 정의·측정 방법)가 **검증 가능**한가?  
- [ ] 대안(더 저렴한 설계·배치)을 **비교**했는가?

재사용 템플릿(비용-가치 판단 시트):
```
[백로그 항목 비용-가치 판단]
- 항목명:
- 비즈니스 가치(1-5):
- 예상 단위당 비용 영향:   [ 증가 / 중립 / 감소 ]
- 자원당 가치 판단:        "이 기능, vCPU당 가치 있나?" → [ 예 / 아니오 / 확인 필요 ]
- 데이터 검증 가능(BA):    [ 가능 / 확인 필요 ]
- 결정:                    [ 진행 / 보류 / 재설계 ]
- 근거 한 줄:
```

---

## S5 타사 실행 사례

> **검증 원칙**: 접근 가능한 출처가 있는 사례만 수록함. 정량 수치는 출처에 명시된 값만 기재하고,  
> 미공개 시 "정량 성과 비공개"로 표기함. 사례별 출처 신뢰도·한계는 절 말미 표에 정리함.

### 국내

#### 우아한형제들(배달의민족)
CUR·CMDB·AD 조직정보 등 10개 이상 데이터 소스를 통합한 사내 FinOps 플랫폼을 구축해  
조직별 비용 자동 할당과 RI/SP 할인 공정 배분을 구현함.  
단위경제 KPI로 "주문 1건 처리 비용", "API 1건당 처리 비용", "트래픽 1GB 처리 비용" 등을 설계해 개발팀에 공유함.  
Slack 기반 일간 비용 알림·이상탐지·조직별 주간 비용 메일로 비용 인식 문화를 형성함.  
정량 성과: EC2 서비스 최대 30%+ 자동 Scale-in(유휴 축소), Tagging 비율 90%+ 유지, RI/SP 커버리지 70% 목표 관리  
(단위비용 추이는 블로그 그래프로만 제시, 구체 수치 미공개).
- **교훈**: "비용 절감 목표"보다 "API 1건당 처리 비용 추이" 같은 단위경제 지표를 제시할 때 개발팀 자발성이 높아짐.  
  중앙 FinOps 조직의 RI/SP 통합 구매·배분이 조직 분산 구매보다 효율적임.
- **출처**: [우아한 Cloud FinOps 여정 | 우아한형제들 기술블로그](https://techblog.woowahan.com/22855/) (2025-09-26)

#### 쿠팡(Coupang)
클라우드 인프라 엔지니어·기술 프로그램 매니저·파이낸스가 협력하는 중앙 팀을 구성해 도메인별 온디맨드 비용을 관리함.  
CloudWatch·Athena·BI로 AWS CUR 기반 대시보드를 구축하고, AMD CPU 전환, 50PB+ S3 Intelligent-Tiering 이전,  
비프로덕션 자동 시작/정지 등을 실행함. 파이낸스=예산 준수, 엔지니어링=기술 최적화로 역할을 분리함.  
정량 성과: 비프로덕션 25% 절감, EMR 25% 절감, AMD 인스턴스 가격 성능 20%↑, 2021년 온디맨드 기준 "수백만 달러" 절감.
- **교훈**: S3 최적화는 오브젝트 크기가 Intelligent-Tiering 효과를 좌우하므로 사전 테스트 필수.  
  "도메인 팀이 비용을 소유"하도록 교육하는 것이 지속 최적화의 기반임.
- **출처**: [Cloud expenditure optimization for cost efficiency | Coupang Engineering Blog](https://www.coupang.jobs/en/life-at-coupang/engineering-blog/cloud-expenditure-optimization-for-cost-efficiency/) (2024-03-21)

#### 인프랩(인프런)
학습 플랫폼 스타트업의 DevOps 파트가 FinOps 도입으로 연간 약 $300,000(약 3.99억 원)를 절감한 사례.  
MediaConvert → 자체 구축(AWS Batch+ECS+ffmpeg) 15 ~ 20배 절감, CircleCI → Jenkins 최대 4.5배 절감,  
ECS Fargate → EC2 월 약 $1,500 절감, RDS Aurora Scale-Down·슬로 쿼리 최적화, RI/SP 10 ~ 30% 추가 절감.  
월간 비용 지표 리뷰 회의·최적화 현황표·Jira 티켓 기반 과제 관리로 팀 협업 체계를 운영함.
- **교훈**: 초기 빠른 출시를 위한 매니지드 서비스는 조직 성장 후 "적정 기술 선택" 관점에서 재검토 필요.  
  비용 최적화는 한 번에 끝나지 않으며 지속적 관심·팀 문화가 전제됨.
- **출처**: [스타트업 엔지니어의 AWS 비용 최적화 경험기 | 인프랩 기술블로그](https://tech.inflab.com/20240227-finops-for-startup/) (2024-02-27)

> **국내 공개 사례 한계**: 카카오·토스·당근·라인 등의 FinOps·단위경제 관련 공개 사례는  
> 본 검색 범위에서 접근 가능한 1차 출처를 확보하지 못해 제외함.

### 글로벌

#### Airbnb
2020년 팬데믹 예약 급감 위기에서 클라우드 비용 효율화를 집중 운영함.  
S3 보존 정책 강화·저비용 티어(Glacier 등) 전환, Kubernetes Cluster Autoscaler 도입, Convertible Savings Plan 구매를 실행함.  
거버넌스: CFO 주관 "재무 규율 표창", 비용 절감 해커톤, 상품팀별 "AWS 비용 챔피언" 지정.  
정량 성과: 2020년 9개월 기준 호스팅 비용 연 $63.5M 감소, 매출원가 26% 하락 기여.
- **교훈**: 위기 상황이 오히려 엔지니어링·파이낸스·경영진의 비용 협업 문화를 신속히 형성함.  
  조직적 인센티브(해커톤·표창)와 기술적 자동화(Autoscaler)를 병행해야 지속성이 확보됨.
- **출처**: [Our Journey Towards Cloud Efficiency | Airbnb Tech Blog](https://medium.com/airbnb-engineering/our-journey-towards-cloud-efficiency-9c02ba04ade8) (2021-04-06)

#### Spotify
내부 개발자 플랫폼(Backstage)에 **Cost Insights** 플러그인을 직접 개발·통합해 엔지니어 주도 비용 관리 문화를 구현함.  
비용을 클라우드 제공사 언어 대신 "팀·서비스·제품" 언어로 표현하고, 비용 대 비즈니스 지표(DAU당 비용 등) 그래프·임계값 알림을 제공함.  
하향식 절감 지시가 아닌 "엔지니어가 자신의 클라우드 비용을 소유"하는 문화로 전환해 스쿼드 단위 자율 최적화를 유도함.  
정량 성과: "수개월 내 수백만 달러 절약"(공식 수치 미공개 → 정량 성과 비공개).
- **교훈**: 클라우드 제공사 언어 대신 조직 내부 언어(팀·서비스·비즈니스 지표)로 비용을 표현할 때 엔지니어 참여도가 높아짐.  
  단위경제(DAU당 비용) 시각화가 자발적 최적화의 트리거가 됨.
- **출처**: [Cost Engineering at Spotify | Spotify Engineering Blog](https://engineering.atspotify.com/2020/09/managing-clouds-from-the-ground-up-cost-engineering-at-spotify) (2020-09-28) ·
  [Cost Insights plugin | Backstage Blog](https://backstage.io/blog/2020/10/22/cost-insights-plugin/)

#### Just Eat Takeaway(JET)
복수 기업 M&A 후 태깅 불일치로 인프라 비용의 45%가 미할당이던 문제를 FinOps 플랫폼 도입으로 해결함.  
Virtual Tags API로 일간 비용 할당을 자동화하고, FinOps 팀이 250개+ 표준화 대시보드를 배포해  
엔지니어가 복제·즉시 사용하는 셀프서비스 환경을 구축함. "shift-left"로 개발 초기에 재무 검토를 통합함.  
정량 성과: 플랫폼 활성 사용자 40명 → 700명+(약 20배), 미할당 지출 45% → 21%(24%p↓), 대시보드 250개+ 배포.
- **교훈**: M&A 후 비용 거버넌스 공백은 "셀프서비스 대시보드 + 표준 태깅 모델"로 단기에 가시성을 확보 가능함.  
  성공 지표는 도구 수가 아니라 "비용 데이터를 의사결정에 쓰는 사람 수"임.
- **출처**: [Just Eat Takeaway.com & Finout Success Story | AWS Case Studies](https://aws.amazon.com/solutions/case-studies/justeattakeaway-finout/)

#### Nubank
브라질 핀테크가 45개+ 비즈니스 유닛의 클라우드 사용량 최적화를 거버넌스 의사결정 주제로 격상시킨 사례.  
3,000개 DynamoDB 테이블 스토리지 클래스 최적화 자동화(중앙 플랫폼 팀 GraphQL API), 미사용 EBS 자동 정리, 옵트아웃 책임 모델을 도입함.  
재무 엔지니어링(요율·약정)과 사용량 최적화(효율 엔지니어링)를 별도 조직으로 분리 운영함.  
정량 성과: 총 클라우드 지출 8 ~ 12% 절감(청구서 대조 검증), DynamoDB 단독 최적화로 총지출 1% 절감,  
불필요 EBS 볼륨 해결 기간 210일 → 16일, 해결 시간 72% 단축, 92% 유닛이 자체 엔지니어 개입 없이 수혜.
- **교훈**: "추정 절감액"이 아닌 청구서 대조 검증 수치가 리더십 신뢰의 핵심임.  
  중앙 플랫폼 팀이 자동화를 주도하면 개별 팀 개입 없이도 92% 조직에 효과가 전파됨.
- **출처**: [How Nubank made usage optimization a boardroom topic | PointFive Blog](https://www.pointfive.co/blog/how-nubank-made-usage-optimization-a-boardroom-topic) (2026-07-16)

### 사례 출처 신뢰도·한계

| 사례 | 출처 유형 | 신뢰도 | 한계 |
|------|-----------|--------|------|
| 우아한형제들 | 1차 공식 기술블로그 | 높음 | 단위비용 수치는 그래프로만 제시, 본문 구체 수치 없음 |
| 쿠팡 | 공식 엔지니어링 블로그 | 높음 | 2021년 절감 총액은 "수백만 달러"로만 표현 |
| 인프랩 | 1차 공식 기술블로그 | 높음 | 스타트업 단일 사례, 대규모 조직 일반화 시 주의 |
| Airbnb | 1차 공식 기술블로그 | 높음 | 2020년 위기 특수 맥락, $63.5M은 매체 보도 교차 확인 |
| Spotify | 1차 공식 엔지니어링 블로그 | 높음 | 정량 성과 수치 미공개("millions saved") |
| Just Eat Takeaway | AWS 공식 사례(파트너 Finout) | 중간 | AWS·Finout 공동 게재(마케팅 성격), 수치는 명확 |
| Nubank | 벤더(PointFive) 블로그, 담당자 인용 포함 | 중간 | 2차 출처, 수치는 "청구서 대조 검증"으로 명시 |

> **한국 현실 참고**: 국내 다수 조직은 성숙도 Crawl ~ Walk 단계이며, 국내+글로벌 CSP 멀티클라우드로 통합 복잡도가 높음.  
> 위 국내 사례(우아한형제들·쿠팡·인프랩)는 단위경제·중앙 거버넌스·셀프서비스 문화를 앞서 실천한 참고 모델임.

---

## 실무 실행 체크리스트

- [ ] **비즈니스 단위(1건)** 정의 워크숍 완료 및 단위 정의서 승인(S1-a)  
- [ ] **단위당 비용** 계산 파이프라인 구성(데이터 소스·주기 확정)(S1-b)  
- [ ] **KPI 6종** 오너·목표·수집주기 지정(교육용 수치는 baseline 기준 재설정)(S1-c/d)  
- [ ] **Capability 자가진단** 후 약한 영역 → 우선 KPI 연결(S2)  
- [ ] **RACI 매트릭스**를 우리 조직 페르소나로 확정(활동당 A 1명)(S3)  
- [ ] **비용 리뷰 주기**(주/월/분기) 참석·안건·산출물 설계(S3)  
- [ ] **PO/BA 역할** 정의(PO=Product 페르소나, BA=조직 역할)(S4)  
- [ ] **Shift-Left** 요구사항 템플릿에 "비용 영향" 항목 반영(S4)  

## 예상 Q&A

- **"비용이 늘었는데 잘한 건가요?"** → 트래픽이 더 빨리 늘어 *단위당 비용이 줄면* 효율 개선 = 건강.  
- **"단위(1건)는 무엇으로?"** → 비즈니스마다 다름: 이커머스=주문, SaaS=MAU, AI=추론 요청. PO가 정의(S1-a).  
- **"FinOps팀이 다 해야 하나요?"** → ❌. 중앙은 정책·도구, 실행은 각 팀(연합 거버넌스). 엔지니어 오너십 필수(원칙 "Everyone takes ownership for their technology usage").  
- **"BA도 공식 FinOps 페르소나인가요?"** → ❌. 공식 페르소나는 Core 6 + Allied 5. BA는 조직 역할로 Product·FinOps Practitioner를 지원(S4).  
- **"목표 수치(95%·70% 등)는 그대로 쓰면 되나요?"** → ❌. 교육용 예시일 뿐, 반드시 baseline 측정 후 조직에 맞게 재설정(S1-d).

## 부록 — 1차 출처 및 참고

- FinOps Foundation Framework — [Domains](https://www.finops.org/framework/domains/) ·
  [Capabilities](https://www.finops.org/framework/capabilities/) ·
  [Principles](https://www.finops.org/framework/principles/) ·
  [Personas](https://www.finops.org/framework/personas/)  
- 원본 교재: [`textbook/m6.단위경제KPI협업.md`](../textbook/m6.단위경제KPI협업.md)  
- 관련 교재: [`finops-on-azure/03.비즈니스-가치-정량화.md`](../finops-on-azure/03.비즈니스-가치-정량화.md)  
- S5 사례 출처: 각 사례 하단 링크 및 "사례 출처 신뢰도·한계" 표 참조
