# ADO 마스터 스펙 한국어 해설본

이 문서는 `ADO_MASTER_SPEC.md`의 사람 읽기용 한국어 해설본이다.

중요한 기준:

```text
Canonical Spec = ADO_MASTER_SPEC.md
Human-readable Korean Guide = ADO_MASTER_SPEC.ko.md
```

즉 실제 DB, 코드, Agent prompt, schema, 상태값, 자동화 계약은 영어 canonical 문서를 기준으로 한다. 이 문서는 사람이 전체 구조와 의도를 빠르게 이해하기 위한 해설본이다.

## 0. 정체성

**ADO = Agent Development Orchestrator**

ADO는 특정 서비스 하나만을 위한 자동화가 아니다. 여러 프로젝트에서 반복적으로 사용할 수 있는 범용 개발 오케스트레이션 시스템이다.

대상 작업은 매번 달라질 수 있다.

- UI/UX 구현
- 백엔드 구현
- 모바일 앱
- 웹
- 관리자 페이지
- 디자인 이미지/에셋
- 문서/리서치
- 인프라 설계

핵심 목표는 긴 대화를 계속 들고 가는 것이 아니라, 사람의 로드맵을 작고 검증 가능한 작업 단위로 나누고, 각 단위의 설계/구현/검증/리뷰/PR 생성을 추적 가능하게 자동화하는 것이다.

ADO의 공통 규약은 별도 ADO Spec Library 저장소에 있고, 실제 NestJS
애플리케이션은 별도 `ado-platform` 저장소에 있다. Platform은 승인된 불변
Spec Library revision만 고정해 실행한다. 경계와 revision 규칙은
`SPEC_LIBRARY_PLATFORM_BOUNDARY.md`를 따른다.

## 1. 기본 계층

ADO의 작업 계층은 다음과 같다.

```text
Project -> Roadmap -> Feature Unit -> Component Work -> Agent Run
```

- `Project`: 최상위 제품 또는 서비스.
- `Roadmap`: 사람이 가져오는 원본 계획.
- `Feature Unit`: 사람이 기능적으로 검증할 수 있는 최소 목표 단위.
- `Component Work`: 하나의 레포지토리에서 수행되는 구현/PR 단위. 단일
  컴포넌트 작업 또는 여러 컴포넌트 root를 묶는 coordinated 작업이다.
- `Agent Run`: Codex, 로컬 모델, Claude import, 시스템 runner, 사람의 실행 기록.

중요한 구분:

```text
Feature Unit = 기능 검증 단위
Component Work = 구현/PR 단위
```

예를 들어 “닉네임 변경”이라는 Feature Unit 하나가 있다면, 그 아래에 server/app/web Component Work가 생길 수 있다.

## 2. 단일 진실 소스

ADO에서 단일 진실 소스는 DB다.

```text
DB = source of truth
Markdown = 작업 문서 / 파생 산출물
Worktree = 실행 공간
```

따라서 Markdown 문서가 DB보다 우선하지 않는다. 문서가 오래되었거나 수동 수정되어 DB와 불일치하면 stale 처리한다.

stale 또는 quarantined artifact는 Agent 입력이나 완료 근거로 사용할 수 없다.

## 3. 역할 분리

ADO는 역할을 강하게 분리한다.

- `Human Owner`: 최종 승인자, 최종 merge 권한자.
- `Nest API + Next Control`: REST/OpenAPI/SSE 기반 상태 확인, 승인, 검증 관제면.
- `Worker`: 실제 Job 실행자.
- `Policy Engine`: 정책/권한/안전성 판단.
- `State Machine`: 상태 전환을 실제로 적용하는 유일한 계층.
- `Codex Planner`: 로드맵 분석과 Feature Unit/Component Work 초안 생성.
- `Codex Implementer`: 승인된 Component Work 구현.
- `Codex Review Arbiter`: 로컬 리뷰 결과를 중재하고 최종 리뷰 판단 후보 생성.
- `Local Review Council`: Qwen Coder, Devstral, Llama 리뷰어.
- `Claude Interactive`: 사람이 수동으로 사용하는 외부 조언자.
- `System Verifier`: 테스트/빌드/검증 명령 실행.
- `GitHub PR Manager`: PR 생성과 상태 동기화. merge는 하지 않음.

