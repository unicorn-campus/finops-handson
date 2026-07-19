# Spot 혼합 전략 (Spot VM · VMSS · Baseline+Spot Mix)

> 실습 환경: **EA(Enterprise Agreement)** · 리전 **Korea Central** 기준  
> 스크린샷: 각 캡처 지점에 `📷 스크린샷` 자리표시 콜아웃 표기 — 실습 중 실제 화면 캡처 후 교체  
> 1차 출처: Microsoft Learn(문서 하단 '용어·출처'에 링크 명시)  
> 📌 한 줄 요약(TL;DR): **미사용 용량을 큰 폭 할인가로 쓰되 언제든 축출(evict)됨 → 중단 허용 워크로드만 Spot →
> 표준 VM 베이스라인 위에 Spot을 얹어(Spot Priority Mix) 가용성과 절감을 동시에 확보** 순으로 설계함.

---

## 왜 Spot인가 (WHY)

- Spot VM은 Azure의 **미사용 용량(unused capacity)**을 온디맨드 대비 큰 폭 할인가로 제공함(출처: spot-vms).
- 대가로 **SLA·고가용성 보장 없음**. Azure가 용량을 회수하면 언제든 축출됨(축출 시 **30초 사전 통지**, best-effort).
- 따라서 Spot은 "싸지만 언제 꺼질지 모르는 컴퓨트"임 → **중단 허용·무상태 워크로드**에 한정해 쓰고,
  가동시간이 필요한 부분은 **표준 VM 베이스라인**으로 받쳐야 함(출처: spot-arch).
- FinOps **Optimize(요율 최적화)** 활동임. 낭비 제거(Right-size)를 먼저 하고, 남은 상시 부하는 약정,
  중단 허용 버스트 부하는 Spot으로 배치하는 것이 원칙임.

### 1차 출처 (MS Learn)

- Spot VM 개요: <https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms>
- Spot 포털 배포: <https://learn.microsoft.com/en-us/azure/virtual-machines/spot-portal>
- VMSS + Spot: <https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/use-spot>
- Spot Priority Mix: <https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/spot-priority-mix>
- Scheduled Events(회수 인지): <https://learn.microsoft.com/en-us/azure/virtual-machines/linux/scheduled-events>
- Spot 축출 아키텍처: <https://learn.microsoft.com/en-us/azure/architecture/guide/spot/spot-eviction>

---

## 파트 0. 사전 확인

**왜 이 단계인가**: Spot은 지원 조건·미지원 SKU가 명확함. 시작 전에 걸러야 실습 중 배포 실패를 피함.

- **계약 유형**: 본 실습은 **EA(Enterprise Agreement)** 기준. Spot은 EA·종량제(003P)·Sponsored 등에서 지원됨(출처: spot-vms).
  - ※ 주의: 오퍼 유형에 따라 Spot 가용 여부가 갈림. 본 문서는 **EA 전제**로 작성됨(MCA 등은 별도 확인 필요).
- **미지원 SKU 확인**: **B-시리즈는 Spot 미지원**. Promo 버전(Dv2/NV/NC/H promo 등)도 미지원(출처: spot-vms).
  - ※ 주의: 기존 VMSS 실습(`hands-on/03.policy.md`)은 **B1s**를 사용함 → **Spot 실습에는 사용 불가**.
    Spot 실습은 **D-시리즈(예: `Standard_D2s_v5`)** 등 지원 SKU로 교체해야 함.
- **리전**: 본 실습 리전은 **Korea Central**. Spot은 21Vianet(중국) 제외 모든 리전에 배포 가능(출처: spot-vms).
- **워크로드 전제**: **무상태(stateless)·중단 허용(interruptible)** 워크로드만 대상으로 함.
  상태 저장·상시 가동이 필요한 부분은 Spot에 올리지 않음.

> 📷 스크린샷: 구독 오퍼 유형(EA) 확인 화면

---

## 파트 1. Spot VM 단일 생성 (포털)

**왜 이 단계인가**: 혼합 전략 이전에, Spot 단일 VM으로 축출 옵션(유형·정책·최대가)의 의미를 손으로 익힘.

