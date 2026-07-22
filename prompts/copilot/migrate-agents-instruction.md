# Copilot Studio 지침 이관 프롬프트

[목표]

로컬 에이전트용 지침을 Microsoft Copilot Studio에서 사용할 수 있는 지침으로 변환하여  
마크다운 파일 1개로 생성함.

[역할]

당신은 대화형 AI 지침 설계와 기술 문서 검수 경력 13년의 Copilot Studio 지침 아키텍트임.  
원본의 업무 의도와 팀 페르소나는 보존하고, 실행 환경에 종속된 표현은 Copilot Studio에 맞게 변환함.

[맥락]

- 내 상황: 로컬 저장소의 `AGENTS.md`에는 파일 경로, Agent 호출, Claude Code, Git 등
  Copilot Studio 지침에서 직접 실행할 수 없는 규칙이 포함되어 있음.  
- 이관 목적: 원본의 팀 운영 방식과 콘텐츠 품질 기준을 유지하면서 Copilot Studio의 지침과
  Knowledge 중심 실행 방식으로 전환함.  
- 결과물 독자: Copilot Studio 에이전트를 구성하고 운영하는 FinOps 교육 콘텐츠 담당자

[입력]

- 원본 지침: `{원본_지침_경로}`
  - 기본값: `AGENTS.md`  
  - 경로로 원본을 읽을 수 없으면 아래 XML 태그 안에 원문 전체를 직접 입력함.  
- 프롬프트 작성 가이드: `references/prompt-guide.md`
- 변환 결과 참고본: `{참고_지침_경로}`
  - 기본값: `hands-on/copilot/instruction.md`  
  - 파일이 존재하면 구조와 표현만 참고하고, 내용 충돌 시 원본 지침과 추가 요구사항을 우선함.  
- 추가 변환 요구사항: 해당 없음이면 `해당 없음`으로 처리함.

<원본_지침_텍스트>
{경로 대신 직접 입력하는 Claude 프로젝트 지침 원문}
</원본_지침_텍스트>

<추가_변환_요구사항>
{사용자가 추가로 지정한 보존·치환·삭제 규칙}
</추가_변환_요구사항>

[처리]

1. `{원본_지침_경로}`를 UTF-8로 읽음.  
   경로를 읽을 수 없으면 `<원본_지침_텍스트>`를 사용하고, 두 입력이 모두 없으면 작업을 중단함.  
   확보한 원문에서 제목, 팀 목표, 페르소나, 역할 매핑, 수행 규칙을 식별함.
2. 원본 내용을 다음 세 범주로 분류함.
   - 보존: 팀 목표, M 행동 원칙, 팀원 페르소나, 담당 매핑, 대화 가이드, 최적안 도출,
     문서 작성 가이드, 정직한 보고 규칙  
   - 치환: 로컬 파일 경로, 런타임 이름, Agent 도구·호출, 시스템 프롬프트 등 실행 환경 종속 표현  
   - 삭제: Copilot Studio에서 사용할 수 없으며 대체 규칙이 필요하지 않은 운영·기억·개발 환경 규칙  
3. 다음 Knowledge 참조 치환을 모든 섹션에 일관되게 적용함.

| 원본 표현 | 변환 표현 |
|---|---|
| `references/prompt-guide.md` | Knowledge의 `prompt-guide.md` |
| `references/ppt-guide.md` | Knowledge의 `pptx-guide.md` |
| `references/pptx-guide.md` | Knowledge의 `pptx-guide.md` |
| `references/xlsx-guide.md` | Knowledge의 `xlsx-guide.md` |

4. Copilot Studio에 맞게 다음 표현을 치환함.

| 원본 표현 유형 | 변환 기준 |
|---|---|
| Claude 또는 Claude Code 기반 대표 에이전트 | Copilot 또는 Microsoft Copilot Studio 기반 대표 에이전트 |
| Agent 도구로 위임·Agent 호출 | 가장 적합한 팀원을 선정하여 위임 |
| 시스템 프롬프트에 프로파일 반영 | 팀원의 프로파일·성향·경력을 작업 관점과 응답에 반영 |
| Agent 도구 기반 실행 방식 | 역할별 수행 방식 |
| 로컬 디렉터리를 참조 자료로 사용 | Knowledge에 등록된 해당 자료 참조 |
| 사용할 수 없는 파일 생성 기능 | 산출물 구조를 응답으로 제공하고 미실행 범위를 명시 |

5. 다음 섹션이나 규칙은 원본에 존재할 때 삭제함.
   - 플러그인 슬래시 명령, marketplace 탐색, 로컬 플러그인 캐시 규칙  
   - `워크플로우 진행상황` 섹션  
   - `Git 연동` 섹션  
   - `URL링크 참조` 섹션  
   - `Lessons Learned` 전체와 하위 `기록 규칙`, `교훈 목록`  
   - Claude Code 런타임 전용 `Advisor 활용 규칙`  
   - MCP, 플러그인, 셸 명령 등 특정 로컬 런타임에서만 실행 가능한 규칙  
   - 단, Azure 서비스인 `Azure Advisor` 등 업무 도메인 용어는 삭제하지 않음.  
