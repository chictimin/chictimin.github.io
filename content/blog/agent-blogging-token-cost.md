+++
title = "에이전트한테 블로그 맡겼더니 토큰이 이렇게 나갔다: 세션 로그 실측 분석"
date = "2026-07-11T11:50:00+09:00"
draft = false
tags = ["hugo", "claude-code", "ai-agent", "token-cost", "회고"]
categories = ["일지"]
description = "1편 이후 별도 세션에서 어제 블로그 작업 로그를 실측 분석해본 결과 — 토큰 사용량, 비용, 병목 원인과 개선 방안 정리"
+++

## 1. 들어가며

[1편](/blog/building-this-blog-with-claude-code/)에서 이 devlog를 Claude Code한테 맡겨서 만든 과정을 적었는데, 그 글 마지막에 슬쩍 "시간도 토큰도 꽤 썼다"고 썼었다. 사실 그 한 줄로 넘기기엔 계속 마음에 걸리는 게 있었다.

토큰 소모는 결국 비용이고, 조금 더 넓게 보면 그 비용 뒤엔 연산에 들어가는 에너지 소비가 있다. 평소에도 관심 있던 부분이라, 어제 작업하면서 "이 정도로 간단한 수정 하나에도 왜 이렇게 오래 걸리지" 싶었던 순간들이 그냥 넘어가지지 않았다. 체감상 확실히 느렸다.

그래서 별도 세션을 하나 열어서, 어제 세션 로그(JSONL 원본)를 AI한테 파싱시켜 실측치를 뽑아봤다. 이 글은 그 결과를 정리한 것 — 숫자로 확인한 병목과, 그래서 앞으로 어떻게 쓸지에 대한 회고다.

## 2. 실측 수치

분석 스코프는 어제 로그 전체 중 블로그 작업 구간만 잘라낸 것이다 (`07:44` 깃헙 페이지 블로그 문의 시작 ~ `11:05` 최종 커밋+푸시, 약 **3시간 23분**).

| 항목                          | 값                                            |
| ----------------------------- | --------------------------------------------- |
| 사용자 표기 turn 수           | 74                                            |
| 실제 API 호출 수 (중복 제거)  | **456회**                                     |
| raw 토큰 총합                 | 약 1억 2,265만                                |
| 컨텍스트 크기 변화            | 세션 시작 ~34,000 → 종료 시점 ~450,000        |
| 추정 비용 (캐시 할인 적용 후) | **약 $42.6 (약 63,850원, 1,499원/달러 기준)** |

가장 먼저 눈에 띈 건 **74 대 456**이라는 차이였다. 내가 체감한 "대화 74번"과 실제로 서버에 날아간 API 호출 456회 사이엔 6배 넘는 간극이 있었다. 이 간극이 어디서 오는지가 이번 분석의 핵심이었다.

## 3. 토큰이 어디서 이렇게 늘어났나 — 구성 비율

raw 토큰 항목을 4가지로 쪼개서 보면 답이 바로 나온다.

