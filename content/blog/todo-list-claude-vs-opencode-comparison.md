+++
title = "같은 PRD, 다른 결과: Claude Code와 OpenCode로 만든 Tauri to-do list 앱 코드 품질 비교"
date = "2026-07-20T21:00:00+09:00"
draft = false
tags = ["claude-code", "opencode", "tauri", "vibe-coding", "code-quality", "comparison"]
categories = ["일지"]
description = "동일한 요구사항 문서(PRD)를 Claude Code와 OpenCode에게 각각 주고 Tauri 2.0 to-do list 앱을 구현하게 한 뒤, 코드 품질을 5개 축에서 비교 분석한 기록"
+++

# 같은 PRD, 다른 결과: Claude Code와 OpenCode로 만든 Tauri to-do list 앱 코드 품질 비교

> 동일한 PRD를 두 AI 코딩 도구(Claude Code, OpenCode)에게 각각 주고 Tauri 2.0 + React + SQLite to-do list 앱을 구현하게 했다. 그리고 나서 코드 품질을 5개 축에서 비교해봤다 -- 아키텍처, 타입 안전성, 기능 완성도, 디자인, 백엔드 품질.

![Claude Code 버전 (다크 테마, 네온 그린)](/images/todo-compare-claude.jpg)
![OpenCode 버전 (라이트 테마, violet)](/images/todo-compare-opencode.jpg)

---

## 들어가며

최근에 나는 aiffel 교육과정 과제로 macOS to-do list 앱을 만들기로 했다. Tauri 2.0을 프레임워크로, React + TypeScript를 프론트엔드로, SQLite를 저장소로 쓰기로 하고 PRD(제품 요구사항 문서)를 먼저 작성했다.

이 PRD를 가지고 두 가지 코딩 도구로 각각 구현해보기로 했다.

- **OpenCode**: 평소에 주력으로 쓰는 AI 코딩 도구. 무료 모델도 물릴 수 있는 게 장점.
- **Claude Code**: Anthropic의 공식 CLI 코딩 에이전트. 유료이지만 Sonnet 모델의 성능이 꾸준히 좋은 평가를 받고 있다.

PRD는 동일했지만, 두 도구가 만든 결과물은 상당히 달랐다. 한쪽은 완성도에 충실했고, 다른 한쪽은 구조에 충실했다. 어느 쪽이 "더 낫다"고 단정하기보다, 이 글에서는 각 도구가 어떤 결정을 내렸고 그 결과가 코드 품질에 어떻게 반영되었는지를 다섯 가지 기준으로 분석해본다.

다만 한 가지 짚고 넘어갈 점이 있다. 사용한 AI 모델의 급이 완전히 달랐다. Claude Code는 `opus`(Claude 3 Opus 계열, 유료 최상위 모델)를 썼고, OpenCode는 `deepseek-v4-flash-free`(무료 추론 모델)를 비롯한 전액 무료 모델만 사용했다. Oracle 같은 고난도 하위 에이전트는 `big-pickle`(무료), 탐색/경량 작업은 `north-mini-code-free`(무료)가 담당했다. 이 차이는 결과 해석에 반드시 고려해야 할 변수다.

---

## 비교 방법

두 코드베이스를 모두 로컬에 내려받고, 소스 코드 전체를 읽으며 5개 축에서 점수를 매겼다.

1. **아키텍처 및 모듈화**: 파일 분할, 관심사 분리, 의존성 방향
2. **타입 안전성**: TypeScript 타입 활용도, SQL 인젝션 방어, 에러 처리
3. **기능 완성도**: PRD 요구사항 대비 실제 구현 여부
4. **디자인**: UI/UX 완성도, 일관성, 시그니처 요소
5. **백엔드 품질**: DB 마이그레이션, 인덱스, 설정 관리

---

## 1. 아키텍처: OpenCode의 완승

가장 큰 차이는 App.tsx의 크기에서 드러났다.

**OpenCode**는 75줄의 얇은 App.tsx를 가지고 있다. 주요 로직은 `useTodos` 커스텀 훅으로 캡슐화했고, TodoForm/TodoList/TodoItem 세 개의 컴포넌트로 UI를 분할했다. 새 파일을 만들 때마다 "이 책임은 어디에 속하는가"를 고민한 흔적이 보인다.

**Claude Code**의 App.tsx는 297줄이다. DB 호출, 상태 관리, 렌더링, 이펙트가 한 파일에 모두 담겨 있다. 컴포넌트 분할이 없고, 모든 UI를 함수 하나가 책임진다.

물론 Claude Code 쪽도 `db.ts`, `notify.ts`, `todoUtils.ts`로 관심사는 나름 분리되어 있다. 문제는 **UI 레이어까지의 분할이 없다**는 점이다. 알림 모듈, 유틸리티 함수는 분리되어도, 화면을 구성하는 코드가 300줄에 가까운 단일 파일에 몰려 있으면 유지보수에 분명한 부담이 된다.