- 포털에서 **Virtual machines(가상 머신)** → **만들기 > Azure virtual machine** 진입
- **기본(Basics) 탭 > 인스턴스 정보**에서 Spot 옵션을 켬
  - **"Run with Azure Spot discount"(Azure Spot 할인으로 실행)**에서 체크/'예' 선택
    - ※ 포털 버전에 따라 라벨이 **'Azure Spot 인스턴스로 실행'**으로 표기될 수 있음(포털 버전에 따라 명칭 상이).
  > 📷 스크린샷: Spot 할인 옵션 확장 화면

- **크기(Size)**: **D-시리즈(예: `Standard_D2s_v5`)** 선택
  - ※ 주의: **B-시리즈 선택 금지**(Spot 미지원). B1s를 고르면 Spot 옵션이 적용되지 않음.

- **축출 유형(Eviction type)** 선택(출처: spot-arch)
  - **"Capacity only"(용량만)**: 최대 가격 = **-1**. 용량이 필요할 때만 축출, 표준가 초과 청구 없음.
  - **"Price or capacity"(가격 또는 용량)**: 최대 가격을 **직접 설정**. 용량 부족 **또는** 현재가 > 최대가일 때 축출.

- **축출 정책(Eviction policy)** 선택(출처: spot-vms, use-spot)
  - **"Deallocate"(할당 취소, 기본값)**: 축출 시 VM을 **stopped-deallocated** 상태로 둠.
    재배포 가능(성공 보장 없음). **quota 차감 유지**, **디스크 스토리지 계속 과금**.
  - **"Delete"(삭제)**: 축출 시 **VM + 디스크 삭제** → 스토리지 과금 없음.

- **최대 가격(Maximum price)** 설정(출처: spot-vms, spot-portal)
  - 값은 **USD 소수점 5자리**(예: `0.05701` = 시간당 $0.05701).
  - **-1 권장**: 가격 사유로는 축출되지 않음. 요금 = (현재 Spot가, 표준가) 중 **낮은 값**, 표준가 초과 청구 없음.
  - 규칙: 최대가 **>=** 현재가 → 배포됨 / 최대가 **<** 현재가 → **배포 실패(오류)**.
  - ※ 주의: **최대 가격을 변경하려면 먼저 deallocate(할당 취소) 후** 구성에서 변경해야 함.
  > 📷 스크린샷: 축출 유형·정책·최대 가격 입력 화면

- 나머지 탭(디스크·네트워킹·관리)은 표준 VM과 동일하게 진행 후 **검토+만들기**로 배포

> 📷 스크린샷: 검토+만들기 요약 화면(Spot 옵션 표기 확인)

---

## 파트 2. Spot VMSS 생성 (포털)

**왜 이 단계인가**: 단일 VM은 축출되면 끝이지만, VMSS는 자동 확장·교체가 가능함. 혼합 전략의 그릇이 VMSS임.

- 포털에서 **Virtual machine scale sets(가상 머신 확장 집합)** → **만들기** 진입
  - 연결 참조: `hands-on/03.policy.md`의 'VMSS 작성' 섹션과 절차 동일하나, **크기·우선순위**가 다름.

- **오케스트레이션 모드(Orchestration mode)**: **Flexible(유연)** 선택
  - ※ 이유: 뒤의 **Spot Priority Mix(혼합)**은 **Flexible 필수**임(출처: spot-priority-mix).
    2023-11부터 CLI/PowerShell 생성 scale set은 mode 미지정 시 Flexible이 기본임(출처: use-spot).

- **인스턴스 정보 > 크기(Size)**: **D-시리즈(예: `Standard_D2s_v5`)** 선택
  - ※ 주의: `hands-on/03.policy.md`는 B1s를 쓰지만, **B-시리즈는 Spot 미지원** → Spot VMSS에는 D-시리즈로 교체.