<div style="overflow-x:auto; margin: 2rem 0;">
<svg viewBox="0 0 700 220" width="100%" style="max-width:700px; display:block; margin:0 auto;" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="chart1-title">
  <title id="chart1-title">세션 전체 raw 토큰 122,652,202개의 구성 비율</title>
  <text x="30" y="20" font-size="13" fill="var(--color-text-muted)">raw 토큰 122,652,202개 중 구성 비율</text>
  <rect x="30" y="34" width="640" height="40" rx="4" fill="var(--color-border)" opacity="0.35" />
  <rect x="30" y="34" width="4.9" height="40" fill="var(--color-text-muted)" />
  <rect x="34.9" y="34" width="635.1" height="40" rx="4" fill="var(--color-accent)" />
  <text x="365" y="20" font-size="12" fill="var(--color-accent)" text-anchor="middle">cache_read_input_tokens — 99.24%</text>
  <text x="345" y="59" font-size="12" fill="var(--color-bg)" text-anchor="middle">121,718,753</text>
  <line x1="32" y1="74" x2="32" y2="104" stroke="var(--color-text-muted)" stroke-dasharray="2,2" />
  <text x="30" y="120" font-size="12" fill="var(--color-text-muted)">나머지 0.76%(933,449개)를 확대해서 보면 —</text>
  <text x="30" y="150" font-size="12" fill="var(--color-text)">cache_creation_input_tokens</text>
  <rect x="250" y="138" width="300" height="16" rx="3" fill="var(--color-accent)" opacity="0.9" />
  <text x="560" y="150" font-size="12" fill="var(--color-text-muted)">673,627</text>
  <text x="30" y="178" font-size="12" fill="var(--color-text)">output_tokens</text>
  <rect x="250" y="166" width="103" height="16" rx="3" fill="var(--color-accent)" opacity="0.55" />
  <text x="560" y="178" font-size="12" fill="var(--color-text-muted)">231,204</text>
  <text x="30" y="206" font-size="12" fill="var(--color-text)">input_tokens (신규)</text>
  <rect x="250" y="194" width="13" height="16" rx="3" fill="var(--color-accent)" opacity="0.3" />
  <text x="560" y="206" font-size="12" fill="var(--color-text-muted)">28,618</text>
</svg>
</div>

전체의 99.24%가 `cache_read_input_tokens`다. 이게 뭔 소리냐면, **매 API 호출마다 그동안 쌓인 컨텍스트 전체를 캐시로 다시 읽어들여서 보낸다**는 뜻이다.

새로 생성되는 토큰(input+output+cache_creation을 다 합쳐도 93만 개, 0.76%)은 미미한 수준이다. 진짜 무게추는 "누적된 컨텍스트 × 호출 횟수"에 있었다.

세션 시작 시점 컨텍스트는 34,000토큰 정도였는데, 끝날 땐 450,000토큰까지 불어나 있었다. 평균으로 잡아도 호출당 cache_read가 약 267,000토큰씩이었고, 이걸 456번 반복하니 1억 2천만이 됐다.

**모델이 토큰을 많이 "생성"해서가 아니다. 대화를 주고받을 때마다 이미 쌓인 걸 반복해서 다시 실어 나른 게 대부분이었다.**

## 4. 그 456번은 어디에 쓰였나 — Tool 호출 분포

