# 적정 SKU 판단법

## 왜 적정 SKU 판단이 필요한가

클라우드 리소스는 "필요할 때 즉시 크게" 잡기 쉬워 실제 사용량 대비 과도한 SKU가 방치되는 경우가 많음.  
과대 프로비저닝(Over-provisioning)은 쓰지 않는 vCPU·메모리에 매달 요금을 내는 직접적 낭비이며,  
과소 프로비저닝(Under-provisioning)은 성능 저하·장애로 이어지는 리스크임.  
적정 SKU 판단(Right-sizing)은 FinOps Optimize 단계의 핵심 역량으로,  
"측정된 사용량에 근거해 리소스 크기를 워크로드에 맞추는" 활동임. 감이 아닌 메트릭이 판단 근거가 되어야 함.

## 판단 프로세스 개관

적정 SKU 판단은 아래 4단계 흐름을 고정 순서로 수행함.

| 단계 | 활동 | 산출물 |
|------|------|--------|
| STEP 1 | 메트릭 관찰 (Azure Monitor) | 관찰 기간별 CPU·메모리·IO·네트워크 P95 값 |
| STEP 2 | Azure Advisor 권고 확인 | Right-size·유휴 리소스 권고 목록(참고치) |
| STEP 3 | 적정 SKU 3대안 비교 | 월비용·성능마진·리스크 동일 기준 비교표 |
| STEP 4 | 선정·축소 적용·재검증 | 선정 SKU + 재관찰 결과 |

- STEP 1 ~ STEP 2는 "현재 상태 진단", STEP 3 ~ STEP 4는 "대안 선택과 검증"에 해당함.
- 단발 스냅샷 한 번으로 STEP 3까지 건너뛰지 말 것 — 관찰이 부실하면 이후 판단 전부가 오염됨.

## STEP 1. 메트릭 관찰

리소스가 실제로 얼마나 쓰이는지 데이터로 확인하는 단계임.  
포털 경로: **Azure Portal > 대상 리소스(VM 또는 SQL DB) > 모니터링 > 메트릭**  
(또는 **Azure Monitor > 메트릭 > 범위(Scope)에서 대상 리소스 선택**)

> 📷 (스크린샷 위치: VM 리소스 블레이드 좌측 "모니터링 > 메트릭" 진입 화면)

### 관찰 기간·집계 방식 (모든 리소스 공통)

- **관찰 기간**: 최소 1 ~ 2주. 월말 배치·주간 피크 등 **피크 시간대가 반드시 포함**되어야 함.
- **집계 방식**: 평균(Avg)만으로 판단 금지. **P95(95백분위수)를 주 판단값**으로 사용하고,  
  Max(순간 피크)·Avg(기저 부하)를 병행 확인함.
- P95를 쓰는 이유: 극단적 순간 피크(Max)에 과잉 대응하지 않으면서,  
  실사용의 상한을 놓치지 않는 균형점이기 때문임.
- **CPU 단일 지표 판단 금지**: 메모리·디스크 IO·네트워크를 병행 관찰.  
  실무에서는 CPU보다 메모리·IO 병목이 더 흔함.

### VM 관찰 메트릭

| 메트릭명 | 의미 | 판단 임계 가이드(예시) |
|----------|------|------------------------|
| Percentage CPU | vCPU 사용률 | P95 < 40% 지속 시 축소 검토 / P95 > 80% 시 축소 보류 |
| Available Memory Bytes | 사용 가능 메모리(여유분) | 여유가 총량의 50% 이상 지속 시 메모리 과대 의심 |
| Disk Read/Write Operations/Sec (IOPS) | 디스크 초당 IO 횟수 | SKU·디스크 티어 IOPS 한도 대비 여유율 확인 |
| Disk Read/Write Bytes/Sec | 디스크 처리량(throughput) | 처리량 한도 근접 시 축소 시 병목 위험 |
| Network In/Out Total | 네트워크 수신·송신량 | 대역폭 상한 대비 여유 확인 |

- 메모리는 플랫폼 메트릭 **Available Memory Bytes**(호스트 관점 여유 바이트)로 게스트 에이전트 없이도 관찰 가능함.  
  단, 사용률(%)·커밋 등 정밀 지표는 게스트 OS 에이전트(Azure Monitor Agent) 설치 시 더 정확함.