중요한 원칙:

```text
어떤 Actor도 자기 작업을 최종 승인할 수 없다.
```

Codex가 구현했다고 해서 Codex가 최종 승인할 수 없다. Arbiter도 상태를 직접 바꾸지 않는다. 최종 상태 전환은 Policy Engine과 State Machine을 통과해야 한다.

## 4. 안전 경계

ADO의 기본 안전 경계는 보수적으로 설계한다.

- `main`은 보호 영역이다. ADO가 직접 수정하거나 PR target으로 삼지 않는다.
- `integrate`는 ADO PR target이다. ADO가 직접 push하지 않는다.
- 작업 branch는 `integrate`에서 생성한다.
- ADO는 PR 생성까지만 자동화한다.
- merge는 사람이 한다.

Branch 형식:

```text
ado/{project_key}/{work_type}/{feature_unit_key}/{component_work_key}
```

절대 자동으로 다루면 안 되는 것:

- secret raw value
- API key
- OAuth token
- production DB row
- 사용자 PII
- billing credential
- production deploy
- production DB 접근
- destructive infra apply
- main merge

외부 전송은 반드시 redaction과 ExternalTransferEvent를 거쳐야 한다.

## 5. 전체 실행 흐름

ADO의 기본 흐름:

```text
Roadmap
-> Planning
-> Human planning approval
-> Component Work creation
-> Branch/worktree creation
-> Codex implementation
-> Verification
-> Local Review Council
-> Codex Arbiter
-> Revision loop if needed
-> PR creation to integrate
-> Human verification
-> Human merge outside ADO
```

v1의 자동화 완료 기준은 PR 생성과 human verification workflow까지다. main merge 자동화는 v1 범위가 아니다.

## 6. 상태 머신

모든 중요한 상태 변경은 다음 흐름을 거친다.

```text
TransitionRequest -> PolicyDecision -> StateMachine -> StateTransition + AuditEvent
```

즉 UI 버튼이나 Runner가 직접 상태를 바꾸면 안 된다.

Feature Unit 주요 상태:

```text
draft -> ready_for_human_review -> approved -> active
-> implementation_done -> verification_running -> review_running
-> needs_revision -> ready_for_pr -> pr_created
-> human_verification_pending -> human_verified -> closed
```

Component Work 주요 상태:

```text
draft -> ready -> branch_created -> implementation_running
-> implementation_done -> verification_running -> verification_failed
-> local_review_running -> local_review_done -> arbiter_review_running
-> needs_revision -> ready_for_pr -> pr_created -> closed
```

예외 상태:

```text
blocked | cancelled | incident_hold
```

`paused`는 상태가 아니라 flag/record로 둔다.

정책/상태 제약의 상세 규칙은 다음 문서에 둔다.

- `POLICY_STATE_CONSTRAINTS.md`
- `STATE_TRANSITION_RULES.md`
- `EVIDENCE_GATES.md`

## 7. 핵심 DB 모델

ADO의 주요 DB 모델:

- Actor, Project, Component, Repository, ComponentRepository, EnvironmentProfile,
  ProjectConstraintProfile
- Roadmap, RoadmapSource, FeatureUnit, AcceptanceCriterion,
  HumanVerificationItem, HumanVerificationResult, FeatureUnitRelation
- ComponentWork, ComponentContract, ComponentWorkRelation, AllowedPathRule
- Artifact, ArtifactSourceRef, DocumentArtifact, ContextPacket, ReviewPacket,
  ArtifactEmbedding
- Job, JobAttempt, JobOutboxEvent, WorkerRegistration, AgentRun, CommandRun,
  VerificationRun, VerificationProfile, VerificationCommand
- ReviewGroup, ReviewResult, ReviewFinding, ArbiterDecision, RevisionTask
- HumanDecision, ApprovalEvent, ManualOverride
- StateSubject, TransitionRequest, PolicyDecision, EvidenceGateResult,
  StateTransition, PauseRecord, AuditEvent
- BudgetPolicy, UsageEvent, SafetyEvent, IncidentReport
- GitWorktree, GitSnapshot, PullRequest, ExternalTransferEvent