<div style="overflow-x:auto; margin: 2rem 0;">
<svg viewBox="0 0 700 480" width="100%" style="max-width:700px; display:block; margin:0 auto;" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="chart2-title">
  <title id="chart2-title">스코프 내 456회 API 호출 중 tool 호출 분포 (상위 14개)</title>
  <rect x="170" y="8" width="12" height="12" fill="var(--color-accent)" />
  <text x="188" y="18" font-size="12" fill="var(--color-text)">프리뷰 관련 호출</text>
  <rect x="320" y="8" width="12" height="12" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="338" y="18" font-size="12" fill="var(--color-text)">그 외</text>
  <text x="170" y="55" font-size="12" fill="var(--color-text)" text-anchor="end">Bash</text>
  <rect x="170" y="42" width="420" height="20" rx="3" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="598" y="57" font-size="12" fill="var(--color-text-muted)">180</text>
  <text x="170" y="85" font-size="12" fill="var(--color-text)" text-anchor="end">Read</text>
  <rect x="170" y="72" width="119" height="20" rx="3" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="297" y="87" font-size="12" fill="var(--color-text-muted)">51</text>
  <text x="170" y="115" font-size="12" fill="var(--color-text)" text-anchor="end">preview_eval</text>
  <rect x="170" y="102" width="112" height="20" rx="3" fill="var(--color-accent)" />
  <text x="290" y="117" font-size="12" fill="var(--color-text-muted)">48</text>
  <text x="170" y="145" font-size="12" fill="var(--color-text)" text-anchor="end">preview_screenshot</text>
  <rect x="170" y="132" width="77" height="20" rx="3" fill="var(--color-accent)" />
  <text x="255" y="147" font-size="12" fill="var(--color-text-muted)">33</text>
  <text x="170" y="175" font-size="12" fill="var(--color-text)" text-anchor="end">Edit</text>
  <rect x="170" y="162" width="63" height="20" rx="3" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="241" y="177" font-size="12" fill="var(--color-text-muted)">27</text>
  <text x="170" y="205" font-size="12" fill="var(--color-text)" text-anchor="end">TaskUpdate</text>
  <rect x="170" y="192" width="40" height="20" rx="3" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="218" y="207" font-size="12" fill="var(--color-text-muted)">17</text>
  <text x="170" y="235" font-size="12" fill="var(--color-text)" text-anchor="end">AskUserQuestion</text>
  <rect x="170" y="222" width="35" height="20" rx="3" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="213" y="237" font-size="12" fill="var(--color-text-muted)">15</text>
  <text x="170" y="265" font-size="12" fill="var(--color-text)" text-anchor="end">preview_start</text>
  <rect x="170" y="252" width="21" height="20" rx="3" fill="var(--color-accent)" />
  <text x="199" y="267" font-size="12" fill="var(--color-text-muted)">9</text>
  <text x="170" y="295" font-size="12" fill="var(--color-text)" text-anchor="end">preview_inspect</text>
  <rect x="170" y="282" width="21" height="20" rx="3" fill="var(--color-accent)" />
  <text x="199" y="297" font-size="12" fill="var(--color-text-muted)">9</text>
  <text x="170" y="325" font-size="12" fill="var(--color-text)" text-anchor="end">Write</text>
  <rect x="170" y="312" width="19" height="20" rx="3" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="197" y="327" font-size="12" fill="var(--color-text-muted)">8</text>
  <text x="170" y="355" font-size="12" fill="var(--color-text)" text-anchor="end">TaskCreate</text>
  <rect x="170" y="342" width="19" height="20" rx="3" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="197" y="357" font-size="12" fill="var(--color-text-muted)">8</text>
  <text x="170" y="385" font-size="12" fill="var(--color-text)" text-anchor="end">preview_stop</text>
  <rect x="170" y="372" width="16" height="20" rx="3" fill="var(--color-accent)" />
  <text x="194" y="387" font-size="12" fill="var(--color-text-muted)">7</text>
  <text x="170" y="415" font-size="12" fill="var(--color-text)" text-anchor="end">WebSearch</text>
  <rect x="170" y="402" width="5" height="20" rx="3" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="183" y="417" font-size="12" fill="var(--color-text-muted)">2</text>
  <text x="170" y="445" font-size="12" fill="var(--color-text)" text-anchor="end">WebFetch</text>
  <rect x="170" y="432" width="5" height="20" rx="3" fill="var(--color-text-muted)" opacity="0.45" />
  <text x="183" y="447" font-size="12" fill="var(--color-text-muted)">2</text>
  <text x="170" y="470" font-size="11" fill="var(--color-text-muted)">※ 브리핑 원본 표 기준 상위 14개 tool. 456회 중 일부 미분류 호출은 표에서 제외됨.</text>
</svg>
</div>

브리핑에 따르면 프리뷰 관련 호출(`preview_*`) 합계는 **112회로 전체의 약 25%**다. 작은 CSS 값 하나, 텍스트 한 줄 바꾸는 데도 로컬 서버 기동 → 브라우저 프리뷰 → 스크린샷 → 확인이라는 풀 사이클을 계속 돌린 게 그대로 호출 수에 찍혔다.

스크린샷 한 장 자체는 토큰이 크지 않다(800x973px 기준 장당 약 1,038토큰). 문제는 한번 컨텍스트에 들어간 이미지가 **세션이 끝날 때까지 매 호출마다 캐시로 계속 재전송**된다는 것이다.

33장이 누적되면서 그것만으로 약 720만 토큰, 전체 cache_read의 약 6%를 차지했다.