> 📷 (스크린샷 위치: Percentage CPU 메트릭에 집계를 "P95/95th percentile"로 설정하고 기간을 지난 14일로 지정한 차트)

### Azure SQL Database 관찰 메트릭

| 메트릭명 | 모델 | 의미 | 판단 임계 가이드(예시) |
|----------|------|------|------------------------|
| CPU percentage | vCore·DTU 공통 | 컴퓨팅 사용률 | P95 < 40% 지속 시 축소 검토 |
| Data IO percentage | vCore·DTU 공통 | 데이터 파일 IO 사용률 | 100% 근접 잦으면 IO 병목 → 축소 보류 |
| Log IO percentage | vCore·DTU 공통 | 트랜잭션 로그 IO 사용률 | 쓰기 부하 지표, 100% 근접 주의 |
| SQL Server process memory percent (sql_instance_memory_percent) | vCore | 메모리 사용률 | 지속 저활용 시 vCore 축소 근거 |
| DTU Percentage (dtu_consumption_percent) / DTU used (dtu_used) | DTU | DTU 종합 소비율(%)·사용량 | P95 기준 상위 티어 축소 검토 |

- vCore 모델은 CPU·메모리·IO를 개별 지표로 분해 관찰 가능함(축소 판단이 정밀함).
- DTU 모델은 CPU·IO·메모리를 묶은 DTU Percentage 단일 지표 중심이라, 병목 원인 분해가 어려움.
- 서버리스(Serverless) 계층은 앱 메모리 사용률 지표 **App memory percentage (app_memory_percent)** 를 별도 제공함.

## STEP 2. Azure Advisor 권고 확인

관찰과 별개로, Azure가 자동 분석한 권고를 "참고치"로 확인하는 단계임.  
포털 경로: **비용 관리 + 청구 > Advisor(권장 사항)** 또는 **Advisor > 비용(Cost) 탭**

> 📷 (스크린샷 위치: Advisor > 비용 탭의 권장 사항 목록 화면)

### 대표 권고 유형

| 권고 유형 | 대상 | 내용 |
|-----------|------|------|
| VM/VMSS 크기 조정(Right-size) | VM, 가상 머신 확장 집합 | 저활용 인스턴스에 더 작은 SKU 제안 |
| 미사용 리소스 종료·삭제 | VM, 공용 IP, 디스크 등 | 유휴(idle) 리소스의 중지·삭제 제안 |
| 예약·절감 계획 전환 | 상시 가동 리소스 | RI(예약)·Savings Plan 구매로 요금 절감 제안 |

### 동작 원리와 한계

- Advisor는 실제 리소스 사용 telemetry를 **기본 7일**(구성에서 7/14/21/30/60/90일로 조정 가능, 반영 최대 48시간)  
  관찰해 "저활용·유휴" 패턴을 탐지함. 관찰 데이터 부족·대상 리소스 미사용 시 권장 사항이 비어 있을 수 있음.
- **크기 조정(Right-size) 권고는 CPU·아웃바운드 네트워크 사용률만 근거로 하며 메모리는 고려하지 않음.**  
  메모리 병목·고사용 워크로드는 Advisor가 축소를 권해도 STEP 1의 메모리 P95를 반드시 별도 확인해야 함.
- Advisor 권고는 **참고치일 뿐, 자동 적용 대상이 아님**. 적용 전 아래를 반드시 재확인함.
  - 재부팅·재배포로 인한 **다운타임** 허용 여부
  - 향후 부하 증가·이벤트를 감안한 **성능 여유(헤드룸)**
  - 배치·피크 등 관찰 창에 안 잡힌 **워크로드 특성**

## STEP 3. 적정 SKU 3대안 비교·선정

관찰값(STEP 1)과 Advisor 권고(STEP 2)를 근거로, 후보 SKU 3개를 동일 기준으로 비교해 1개를 선정함.

### 비교 기준 정의 (4개 축을 동일하게 적용)