UUID를 PK로 사용하고, 사람이 읽는 key는 별도 필드로 둔다.

DB 상세 계약은 다음 문서에서 확정한다.

- `DB_MODEL_SPEC.md`: 테이블, 필드, 관계, 상태 소유권
- `DATABASE_CONSTRAINTS.md`: PostgreSQL 제약, lease, 인덱스, append-only 보호
- `DATA_LIFECYCLE.md`: 생성, stale, quarantine, 보존, 복구, 삭제
- `DATABASE_ERD.md`: 관계도

## 8. 산출물

Artifact는 크게 네 종류다.

- Source Record
- Generated Artifact
- Human Artifact
- External Artifact

중요 산출물:

- RoadmapSource
- RoadmapAnalysis
- FeatureUnitSpec
- ComponentWorkSpec
- ContextPacket
- ReviewPacket
- LocalReviewResult
- ArbiterDecision
- ImplementationArtifact
- GitDiffArtifact
- VerificationRun
- CommandRunLog
- PullRequestPacket
- PullRequestArtifact
- HumanVerificationResult
- SafetyEvent
- IncidentReport
- AuditReport

모든 Artifact는 다음 정보를 가져야 한다.

- source refs
- actor
- version/hash
- status
- policy/redaction metadata

구조화 출력 스키마의 상세 규칙은 `SCHEMA_CONSTRAINTS.md`에 둔다.

## 9. 문서와 Markdown

Markdown은 기본적으로 DB에서 생성되는 파생 문서다.

문서 제약의 상세 규칙은 `DOCUMENT_CONSTRAINTS.md`에 둔다.

Agent별 역할 계약은 `AGENT_ROLE_SPECS.md`에 둔다.

문서 frontmatter에는 다음이 포함된다.

```yaml
ado_doc_type:
ado_id:
human_key:
source_version:
context_hash:
status:
stale:
generated_from_db: true
manual_edit_detected:
generated_at:
```

Agent 입력으로 사용할 수 있는 문서는 `valid` 상태여야 한다.

## 10. 코드 제약과 프로젝트별 서비스 제약

ADO에는 서로 다른 두 제약 층이 있다.

```text
ADO 구현 제약
프로젝트/서비스 제약
```

ADO 구현 제약은 ADO 자체를 어떤 코드 구조와 아키텍처로 만들지 정의한다.

- `CODING_STANDARDS.md`
- `NESTJS_MONOREPO_ARCHITECTURE.md`
- `CONTROL_ROOM_API_UI_SPEC.md`
- `CONTROL_ROOM_DESIGN_SYSTEM.md`
- `BOOTSTRAP_PROTOCOL.md`
- `SPEC_LIBRARY_PLATFORM_BOUNDARY.md`
- `MANAGED_PROJECT_MONOREPO_POLICY.md`
- `SERVICE_LAYER_RULES.md`
- `TESTING_STRATEGY.md`

프로젝트/서비스 제약은 각 서비스가 어떤 톤, UX, 아키텍처, 검증 기준을 가져야 하는지 정의한다.

즉 ADO 자체의 제약과, ADO가 만들 대상 서비스의 제약은 섞지 않는다.

ADO 관제실 디자인 시스템과 각 관리 Project의 디자인 시스템은 서로 다른
제품 규약이다. 서로의 색상, 폰트, 컴포넌트, 레이아웃을 상속하지 않는다.
접근성, 반응형 무결성, 명시적 상태, 오류/복구 동작, 증거 기반 UI 검증만
공통 품질 하한선으로 적용한다.

프로젝트별 제약은 사람이 답변한 questionnaire와 human-approved ProjectConstraintProfile에서 생성된다.

- `PROJECT_CONSTRAINT_GENERATION.md`
- `PROJECT_DESIGN_GOVERNANCE.md`
- `SERVICE_CONSTRAINT_QUESTIONNAIRE.md`
- `templates/PROJECT_DESIGN_CONSTRAINTS_TEMPLATE.md`
- `templates/PROJECT_ARCHITECTURE_TEMPLATE.md`
- `templates/PROJECT_VERIFICATION_PROFILE_TEMPLATE.md`
- `schemas/project_constraint_profile.schema.json`

제약 우선순위:

```text
ADO 공통 제약
> Project 제약
> Feature Unit 제약
> Component Work 제약
> Agent 제안
```

ADO 공통 제약은 안정적으로 유지하고, 서비스 제약은 Project마다 새로 만든다.

## 11. Worker

v1-alpha Worker 구조:

```text
NestJS standalone Worker + PostgreSQL DB queue
```

v1-stable에서는 Job, JobAttempt, JobOutboxEvent, StateMachine, Audit 계약을
보존하는 범위에서만 wake-up, 확장, 원격 artifact 어댑터를 검토할 수 있다.
PostgreSQL은 계속 큐의 단일 진실 소스다.

Worker 책임:

- Job lease
- heartbeat
- policy/state precheck
- runner 호출
- artifact 저장
- transition request 생성
- 승인된 JobOutboxEvent publish
- audit event 기록

Worker는 상태를 직접 바꾸지 않는다.

Worker의 상세 실행 계약은 다음 문서에서 정한다.

- `WORKER_EXECUTION_CONTRACT.md`: lease, outbox, handler lifecycle, timeout, recovery
- `JOB_HANDLER_CATALOG.md`: 허용 Job type별 입력, 증거, side effect, retry 계약
- `WORKER_OPERATIONS.md`: 관리 명령, 관제 UI 상태, shutdown/recovery runbook

Runner 종류:

- CodexRunner
- LocalModelRunner
- VerificationRunner
- GitWorkspaceManager
- GitHubPRManager
- DocumentGenerator
- ClaudeImportHandler

실행 제약의 상세 규칙은 다음 문서에 둔다.

- `RUNTIME_CONSTRAINTS.md`
- `RUNTIME_RULES.md`
- `REPOSITORY_RULES.md`
- `SECURITY_POLICY.md`

## 12. Codex CLI

ADO는 Codex 자동 실행에 `codex exec`를 사용한다.

역할별 sandbox:

- Planner: read-only
- Implementer: workspace-write
- Arbiter: read-only

기본 실행 형태:

```bash
codex exec \
  --cd {workdir} \
  --sandbox {read-only|workspace-write} \
  --ask-for-approval never \
  --output-schema {schema_file} \
  --output-last-message {output_file} \
  --json \
  --ephemeral \
  -
```

`danger-full-access`와 bypass flag는 v1 기본에서 금지한다.

AGENTS.md에는 짧은 실행 규칙만 담고, 긴 컨텍스트는 ContextPacket에 둔다.

## 13. Claude Interactive

Claude는 v1에서 자동 실행 Agent가 아니다.

Claude는 사람이 직접 사용하는 외부 조언자다.

흐름:

```text
ADO exports external-safe packet
-> human pastes into Claude
-> human imports Claude response
-> ADO stores CandidateArtifact
-> human promotes/rejects
```

Claude 응답은 CandidateArtifact다. 자동 상태 전환 evidence가 아니다.

v1 제외:

- Claude SDK 자동 실행
- Claude MCP 자동 연동
- callback endpoint 자동 import

## 14. Local Review Council

로컬 리뷰어는 세 개다.

- Qwen Coder: 코드/로직/API/테스트 관점
- Devstral: 요구사항 충족/파일 간 일관성/완성도 관점
- Llama: UX/요구사항/사람 체크리스트 관점

일반 작업은 3개 중 2개 성공이면 ReviewGroup completed 가능하다.

고위험/보안 민감 작업은 3개 모두 성공해야 한다.

Local reviewer는 결함 탐색자다. 최종 승인자가 아니다.

Codex Arbiter가 finding을 accepted/rejected/merged/human_required로 중재한다.

accepted P0/P1 finding이 있으면 PR 생성 불가다.

## 15. 검증

ADO는 Agent의 “구현 완료” 주장을 믿지 않는다. 검증 evidence만 믿는다.

VerificationProfile은 Component별 검증 계약이다.

검증 명령은 allowlist 기반 argv list로 저장한다.

검증 단위:

```text
Component Work Verification = PR 생성 전 검증
Feature Unit Verification = 사람이 기능 단위로 최종 확인
```

Risk level:

```text
low | normal | high | security_sensitive | production_data_related
```

테스트 명령이 없는 작업도 artifact_review 방식으로 검증한다.