- **Spot 우선순위 활성화**: **"Run with Azure Spot discount"(Azure Spot 할인으로 실행)** 체크
  - 축출 정책 선택: **Deallocate(기본)** 또는 **Delete**.
  - ※ 주의: **autoscale(자동 크기 조정) 사용 시 Delete 권장**(출처: use-spot).
    Deallocate는 축출된 인스턴스가 quota를 계속 차감하고 디스크 과금이 지속돼, 스케일 확장 시 quota 한계·비용 누수를 유발함.
  > 📷 스크린샷: VMSS Spot 우선순위·축출 정책 설정 화면

- **네트워킹 탭**: 부하 분산 장치(Load Balancer) 생성·연결(`hands-on/03.policy.md` 절차 동일)
- **검토+만들기**로 배포

> 📷 스크린샷: VMSS 검토+만들기 요약 화면

---

## 파트 3-A. 회수 위험·할인율 관찰

**왜 이 단계인가**: Spot의 핵심 리스크는 "얼마나 자주 축출되나(축출률)"임. 배포 전에 SKU·리전별 위험을 눈으로 확인함.

### 가격 이력·축출률 확인 (포털)

- VM/VMSS 생성 중 **"Run with Azure Spot discount"** 체크 후, **크기 선택 화면 아래**의
  **"View pricing history and compare prices in nearby regions"(가격 이력 보기 및 인접 리전 가격 비교)** 링크 클릭(출처: spot-portal)
  > 📷 스크린샷: 가격 이력·인접 리전 비교 링크 위치

- 팝업에서 해당 **SKU의 Spot 가격 표/그래프 + 축출률(eviction rate)** 확인
  - **축출률 = 시간당(per hour) 축출 확률**. 예: 10% = **지난 7일 이력** 기준, 다음 1시간 내 축출 확률 10%(출처: spot-vms).
  - **인접 리전 비교**: 같은 SKU라도 리전별로 가격·축출률이 다름
    (문서 예시: **Canada Central이 East US보다 저렴 + 낮은 축출률**, 출처: spot-portal).
    Korea Central 축출률이 높으면 인접 리전과 비교해 판단.
  > 📷 스크린샷: SKU 가격 그래프·축출률·인접 리전 표

- **할인율은 리전·SKU별로 상이하며 시점에 따라 변동함** → 구체 할인율은 **포털에서 실측**해 확인함
  (본 문서는 특정 할인율 수치를 단정하지 않음).
- 프로그래밍 조회: **Azure retail prices API**로도 Spot 가격 조회 가능(meterName·skuName에 `Spot` 포함, 출처: spot-vms).
- 이력 조회: **Azure Resource Graph(ARG)의 `SpotResources` 테이블**로 가격 이력(최근 90일)·축출률(최근 28일)을
  조회 가능 → 더 낮은 가격·축출률의 SKU/리전 탐색에 활용(출처: spot-vms).

### Cost Management에서 Spot(변동 요율) 식별

- 비용 관리 + 청구 → **Cost Management > Cost analysis** 진입
- 필터/그룹화에서 **Pricing category = Dynamic(스팟·변동 요율)**로 Spot 사용분을 분리해 확인
  - ※ EA 기준 **관리 그룹·구독 범위 비용 가시성이 정상 노출**됨(스코프 이동으로 상위 집계 확인 가능).
  - ※ 'Pricing category(가격 분류: Standard/Committed/Dynamic)'는 **FOCUS·Cost Management 용어**임
    (본 프로젝트 비용 분석 문서 기준). 위 Spot 6개 MS Learn 출처 범위 밖의 일반 지식임.
  > 📷 스크린샷: Cost analysis에서 Pricing category=Dynamic 필터 결과

### 실행 중 축출 인지 — Scheduled Events

- Spot이 축출될 때 **VM 내부**에서 사전 신호를 받는 메커니즘임(출처: scheduled-events, spot-arch).
  - EventType **"Preempt"** = 이 Spot VM이 곧 삭제됨(ephemeral disk 손실), best-effort, **최소 30초** 통지.
  - VM 내부 앱이 **IMDS Scheduled Events 엔드포인트를 폴링**(최대 초당 1회 권장)해 감지함. **외부에서는 조회 불가**.
  - 수신 시 **graceful shutdown**(연결 드레인·로그 flush·체크포인트 저장) 후 교체 오케스트레이션 수행.