| 기준 | 설명 |
|------|------|
| 월 예상 비용 | 리전·환율·시점별 상이한 **예시값** 기준. 상대 비교용 |
| vCPU · 메모리 | SKU가 제공하는 컴퓨팅·메모리 스펙 |
| 예상 성능마진(헤드룸) | 관찰 P95 대비 SKU 용량의 여유율(권장 20 ~ 30% 이상) |
| 리스크 · 다운타임 | 변경 시 재부팅·재배포 필요 여부, 병목 발생 가능성 |

> 아래 비용은 모두 **예시값(실제는 리전·환율·시점별 상이)** 이며, 상대 비교 학습용임.  
> 실제 판단 시 반드시 현재 시점 Azure 가격 계산기(Pricing Calculator)로 재확인할 것.

### VM 3대안 비교 예시

관찰 결과 가정: 현재 Standard_D4s_v5(4 vCPU / 16GB), CPU P95 약 22%, 메모리 사용 P95 약 4GB(사용률 25%, 여유 75%).

| 구분 | SKU | vCPU / 메모리 | 월 예상 비용(예시) | 헤드룸(CPU / 메모리) | 리스크 · 다운타임 |
|------|-----|---------------|--------------------|----------------------|-------------------|
| 현재 | Standard_D4s_v5 | 4 / 16GB | 기준(100%) | CPU 78% / 메모리 75% 유휴(과잉) | 없음(현행 유지) |
| 후보 A | Standard_D2s_v5 | 2 / 8GB | 약 50% 수준 | CPU 22%→44%(여유 56%) / 메모리 50%(여유 50%) | 크기 변경 재부팅 1회 |
| 후보 B | Standard_B2ms | 2 / 8GB | 약 35% 수준 | 메모리 여유 50%이나 CPU 버스트형(지속 고부하 부적합) | 재부팅 + 버스트 크레딧 소진 리스크 |

- 후보 A: CPU·메모리 모두 헤드룸 20 ~ 30% 이상 확보 → 범용(D-계열) 유지로 성능 예측 용이한 안정적 축소안.
- 후보 B: 버스트(B-계열)로 비용 최저이나, 지속 CPU 사용 시 크레딧 소진 → 성능 저하 리스크.  
  간헐적 저부하 워크로드에만 적합.
- **주의**: 메모리 P95가 높았다면(예: 12GB 사용) CPU가 낮아도 8GB SKU는 메모리 부족으로 탈락함.  
  CPU만 보고 축소하면 안 되는 이유(함정 "CPU만 보기")가 이 지점에서 드러남.

### Azure SQL Database 3대안 비교 예시

관찰 결과 가정: 현재 vCore 4(범용) CPU P95 약 30%, Data IO P95 약 25%.

| 구분 | 구성 | vCore / 계층 | 월 예상 비용(예시) | 헤드룸 | 리스크 · 다운타임 |
|------|------|--------------|--------------------|--------|-------------------|
| 현재 | 범용 vCore 4 | 4 / General Purpose | 기준(100%) | 과잉(P95 30%) | 없음 |
| 후보 A | 범용 vCore 2 | 2 / General Purpose | 약 50% 수준 | P95 30%→약 60%, 여유 확보 | 스케일 변경 시 짧은 연결 끊김 |
| 후보 B | 서버리스 vCore(자동 일시 중지) | 최대 2 / GP Serverless | 사용량 기반(간헐 부하 시 최저) | 부하 따라 자동 조정 | 콜드 스타트 지연, 상시 부하엔 불리 |

- vCore 축소(후보 A)는 스케일 변경 시 수 초 ~ 수십 초의 연결 재설정이 발생할 수 있음.
- 서버리스(후보 B)는 트래픽이 간헐적일 때 유리하나, 상시 부하나 지연 민감 워크로드엔 부적합.

### 선정 로직과 재검증 루프

1. **성능마진 기준 통과**: 관찰 P95를 대안 SKU에 대입했을 때 **헤드룸 20 ~ 30% 이상** 확보되는 후보만 남김.
2. **리스크·다운타임 수용 가능성 확인**: 재부팅·연결 끊김이 서비스 SLA 내 허용되는지 확인.
3. **동일 기준 비교로 최소 비용 후보 선정**: 위 2개를 통과한 후보 중 월 예상 비용이 가장 낮은 안을 선정.
4. **축소 적용 → 재관찰·재검증 루프**:
   - 선정안을 **비운영 환경 또는 저위험 시간대**에 적용함.
   - 적용 후 다시 **1 ~ 2주 재관찰**하여 P95·병목 지표가 안전 범위인지 검증함.
   - 성능 문제 발생 시 즉시 한 단계 상향 롤백, 여유가 여전히 크면 추가 축소 재검토함.
   - 한 번에 목표까지 크게 줄이지 말고 **점진적 축소 → 재검증**을 반복함.