```typescript
// OpenCode: 얇은 App.tsx가 위임만 한다
function App() {
  const { sortedTodos, loading, addTodo, toggleTodo, ... } = useTodos();
  return (
    <TodoForm onSubmit={addTodo} />
    <TodoList todos={sortedTodos} ... />
  );
}

// Claude Code: App.tsx가 모든 렌더링을 직접 한다
function App() {
  const [todos, setTodos] = useState<Todo[]>([]);
  // ... 280줄의 핸들러 + renderRow + JSX
}
```

이 차이는 AI 코딩 도구의 기본 코드 생성 전략 차이로 보인다. OpenCode는 "분할하고 위임하라"는 방향으로 유도하고, Claude Code는 "일단 돌아가게 만들고 나중에 리팩터하라"는 방향에 가깝다. 단기적으로는 후자가 빠르지만, 이 프로젝트처럼 규모가 조금만 커져도 전자가 유리해진다.

---

## 2. 타입 안전성: Claude Code의 안정적 DB 계층

표면적인 TypeScript 타입 활용은 비슷했다. 두 버전 모두 인터페이스를 정의하고, 파라미터화된 SQL 쿼리를 사용한다. 하지만 DB 접근 계층에서 결정적인 차이가 있었다.

**Claude Code**는 `db.ts`에 모든 CRUD를 타입드 함수로 래핑했다. 외부에서는 SQL을 전혀 볼 필요 없이 `addTodo(title, parentId, dueAt)`만 호출하면 된다. 위험한 문자열 조합이 발생할 여지가 없다.

**OpenCode**는 `useTodos` 훅 안에서 직접 SQL을 실행한다. 더 문제는 `updateTodo` 함수가 컬럼명을 동적으로 조합한다는 점이다.

```typescript
// OpenCode: 컬럼명을 문자열로 조합
const setClauseNumbered = entries
  .map(([key], i) => `${key} = $${i + 1}`)
  .join(", ");
```

컬럼명을 `$1, $2` 같은 파라미터 바인딩으로 처리하지 않고 문자열 조합으로 처리하면, `title`처럼 안전한 값이 아니라 악의적인 문자열이 들어왔을 때 인젝션으로 이어질 수 있다. 물론 컬럼명은 `keyof Todo`로 제한되어 있지만, 런타임에 `Object.entries(data)`에서 추출되는 값이라 타입 시스템만으로 완전히 안전하다고 보기 어렵다.

---

## 3. 기능 완성도: Claude Code의 완전 구현

여기서 가장 큰 격차가 벌어졌다.

| 기능 | Claude Code | OpenCode |
|---|---|---|
| 알림 (1h/30m/15m 전) | 30초 스캔 루프 완전 구현 | 미구현 |
| 설정 영속화 | DB settings 테이블 저장 | React state만 (재시작 리셋) |
| 메뉴바 타이틀 | set_title + set_tooltip | set_tooltip만 |

Claude Code가 PRD의 요구사항을 모두 구현한 반면, OpenCode는 알림 기능과 설정 영속화를 구현하지 않았다. 특히 알림은 PRD에서 "마감 시간 1시간 전, 30분 전, 15분 전 알림"이라고 명시되어 있었고 DB 스키마에도 `notifications` 테이블을 정의해두었지만, 실제 알림 전송 로직은 비어 있었다.

Claude Code가 알림을 더 단순한 방식으로 처리한 점도 흥미롭다. OpenCode의 PRD는 `notifications`라는 별도 테이블을 설계했지만, Claude Code는 `todos` 테이블에 `notified_60m`, `notified_30m`, `notified_15m`라는 세 개의 boolean 플래그 컬럼을 추가했다.

```sql
-- Claude Code: 단순한 알림 추적
notified_60m INTEGER NOT NULL DEFAULT 0,
notified_30m INTEGER NOT NULL DEFAULT 0,
notified_15m INTEGER NOT NULL DEFAULT 0
```

알림 종류가 3개로 고정되어 있다면 별도 테이블을 만드는 것보다 이 방식이 훨씬 단순하다. 조인도 필요 없고, 한 줄 UPDATE로 모든 알림 플래그를 초기화할 수 있다. **같은 요구사항이라도 설계 철학에 따라 전혀 다른 스키마로 이어질 수 있다**는 좋은 사례다.

---

## 4. 디자인: 차이가 가장 극명했던 영역

두 버전의 디자인 접근 방식은 완전히 달랐다.

