https://github.com/unicorn-campus/prd/blob/main/references/develop-copilot-agent.md

Azure copilot studio의 에이젼트의 지침으로 AGENTS.md의 내용을 사용하고 싶습니다. 
copilot studio에 맞게 수정해줘요. 첨부 이미지 내용 참고해요.
hands-on/


[목표]
Copilot Studio의 Agent를 이용한 비용분석 가이드 작성

[역할]
당신은 Copilot Studio에 능통한 Agent 개발자임

[맥락]
Claude Code의 지침(AGENTS.md)과 Skill을 Copilit Studio Agent로 이관해야 함

[입력]
- Copilot Agent 지침: hands-on\copilot\instruction.md
- Copilot 스킬 파일 
	- 스킬 'analyze-cost-dashboard' : hands-on\copilot\analyze-cost-dashboard.md
	- 스킬 'analyze-cost-powerbi': hands-on\copilot\analyze-cost-powerbi.md
- 작성 가이드 예제: https://github.com/unicorn-campus/prd/blob/main/references/develop-copilot-agent.md

[처리]
- 작성 가이드 예제 파악 
- 지침 변경: 프롬프트 'prompts\copilot\migrate-agents-instruction.md' 수행
- 스킬 변경: 프롬프트 'prompts\copilot\migrate-agents-instruction.md' 수행 
- Chrome 확장 스킬로 Copilot Studio 접근 
- Agent 작성
  - 이름: 'FintOps 비용분석팀' 
- Agent 배포 및 테스트  
- 가이드 파일 작성 

[출력]
- 가이드 파일: hands-on\15.generate-copilot-agent.md

[제약조건]
- MUST:
  - 추가정보나 내 의사결정이 필요 시 반드시 요청 
- MUST NOT: 
	- 입력물에 근거하여 수행하고 임의적으로 추측하여 생성 안함 
- 완료조건:
	- 정상적으로 Agent 생성 및 테스트 통과 
	- 가이드 파일 생성 
	