## 판단 체크리스트

- [ ] 관찰 기간이 최소 1 ~ 2주이며 피크 시간대를 포함하는가
- [ ] 집계를 P95 기준으로 판단하고 Max·Avg를 병행 확인했는가
- [ ] CPU 외 메모리·디스크 IO·네트워크를 함께 관찰했는가
- [ ] Azure Advisor 권고를 "참고치"로만 취급하고 무조건 적용하지 않았는가
- [ ] 3대안을 월비용·성능마진·리스크의 동일 기준으로 비교했는가
- [ ] 선정안이 헤드룸 20 ~ 30% 이상을 확보하는가
- [ ] 변경 시 재부팅·연결 끊김 등 다운타임을 SLA 내에서 허용 가능한가
- [ ] 축소 적용 후 재관찰·재검증 루프 계획을 세웠는가
- [ ] 비용 수치가 예시값임을 인지하고 실제 가격 계산기로 재확인했는가

## 흔한 함정

| 함정 | 왜 위험한가 | 대응 |
|------|-------------|------|
| 단발 스냅샷 판단 | 특정 순간 부하만 보고 축소하면 피크에서 장애 발생 | 1 ~ 2주 관찰, 피크 포함 |
| 평균(Avg)만 보기 | 평균은 낮아도 피크가 높으면 성능 부족 | P95 주 판단 + Max 병행 |
| CPU만 보기 | 메모리·IO 병목을 놓쳐 축소 후 성능 저하 | 멀티 메트릭 병행 관찰 |
| Advisor 맹신 | 크기 조정 권고는 **CPU·아웃바운드 네트워크만 반영·메모리 미고려** → 메모리 병목 워크로드 축소 시 위험 | 메모리 P95 등 멀티 메트릭 병행, 다운타임·헤드룸 재확인 후 적용 |
| 다운타임 미고려 | 재부팅·연결 끊김이 운영 서비스에 영향 | 저위험 시간대·비운영 환경 우선 적용 |
| 한 번에 과도 축소 | 목표까지 급격히 줄이면 롤백 비용·리스크 증가 | 점진적 축소 → 재검증 루프 |

## 참고 출처 (References)

메트릭 표시명·Advisor 동작·SKU 스펙은 아래 Azure 공식 문서(MS Learn) 기준으로 검증함(검증일: 2026-07-19).

- Azure Monitor 지원 메트릭 — 가상 머신:  
  <https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-compute-virtualmachines-metrics>
- Azure Monitor 지원 메트릭 — Azure SQL Database:  
  <https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-sql-servers-databases-metrics>
- Azure SQL Database 메트릭·경고 모니터링:  
  <https://learn.microsoft.com/en-us/azure/azure-sql/database/monitoring-metrics-alerts?view=azuresql>
- Azure Advisor 비용 권고(VM/VMSS 크기 조정):  
  <https://learn.microsoft.com/en-us/azure/advisor/advisor-cost-recommendations>
- Azure VM Dsv5 시리즈 스펙:  
  <https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dsv5-series>
- Azure VM B-시리즈 버스터블 스펙:  
  <https://learn.microsoft.com/en-us/azure/virtual-machines/sizes-b-series-burstable>
- Azure VM B-시리즈 CPU 크레딧 모델:  
  <https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/b-series-cpu-credit-model>
- Azure SQL Database 서버리스 계층:  
  <https://learn.microsoft.com/en-us/azure/azure-sql/database/serverless-tier-overview?view=azuresql>
- Azure SQL Database 리소스 스케일(연결 중단):  
  <https://learn.microsoft.com/en-us/azure/azure-sql/database/scale-resources?view=azuresql>

> 비용 수치는 실시간 가격표를 대조하지 않은 **상대 비교용 예시값**임. 실제 판단 시 Azure 가격 계산기(Pricing Calculator)로 재확인 필요.