- 테스트: **Simulate eviction** API(`POST .../simulateEviction?api-version=2020-06-01`) 호출 → **204**면 성공(출처: spot-portal).
  - ※ 실습에서는 개념만 이해하고, 실제 폴링 코드는 애플리케이션 구현 영역으로 이관.

---

## 파트 3-B. 베이스라인 + Spot 혼합 설계 (Spot Priority Mix)

**왜 이 단계인가**: 100% Spot은 전량 축출 위험이 있음. 표준 VM으로 **최소 가용량**을 깔고 초과분만 Spot으로 채워
가용성과 절감을 동시에 얻는 것이 혼합 전략의 핵심임(출처: spot-priority-mix).

- **전제**: Spot Priority Mix는 **단일 VMSS**에서 표준 VM + Spot VM을 섞는 기능. **Flexible orchestration mode 필수**.
- **두 파라미터**로 표준/Spot 비율을 제어함(출처: spot-priority-mix)
  - **`baseRegularPriorityCount`** — 포털 라벨 **"Base VM (uninterruptible) count"**  
    항상 유지되는 **표준(비축출) VM 최소 개수**. = 최소 가용량 보장선.
  - **`regularPriorityPercentageAboveBase`** — 포털 라벨 **"Instance distribution"**  
    base **초과분**에서 표준 VM이 차지하는 **비율(0 ~ 100)**. `50` = 초과분을 표준:Spot = 50:50으로 분배.
  - ※ 포털 버전에 따라 위 라벨 명칭이 상이할 수 있음(포털 버전에 따라 명칭 상이).

- **포털 설정 위치**: VMSS 만들기에서 **"Scale with VMs and discounted Spot VMs"** 아래
  **"Scale with VMs and Spot VMs"(표준 VM + Spot 혼합)**를 선택한 뒤 위 두 값(Base VM count · Instance
  distribution)을 입력함(출처: spot-priority-mix).
  > 📷 스크린샷: Spot Priority Mix 설정(Base VM count · Instance distribution)

- **예시(출처: spot-priority-mix)**: `base=10`, `pct=50`인 VMSS를 **30대**로 확장 시(원문 canonical 예시)
  - 처음 **10대는 전부 표준 VM**(base 보장분).
  - 초과 **20대**는 표준:Spot = 50:50 → 표준 10대 + Spot 10대.
  - 합계 = 표준 20대 + Spot 10대(= base 10 + 초과분 표준 10 + Spot 10).
- **scale-in(축소) 시에도 설정 비율을 유지**함(비율 기준으로 지능적으로 인스턴스 제거, 출처: spot-priority-mix).

### 혼합 아키텍처 (ASCII 다이어그램)

```text
                       [ Load Balancer / 트래픽 진입 ]
                                   │
                 ┌─────────────────┴──────────────────┐
                 ▼                                     ▼
        ┌─────────────────┐                 ┌──────────────────────┐
        │  BASELINE 계층   │                 │   BURST(버스트) 계층   │
        │  표준 VM (비축출)  │                 │   Spot VM (축출 가능)   │
        │                 │                 │                      │
        │  [STD][STD][STD]│                 │  [SPOT][SPOT][SPOT]  │
        │  base=최소 가용량  │                 │  초과분 pct 비율로 채움  │
        │  SLA·상시 가동 보장 │                 │  큰 폭 할인 · 30초 통지   │
        └─────────────────┘                 └──────────┬───────────┘
                 ▲                                     │ Preempt 이벤트(30초)
                 │  축출돼도 baseline이 서비스 지속          ▼
                 └──────────────  graceful shutdown → 재확장으로 교체
             (단일 VMSS · Flexible orchestration · Spot Priority Mix)
```

- 읽는 법: **왼쪽(표준)은 절대 축출 안 됨** → 최소 서비스 지속 보장.
  **오른쪽(Spot)은 축출 가능** → 축출돼도 baseline이 버티고, VMSS가 재확장으로 교체함.

