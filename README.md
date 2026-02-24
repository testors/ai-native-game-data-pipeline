# AI-Native Game Data Architecture & Pipeline Specification

## 1. 아키텍처 철학 (Architecture Philosophy)
본 아키텍처는 게임 개발의 기획 데이터를 확률론적 AI(LLM) 환경과 결정론적 엔진 런타임 환경에 완벽하게 통합하기 위한 **스키마 주도(Schema-Driven) 및 GitOps 기반 프레임워크**다. 기존 모놀리식 엑셀(Excel) 기반 데이터 관리의 한계(버전 관리 불가, RAG 비호환성, 비효율적 파싱)를 제거하고, 데이터의 무결성, 추적성, 그리고 AI 주도 생산성을 극대화한다.

## 2. 데이터 분류 및 저장소 이원화 (Data Bifurcation)

### 2.1 비정형/관계형 텍스트 데이터 (Narrative & System Design)
* **저장소:** Outline (또는 동급의 Native Markdown 기반 위키)
* **목적:** 기획 의도, 시스템 플로우차트(Mermaid.js), 로어(Lore), UI/UX 명세 저장.
* **활용:** 구조화된 마크다운을 의미론적 청킹(Semantic Chunking)하여 RAG 벡터 데이터베이스(Milvus, Qdrant 등)로 직접 인덱싱.
* **효과:** 시각적 암묵지 소실 방지 및 고정밀 하이브리드 검색 구현.

### 2.2 정형 파라미터 데이터 (Structured Specs & Balancing)
* **저장소:** 원자화(Atomized)된 JSON 파일 + Git Repository
* **목적:** 스킬 수치, 몬스터 스탯, 아이템 테이블 등 엔진 구동용 정량 데이터.
* **원칙:** 단일 객체(Row/Key)당 1개의 JSON 파일로 분리하여 병합 충돌(Merge Conflict) 방지 및 코드 레벨의 Diff 추적 확보.

---

## 3. 스키마 주도 프레임워크 4단계 설계 (Schema-Driven Framework Design)

모든 컨텐츠(스킬, 몬스터, 퀘스트 등)의 파이프라인을 하드코딩하지 않고, 프로그래머가 스키마만 제공하면 전체 파이프라인이 동적(Dynamic)으로 작동하는 제네릭 프레임워크 구조다.



### Phase 1: 중앙 스키마 레지스트리 (Single Source of Truth)
모든 게임 데이터의 형태와 제약 조건을 프로그래머가 코드로 정의하고 중앙에 등록한다.
* **구현:** Python `Pydantic` 모델 또는 `JSON Schema` 활용.
* **역할:** LLM 추론 강제, JSON 무결성 검증, 빌드 직렬화의 유일한 기준점.
* **특징:** 컨텐츠가 추가될 때마다 새로운 스키마 클래스를 레지스트리(Dictionary 형태)에 매핑하는 작업만 수행.

### Phase 2: 제네릭 LLM 프로세서 (Generic LLM Processor & HITL)
기획자의 자연어(선언적 기획)를 결정론적 JSON으로 변환하는 인터페이스.
* **동적 스키마 주입:** 사용자가 스킬 기획을 요청하면 레지스트리에서 `SkillSchema`를 찾아 LLM(OpenAI Structured Outputs 등)에 주입.
* **환각 통제:** LLM은 스키마에 정의된 키(Key), 데이터 타입(Type), 열거형(Enum), 값의 범위(Min/Max)를 100% 준수하는 JSON 데이터만 반환하도록 강제됨.
* **Human-in-the-Loop:** 생성된 JSON 초안은 즉시 반영되지 않고, 기획자에게 Diff 형태로 제공되어 승인(Approve) 절차를 거침.

### Phase 3: 동적 라우팅 및 스토리지 매니저 (Dynamic Routing)
승인된 데이터를 Git 저장소의 규칙에 맞게 원자화하여 자동 저장.
* **라우팅 규칙:** 스키마 메타데이터를 기반으로 디렉토리 결정. (예: `type: monster` -> `/data/monsters/`)
* **식별자 강제:** 객체의 `ID` 필드를 추출하여 파일명으로 자동 할당 및 덮어쓰기. (예: `SKILL_FIRE_001.json`)

### Phase 4: 유니버설 CI/CD 빌더 (Universal Compiler)
Git에 머지된 수만 개의 파편화된 JSON을 엔진 런타임에 최적화된 단일/다중 파일로 직렬화(Serialization)하는 결정론적 컴파일 과정.
* **무결성 재검증:** 빌드 스크립트가 전체 JSON을 순회하며 레지스트리 스키마를 통해 런타임 에러 요소 사전 차단.
* **의존성 검증:** 참조키 무결성 체크 (예: 몬스터 드랍 테이블에 존재하지 않는 아이템 ID가 있는지 검증).
* **바이너리 직렬화:** JSON의 파싱 오버헤드와 메모리 파편화를 제거하기 위해 `FlatBuffers`, `MessagePack`, 또는 `SQLite`로 변환하여 최종 배포.



---

## 4. 데이터 생명주기 플로우 (Data Lifecycle Flow)

1.  **[Authoring]** 기획자가 웹 UI 또는 슬랙/디스코드 봇에 자연어로 기획 의도 입력.
    * *Input:* "전방 부채꼴 적에게 물리 데미지를 주고 3초 출혈. 데미지는 표준."
2.  **[Processing]** 프레임워크가 `SkillSchema`와 밸런스 기준치(System Prompt)를 LLM에 전달.
3.  **[Validation]** LLM이 스키마 제약조건을 통과한 정확한 JSON Draft 반환.
4.  **[Approval]** 기획자가 수치 확인 후 '승인'.
5.  **[Commit]** 시스템이 `/data/skills/SKILL_CLEAVE_001.json`으로 Git 커밋 및 PR 생성.
6.  **[Build]** CI/CD(GitHub Actions 등)가 트리거되어 전체 JSON 무결성 검사 수행.
7.  **[Compile]** 엔진이 $O(1)$ 속도로 읽을 수 있는 `game_data.bytes` (FlatBuffers 등)로 직렬화 후 클라이언트/서버 배포.

---

## 5. 결론 및 엔지니어링 이점
이 아키텍처는 프로그래머와 기획자의 역할을 완벽히 분리한다. **프로그래머는 데이터 계약(Schema)과 파이프라인만 통제하고, 기획자는 엑셀 수식이나 포맷에 얽매이지 않고 게임의 본질적 재미와 로직(Natural Language)에만 집중할 수 있다.** 최종 런타임 리소스는 바이너리로 최적화되어 엔진의 로딩 속도와 메모리 효율을 극대화한다.
