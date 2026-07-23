# ConnectingRoad — Web

**배포**: [dev-web.connectforme.com](https://dev-web.connectforme.com) | **개발 기간**: 2025.05 ~ 진행 중

> 커리어 목표 및 퀴즈 기반 역량 분석으로 맞춤 로드맵을 추천하는 서비스
> 사이드 프로젝트로 시작해 현재 법인화를 진행 중인 팀 프로젝트입니다. 본 레포지토리는 비공개 팀 프로젝트의 웹 담당 기여를, 코드 대신 기술적 의사결정 중심으로 정리한 문서 레포입니다.

## 목차
- [프로젝트 소개](#프로젝트-소개)
- [FE 기술 스택](#fe-기술-스택)
- [아키텍처](#아키텍처)
- [기술적 의사결정](#-기술적-의사결정)
  - [1. FSD 아키텍처 전환 제안·주도](#1-fsd-아키텍처-전환-제안주도)
  - [2. 배럴 export의 부작용 코드가 Tree Shaking을 막던 문제](#2-배럴barrel-export의-부작용-코드가-tree-shaking을-막던-문제-역추적제거)
  - [3. PostHog 에러 모니터링 4개 지점 캡처 구조 설계](#3-posthog-에러-모니터링-4개-지점-캡처-구조-설계)
  - [4. 로드맵 생성 폼 필드 검증 및 에러 복구 플로우 설계](#4-로드맵-생성-폼-필드-검증-및-에러-복구-플로우-설계)
- [화면](#-화면)

## 프로젝트 소개
ConnectingRoad는 퀴즈 기반 역량 분석으로 사용자의 부족 역량을 진단하고, 맞춤 커리어 로드맵을 추천하는 서비스입니다. 사이드 프로젝트로 출발했지만 현재 법인화가 진행 중이라, 실제 서비스 운영을 전제로 한 의사결정(에러 모니터링, 데이터 검증 등)이 요구되는 환경이었습니다.

- **팀 구성**: 6인 사이드 팀 프로젝트 (기획 1 · 마케팅 1 · 백엔드 2 · 디자인 1 · 프론트엔드 1)
- **본인 역할**: 웹 프론트엔드 전담 (홈 라우팅 제외 web 전체)



## FE 기술 스택

![Next.js](https://img.shields.io/badge/Next.js-000000?style=flat-square&logo=next.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-06B6D4?style=flat-square&logo=tailwindcss&logoColor=white)
![shadcn/ui](https://img.shields.io/badge/shadcn%2Fui-000000?style=flat-square&logo=shadcnui&logoColor=white)
![TanStack Query](https://img.shields.io/badge/TanStack_Query-FF4154?style=flat-square&logo=reactquery&logoColor=white)
![React Hook Form](https://img.shields.io/badge/React_Hook_Form-EC5990?style=flat-square&logo=reacthookform&logoColor=white)
![Zod](https://img.shields.io/badge/Zod-3E67B1?style=flat-square&logo=zod&logoColor=white)
![Zustand](https://img.shields.io/badge/Zustand-443E38?style=flat-square)
![PostHog](https://img.shields.io/badge/PostHog-000000?style=flat-square&logo=posthog&logoColor=white)



## 아키텍처
`pnpm workspace` 기반 모노레포 구조로, 백엔드팀이 주도해 웹/서버 패키지를 분리했습니다. 이 과정에서 웹 패키지의 빌드·배포 스크립트를 워크스페이스 루트 `package.json`에서 웹 패키지만 지정해 독립적으로 빌드·배포할 수 있도록 재구성했습니다.

FSD 레이어 구조는 다음과 같습니다.

​<img width="650" height="553" alt="image" src="https://github.com/user-attachments/assets/afd4b5cf-9623-418d-90f6-40780eb4d9c6" />



## 🔍 기술적 의사결정

아래 6개는 이력서에 압축된 의사결정을 이 프로젝트 안에서 서로 어떻게 이어졌는지 볼 수 있도록 정리한 것입니다. 이 중 4개(FSD·배럴 최적화·PostHog·로드맵 가드)는 코드·버그 레벨의 상세 근거를 노션 문서에 별도로 정리해뒀고, 여기서는 프로젝트 흐름 안에서의 판단만 짚습니다. 나머지 2개(RHF+Zod·퀴즈 신뢰도)는 다른 기록이 없어 여기서 전체 과정을 다룹니다.

### 1. FSD 아키텍처 전환 제안·주도

#### 왜 문제였는가
원래 ConnectingRoad는 바닐라 JS 기반 코드베이스였고, 팀은 이를 React로 마이그레이션하는 작업을 진행하고 있었습니다. 마이그레이션 과정에서 폴더가 components/, hooks/, apis/처럼 역할별로 나뉘고 있었는데, 폴더 하나에 컴포넌트가 10개 이상 섞여 있어서 기능 하나를 이해하려면 서로 다른 세 개 폴더를 오가며 코드를 찾아야 했습니다. 기능이 늘어날수록 이 탐색 비용이 선형으로 커질 게 분명했습니다.

#### 어떻게 결정했는가
마이그레이션이 아직 진행 중이라 구조를 다시 잡을 수 있는 사실상 마지막 타이밍이라고 보고, React 마이그레이션 계획을 Next.js + FSD 아키텍처 도입으로 확장할 것을 제안했습니다. 점진적 개선과 Atomic Design도 함께 검토했지만, 둘 다 탐색 비용 문제의 근본 원인(관심사가 역할로 흩어지는 것)을 해결하지 못한다고 판단했습니다. 다른 팀원의 "기존 구조를 점진적으로 개선하는 게 낫지 않냐"는 반대 의견에는, 현재 구조에서 겪고 있는 구체적 문제를 먼저 정리하고 FSD가 그 문제를 어떻게 해결하는지 하나씩 매핑해서 제시했고, 마침 프레임워크 전환 시점과 맞물려 합의는 비교적 빠르게 이뤄졌습니다.

#### 결과, 그리고 트레이드오프
기능 단위 응집 구조로 전환해 탐색 비용을 낮췄습니다. 다만 공짜는 아니었습니다. FSD 레이어 규칙(하위 레이어만 참조, Public API로만 노출)을 팀 전체가 처음부터 완벽히 지키긴 어려워서, 이후 작업(에러 모니터링 도입 등)에서도 배럴 export 규칙 위반이 종종 발견됐고, 지금도 레이어 경계는 꾸준히 점검해야 하는 항목입니다. 또한 FSD 적용 과정에서 슬라이스마다 진입점(Public API, `index.ts`)을 두는 관례가 함께 들어왔는데, 이 관례가 바로 다음 항목의 번들링 문제로 이어졌습니다.

> FSD 레이어 구조, thin shell 패턴, 실제 리팩토링 중 발견한 버그(Rules of Hooks 위반, `ensureQueryData` 캐시 null 이슈)는 [노션 문서 - 기술 상세](https://ahead-bug-3de.notion.site/FSD-39db438e5ecb8110837ddcd2c37fea7a?source=copy_link)에 정리했습니다.

### 2. 배럴(barrel) export의 부작용 코드가 Tree Shaking을 막던 문제 역추적·제거

#### 왜 문제였는가

로드맵 섹션은 산업군 선택부터 결과 분석까지 10개 넘는 화면으로 구성돼 있습니다. 화면별 초기 로딩 용량을 점검하다가, 결과 분석 화면에만 필요한 레이더 차트가 있는데도 그래프가 전혀 없는 다른 화면들의 초기 로딩 용량이 거의 똑같이 크다는 걸 발견했습니다.

#### 어떻게 결정했는가

여러 화면이 공통으로 참조하는 배럴(index.ts)이 그래프 컴포넌트를 최상단에서 export하고 있었고, 그 안의 차트 라이브러리가 모듈 로드 즉시 초기화 코드를 실행하는 부작용 때문에 번들러가 트리셰이킹을 안전하게 수행하지 못하고 있었습니다. 이 배럴을 전혀 참조하지 않는 순수 정적 페이지가 수정 전에도 공유 베이스라인(103KB)에 머물러 있었다는 걸 대조군으로 확인해 원인을 교차 검증한 뒤, 배럴에서 그래프 컴포넌트 export를 제거하고 실제로 필요한 화면 하나에서만 동적 import로 분리했습니다.

#### 결과와 트레이드오프
<img width="628" height="406" alt="image" src="https://github.com/user-attachments/assets/45bfce8b-1960-4145-9fc5-860359cd9f71" />

`next build` 실측 기준(2026-07-19), 로드맵 섹션 전체 화면의 초기 로딩 용량이 평균 15% 줄었습니다(예: `/home/roadmap` 436KB→367KB). 그래프가 필요한 화면조차 "필요할 때만" 코드를 불러오는 방식으로 바뀌며 초기 로딩이 더 가벼워졌습니다.

다만 배럴에서 export를 걷어내면서, 이 컴포넌트를 쓰는 쪽의 import 경로가 `@/features/roadmap`(배럴)에서 `@/features/roadmap/ui/CompetencyRadarChart`(파일 직접 경로)로 길어졌습니다. 배럴이 주던 "한 곳에서 다 가져다 쓰는" 편의는 포기한 대신, 비슷한 부작용 있는 모듈이 배럴에 다시 섞여 들어가는 걸 막을 린트/CI 게이트가 필요하다고 보고 있습니다(아직 미도입).

> 화면별 수정 전/후 실측 표, 대조군 교차 검증 근거, 배럴 export 코드는 [노션 문서 - 기술 상세](https://ahead-bug-3de.notion.site/3a2b438e5ecb816783aae92bf2ce1bfa?source=copy_link)에 정리했습니다.

### 3. PostHog 에러 모니터링 4계층 캡처 구조 설계

#### 왜 문제였는가

에러가 발생해도 Slack으로 공유받는 네트워크 탭 스크린샷이 유일한 추적 수단이었습니다. 어떤 API가 어떤 상태 코드로 실패했는지 스크린샷만으로 특정하기 어려웠고, 프론트엔드 런타임 에러는 재현조차 힘들었습니다. 무엇보다 실제 사용자가 마주친 에러는 데이터를 수집할 방법 자체가 없었습니다.

#### 어떻게 결정했는가

Sentry 도입을 검토하다가 백엔드팀이 비용 정책상 이미 PostHog Cloud로 전환한 것을 확인해, 새 도구를 추가하는 대신 조직이 검증한 도구를 재사용하기로 했습니다. 도구를 붙이기 전에 에러가 발생할 수 있는 지점부터 선별했는데, API 요청 실패·클라이언트 렌더링 에러·서버 렌더링 크래시가 서로 다른 레이어에서 처리된다는 걸 확인했습니다. axios 인터셉터에서 모든 에러를 잡으면 react-query의 재시도 로직과 겹쳐 같은 에러가 중복 리포트되는 것도 발견해, react-query 전역 에러 핸들러 한 곳에서만 캡처하도록 구조를 좁혔습니다.

#### 결과
`QueryCache`/`MutationCache` 전역 핸들러, 라우트 세그먼트 에러 바운더리(`error.tsx`), 루트 크래시 폴백(`global-error.tsx`), 서버 인스트루먼테이션(`instrumentation.ts`)까지 4개 지점에 캡처 로직을 배치했습니다. 클라이언트 렌더링 에러는 `error.tsx`와 `global-error.tsx` 두 메커니즘으로 나뉘는데, 전자는 라우트 세그먼트 단위 폴백이고 후자는 루트 레이아웃 자체가 깨져 `<html>`/`<body>`를 다시 그려야 하는 경우에만 발동하는 별도 메커니즘이라 구분해서 캡처했습니다. 프로덕션 배포 환경에 적용해 4개 지점 전부 PostHog 대시보드에 정상 도달함을 테스트 이벤트로 확인했습니다. 
<img width="645" height="540" alt="image" src="https://github.com/user-attachments/assets/c986eb5d-0484-461e-a2e3-b0ed309f7dea" />
<img width="1461" height="830" alt="posthog 에러 캡쳐본" src="https://github.com/user-attachments/assets/7150e055-5d4c-4427-bc0b-dbfdbe7ba390" />

아직 출시 전이라 실사용자 트래픽 기준 검증은 아닙니다.

정량적으로 측정하진 않았지만, 요청마다 에러 여부를 확인하고 필요 시 PostHog로 전송하는 로직이 추가되는 구조라 이론적으로는 약간의 런타임 오버헤드가 있습니다. 다만 실패 경로에서만 동작하고 성공 경로에는 개입하지 않는 구조라 정상 트래픽에 미치는 영향은 제한적이라고 판단했습니다.



> 레이어별 캡처 전략의 상세 근거, 정상 미인증과 진짜 장애를 구분한 방식, 인스트루먼테이션 훅 코드는 [노션 문서 - 기술 상세](https://ahead-bug-3de.notion.site/PostHog-39eb438e5ecb81718633ed40e262c90f?source=copy_link)에 정리했습니다.

### 4. 로드맵 생성 폼 필드 검증 및 에러 복구 플로우 설계

#### 왜 문제였는가
로드맵 생성이 실패하면 사용자에게 보여주는 건 에러 토스트 하나가 전부였고, 어떤 값이 잘못됐는지, 어디로 가서 무엇을 다시 선택해야 하는지는 알려주지 않았습니다. 사용자 입장에서는 정성껏 입력한 값이 몇 단계 뒤에서 알 수 없는 이유로 튕겨나가는 경험이라, 이 상태로 두면 이탈로 이어질 가능성이 높다고 판단했습니다. 백엔드도 필드별 에러 코드를 아직 정리해두지 않은 상태였고, Swagger로 스펙을 다시 확인해보니 일부 필드가 optional로 처리돼 있어 값이 비어 있어도 API가 200으로 응답한다는 사각지대까지 발견했습니다.

#### 어떻게 결정했는가
백엔드 코드가 확정되기 전이라 프론트엔드가 먼저 필드별 코드를 자체적으로 부여하고, 나중에 매핑 테이블 키만 바꾸면 되도록 설계했습니다. 처음에는 각 페이지 "다음" 버튼마다 전체 필드를 재검증하는 가드를 뒀는데, 정상 플로우에서도 이 가드가 오작동할 수 있다는 걸 발견해 제거하고 페이지 버튼의 활성화 조건으로 역할을 옮겼습니다. 구현은 AI 페어 프로그래밍으로 속도를 냈지만 설계와 검증은 직접 했는데, 코드를 리뷰하는 과정에서 `detectMissingFieldCode`가 Zustand 상태를 검사하고 실제 API 전송은 React Hook Form의 data를 쓰는 걸 발견해, 두 소스가 어긋나면 검증과 실제 전송 값이 불일치할 수 있다고 보고 API에 실제 전송되는 data를 직접 검사하도록 통일했습니다.

#### 결과
에러 발생 시 문제 필드로 자동 이동 + 해당 필드만 선택적으로 초기화하는 복구 플로우를 구현했고, 백엔드가 아직 처리하지 못하는 케이스까지 프론트엔드에서 방어했습니다. 6개 필드×4가지 무효값 매트릭스 24개에 사각지대 재현 테스트 3건, baseline 1건을 더해 테스트 28개로 검증했고, `detectMissingFieldCode` 기준 100% 커버리지를 확보했습니다 (사각지대 3건 중 targetPositionId 음수·industry 공백 문자열 2건은 현재 가드가 못 잡아 향후 과제로 남겨뒀고, targetCompanyId 음수는 detectMissingFieldCode는 못 잡지만 별도의 Zod 스키마 검증(roadmapSchema.superRefine)이 방어하고 있음을 재현 테스트로 확인했습니다)

<img width="906" height="716" alt="image" src="https://github.com/user-attachments/assets/702e4f7a-0333-49f5-b070-d9843b1f069e" />


> 통합 에러 라우팅 맵, 사전 검증·사후 처리 코드, `returnTo` 복귀 메커니즘은 [노션 문서 - 기술 상세](https://ahead-bug-3de.notion.site/39db438e5ecb8181bf05f5eadd3a38ef?source=copy_link)에 정리했습니다.



## 📸 화면

| 퀴즈 & 역량 분석 | 로드맵 |
|:---:|:---:|
| **퀴즈 화면** | **로드맵 - 추천 콘텐츠** |
| <img width="259" height="743" alt="image" src="https://github.com/user-attachments/assets/816e08b2-fdc1-4e43-b2c1-1d19340e892a" /> | <img width="258" height="742" alt="image" src="https://github.com/user-attachments/assets/818d5312-7509-4d5b-a134-d4a3590cff47" /> |
| **역량 분석 결과** | **로드맵 - 일정 관리** |
| <img width="259" height="743" alt="image" src="https://github.com/user-attachments/assets/88a700e5-3bf7-4a61-b1b7-fc01b1b3b149" /> | <img width="259" height="740" alt="image" src="https://github.com/user-attachments/assets/8868148e-1bd4-4acf-8e34-a5437060c438" /> |

