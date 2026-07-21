# ConnectingRoad — Web

**배포**: [dev-web.connectforme.com](https://dev-web.connectforme.com) | **개발 기간**: 2025.05 ~ 진행 중
> 커리어 목표 및 퀴즈 기반 역량 분석으로 맞춤 로드맵을 추천하는 서비스
> 본 레포지토리는 비공개 팀 프로젝트(법인화 진행 중)의 웹 담당 기여를 코드 없이 기술적 의사결정 중심으로 정리한 문서 레포입니다.



## 프로젝트 소개

ConnectingRoad는 퀴즈 기반 역량 분석으로 사용자의 부족 역량을 진단하고, 맞춤 커리어 로드맵을 추천하는 서비스입니다.

- **팀 구성**: 6인 사이드 팀 프로젝트 (디자인 1, 백엔드 2, 프론트엔드 웹 1, 백엔드 1, 테크리더 1)
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

모노레포 구조(테크리더 적용) 환경에서 web 패키지를 담당했습니다.
모노레포 전환 이후 프로젝트 상황에 맞게 공용 `package.json`의 web 관련 명령어를 수정했습니다.



## 🔍 기술적 의사결정

아래 6개는 이력서에 압축된 의사결정을 이 프로젝트 안에서 서로 어떻게 이어졌는지 볼 수 있도록 정리한 것입니다. 이 중 4개(FSD·배럴 최적화·PostHog·로드맵 가드)는 코드·버그 레벨의 상세 근거를 노션 문서에 별도로 정리해뒀고, 여기서는 프로젝트 흐름 안에서의 판단만 짚습니다. 나머지 2개(RHF+Zod·퀴즈 신뢰도)는 다른 기록이 없어 여기서 전체 과정을 다룹니다.

### 1. FSD 아키텍처 전환 제안·주도

#### 왜 문제였는가

React에서 Next.js로 재구축하는 시점이었고, 기존 코드는 components/, hooks/, utils/처럼 역할별로 폴더를 나누는 구조였습니다. 기능이 늘어날수록 기능 하나를 수정하려면 여러 폴더를 동시에 열어야 했고, "이 기능이 어디까지 영향을 주는지"를 한눈에 파악할 수 없었습니다.

#### 어떻게 결정했는가

점진적 개선과 Atomic Design도 함께 검토했지만, 둘 다 탐색 비용 문제의 근본 원인(관심사가 역할로 흩어지는 것)을 해결하지 못한다고 판단했습니다. FSD는 기능 단위로 코드를 응집시켜 이 문제를 구조 차원에서 없앨 수 있다고 봤습니다. 다른 팀원의 "점진적 개선이 낫지 않냐"는 반대 의견에는, 현재 구조에서 겪고 있는 구체적 문제를 먼저 정리하고 FSD가 그 문제를 어떻게 해결하는지 하나씩 매핑해서 제시했습니다.

#### 결과, 그리고 다음 문제로 이어진 지점

기능 단위 응집 구조로 전환해 탐색 비용을 낮췄습니다. 다만 FSD를 적용하면서 슬라이스마다 진입점(Public API, `index.ts`)을 두는 관례도 함께 들어왔는데, 이 관례가 바로 다음 항목의 번들링 문제로 이어졌습니다.

> FSD 레이어 구조, thin shell 패턴, 실제 리팩토링 중 발견한 버그(Rules of Hooks 위반, `ensureQueryData` 캐시 null 이슈)는 [노션 문서 - 기술 상세](https://ahead-bug-3de.notion.site/FSD-39db438e5ecb8110837ddcd2c37fea7a?source=copy_link)에 정리했습니다.

### 2. 배럴(barrel) export의 부작용 코드가 Tree Shaking을 막던 문제 역추적·제거

#### 왜 문제였는가

로드맵 섹션은 산업군 선택부터 결과 분석까지 10개 넘는 화면으로 구성돼 있습니다. 화면별 초기 로딩 용량을 점검하다가, 결과 분석 화면에만 필요한 레이더 차트가 있는데도 그래프가 전혀 없는 다른 화면들의 초기 로딩 용량이 거의 똑같이 크다는 걸 발견했습니다.

#### 어떻게 결정했는가

여러 화면이 공통으로 참조하는 배럴(index.ts)이 그래프 컴포넌트를 최상단에서 export하고 있었고, 그 안의 차트 라이브러리가 모듈 로드 즉시 초기화 코드를 실행하는 부작용 때문에 번들러가 트리셰이킹을 안전하게 수행하지 못하고 있었습니다. 이 배럴을 전혀 참조하지 않는 순수 정적 페이지가 수정 전에도 공유 베이스라인(103KB)에 머물러 있었다는 걸 대조군으로 확인해 원인을 교차 검증한 뒤, 배럴에서 그래프 컴포넌트 export를 제거하고 실제로 필요한 화면 하나에서만 동적 import로 분리했습니다.

#### 결과

로드맵 섹션 전체 화면의 초기 로딩 용량이 실측 기준 평균 15% 줄었습니다. 그래프가 필요한 화면조차 "필요할 때만" 코드를 불러오는 방식으로 바뀌며 초기 로딩이 더 가벼워졌습니다.

> 화면별 수정 전/후 실측 표, 대조군 교차 검증 근거, 배럴 export 코드는 [노션 문서 - 기술 상세](https://ahead-bug-3de.notion.site/3a2b438e5ecb816783aae92bf2ce1bfa?source=copy_link)에 정리했습니다.

### 3. PostHog 에러 모니터링 4계층 캡처 구조 설계

#### 왜 문제였는가

에러가 발생해도 Slack으로 공유받는 네트워크 탭 스크린샷이 유일한 추적 수단이었습니다. 어떤 API가 어떤 상태 코드로 실패했는지 스크린샷만으로 특정하기 어려웠고, 프론트엔드 런타임 에러는 재현조차 힘들었습니다. 무엇보다 실제 사용자가 마주친 에러는 데이터를 수집할 방법 자체가 없었습니다.

#### 어떻게 결정했는가

Sentry 도입을 검토하다가 백엔드팀이 비용 정책상 이미 PostHog Cloud로 전환한 것을 확인해, 새 도구를 추가하는 대신 조직이 검증한 도구를 재사용하기로 했습니다. 도구를 붙이기 전에 에러가 발생할 수 있는 지점부터 선별했는데, API 요청 실패·클라이언트 렌더링 에러·서버 렌더링 크래시가 서로 다른 레이어에서 처리된다는 걸 확인했습니다. axios 인터셉터에서 모든 에러를 잡으면 react-query의 재시도 로직과 겹쳐 같은 에러가 중복 리포트되는 것도 발견해, react-query 전역 에러 핸들러 한 곳에서만 캡처하도록 구조를 좁혔습니다.

#### 결과

react-query 핸들러·클라이언트 에러 바운더리·서버 인스트루먼테이션까지 4개 레이어로 캡처 구조를 구현했고, 프로덕션 배포 환경에 4개 레이어를 적용해 테스트 이벤트가 PostHog까지 정상 도달하는 것을 확인했습니다. 아직 출시 전이라 실사용자 트래픽 기준 검증은 아닙니다.

> 레이어별 캡처 전략의 상세 근거, 정상 미인증과 진짜 장애를 구분한 방식, 인스트루먼테이션 훅 코드는 [노션 문서 - 기술 상세](https://ahead-bug-3de.notion.site/PostHog-39eb438e5ecb81718633ed40e262c90f?source=copy_link)에 정리했습니다.

### 4. 로드맵 생성 폼 필드 검증 및 에러 복구 플로우 설계

#### 왜 문제였는가

로드맵 생성이 실패하면 사용자에게 보여주는 건 에러 토스트 하나가 전부였고, 어떤 값이 잘못됐는지, 어디로 가서 무엇을 다시 선택해야 하는지는 알려주지 않았습니다. 백엔드도 필드별 에러 코드를 아직 정리해두지 않은 상태였고, Swagger로 스펙을 다시 확인해보니 일부 필드가 optional로 처리돼 있어 값이 비어 있어도 API가 200으로 응답한다는 사각지대까지 발견했습니다.

#### 어떻게 결정했는가

백엔드 코드가 확정되기 전이라 프론트엔드가 먼저 필드별 코드를 자체적으로 부여하고, 나중에 매핑 테이블 키만 바꾸면 되도록 설계했습니다. 처음에는 각 페이지 "다음" 버튼마다 전체 필드를 재검증하는 가드를 뒀는데, 정상 플로우에서도 이 가드가 오작동할 수 있다는 걸 발견해 제거하고 페이지 버튼의 활성화 조건으로 역할을 옮겼습니다. API 호출 전 자체 검증(사각지대 방어)과 호출 후 에러 처리를 역할별로 분리하고, 에러로 이동한 페이지에서 값을 고친 뒤 원래 페이지로 바로 복귀하는 파라미터를 추가했습니다.

#### 결과

에러 발생 시 문제 필드로 자동 이동 + 해당 필드만 선택적으로 초기화하는 복구 플로우를 구현했고, 백엔드가 아직 처리하지 못하는 케이스까지 프론트엔드에서 방어했습니다.

> 통합 에러 라우팅 맵, 사전 검증·사후 처리 코드, `returnTo` 복귀 메커니즘은 [노션 문서 - 기술 상세](https://ahead-bug-3de.notion.site/39db438e5ecb8181bf05f5eadd3a38ef?source=copy_link)에 정리했습니다.



## 📸 화면

| 화면 |
|:---:|
| **퀴즈 화면** |
| <img width="330" alt="퀴즈 화면" src="https://github.com/user-attachments/assets/10d1f7ec-7b0c-4c9f-8470-1c6301e3c868" /> |
| **역량 분석 결과** |
| <img width="330" alt="역량 분석 결과 1" src="https://github.com/user-attachments/assets/9f3a2c33-d84d-4d41-84b6-bed8c0313611" /> <img width="330" alt="역량 분석 결과 2" src="https://github.com/user-attachments/assets/b175892b-589b-4c9f-8865-582bac422c30" /> |
| **로드맵 - 추천 콘텐츠** |
| <img width="330" alt="로드맵 - 추천 콘텐츠 1" src="https://github.com/user-attachments/assets/1f9115d6-97e6-4290-ad60-1fe2129d742c" /> <img width="330" alt="로드맵 - 추천 콘텐츠 2" src="https://github.com/user-attachments/assets/de37a554-176b-4fe3-940a-f0dd4408654f" /> |
| **로드맵 - 일정 관리** |
| <img width="330" alt="로드맵 - 일정 관리 1" src="https://github.com/user-attachments/assets/971a9221-2e5b-4df8-9411-b0aed406dbab" /> <img width="330" alt="로드맵 - 일정 관리 2" src="https://github.com/user-attachments/assets/6d912651-967a-4711-81fc-5ac945072e24" /> |

