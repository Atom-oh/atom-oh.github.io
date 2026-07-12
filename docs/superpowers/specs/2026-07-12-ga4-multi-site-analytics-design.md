# GA4 통합 트래킹 + 비공개 통계 대시보드 설계

## Context

메인 프로필 사이트(`atom-oh.github.io`)에는 이미 GA4(측정 ID `G-GWVLEW5JLL`)가 붙어 있다. 사이트의 Projects 섹션에서 소개하는 하위 7개 GitHub Pages 프로젝트(kubernetes-docs, multi-region-architecture, awsops, aws-ec2-benchmark, reactive_presentation, oh-my-cloud-skills, blog)는 별도 GitHub 리포에서 호스팅되며 트래킹이 없다. 사용자는 이 7개 사이트에도 GA4를 붙이고, 전체 트래픽 통계를 메인 프론트엔드에서(본인만) 확인하고 싶어한다.

조사 결과 이 환경에는 GA4 Data API를 직접 조회하는 MCP/플러그인이 없고, GA4 Data API는 OAuth/서비스 계정 인증이 필요해 정적 GitHub Pages에서 클라이언트 사이드로 안전하게 호출할 수 없다. 대안으로 백엔드(Lambda 등)를 새로 구축하는 방법도 있지만, 이번 목적(본인만 보는 대시보드)에는 과합니다.

## 결정 사항

- **공개 대상**: 본인만. 방문자에게는 통계를 노출하지 않는다.
- **구현 방식**: Looker Studio 리포트를 GA4에 연결하고, 공유 설정을 "나만"으로 지정해 구글 로그인으로 실제 접근 제어. 백엔드/커스텀 인증 코드 없음.
- **GA4 속성 구성**: 기존 측정 ID(`G-GWVLEW5JLL`) 하나로 통일해 7개 하위 사이트에 그대로 부착. GA4 안에서는 hostname/경로로 사이트를 구분하고, Looker Studio에서는 데이터소스 하나로 통합 대시보드를 구성한다.

## 아키텍처

```
[하위 7개 GitHub Pages 사이트] --gtag.js(G-GWVLEW5JLL)--> [GA4 속성 (기존)]
                                                                  │
                                                      Looker Studio 리포트
                                                      (공유 설정: "나만")
                                                                  │
[atom-oh.github.io/stats.html] --iframe embed-->  구글 로그인 검사
   (nav에는 링크 없음, 직접 URL 접근만)              본인 아니면 "접근 권한 없음" 화면
```

## 작업 분담

### 1. 하위 7개 리포에 GA4 태그 부착 (담당: 나)

각 리포는 정적 사이트 생성기가 달라 삽입 지점이 다르다. 확인된 것:

| 리포 | 생성기/구조 | 삽입 방법(예상) |
|---|---|---|
| kubernetes-docs | VitePress | `.vitepress/config.*`의 `head` 배열에 gtag 스크립트 추가 |
| multi-region-architecture | `docs/` 하위 순수 HTML/MD | 배포되는 각 HTML 파일에 직접 `<script>` 삽입 (또는 공유 헤더 include가 있으면 거기에) |
| awsops | `docs-site/` 하위 별도 생성기(미확인) | 리포 진입 후 생성기 확인 후 결정 |
| aws-ec2-benchmark | `site/` 하위 별도 생성기(미확인) | 리포 진입 후 생성기 확인 후 결정 |
| oh-my-cloud-skills | `doc-sites/` 하위 별도 생성기(미확인) | 리포 진입 후 생성기 확인 후 결정 |
| reactive_presentation | 로컬 클론 없음, 구조 미확인 | 클론 후 확인 |
| blog | 로컬 클론 없음, 구조 미확인 | 클론 후 확인 |

원칙: 생성기가 GA4용 네이티브 옵션(config 필드 등)을 제공하면 그것을 사용하고, 순수 정적 HTML이면 메인 사이트와 동일한 `<script async src="...gtag/js?id=G-GWVLEW5JLL">` 스니펫을 직접 삽입한다. 리포별로 커밋 후 각각 개별 push (사용자 확인 후).

### 2. Looker Studio 리포트 생성 (담당: 사용자, 수동)

구글 로그인이 필요한 단계라 내가 대신 할 수 없다. 안내할 절차:
1. lookerstudio.google.com 접속, 본인 구글 계정으로 로그인
2. 새 데이터소스 추가 → Google Analytics 커넥터 → GA4 속성(`G-GWVLEW5JLL`) 선택
3. 새 리포트 생성 (방문자 수, 페이지별 트래픽, 호스트명별 트래픽 등 기본 차트)
4. 공유 설정을 "특정 사용자"로 바꾸고 본인 구글 계정만 추가 (링크 소지자 전체 공개 금지)
5. 파일 → 삽입 리포트 → 임베드 URL 복사해서 나에게 전달

### 3. `stats.html` 추가 (담당: 나, 임베드 URL 받은 후)

- `atom-oh.github.io/stats.html` 신규 생성. 기존 다크 테마(`--bg`, `--card` 등 CSS 변수) 안에 Looker Studio iframe을 얹는 최소 래퍼 페이지.
- 헤더/사이드바 nav에는 링크를 걸지 않음 (직접 URL 접근만). 나중에 원하면 링크 추가는 한 줄로 가능.

## 테스트/검증

- 각 하위 사이트 배포 후 GA4 실시간 보고서에서 해당 hostname의 이벤트 수신 확인 (`curl`로 확인 불가능한 부분이라 GA4 콘솔에서 사용자가 직접 확인).
- `stats.html`은 로그인 상태/비로그인 상태 브라우저 두 가지로 열어 접근 제어가 실제로 동작하는지 확인 (비로그인 시 "권한 없음" 화면 뜨는지).

## 범위 제외

- 공개 방문자용 카운터/배지는 만들지 않음 (본인만 보는 대시보드로 확정).
- GA4 Data API를 감싸는 커스텀 백엔드(Lambda 등)는 만들지 않음 — 필요해지면 추후 별도 스펙으로 진행.