### 표 1. 축출 정책 비교 (Deallocate vs Delete)

| 구분 | Deallocate(할당 취소, 기본값) | Delete(삭제) |
|---|---|---|
| 축출 시 상태 | VM → stopped-deallocated | VM + 디스크 삭제 |
| 재배포 | 가능(성공 보장 없음) | 불가(새로 생성) |
| quota | 계속 차감 유지 | 반환 |
| 디스크 스토리지 과금 | **계속 과금** | 없음 |
| 권장 상황 | 단발 Spot VM, 상태 재개 필요 시 | **VMSS autoscale 사용 시 권장** |

> 출처: spot-vms, use-spot

### 표 2. 적합 / 부적합 워크로드

| 구분 | 특징 | 예시 |
|---|---|---|
| **적합**(Spot에 올림) | 무상태 · 중단 허용 · 짧은 처리 · 중단 후 재개 가능 · 우선순위 낮음 | 배치 처리, 데이터 분석, dev/test, 비프로덕션 CI/CD, 대규모 컴퓨트 |
| **부적합**(표준 VM 유지) | 상태 저장 · 상시 가동 · SLA 요구 · 중단 시 데이터 손실 | 프로덕션 DB, 세션 상태 보관 서버, 결제·트랜잭션 처리 |

> 출처: spot-vms, spot-arch

#### Spot 적재 판단 (실습 마무리 — 이 워크로드를 Spot에 올려도 되는가)

관찰(파트 3-A)에서 끝내지 말고, 그 결과로 **"base로 남길지 Spot에 올릴지"**를 직접 결정함.  
아래 4문에 **모두 예**면 Spot 후보, **하나라도 아니오**면 baseline(표준 VM)으로 남김.

- [ ] 축출을 견디고 **나중에 재개**할 수 있는가? (무상태 · 체크포인트를 Spot 외부에 저장)
- [ ] 재처리해도 결과가 같은 **idempotent** 워크로드인가?
- [ ] **SLA · 상시 가동** 요구가 없는가? (프로덕션 실시간 서비스 아님)
- [ ] 관찰한 **축출률**이 감내 가능한 수준인가? (높으면 인접 리전 · 다른 SKU 검토)

### 표 3. 혼합 비율 판단 기준

| 판단 축 | 값이 강할수록 | 설계 방향 |
|---|---|---|
| SLA 요구(가동시간) | 높음 | base(표준) 개수 ↑ — 최소 가용량 크게 |
| 상태성(stateful) | 강함 | Spot 비율 ↓ — 상태 부분은 baseline으로 |
| 중단 허용도 | 높음 | Spot 비율 ↑ — 초과분 pct를 낮춰 Spot 비중 확대 |
| 축출률(관찰값) | 높음 | Spot 비율 ↓ 또는 인접 리전 이전 |
| 비용 절감 목표 | 높음 | Spot 비율 ↑ (단, base로 하한 보장 유지) |

> 판단 요령: **base로 "무슨 일이 있어도 지켜야 할 최소 용량"을 먼저 고정** → 그 위 초과분에서 pct로 Spot 비중을 조절함.

### 안티패턴 (하지 말 것)

- **상태 저장(stateful) 워크로드를 Spot에 배치**: 축출 시 데이터·세션 손실. 체크포인트는 Spot 외부(스토리지 계정)에 저장해야 함.
- **단일 Spot에만 의존(유일 컴퓨트원)**: 전량 축출 시 서비스 중단. **가동시간 요구를 충족할 표준 VM(baseline)을 반드시 확보**(출처: spot-arch).
- **autoscale + Deallocate 조합**: 축출 인스턴스가 quota·디스크 과금을 계속 점유 → 확장 시 quota 한계·비용 누수. autoscale은 **Delete**로.

---

## 파트 4. 실습 체크리스트

**왜 이 단계인가**: 생성·관찰·설계 3영역을 스스로 검증해, "문서만 읽음"이 아니라 "실제로 재현함"을 확인함.