6. 팀원 위임 규칙을 다음 원칙으로 재작성함.
   - 클로니가 요청을 분석하여 가장 적합한 팀원을 선정하여 위임함.  
   - 해당 팀원의 프로파일·성향·경력을 작업 관점과 응답에 반영함.  
   - 적합한 팀원이 없으면 클로니가 직접 수행함.  
   - 여러 관점의 결과는 클로니가 하나의 일관된 응답으로 통합함.  
   - 역할을 선언한 것과 실제 산출물·검증을 완료한 것을 구분함.  
7. 오케스트레이터 프로파일을 Copilot Studio 환경에 맞게 조정함.
   - 이름은 `코파일럿`, 별명은 `클로니`로 표기함.  
   - `Anthropic Claude Code 런타임 기반` 표현을 `Microsoft Copilot Studio 기반`으로 변경함.  
   - 로컬 도구 운영 경력 대신 지침·Knowledge를 활용한 콘텐츠 조율 전문성을 기술함.  
8. Copilot Studio에 기능이 있는 것으로 확인되지 않은 외부 시스템 조작은 완료로 단정하지 않도록
   정직한 보고 규칙을 보완함.
9. 중복 문장을 통합하고 Copilot Studio가 우선순위를 해석할 수 있도록 제목과 목록 구조를 단순화함.
10. 최종 지침을 `{출력_지침_경로}`에 UTF-8 마크다운으로 생성하거나 업데이트함.
11. 저장 후 다음 항목을 정적 검증함.
    - 파일이 존재하며 비어 있지 않음.  
    - `Knowledge의 prompt-guide.md`, `Knowledge의 pptx-guide.md`,
      `Knowledge의 xlsx-guide.md` 참조가 모두 존재함.  
    - `references/prompt-guide.md`, `references/ppt-guide.md`, `references/pptx-guide.md`,
      `references/xlsx-guide.md`가 남아 있지 않음.  
    - 삭제 대상 섹션 제목과 Claude Code 전용 규칙이 남아 있지 않음.  
    - 팀 목표, 팀원 5개 역할, 담당 매핑, 대화 가이드, 최적안 도출, 정직한 보고 규칙이 존재함.  
    - 모든 줄이 120자 이내임.  
    - 원본 파일의 내용이 변경되지 않음.  
- 출력파일: `instruction.md`
- 톤앤매너: 간결하고 객관적인 한국어 기술 문서, 명사체 종결
- 작성 규칙:
  - 지침 본문에는 이관 과정의 설명이나 변경 이력을 포함하지 않음.  
  - 확인하지 않은 Copilot Studio 기능이나 Knowledge 자료를 존재한다고 단정하지 않음.  
  - 긴 입력 텍스트를 직접 포함해야 하면 XML 태그로 감싸 지시문과 구분함.  

[출력]

- Copilot Studio 지침: `{출력_지침_경로}`
  - 기본값: `hands-on/copilot/instruction.md`  
  - 형식: UTF-8 마크다운  
- 작업 응답: 생성·수정한 경로, 적용한 주요 변환, 정적 검증 결과, 미검증 범위 요약

[제약조건]

- MUST: 원본의 업무 목표, 팀 페르소나, 역할 경계, 대화 규칙, 품질 원칙을 의미상 보존함.
- MUST: 세 개의 작성 가이드 참조를 `Knowledge의 {파일명}` 형식으로 통일함.
- MUST: `Agent 도구로 위임`을 `가장 적합한 팀원을 선정하여 위임`으로 변경함.
- MUST: 명사체 종결, 한 줄 120자 이내, 간결한 마크다운 구조를 적용함.
- MUST: 실제 수행한 파일 생성과 정적 검증 결과만 완료로 보고함.
- MUST NOT: `{원본_지침_경로}`를 수정함.
- MUST NOT: Git, URL 저장, 기억 기록, 교훈 목록, Claude Advisor 규칙을 결과에 포함함.
- MUST NOT: Azure Advisor 같은 업무 용어를 Claude Advisor 규칙과 혼동하여 삭제함.
- MUST NOT: Copilot Studio 게시, Knowledge 등록 또는 실제 에이전트 동작을 수행하지 않고 완료로 보고함.
- 완료조건: 출력 파일 생성, 필수 Knowledge 참조 3개 확인, 삭제 대상 0건, 필수 보존 섹션 확인,
  120자 초과 줄 0건, 원본 무변경을 검증한 결과가 제시됨.

[예시]

입력:

```text
{원본_지침_경로}=AGENTS.md
{참고_지침_경로}=hands-on/copilot/instruction.md
{출력_지침_경로}=hands-on/copilot/instruction.md
```

치환 예시:

```text
변환 전: 위임 프롬프트는 `references/prompt-guide.md`의 8섹션 표준을 따름
변환 후: 팀원·작업 지시용 프롬프트는 Knowledge의 `prompt-guide.md`를 준수함

변환 전: 가장 적합한 팀원을 선정하여 Agent 도구로 위임
변환 후: 가장 적합한 팀원을 선정하여 위임함
```

삭제 예시:

```text
변환 전: ## Git 연동
변환 후: 해당 섹션 없음

변환 전: ## Advisor 활용 규칙: 런타임이 Claude Code인 경우만 수행
변환 후: 해당 섹션 없음
```

완료 응답 예시:

```text
Copilot Studio용 지침을 {출력_지침_경로}에 생성함.
필수 Knowledge 참조 3개, 삭제 대상 0건, 필수 보존 섹션, 120자 제한, 원본 무변경 검증을 통과함.
Copilot Studio 게시와 실제 에이전트 응답 검증은 수행하지 않음.
```
