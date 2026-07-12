# GA4 통합 트래킹 + 비공개 통계 대시보드 설계

## Context

메인 프로필 사이트(`atom-oh.github.io`)에는 이미 GA4(측정 ID `G-GWVLEW5JLL`)가 붙어 있다. 사이트의 Projects 섹션에서 소개하는 하위 7개 GitHub Pages 프로젝트(kubernetes-docs, multi-region-architecture, awsops, aws-ec2-benchmark, reactive_presentation, oh-my-cloud-skills, blog)는 별도 GitHub 리포에서 호스팅되며 트래킹이 없다. 사용자는 이 7개 사이트에도 GA4를 붙이고, 전체 트래픽 통계를 메인 프론트엔드에서(본인만) 확인하고 싶어한다.

조사 결과 이 환경에는 GA4 Data API를 직접 조회하는 MCP/플러그인이 없고, GA4 Data API는 OAuth/서비스 계정 인증이 필요해 정적 GitHub Pages에서 클라이언트 사이드로 안전하게 호출할 수 없다.

## 리비전 노트 (2026-07-12)

최초 설계는 백엔드 없이 Looker Studio 임베드로 가는 방향이었다. 목업 두 가지(Looker Studio 다크테마 vs awsops 스타일 커스텀 대시보드)를 비교한 뒤 사용자가 **awsops 스타일 커스텀 대시보드**를 선택했고, 이 룩앤필로 실제 GA4 숫자를 보여주려면 백엔드가 필요하다는 트레이드오프를 인지한 채로 **Lambda + Google Sign-In 구축을 진행**하기로 결정했다. 아래 내용은 그 결정을 반영한 최종 설계다.

## 결정 사항

- **공개 대상**: 본인만(`ojs0106@gmail.com`). 방문자에게는 통계를 노출하지 않는다.
- **UI**: 기존 프로필 사이트 다크 테마(`--bg:#0d1117`, `--accent:#FF9900` 등)를 그대로 쓰고, awsops 대시보드 스타일(스탯 카드 그리드 + 시안 바차트 + 컬러 도넛)을 재현. 목업 파일: `.superpowers/brainstorm/`에 남긴 `stats-mockup-compare.html`의 Option B.
- **접근 제어**: 구글 로그인(Google Identity Services) — 커스텀 백엔드 인증 코드를 최소화하면서도 "본인 구글 계정만" 이라는 보안 속성은 유지한다.
- **GA4 속성 구성**: 기존 측정 ID(`G-GWVLEW5JLL`) 하나로 통일해 7개 하위 사이트에 그대로 부착. GA4 안에서는 hostname/경로로 사이트를 구분.
- **인프라 도구**: Terraform. 사용자의 다른 리포(awsops, aws-ec2-benchmark, multi-region-architecture)가 모두 Terraform 기반이라 관례를 따른다. (CDK/SAM으로 바꾸고 싶으면 언제든 조정 가능)
- **Lambda 런타임**: Python. GA4 Data API/Google 인증 라이브러리가 성숙해 있고, `aws-core:aws-sdk-python-usage` 스킬과도 맞음.

## 아키텍처

```
[하위 7개 GitHub Pages 사이트] --gtag.js(G-GWVLEW5JLL)--> [GA4 속성 (기존)]
                                                                  │
                                                        GA4 Data API
                                                                  │
[atom-oh.github.io/stats.html]                         [Lambda Function URL]
   "Sign in with Google" 버튼                          1) 구글 ID 토큰 서명/발급자 검증
   (Google Identity Services, 무료)                    2) email == ojs0106@gmail.com 확인
        │  ID token                                    3) GA4 서비스 계정으로 Data API 호출
        └──────────────── Authorization: Bearer ──────>  4) JSON 응답 (방문자/PV/디바이스 등)
                                                                  │
[stats.html] <── JSON ── 카드+바차트+도넛(awsops 스타일)로 렌더링
```

**자격증명 저장**: GA4 서비스 계정 키는 AWS Secrets Manager에 저장하고 Lambda가 런타임에 조회. 코드/리포에는 절대 평문으로 두지 않는다 (`aws-core:aws-secrets-manager` 스킬 원칙 준수 — 시크릿 값이 LLM 컨텍스트에 노출되지 않도록 런타임 동적 참조 사용).

## 작업 분담

### 1. 하위 7개 리포에 GA4 태그 부착 (담당: 나)

각 리포는 정적 사이트 생성기가 달라 삽입 지점이 다르다. 확인된 것:

| 리포 | 생성기/구조 | 삽입 방법(예상) |
|---|---|---|
| kubernetes-docs | VitePress | `.vitepress/config.*`의 `head` 배열에 gtag 스크립트 추가 |
| multi-region-architecture | `docs/` 하위 순수 HTML/MD | 배포되는 각 HTML 파일에 직접 `<script>` 삽입 (또는 공유 헤더 include가 있으면 거기에) |
| awsops | `docs-site/` (Docusaurus) | `docusaurus.config.ts`의 `gtag` 프리셋 옵션 사용 |
| aws-ec2-benchmark | `site/` 하위 별도 생성기(미확인) | 리포 진입 후 생성기 확인 후 결정 |
| oh-my-cloud-skills | `doc-sites/` 하위 별도 생성기(미확인) | 리포 진입 후 생성기 확인 후 결정 |
| reactive_presentation | 로컬 클론 없음, 구조 미확인 | 클론 후 확인 |
| blog | 로컬 클론 없음, 구조 미확인 | 클론 후 확인 |

원칙: 생성기가 GA4용 네이티브 옵션(config 필드 등)을 제공하면 그것을 사용하고, 순수 정적 HTML이면 메인 사이트와 동일한 `<script async src="...gtag/js?id=G-GWVLEW5JLL">` 스니펫을 직접 삽입한다. 리포별로 커밋 후 각각 개별 push (사용자 확인 후).

### 2. Google Cloud 쪽 수동 설정 (담당: 사용자, 구글 로그인 필요해서 대행 불가)

1. Google Cloud 프로젝트 준비, **Google Analytics Data API** 활성화
2. **OAuth 2.0 클라이언트 ID**(웹 애플리케이션) 생성 — 승인된 자바스크립트 원본에 `https://www.atomai.click` 추가. 이 클라이언트 ID는 비밀값 아님 (프론트엔드 코드에 그대로 들어감).
3. **서비스 계정** 생성 → GA4 속성의 "속성 액세스 관리"에서 이 서비스 계정 이메일을 뷰어로 추가 → JSON 키 발급
4. 발급받은 JSON 키는 **사용자가 직접** `aws secretsmanager create-secret --secret-string file://키파일경로` 로 업로드 (내가 명령어만 안내). 키 내용을 대화창에 붙여넣거나 나에게 전달하지 않는다 — 시크릿 값은 절대 LLM 컨텍스트에 들어오지 않게 한다.

### 3. Lambda 백엔드 구축 (담당: 나)

- Terraform으로 Lambda + Function URL 정의. 코드: 요청의 `Authorization: Bearer <id_token>`을 Google의 공개키로 검증 → `email`/`email_verified` 클레임이 `ojs0106@gmail.com`인지 확인 → Secrets Manager에서 서비스 계정 키 조회 → GA4 Data API(`runReport`)로 방문자 수/페이지별 PV/디바이스 비중 조회 → JSON 반환.
- CORS는 `https://www.atomai.click` 오리진만 허용.

### 4. `stats.html` 추가 (담당: 나)

- `atom-oh.github.io/stats.html` 신규 생성. 목업 Option B 스타일(스탯 카드 4개 + 사이트별 PV 바차트 + 디바이스 도넛) 그대로 구현.
- Google Identity Services 스크립트로 로그인 버튼 렌더링 → 로그인 성공 시 ID 토큰을 Lambda Function URL로 전달 → 응답 JSON으로 카드/차트 채움.
- 로그인 실패/타인 계정이면 "접근 권한 없음" 문구만 표시, 데이터는 아예 요청하지 않음.
- 헤더/사이드바 nav에는 링크를 걸지 않음 (직접 URL 접근만).

## 테스트/검증

- 각 하위 사이트 배포 후 GA4 실시간 보고서에서 해당 hostname의 이벤트 수신 확인 (`curl`로 확인 불가능한 부분이라 GA4 콘솔에서 사용자가 직접 확인).
- Lambda: 다른 구글 계정으로 로그인해 401/403 응답 확인, 본인 계정으로 로그인해 정상 JSON 응답 확인.
- `stats.html`을 로그인 전/후 상태로 열어 렌더링이 기대대로 전환되는지 확인.

## 범위 제외

- 공개 방문자용 카운터/배지는 만들지 않음 (본인만 보는 대시보드로 확정).
- Workload Identity Federation 등 서비스 계정 키를 대체하는 고급 인증 방식은 이번 스코프에서 제외 (필요해지면 후속 스펙).