## 5. 확인된 병목 3가지

1. **프리뷰 확인 루프 과다** — 위에서 본 것처럼 전체 호출의 25%. 사소한 수정 하나에도 매번 "로컬 서버 → 프리뷰 → 스크린샷 → 확인" 사이클을 새로 돌렸다.
2. **단일 세션 과잉 지속** — 3시간 23분을 끊지 않고 한 세션에서 진행하다 보니 컨텍스트가 34K에서 450K까지 단조 증가했다. 스캐폴딩 완료, 첫 커밋처럼 자연스러운 체크포인트가 있었는데도 세션을 새로 열거나 `/compact`를 실행하지 않았다.
3. **요청 단위가 지나치게 잘게 쪼개짐** — "명도 낮춰볼래" → 확인 → "마퀴 텍스트 줄여줘" → 확인, 이런 식으로 수정 하나당 한 번씩 왕복했다. 왕복 횟수 자체가 캐시 재전송 배수라서, 관련 수정을 배치로 묶지 않은 게 456회라는 총 호출 수를 불필요하게 늘렸다. (74 turn이 456 API 호출이 된 것도 결국 이 문제다.)

부가 요인으로, 초반에 "클로드코드 아티팩트가 뭐야", "SSG 뭘로 할지" 같은 사전 리서치성 질문을 본 작업과 같은 세션에서 시작한 것도 있다 — 시작 시점부터 컨텍스트가 이미 무거워진 상태였다.

## 6. 앞으로 이렇게 써보려고 한다

### 6.1 브리핑에서 나온 첫 제안

AI가 세션 로그를 실제로 까보고 나서 개선 방안을 우선순위대로 정리해줬다. 그대로 옮겨두면 이렇다.

1. 프리뷰는 로컬 브라우저로 직접 확인하고, 에이전트한테는 프리뷰 도구를 아예 안 쓰게 한다. 개발 서버는 켜둔 채로 코드만 고치게 하고 "반영했음, 확인해줘"로 마무리 — 이건 사실 세션 중에 나도 이미 스스로 도출했던 방향이다.
2. 스캐폴딩 완료, 첫 커밋 같은 자연스러운 마일스톤 단위로 세션을 분리하거나 `/compact`를 실행한다.
3. 관련된 수정 요청은 배치로 모아서 한 번에 전달한다 (왕복 횟수 최소화).
4. Edit 직후 전체 파일을 다시 Read하는 습관을 지양하고, diff를 신뢰하거나 필요한 라인 범위만 확인한다.
5. 개념 질문·기술 비교 같은 사전 리서치성 대화는 본 작업 세션 밖에서 처리하고 결론만 가져온다.

1번은 세션 도중에 이미 스스로 깨달은 부분이라 그대로 두기로 했다. 문제는 2번과 5번이었다. 둘 다 "이게 말로는 그럴듯한데, 실제로 되는 얘기인가?"가 걸렸다.

### 6.2 제안을 검증해보다

2번부터. "마일스톤 단위로 세션을 분리하거나 컴팩트를 실행한다"는 좋은데, 그걸 매번 내가 손으로 판단해서 실행해야 하나 싶었다. 에이전트가 알아서 "지금 커밋도 끝났고 컴팩트하기 좋은 타이밍이네"라고 판단해줄 수는 없을까 궁금해서 물어봤다.

결과는 반반이었다. Claude Code에는 이미 컨텍스트가 한계에 가까워지면 자동으로 요약하는 auto-compaction이 내장돼 있다. 다만 이건 순수 토큰 비율 기준이지, 커밋 같은 의미 있는 지점을 인식하는 건 아니다.

그렇다고 매 턴마다 "지금이 컴팩트할 시점인가?"를 에이전트한테 판단시키는 것도 생각해봤다. 계산해보니 이건 오히려 손해였다. 판단 자체에 드는 토큰이 456번의 호출 규모에서는 아끼려는 것보다 더 나갈 수 있다.