- 생성(Create)
  - [ ] Spot VM을 **D-시리즈(B-시리즈 아님)**로 생성함
  - [ ] 축출 유형(Capacity only / Price or capacity)과 정책(Deallocate / Delete)을 이해하고 선택함
  - [ ] 최대 가격을 **-1**로 설정함(또는 의도적으로 직접 설정하고 이유 설명 가능)
  - [ ] VMSS를 **Flexible** 모드 + Spot 우선순위로 생성함(autoscale 시 Delete 선택)
- 관찰(Observe)
  - [ ] "View pricing history..." 링크로 **SKU 가격·축출률**을 확인함
  - [ ] 축출률을 **시간당 확률(지난 7일)**로 해석할 수 있음
  - [ ] 인접 리전과 가격·축출률을 비교함
  - [ ] Cost Management에서 **Pricing category = Dynamic**으로 Spot 사용분을 식별함
  - [ ] Scheduled Events **Preempt** 개념(30초 통지·IMDS 폴링·simulateEviction)을 설명할 수 있음
- 설계(Design)
  - [ ] **base(표준 최소 가용량) + pct(초과분 Spot 비율)** 개념으로 혼합 비율을 설계함
  - [ ] 적합/부적합 워크로드를 구분하고, 안티패턴 3종을 회피함

---

## 파트 5. 정리 (과금 방지)

**왜 이 단계인가**: 실습 리소스를 남기면 과금이 지속됨. 특히 **Deallocate는 디스크 과금이 계속됨** → 반드시 삭제해야 함.

- **Deallocate 상태 Spot VM 주의**: VM이 정지(deallocated)돼 있어도 **디스크 스토리지는 계속 과금됨**(출처: spot-vms).
  정지만으로는 과금이 멈추지 않음 → **리소스를 삭제**해야 함.
- 삭제 절차
  - [ ] Spot **VM 삭제**(연결 디스크·NIC·공용 IP 포함 삭제 확인)
  - [ ] Spot **VMSS 삭제**(Deallocate 정책이었다면 정지 인스턴스의 디스크가 남지 않도록 삭제 확인)
  - [ ] 네트워킹 리소스(Load Balancer·공용 IP·VNet 등) 중 실습 전용으로 만든 것 삭제
  - [ ] 가장 확실한 방법: **실습용 리소스 그룹 자체를 삭제**해 잔여 리소스를 일괄 제거
  > 📷 스크린샷: 리소스 그룹 삭제 확인 화면

- ※ 주의: 삭제는 되돌릴 수 없음. 삭제 전 보존이 필요한 데이터·체크포인트가 스토리지 계정 등 **Spot 외부**에 저장돼 있는지 확인.

---

## 용어·출처

- **Spot VM**: Azure 미사용 용량을 할인가로 제공하는 VM. SLA 없음, 축출 시 30초 통지.
- **Eviction(축출)**: Azure가 용량 회수를 위해 Spot을 중단시키는 동작.
- **Eviction type(축출 유형)**: Capacity only(용량만) / Price or capacity(가격 또는 용량).
- **Eviction policy(축출 정책)**: Deallocate(할당 취소) / Delete(삭제).
- **Spot Priority Mix**: 단일 VMSS에서 표준 VM + Spot VM을 base·pct로 혼합(Flexible 필수).
- **Preempt**: Scheduled Events의 이벤트 타입, 해당 Spot VM 삭제 예정 신호.

| 출처 | URL |
|---|---|
| Spot VM 개요 | <https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms> |
| Spot 포털 배포 | <https://learn.microsoft.com/en-us/azure/virtual-machines/spot-portal> |
| VMSS + Spot | <https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/use-spot> |
| Spot Priority Mix | <https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/spot-priority-mix> |
| Scheduled Events | <https://learn.microsoft.com/en-us/azure/virtual-machines/linux/scheduled-events> |
| Spot 축출 아키텍처 | <https://learn.microsoft.com/en-us/azure/architecture/guide/spot/spot-eviction> |