**Claude Code**는 순수 CSS에 의존성 제로로 갔다. CSS 커스텀 프로퍼티(토큰)로 색상과 간격을 정의하고, 다크 테마 + 네온 그린(#00D26A) 강조라는 시그니처 컬러를 만들었다. 가장 인상적이었던 요소는 **마감 카운트다운 칩**이었다.

- 마감이 많이 남음: 회색 (날짜 표시)
- 24시간 이내: 앰버
- 60분 이내: 네온 그린 + 글로우
- 마감 지남: 빨간색 + 배경 강조
- 마감 없음: 점선 + "마감 +" 표시

알림 임계값(60분/30분/15분)과 시각적 피드백이 연동되어 있어, 사용자가 마감 상태를 한눈에 파악할 수 있다. 이건 기능과 디자인이 함께 만든 시그니처였다.

**OpenCode**는 Tailwind v4로 보다 무난한 디자인을 선택했다. 밝은 테마에 보라색(violet) 강조, 둥근 카드 형태의 표준적인 UI다. 나쁘지 않지만, "이 앱만의 정체성"이라는 측면에서는 Claude 버전에 비해 약하다. Tailwind의 유틸리티 클래스 접근은 생산성이 높지만, 디자인이 "프레임워크의 기본값"처럼 보이게 만드는 경향이 있다.

---

## 5. 백엔드 품질: Claude Code의 격차

Tauri 앱의 Rust 백엔드 품질에서도 차이가 컸다.

**Claude Code**는 `tauri_plugin_sql::Migration` 구조체를 사용해 정식 마이그레이션을 등록했다. `parent_id`와 `due_at`에 인덱스를 걸었고, `settings` 테이블에 기본값을 INSERT하는 것까지 챙겼다. 또한 `set_title()`으로 트레이 아이콘의 텍스트를 설정해 메뉴바에 실제로 제목이 표시되도록 했다.

**OpenCode**는 프론트엔드 `db.ts`의 `CREATE TABLE IF NOT EXISTS`로 테이블을 생성한다. 인덱스도 없고, 기본값도 없다. 트레이는 `set_tooltip()`만 호출해서 메뉴바에는 아이콘만 뜬다. 불필요한 `serde` + `serde_json` 의존성만 선언되어 있고 실제로 사용되지는 않는다.

```rust
// Claude Code: 정식 마이그레이션
fn migrations() -> Vec<Migration> {
    vec![Migration {
        version: 1,
        description: "create_todos_and_settings",
        sql: "
            CREATE TABLE IF NOT EXISTS todos ( ... );
            CREATE INDEX IF NOT EXISTS idx_todos_parent ON todos(parent_id);
            CREATE INDEX IF NOT EXISTS idx_todos_due ON todos(due_at);
            INSERT OR IGNORE INTO settings ...
        ",
        kind: MigrationKind::Up,
    }]
}
```

프로덕션 앱이라면 Claude Code의 접근이 훨씬 바람직하다. 마이그레이션 시스템이 있으면 스키마 변경을 추적할 수 있고, 인덱스는 데이터가 쌓일수록 성능 차이를 만든다.

---

## 종합 평가

| 평가 영역 | Claude Code | OpenCode |
|---|---|---|
| 아키텍처 | ★★★☆☆ | ★★★★★ |
| 타입 안전성 | ★★★★★ | ★★★☆☆ |
| 기능 완성도 | ★★★★★ | ★★★☆☆ |
| 디자인 | ★★★★★ | ★★★☆☆ |
| 백엔드 | ★★★★★ | ★★☆☆☆ |
| 종합 | 4.6/5 | 3.2/5 |

---

## 배운 점

### 1. AI 코딩 도구는 완성도 vs 구조성 간의 트레이드오프를 만든다

Claude Code는 "일단 끝까지 만든다"에 강했고, OpenCode는 "깔끔하게 쌓아올린다"에 강했다. 어느 쪽이 절대적으로 낫다고 말하기보다, 목적에 따라 선택하는 게 맞다. 프로토타입이나 단기 과제라면 Claude Code의 완성도 높은 접근이 유리하고, 장기 프로젝트라면 OpenCode의 구조적 접근이 유리할 것이다.

### 2. 이상적인 조합이 있다면

OpenCode의 컴포넌트 분할 + 커스텀 훅 구조를 Claude Code의 기능 완성도와 백엔드 품질에 결합하는 것이다. 실제로 두 결과물을 보고 "이 부분은 저쪽에서 가져오면 좋겠다"는 생각이 여러 번 들었다. AI 코딩 도구를 하나만 고집할 필요는 없다는 걸 보여준 사례다.

### 3. 같은 PRD도 다른 해석이 나온다

알림 테이블을 별도로 만들지 말지, settings를 KV 테이블로 둘지, 정렬 기준을 DB sort_order로 할지 프론트 localeCompare로 할지 -- 같은 요구사항도 두 도구는 전혀 다른 선택을 했다. 이는 AI 코딩 도구가 단순히 "요구사항을 코드로 번역"하는 것이 아니라, 자기만의 설계 철학을 가지고 의사결정한다는 증거다.

---

## 참고 자료

- Claude Code 구현체: [GitHub - chictimin/todo-app](https://github.com/chictimin/todo-app) | [Release v0.1.0](https://github.com/chictimin/todo-app/releases/tag/v0.1.0)
- OpenCode 구현체: [GitHub - chictimin/todo-list](https://github.com/chictimin/todo-list) | [Release v0.1.0](https://github.com/chictimin/todo-list/releases/tag/v0.1.0)