대안으로 나온 게 `PostToolUse` 훅이었다. Bash 명령어에 `git commit`이 들어있고 성공했는지만 셸 스크립트로 체크한다. 맞으면 "방금 커밋했으니 컴팩트를 제안하라"는 짧은 문구만 컨텍스트에 끼워 넣는 방식이다. 이건 LLM 판단이 아니라 단순 조건문이라 토큰 비용이 거의 안 든다.

최종적으로는 "내장 auto-compact를 안전망으로 깔아두고, 커밋 감지 훅으로 더 이른 타이밍에 제안받는" 조합으로 정리했다.

5번도 비슷하게 짚어봤다. "리서치성 질문은 세션 밖에서"라고 했는데, 이걸 자동으로 감지해서 라우팅하는 게 기술적으로 가능한지 물어봤다.

Claude Code 공식 문서를 보면 실제로 "테스트 실행이나 로그 처리처럼 컨텍스트를 많이 먹는 작업은 서브에이전트에 위임하라, 장황한 출력은 서브에이전트 안에만 남고 요약만 돌아온다"는 패턴을 권장하고 있었다. 그런데 "이 질문이 개념 질문인지 아닌지"를 자동으로 판별하는 훅을 만드는 건 자연어 판별의 신뢰도 문제로 과한 설계였다.

더 재밌었던 건, 서브에이전트를 세션 안에서 띄우는 것보다 완전히 별도의 대화창을 쓰는 게 오히려 더 가볍다는 점이었다. 서브에이전트도 프로젝트의 CLAUDE.md나 MCP 설정을 그대로 상속해서 불러오기 때문에, 순수 개념 질문 하나 처리하는 데도 그 로딩 비용이 붙는다. 완전히 딴 대화는 그게 아예 없다.

그래서 자동화 훅을 만들기보다 CLAUDE.md에 원칙 한 줄만 적어두는 쪽으로 결론 냈다.

### 6.3 보완된 최종안

검증 결과를 반영해서 다시 정리하면 이렇다.

1. 프리뷰는 로컬 브라우저로 직접 확인, 에이전트는 프리뷰 도구를 아예 안 쓴다. (변경 없음)
2. 컴팩트는 내장 auto-compact를 기본 안전망으로 두고, `git commit` 명령을 감지하는 `PostToolUse` 훅을 추가해서 커밋 직후 컴팩트를 제안받는다. 에이전트에게 매 턴 판단을 맡기는 방식은 배제한다.
3. 관련된 수정 요청은 배치로 모아서 한 번에 전달한다. (변경 없음)
4. Edit 직후 전체 파일 재확인은 지양하고, diff를 신뢰하거나 필요한 라인만 확인한다. (변경 없음)
5. 개념 질문·기술 비교는 서브에이전트가 아니라 아예 별도의 대화창에서 처리하고 결론만 가져온다. 자동 라우팅은 만들지 않고, CLAUDE.md에 이 원칙만 명시해둔다.

결국 남은 건 "자동화할 수 있는 것"과 "자동화할 가치가 없는 것"을 가려내는 일이었다. 손이 좀 가더라도 사람이 판단하는 게 더 싼 경우가 분명히 있었다.

## 7. 그런데 이 글도 그 병목 위에서 쓰고 있었다

<div style="text-align:center; margin: 2rem 0;">
<img src="/images/token-cost-expectation.gif" alt="내가 생각한 내 모습은 이거였는데" style="width:50%; max-width:100%; display:inline-block; border-radius:0.75rem; border:1px solid color-mix(in srgb, var(--color-border) 60%, transparent);" /><div style="font-size:0.85rem; color:var(--color-text-muted);">내가 생각한 내 모습은 이거였는데</div>
</div>

<div style="text-align:center; margin: 2rem 0;">
<img src="/images/token-cost-reality.jpg" alt="사실 이러고 있었던 거다" style="width:60%; max-width:100%; display:inline-block; border-radius:0.75rem; border:1px solid color-mix(in srgb, var(--color-border) 60%, transparent);" /><div style="font-size:0.85rem; color:var(--color-text-muted);">사실 이러고 있었던 거다</div>
</div>

