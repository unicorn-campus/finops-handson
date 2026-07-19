# Azure Blob Storage 액세스 계층 실습 랩 (Hot / Cool / Cold / Archive)

> **목적**: FinOps 학습자가 Blob 액세스 계층의 **배치 기준·수명주기 자동화·계층별 단가**를 포털 실습으로 체득함  
> **핵심 메시지**: "저장 단가만 내리는 것"은 함정 — **접근 빈도 × 트랜잭션·검색·최소보존**을 함께 봐야 실제 절감됨  
> **⚠️ 주의**: 아래 금액·수치는 **리전·시점·계약(EA/MCA)·이중화별로 상이**함.  
> 본 문서 요금은 **리전: Korea Central · 통화: USD · 이중화: LRS · Block Blob(GPv2) · 조회일: 2026-07-19(Azure Retail Prices API 기준)** —  
> 실제 값은 반드시 **Pricing Calculator·Blob 가격 페이지**로 확인 필요  
> 📖 **1차 출처(Microsoft Learn / Azure Pricing)**  
> - 액세스 계층: [Access tiers for blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)  
> - 수명주기: [Optimize costs by automatically managing the data lifecycle](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)  
> - 계층 변경: [Set a blob's access tier](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-online-manage) · [Archive 복원(rehydrate)](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview)  
> - 요금: [Blob Storage 가격](https://azure.microsoft.com/en-us/pricing/details/storage/blobs/) · [Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/)  

---

## ① 한눈에 보기 — 4계층(+Smart) 특성 비교

> **큰 그림**: 계층이 **차가워질수록(Hot → Archive)** 저장 단가는 내려가나 **접근(트랜잭션·검색) 비용·지연·최소보존**은 올라감.  
> 계층 설정은 **Block Blob에만** 적용됨(Append/Page Blob 미지원).

| 항목 | Hot | Cool | Cold | Archive |
|---|---|---|---|---|
| **온라인/오프라인** | 온라인 | 온라인 | 온라인 | **오프라인** |
| **접근 패턴** | 자주 접근·수정 | 드물게 접근 | 매우 드물게 접근(빠른 검색 필요) | 거의 접근 안 함 |
| **저장 단가** | 최고 | ↓(Hot 대비) | ↓↓(Cool보다↓) | **최저** |
| **접근(트랜잭션·검색) 단가** | 최저 | ↑ | ↑↑ | **최고** |
| **최소 보존 기간** | 없음 | 30일(GPv2) | 90일(GPv2) | **180일** |
| **읽기 지연** | 밀리초 | 밀리초 | 밀리초 | **수 시간(복원 필요)** |
| **가용성 SLA** | 99.9% | 99% | 99% | 99% |
| **이중화 지원** | LRS/ZRS/GRS 등 | LRS/ZRS/GRS 등 | LRS/ZRS/GRS 등 | **LRS/GRS/RA-GRS만**(ZRS/GZRS 미지원) |
| **계정 기본 계층 지정** | 가능 | 가능 | 가능 | **불가**(계정 기본값 불가) |

- **(참고) Smart tier**: 사용 패턴에 따라 **Hot/Cool/Cold 간 자동 이동**하는 최근 추가 기능(개별 계층 수동 지정 대신 자동화).
- **기본 계층(default access tier)**: 계정 단위로 Hot/Cool/Cold 지정 가능. 명시 안 하면 신규 Blob은 계정 기본값을 상속하여  
  `Hot (inferred)` 등으로 표기됨. **GPv2 신규 계정 기본은 Hot**.
- **Archive 제약**: 스냅샷 미지원, 메타데이터 저장은 **Cool 요율로 과금**됨.

> 🔑 **FinOps 포인트**: Archive는 "가장 싼 계층"이 아니라 **"거의 안 읽는 데이터 전용 계층"** 임 —  
> 오프라인이라 **읽기 전 복원(rehydrate)** 이 필요하고 지연(최대 15시간)·비용이 커, 접근이 조금이라도 있으면 오히려 손해임.

---

## ② 액세스 패턴별 계층 배치 — "이런 데이터는 이 계층"

계층 선택의 기준은 **"얼마나 자주 읽는가 × 얼마나 빨리 필요한가 × 얼마나 오래 보관하는가"** 임.  
저장 단가만 보고 배치하면 트랜잭션·검색 비용에서 역전됨(→ ③절 손익분기 참조).

### 의사결정 순서(텍스트 플로우)

```
[이 데이터를 얼마나 자주 읽는가?]
   ├─ 자주(매일~수시) ─────────────────────────────▶ Hot
   ├─ 가끔(월 단위) ──────────────────────────────▶ Cool
   ├─ 거의 안 읽지만, 읽을 땐 즉시(밀리초) 필요 ──▶ Cold
   └─ 거의 안 읽고, 읽을 때 수 시간 지연 허용 ────▶ Archive
                                                        │
   [최소 보존 기간(Cool 30 / Cold 90 / Archive 180일)보다
    빨리 지우거나 옮길 가능성이 있는가?] ── 예 ──▶ 조기삭제 위약금 주의(한 단계 위 계층 재검토)
```

### 데이터 유형별 매핑(예시)

| 데이터 예시 | 접근 패턴 | 권장 계층 | 근거 |
|---|---|---|---|
| 운영 중 웹/앱 콘텐츠, 활성 DB 백업(직전) | 자주 읽기·수정 | **Hot** | 저장 단가↑라도 트랜잭션 단가 최저 |
| 월간 리포트 원본, 최근 로그(30 ~ 90일) | 월 단위 조회 | **Cool** | 저장 절감 + 밀리초 지연 유지 |
| 분기·연 단위 참조 데이터, 완료 프로젝트 산출물 | 매우 드물게·즉시 필요 | **Cold** | Cool보다 저장↓, 온라인 유지 |
| 규정 보존(수년) 기록물, 원본 미디어 마스터, 장기 감사 로그 | 거의 안 읽음·지연 허용 | **Archive** | 저장 최저, 복원 지연 감내 가능 |
| 수많은 소형 파일(다수 객체) | 다양 | **주의** | 계층 이동·읽기가 **객체 건당 트랜잭션**으로 과금 → 저장 절감분을 트랜잭션·최소보존이 상쇄 가능 |

> 🔑 **FinOps 포인트**: **"규정 보존 = 무조건 Archive"** 가 항상 정답은 아님 —  
> 보존 중 **간헐적 감사·조회**가 있으면 Cold(온라인·즉시)가 총비용에서 유리할 수 있음. 접근 로그로 실제 패턴을 확인 후 배치함.

---

## ③ 계층별 단가 비교 (Korea Central · USD · LRS · Block Blob)

> **표 라벨**: 리전: Korea Central · 통화: USD · 이중화: LRS · 유형: Block Blob(GPv2) ·  
> 조회: 2026-07-19 · Azure Retail Prices API 기준 · **실제값은 Pricing Calculator 확인**

| 과금 차원 | 단위 | Hot | Cool | Cold | Archive |
|---|---|---|---|---|---|
| **저장(Capacity)** | 1 GB/월 | $0.0192 | $0.011 | $0.0045 | $0.002 |
| **쓰기 작업(Write)** | 10K | $0.05 | $0.10 | $0.18 | $0.1086 |
| **읽기 작업(Read)** | 10K | $0.004 | $0.01 | $0.10 | **$5.50** (우선순위 $67.5) |
| **데이터 검색(Retrieval)** | 1 GB | 없음 | $0.01 | $0.03 | $0.022 (우선순위 $0.135) |
| **데이터 쓰기(ingress)** | per GB | $0.0(무료) | $0.0(무료) | $0.0(무료) | $0.0(무료) |
| **조기삭제 위약금** | 1 GB | 해당 없음 | $0.0144 | $0.0045 | $0.002 |

- **Hot 대비 저장 단가 배수(참고)**: Cool ≈ 0.57x · Cold ≈ 0.23x · Archive ≈ 0.10x.
- **Hot 저장 구간 요율**: 사용량 구간별(첫 50TB 등)로 $0.0184 ~ $0.0192 범위 — 본 표는 $0.0192 사용.
- **이중화 배수(참고)**: GRS는 저장 단가 약 2배, RA-GRS 약 2.5배(리전 복제 추가). Archive는 ZRS/GZRS 미지원.
- **기타 작업 요금**: Hot 기타작업 $0.004/10K · 컨테이너 List/Create $0.05/10K · Index Tags $0.0406/10K월.

### 저장단가↓ ↔ 접근단가↑ "역전(손익분기)" 개념

계층이 차가울수록 **저장은 싸지지만 읽기·검색은 비싸짐** → **자주 접근하면 Cool/Cold/Archive가 오히려 더 비쌈**.

- 예: Archive **읽기 작업 $5.50/10K + 검색 $0.022/GB** 은 Hot 읽기 $0.004/10K의 **약 1,375배** 수준.  
  "저장이 10분의 1"이라는 이점은 **한 번의 대량 읽기·복원**으로 상쇄될 수 있음.
- **판단 기준**: `저장 절감액(월)` vs `예상 읽기·검색·복원 비용` 을 비교함.  
  접근이 잦거나 예측 불가하면 한 단계 **따뜻한** 계층을 선택하는 것이 안전함(정확 계산은 Pricing Calculator 확인 권장).

### 함정 2가지 — 소형 객체 다수 · rehydrate 비용

- **소형 객체 다수(건당 트랜잭션)**: 계층 이동·읽기·복원은 **객체 건당 트랜잭션(per 10K)** 으로 과금됨 →  
  **수많은 소형 파일**을 냉각(cooler) 계층으로 옮기면 트랜잭션·최소보존 부담이 저장 절감분을 상쇄할 수 있음  
  (참고: 일부 3차 자료의 "128 KiB 최소 과금 객체"는 **MS 1차 출처(과금 미터 목록)에서 확인되지 않음** — 트랜잭션·최소보존 관점으로 판단함).
- **rehydrate(복원) 비용·지연**: Archive 읽기 전 **Set Blob Tier 또는 Copy Blob** 으로 복원 필요.  
  우선순위 **표준(최대 15시간)** / **높음(High, 더 빠르나 고비용)** 선택 → 지연·비용 상이. **Lifecycle로는 archive → online 복원 불가**.

### 계층 변경 시 과금 방향

| 이동 방향 | 과금 항목 |
|---|---|
| **cooler로 이동**(예: Hot → Cool) | **대상 계층 쓰기 작업(per 10K)** 과금 |
| **warmer로 이동**(예: Archive → Hot) | **원본 계층 읽기 작업(per 10K) + 데이터 검색(per GB)** + **조기삭제 위약금** 가능 |

> 🔑 **FinOps 포인트**: 계층 이동 자체가 **작업·검색 비용**을 유발함 —  
> 잦은 왕복(Hot↔Archive)은 저장 절감을 까먹음. **한 방향(따뜻→차가움) 단계적 이동**을 수명주기로 자동화하는 것이 정석임.

### 조기 삭제 위약금(Early deletion penalty)

**최소 보존 기간 경과 전** 삭제·덮어쓰기·타 계층 이동 시 부과되며 **일할 계산(prorated)** 됨.

- 예 1: **Cool** 이동 후 **21일 만에 삭제** → 남은 **9일(30-21)** 분 Cool 저장료 부과.
- 예 2: **Archive** 이동 후 **45일 만에 이동** → 남은 **135일(180-45)** 분 부과.
- (참고) **soft delete** 활성 시, 보존 기간 만료 전까지는 위약금 대상이 아님.

---

## ④ 수명주기(Lifecycle) 관리 규칙

수명주기 관리는 **규칙 기반(rule-based) 정책 JSON** 으로, 조건 충족 시 자동으로 **계층 이동·만료(삭제)** 를 수행함.

### 규칙 구조

| 구성 | 값 | 설명 |
|---|---|---|
| **필터(filters)** | `blobTypes: [blockBlob]`, `prefixMatch`(경로 접두사), `blobIndexMatch`(인덱스 태그) | 적용 대상 좁히기 |
| **조건(조합 가능)** | `daysAfterModificationGreaterThan`(최종 수정 기준) / `daysAfterCreationGreaterThan`(생성 기준) / `daysAfterLastAccessTimeGreaterThan`(최종 접근 기준) / `daysAfterLastTierChangeGreaterThan`(archive 이동 후 경과) | 언제 실행할지 |
| **액션(baseBlob)** | `tierToCool` / `tierToCold` / `tierToArchive` / `enableAutoTierToHotFromCool` / `delete` | 무엇을 할지 |
| **스코프** | `baseBlob` / `snapshot` / `version` | 원본·스냅샷·버전 |

- **실행 주기**: 정책 엔진이 **하루 1회 실행**, 최초 적용까지 **최대 24 ~ 48시간** 소요 가능(즉시 반영 아님).
- **최종 접근 기준 사용 조건**: `daysAfterLastAccessTimeGreaterThan` 은 **액세스 추적(last access tracking) 활성화** 필요 —  
  활성화 시 **추가 트랜잭션 비용**이 발생함.

### 대표 정책 JSON 예시 (`logs/` 접두사 — 계단식 이동 후 만료)

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "logs-tiering-and-expire",
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": [ "blockBlob" ],
          "prefixMatch": [ "logs/" ]
        },
        "actions": {
          "baseBlob": {
            "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
            "tierToCold":    { "daysAfterModificationGreaterThan": 90 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 180 },
            "delete":        { "daysAfterModificationGreaterThan": 2555 }
          },
          "snapshot": {
            "delete": { "daysAfterCreationGreaterThan": 90 }
          }
        }
      }
    }
  ]
}
```

- 위 규칙: `logs/` 접두사 Block Blob을 **30일 후 Cool → 90일 후 Cold → 180일 후 Archive → 약 7년(2555일) 후 삭제**,  
  **스냅샷은 생성 90일 후 삭제**. (`daysAfterModificationGreaterThan` = **최종 수정일** 기준)

### 함정 — 최종수정 vs 최종접근 · 실행주기 · Archive 복원 불가

- **기준일 오해**: 조건 기본은 **'최종 수정일(Modification)'** 이지 **'최종 접근일(Access)'** 이 아님 —  
  수정만 안 됐을 뿐 **자주 읽는 데이터**가 조건 충족으로 Archive로 내려갈 수 있음. 접근 기준은 추적 활성화가 별도 필요함.
- **즉시 반영 아님**: 하루 1회 실행 + 최초 최대 24 ~ 48시간 → **"규칙을 저장했는데 안 옮겨졌다"** 는 정상일 수 있음.
- **Archive 복원 불가**: **Lifecycle로는 archive → online 복원 불가** — 복원은 수동(Set Blob Tier/Copy Blob)으로만 가능.
- **위약금 상호작용**: 예로 **Cool 이동 30일 이내에 archive 이동** 규칙이 겹치면 **조기삭제 위약금**이 발생함(→ ③절).

> 🔑 **FinOps 포인트**: 수명주기는 **"설정하고 잊는(set-and-forget) 절감 자동화"** 의 핵심 도구지만,  
> **기준일(수정 vs 접근)·최소보존 위약금·복원 불가**를 모르고 걸면 오히려 비용·지연을 유발함. 규칙은 **접근 로그 확인 후** 설계함.

---

## ⑤ 포털 실습 절차 (핸즈온)

> **사전 준비**: GPv2 스토리지 계정 1개, `logs/` 등 컨테이너·샘플 Blob 몇 개 업로드.  
> 클릭 경로는 **Azure Portal(포털)** 기준이며 UI 명칭은 시점에 따라 일부 상이할 수 있음.

### (a) 스토리지 계정 기본 액세스 계층 확인·변경

1. **포털** → 상단 검색창에 **스토리지 계정** 입력 → 대상 스토리지 계정 선택.
2. 좌측 메뉴 **설정 > 구성(Configuration)** 진입.
3. **기본 액세스 계층(Blob access tier / default)** 항목에서 현재 값 확인(예: `Hot`).  
   → Hot / Cool / Cold 중 선택 가능(**Archive는 계정 기본값으로 지정 불가**).
4. 값 변경 후 상단 **저장(Save)** 클릭.

![](images/블롭계층-01-계정기본계층.png)  
> 📸 실제 실행 시 캡처 삽입 — 스토리지 계정 '구성' 블레이드의 **기본 액세스 계층** 드롭다운(Hot/Cool/Cold)

> ⚠️ **주의**: 기본 계층 변경은 **명시 계층이 없는(`inferred`) 기존 Blob**의 과금 계층에 영향을 줄 수 있음 —  
> `Cool(inferred)` 로 바뀌면 최소보존·검색 요금 대상이 됨. 대량 계정은 변경 전 영향 범위를 확인함.

### (b) 개별 Blob 계층 수동 변경 (Hot → Cool/Cold/Archive)

1. 스토리지 계정 → **데이터 스토리지 > 컨테이너** → 대상 컨테이너 진입.
2. 대상 Blob 클릭 → 우측 상단 또는 **... (기타)** 메뉴에서 **계층 변경(Change tier)** 선택.
3. **액세스 계층(Access tier)** 드롭다운에서 대상 계층 선택(Hot / Cool / Cold / Archive).
4. Archive 선택 시 **"오프라인이 되며 읽기 전 복원 필요"** 안내 확인 → **저장(Save)**.

![](images/블롭계층-02-blob계층변경.png)  
> 📸 실제 실행 시 캡처 삽입 — Blob '계층 변경' 패널에서 **Archive 선택 시 나타나는 rehydrate 경고 문구**

> ⚠️ **주의**: 최소 보존 경과 전 재이동·삭제 시 **조기삭제 위약금**(일할)이 부과됨 —  
> 예로 Cool 이동 21일 후 삭제 시 남은 9일분 저장료가 과금됨(→ ③절).

### (c) 수명주기 관리 규칙 생성

1. 스토리지 계정 → **데이터 관리 > 수명 주기 관리(Lifecycle management)** 진입.
2. **규칙 추가(Add rule)** 클릭.
3. **세부 정보(Details)**: 규칙 이름 입력(예: `logs-tiering-and-expire`),  
   범위는 **접두사로 blob 필터 제한(Limit blobs with filters)** 선택 시 (5)단계에서 `logs/` 지정.
4. **기본 blob(Base blobs)** 탭에서 조건·작업 설정:  
   - "최종 수정(Last modified) 이후 30일 → **Cool로 이동**"  
   - "최종 수정 이후 90일 → **Cold로 이동**"  
   - "최종 수정 이후 180일 → **Archive로 이동**"  
   - "최종 수정 이후 2555일 → **삭제(Delete)**"  
   (기준을 최종 수정 대신 **최종 액세스**로 하려면 **액세스 추적 활성화**가 선행되어야 함)
5. **필터 세트(Filter set)** 탭에서 접두사 `logs/` 입력 → **추가(Add)** → **검토+저장**.

![](images/블롭계층-03-수명주기규칙.png)  
> 📸 실제 실행 시 캡처 삽입 — '기본 blob' 탭의 **이동/삭제 조건 입력 화면**(최종 수정 기준 일수 + 대상 계층)

![](images/블롭계층-04-수명주기필터.png)  
> 📸 실제 실행 시 캡처 삽입 — '필터 세트' 탭의 **`logs/` 접두사 입력** 화면

> ⚠️ **주의**: 규칙 저장 후 **하루 1회 실행 + 최초 최대 24 ~ 48시간** 소요됨 —  
> 저장 직후 계층이 안 바뀌어도 정상임. 실제 이동 여부는 시간 경과 후 Blob 계층 컬럼으로 확인함.

### (d) (선택) Archive 복원(rehydrate) 확인

1. Archive 계층 Blob 선택 → **계층 변경(Change tier)** → 대상 계층 **Hot 또는 Cool** 선택.
2. **복원 우선순위(Rehydrate priority)**: **표준(Standard, 최대 15시간)** / **높음(High, 더 빠름·고비용)** 선택 → 저장.
3. 상태가 **"복원 대기 중(Rehydrate pending)"** 으로 표시됨 → 완료까지 대기(표준은 수 시간 소요).

![](images/블롭계층-05-복원rehydrate.png)  
> 📸 실제 실행 시 캡처 삽입 — Archive Blob의 **복원 우선순위 선택** 및 **'Rehydrate pending' 상태** 표시

> ⚠️ **주의**: 복원은 **읽기 작업 + 데이터 검색 요금**이 발생하며, **높음(High)** 우선순위는 검색 단가가 크게 올라감  
> (Archive 우선순위 읽기 $67.5/10K · 우선순위 검색 $0.135/GB — → ③절). 급하지 않으면 표준을 사용함.

---

## ⑥ FinOps 체크리스트 & 함정 요약

- **① 조기삭제 위약금·복원 함정**: Cool/Cold/Archive는 **최소 보존(30/90/180일)** 이 있어 조기 삭제·이동 시 **일할 위약금** 발생.  
  Archive는 **오프라인** 이라 읽기 전 **복원(최대 15시간·고비용)** 필요 → "저장이 싸다"만 보고 배치하면 지연·비용 폭탄이 됨.
- **② 저장 단가만 보는 착시**: 계층이 차가울수록 **트랜잭션·검색 단가는 급등**함(Archive 읽기 ≈ Hot의 1,375배) →  
  **자주 접근하면 Cool/Cold/Archive가 오히려 더 비쌈**. `저장 절감 vs 읽기·검색·복원 비용`을 반드시 비교함.
- **③ 소형 객체 다수**: 냉각 계층 이동·읽기는 **건당 트랜잭션** 과금 → 수많은 소형 파일은 절감분이 상쇄될 수 있어 묶음(아카이브 패키징)·Hot 유지 검토.
- **④ Lifecycle 기준일 오해**: 조건 기본은 **최종 수정일(≠최종 접근일)** — 자주 읽어도 수정만 없으면 Archive로 내려감.  
  **Lifecycle은 archive → online 복원 불가** · 실행은 하루 1회(최초 24 ~ 48시간).
- **⑤ 실측 우선**: 계층 배치·수명주기 규칙은 **감이 아니라 접근 로그·사용 패턴 확인 후** 설계 → 왕복 이동 최소화로 작업 비용 절감.

---

## ⑦ 참고 (1차 출처 링크)

- Access tiers for blob data — <https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview>
- Optimize costs by automatically managing the data lifecycle — <https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview>
- Set a blob's access tier — <https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-online-manage>
- Blob rehydration from the Archive tier — <https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview>
- Blob Storage 가격 페이지 — <https://azure.microsoft.com/en-us/pricing/details/storage/blobs/>
- Azure Pricing Calculator — <https://azure.microsoft.com/en-us/pricing/calculator/>

> ⚠️ 본 문서의 구체 금액·비율은 **시점·리전·계약·이중화별로 변동**되므로,  
> 반드시 위 **1차 출처(요금 페이지·Pricing Calculator)** 로 최신 값을 확인 후 의사결정에 활용함.