## 16. GitHub / PR

ADO는 `integrate`에서 branch를 만들고, `integrate`로 PR을 생성한다.

PR은 Component Work 단위로 만든다. 단일 Work는 하나의 컴포넌트 root를,
coordinated Work는 여러 선언된 root를 다루지만 둘 다 branch/PR은 하나다.
Feature Unit 화면은 관련 PR을 묶어서 보여준다.

PR 생성 조건:

- ComponentWork ready_for_pr
- branch rule valid
- base branch integrate
- changed paths within allowed paths
- required verification passed
- ReviewGroup completed
- ArbiterDecision permits PR
- PullRequestPacket valid
- stale/quarantined artifact 없음

ADO는 merge하지 않는다.

## 17. UI

DB inspection 도구는 관제실이 아니다. Next.js Control은 Nest REST/OpenAPI/SSE
경계만 사용하며 DB를 직접 수정하지 않는다.

Custom UI는 관제실이다.

필수 화면:

- Project
- Roadmap
- Feature Unit
- Component Work
- Runs / Logs
- Reviews
- PRs
- Artifacts
- Incidents
- Human Decision Inbox
- Settings

UI 버튼은 Runner를 직접 실행하지 않는다. Job을 enqueue하거나 HumanDecision을 생성한다.

## 18. 운영/로그

모든 실행은 trace_id/correlation_id로 묶는다.

로그 종류:

- AuditEvent
- AgentRunLog
- CommandRunLog
- WorkerLog
- PolicyDecisionLog
- StateTransitionLog
- SafetyEventLog
- UsageEventLog
- ExternalTransferLog
- IncidentLog

AuditEvent는 append-only다.

Incident가 열리면 관련 자동화는 pause된다.

비용을 정확히 모르면 숫자를 지어내지 않고 unknown 또는 estimated로 표시한다.

## 19. v1 구현 로드맵

### v1-alpha

목표: DB, 상태, Worker, UI, Artifact 뼈대 구축.

Feature Units:

- FU-A1 NestJS monorepo/PostgreSQL bootstrap
- FU-A2 Core DB models
- FU-A3 State Machine / Policy Engine
- FU-A4 DB Job Queue + Worker
- FU-A5 ArtifactStore + Markdown generation
- FU-A6 Next.js Control Room + Nest API operational slice

### v1-beta

목표: 실제 구현, 검증, PR 생성 루프 연결.

Feature Units:

- FU-B1 Git Workspace Manager
- FU-B2 VerificationRunner
- FU-B3 CodexRunner Implementer
- FU-B4 PR Body + GitHub PR Manager
- FU-B5 End-to-End Beta Loop

### v1-stable

목표: 리뷰 품질, revision loop, Claude import, 운영 안정화.

Feature Units:

- FU-S1 Local Review Council
- FU-S2 Codex Arbiter
- FU-S3 Revision Loop
- FU-S4 Claude Interactive Import
- FU-S5 Safety / Incident / Budget Monitoring
- FU-S6 Stable UI Polish

## 20. v1 제외 범위

v1에서 제외한다.

- automatic merge
- production deploy
- production DB access
- Claude SDK automation
- Claude MCP automatic integration
- Gemini/Grok/Copilot paid reviewer APIs
- multi-user team approval workflow
- Celery/RQ/Redis hard dependency
- Kubernetes operation
- full visual regression platform
- automatic dependency upgrade
- automatic GitHub review comment resolve

## 21. 성공 기준

ADO v1 성공 기준:

```text
Roadmap
-> Feature Unit decomposition
-> human approval
-> Component Work branch/worktree
-> Codex implementation
-> verification
-> Local Review Council + Arbiter
-> PR to integrate
-> human verification workflow
```

v1 complete는 main merge 자동화가 아니다.

```text
v1 complete = PR created + human verification workflow available
```

## 22. 운영 원칙 요약

ADO는 완벽한 Agent를 믿는 시스템이 아니다.

ADO는 다음을 믿는다.

```text
명시적 범위
검증 가능한 evidence
분리된 역할
추적 가능한 로그
사람의 최종 승인
```

자동화는 빠르게 움직일 수 있어야 하지만, 왜 움직였는지 설명할 수 있어야 한다.