여기까지 정리하고 나서 좀 머쓱해진 게 있다. 사실 이 포스트 초안도 처음엔 Claude Code에서 쓰고 있었다. 로그 파싱하고 표 만드는 건 코드 세션에서 했다 쳐도, 그다음에 "이걸 어떻게 서술할지", "문단을 어떻게 배치할지" 같은 순수 글쓰기까지 같은 세션 안에서 계속 이어가고 있었다.

5번 개선안("리서치성 대화는 세션 밖에서")을 정리해놓고 바로 다음 문단에서 똑같은 실수를 하고 있었던 셈이다.

그래서 이번엔 실제로 바꿔봤다. 로그 분석과 브리핑 문서 작성은 별도의 Claude 세션에서 끝내고, 그 브리핑을 근거로 이 포스트 초안도 같은 세션에서 계속 다듬었다. Claude Code로는 최종 파일 배치와 커밋·푸시만 넘길 생각이다.

코드 세션은 프로젝트의 CLAUDE.md, 붙어있는 MCP 서버, 도구 목록을 매번 새로 불러온다. 순수하게 문장을 다듬는 작업엔 이 오버헤드가 전혀 필요 없다. 지금 이 문단도 그렇게 쓰고 있다.

여기서 좀 더 신경 쓰이는 지점이 있었다. 나는 이번에 세션 로그를 직접 까봐서 74턴이 456번의 API 호출이 되는 과정을 눈으로 확인했다. 하지만 Claude와 Claude Code가 토큰을 쌓고 재전송하는 방식이 근본적으로 다르다는 걸 모르는 채로 계속 코드 세션 안에서 글도 쓰고 리서치도 했다면 어땠을까.

그 차이를 인지하지 못한 채로 새는 토큰의 총량은 아마 지금 확인한 것보다 훨씬 컸을 거다. 문제는 이게 겉으로 티가 안 난다는 거다. 에러가 나거나 눈에 띄게 느려지는 게 아니라, 그냥 매 턴 조용히 캐시 재전송량만 불어나는 거라 자각하기가 어렵다.

이번처럼 로그를 실제로 뜯어보지 않았으면 나도 그냥 "느낌상 좀 오래 걸리네" 정도로 넘어갔을 것 같다.

## 8. 마무리

숫자로 뜯어보고 나니 "느리다"는 체감이 막연한 인상이 아니라 구조적인 이유가 있는 문제였다는 게 확실해졌다. 캐시 재전송이라는 메커니즘 자체는 어쩔 수 없다. 결국 줄일 수 있는 건 **호출 횟수와 세션 길이**뿐이다.

그리고 바로 앞에서 확인했듯, 이 글을 쓰는 방식 자체도 줄여야 할 대상 중 하나였다.

다음 포스트부터는 위에서 정리한 다섯 가지를 실제로 적용해볼 생각이다. 다만 같은 방식으로 다시 로그를 파서 실측하고 비교 포스팅까지 할지는 솔직히 미정이다.

내가 변덕이 좀 있는 편이라, 이번처럼 로그 뜯어볼 마음이 다시 들지는 그때 가봐야 알 것 같다. 적용은 하되, 그 결과를 또 숫자로 확인해서 글로 남길지는 그냥 열어두려고 한다.

> 데이터 출처: 사용자가 업로드한 세션 로그(`session-export-*.zip` 내 JSONL)를 별도 세션에서 직접 파싱. 각 라인의 `message.usage`에서 토큰 수치를, `message.content[].type == "tool_use"`에서 tool 호출을 `message.id` 기준 중복 제거 후 집계했다. 스크린샷 토큰은 tool_result 내 이미지의 실제 픽셀 크기에 Anthropic 공식 이미지 토큰 공식((width×height)/750)을 적용해 추정한 값이다.
