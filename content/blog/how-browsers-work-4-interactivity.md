+++
title = "브라우저는 어떻게 화면을 그리는가 4편: 상호작용(Interactivity)과 정리"
date = "2026-07-16T13:15:00+09:00"
draft = false
tags = ["browser-internals", "web-performance", "rendering-pipeline", "frontend", "학습노트"]
categories = ["일지"]
description = "브라우저 동작 원리 4부작 완결편. 입력 이벤트와 적중 테스트, TTI/INP, 그리고 다섯 단계 전체를 관통하는 실무 최적화 요약표"
+++

**브라우저 동작 원리 시리즈 (총 4부작)** — [1편 탐색·응답](/blog/how-browsers-work-1-navigation-response/) · [2편 구문 분석](/blog/how-browsers-work-2-parsing/) · [3편 렌더](/blog/how-browsers-work-3-render/) · **4편 상호작용·정리(이 글)**

지난 편까지 브라우저가 URL을 받아 화면에 픽셀을 그리는 과정(탐색 → 응답 → 구문 분석 → 렌더)을 다뤘다. 이번 편은 그렇게 그려진 화면과 사용자가 실제로 주고받는 마지막 단계, 그리고 시리즈 전체를 관통하는 최적화 요약이다.

---

## 5단계: 상호작용(Interactivity)

상호작용 요소가 전혀 없는 순수 문서형 페이지도 있지만, 대부분의 웹페이지는 사용자와 주고받는 대화형 페이지다. 이 상호작용이 처리되는 단계다.

### 입력 이벤트

브라우저 관점에서는 사용자의 모든 움직임이 입력 이벤트다. `input`에 텍스트를 넣거나 링크를 클릭하는 것뿐 아니라, 마우스 휠 스크롤, 마우스 오버, 탭 포커스까지 전부 포함된다.

사용자가 화면을 클릭했다고 하면, 브라우저는 클릭 좌표와 이벤트 유형(`ClickEvent`)을 수신해 렌더 프로세스로 넘긴다.

### 적중 테스트(Hit Test)

이벤트 데이터를 넘겨받은 렌더러 프로세스는 **적중 테스트**를 실행해서 이벤트 대상을 찾는다. 방금 페인트 단계에서 만들어진 화면 데이터를 활용해, 수신된 좌표에 어떤 요소가 있는지 알아내는 것이다. 대상을 찾으면 거기 연결된 이벤트 리스너를 실행(Trigger)한다.

![적중 테스트 흐름](/images/browser-hit-test-webdev.png "Browser Process → Renderer Process Compositor → 적중 테스트 → Main Thread가 클릭을 인식하는 흐름 — 출처: Mariko Kosaka (web.dev, 2018)")

### Time to Interactive (TTI)

TTI는 브라우저가 페이지에 처음 진입한 순간(DNS 조회 시작)부터, 지금 이 상호작용 단계까지 걸리는 시간을 말한다. 기준은 다음과 같다.

- FCP 시점 이후
- 화면에 보이는 대부분의 요소에 이벤트 핸들러가 등록되어 있어야 함
- 사용자 상호작용에 50ms 이내로 응답 가능한 상태

> **업데이트 포인트**: TTI 역시 "Web Vitals Performance의 핵심 기준"이라고 하기엔 정확하지 않다. TTI는 Lighthouse가 측정하는 **성능 진단 지표**이고, 2024년 개편된 **Core Web Vitals**(LCP·INP·CLS)에는 포함되지 않는다. 특히 TTI가 다루던 "상호작용 응답성"이라는 문제의식은 지금은 **INP**(Interaction to Next Paint)가 이어받아 더 정교하게 측정하고 있다 — 클릭 한 번이 아니라 페이지 생애주기 전체의 상호작용 응답 속도를 본다는 점에서 TTI보다 실사용 체감에 더 가깝다.

---

## 정리: 단계별로 뭘 최적화할 수 있나

| 단계      | 병목 포인트                           | 실무에서 손댈 수 있는 것                                    |
| --------- | -------------------------------------- | ----------------------------------------------------------- |
| 탐색      | 참조 도메인이 많을수록 DNS 조회 반복  | `dns-prefetch`, `preconnect`로 미리 연결 준비               |
| 응답      | TTFB, 서버 응답 지연                  | 서버 응답 속도 개선, CDN, 캐싱 헤더                         |
| 구문 분석 | script 블로킹, 큰 CSS/JS 번들         | `async`/`defer`, 코드 스플리팅, critical CSS 인라인         |
| 렌더      | 잦은 Reflow/Repaint, 큰 렌더 트리     | DOM 깊이·개수 최소화, `transform`/`opacity` 위주 애니메이션 |
| 상호작용  | 무거운 메인 스레드 작업으로 응답 지연 | 긴 태스크 쪼개기, Web Worker로 오프로드                     |

페이지 로드 전체를 타임라인 하나로 겹쳐 보면 이렇다.

{{< chart-load-timeline >}}

브라우저가 URL 하나 받아서 여기까지 오는 동안 하는 일이 이렇게 많다. 결국 "체감 성능을 개선한다"는 건 이 다섯 단계 중 **어디서 시간이 새고 있는지를 정확히 짚어내는 일**이라는 걸, 이번에 자료를 다시 정리하면서 새삼 느꼈다.

---

**시리즈 전체 목록** — [1편 탐색·응답](/blog/how-browsers-work-1-navigation-response/) · [2편 구문 분석](/blog/how-browsers-work-2-parsing/) · [3편 렌더](/blog/how-browsers-work-3-render/) · 4편 상호작용·정리(이 글)

**← 이전 편** [3편: 렌더(Render)](/blog/how-browsers-work-3-render/)
