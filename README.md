# ConnectingRoad — Web

**배포**: [dev-web.connectforme.com](https://dev-web.connectforme.com) | **개발 기간**: 2025.05 ~ 진행 중
> 커리어 목표 및 퀴즈 기반 역량 분석으로 맞춤 로드맵을 추천하는 서비스
> 본 레포지토리는 비공개 팀 프로젝트(법인화 진행 중)의 웹 담당 기여를 코드 없이 기술적 의사결정 중심으로 정리한 문서 레포입니다.



## 프로젝트 소개

ConnectingRoad는 퀴즈 기반 역량 분석으로 사용자의 부족 역량을 진단하고, 맞춤 커리어 로드맵을 추천하는 서비스입니다.

- **팀 구성**: 6인 사이드 팀 프로젝트 (디자인 1, 백엔드 2, 프론트엔드 웹 1, 앱 1, 테크리더 1)
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

아래 6개는 이력서에 압축된 의사결정을 이 프로젝트 안에서 서로 어떻게 이어졌는지 볼 수 있도록 정리한 것입니다. 이 중 3개(FSD·PostHog·로드맵 가드)는 코드·버그 레벨의 상세 근거를 노션 문서에 별도로 정리해뒀고, 여기서는 프로젝트 흐름 안에서의 판단만 짚습니다. 나머지 3개(Tree Shaking·RHF+Zod·퀴즈 신뢰도)는 다른 기록이 없어 여기서 전체 과정을 다룹니다.

### 1. FSD 아키텍처 전환 제안·주도

#### 왜 문제였는가

React에서 Next.js로 재구축하는 시점이었고, 기존 코드는 components/, hooks/, utils/처럼 역할별로 폴더를 나누는 구조였습니다. 기능이 늘어날수록 기능 하나를 수정하려면 여러 폴더를 동시에 열어야 했고, "이 기능이 어디까지 영향을 주는지"를 한눈에 파악할 수 없었습니다.

#### 어떻게 결정했는가

점진적 개선과 Atomic Design도 함께 검토했지만, 둘 다 탐색 비용 문제의 근본 원인(관심사가 역할로 흩어지는 것)을 해결하지 못한다고 판단했습니다. FSD는 기능 단위로 코드를 응집시켜 이 문제를 구조 차원에서 없앨 수 있다고 봤습니다. 다른 팀원의 "점진적 개선이 낫지 않냐"는 반대 의견에는, 현재 구조에서 겪고 있는 구체적 문제를 먼저 정리하고 FSD가 그 문제를 각각 어떻게 해결하는지 하나씩 매핑해서 제시했습니다.

#### 결과, 그리고 다음 문제로 이어진 지점

기능 단위 응집 구조로 전환해 탐색 비용을 낮췄습니다. 다만 FSD를 적용하면서 슬라이스마다 진입점(Public API, `index.ts`)을 두는 관례도 함께 들어왔는데, 이 관례가 바로 다음 항목의 번들링 문제로 이어졌습니다.

> FSD 레이어 구조, thin shell 패턴, 실제 리팩토링 중 발견한 버그(Rules of Hooks 위반, `ensureQueryData` 캐시 null 이슈)는 [노션 문서](https://ahead-bug-3de.notion.site/FSD-39db438e5ecb8110837ddcd2c37fea7a?source=copy_link)에 정리했습니다.

### 2. 중첩 Public API(배럴) 구조가 Tree Shaking을 방해하던 문제 진단·제거

FSD 도입으로 슬라이스마다 배럴을 두는 게 관례가 됐는데, 재구축 후 번들 분석 도구로 실제 결과물을 검증하다가 이 관례가 만든 부작용을 발견했습니다. 예상보다 많은 모듈이 번들에 포함돼 있어서 "왜 이게 들어와 있지?"를 추적하기 시작했습니다.

#### 원인 추적 과정

구조를 따라가 보니 문제는 배럴이 두 겹으로 중첩된 구조에 있었습니다. 로그인 페이지에서 ButtonList를 Public API(배럴)로 호출하고, ButtonList 내부에서 다시 각 소셜 버튼들을 Public API(배럴)로 import하는 구조였습니다. 배럴은 여러 모듈을 한 곳에서 re-export하는데, 이 중첩 구조에서는 실제로 사용하는 소셜 버튼 하나만 필요해도 배럴이 묶고 있는 모든 컴포넌트가 통째로 번들에 끌려 들어왔습니다. Tree Shaking이 배럴의 중첩 구조를 뚫고 불필요한 모듈을 쳐내지 못했던 겁니다.

```tsx
// Before — 배럴이 두 겹으로 중첩된 구조
// login/page.tsx
import { ButtonList } from '@/features/auth/ui'; // 1차 배럴 호출

// ButtonList/index.tsx 내부
import { KakaoButton, GoogleButton, NaverButton } from '@/features/auth/ui'; // 2차 배럴 호출
// → 실제로 KakaoButton만 쓰더라도 Google, Naver까지 번들에 포함
```

#### 왜 이렇게 해결했는가

문제의 본질은 ButtonList라는 래핑 컴포넌트가 중첩 배럴 구조를 만들어내고 있다는 점이었습니다. 래핑 레이어를 유지하면서 배럴만 걷어내는 방법도 있었지만, ButtonList 자체가 독립적인 역할 없이 소셜 버튼들을 감싸기만 하는 컴포넌트라면 존재 이유가 없다고 판단했습니다. 그래서 ButtonList 래핑 컴포넌트를 제거하고 페이지에 소셜 버튼들을 직접 배치하는 방향으로 구조를 평탄화했습니다.

```tsx
// After — ButtonList 제거, 페이지에 직접 배치
// login/page.tsx
import {
  AppleLoginButton,
  GoogleLoginButton,
  KakaoLoginButton,
} from "@/features/auth";
// → 필요한 컴포넌트만 번들에 포함, Tree Shaking 정상 동작
```

#### 결과

중첩 배럴 구조 제거로 Tree Shaking이 정상 동작하는 것을 번들 분석으로 재확인했습니다. "구조가 깔끔하다"와 "번들이 효율적이다"는 별개의 문제라는 걸 체득한 경험이었습니다. FSD를 도입한 것으로 끝이 아니라, 도입한 관례 자체를 계속 검증해야 한다는 것도 함께 배웠습니다.

### 3. PostHog 에러 모니터링 4계층 캡처 구조 설계

#### 왜 문제였는가

에러가 발생해도 Slack으로 공유받는 네트워크 탭 스크린샷이 유일한 추적 수단이었습니다. 어떤 API가 어떤 상태 코드로 실패했는지 스크린샷만으로 특정하기 어려웠고, 프론트엔드 런타임 에러는 재현조차 힘들었습니다. 무엇보다 실제 사용자가 마주친 에러는 데이터를 수집할 방법 자체가 없었습니다.

#### 어떻게 결정했는가

Sentry 도입을 검토하다가 백엔드팀이 비용 정책상 이미 PostHog Cloud로 전환한 것을 확인해, 새 도구를 추가하는 대신 조직이 검증한 도구를 재사용하기로 했습니다. 도구를 붙이기 전에 에러가 발생할 수 있는 지점부터 선별했는데, API 요청 실패·클라이언트 렌더링 에러·서버 렌더링 크래시가 서로 다른 레이어에서 처리된다는 걸 확인했습니다. axios 인터셉터에서 모든 에러를 잡으면 react-query의 재시도 로직과 겹쳐 같은 에러가 중복 리포트되는 것도 발견해, react-query 전역 에러 핸들러 한 곳에서만 캡처하도록 구조를 좁혔습니다.

#### 결과

react-query 핸들러·클라이언트 에러 바운더리·서버 인스트루먼테이션까지 4개 레이어로 캡처 구조를 구현했고, 실제 서비스 환경에서 이벤트가 PostHog까지 정상 도달하는 것을 직접 검증했습니다.

> 레이어별 캡처 전략의 상세 근거, 정상 미인증과 진짜 장애를 구분한 방식, 인스트루먼테이션 훅 코드는 [노션 문서](https://ahead-bug-3de.notion.site/PostHog-39eb438e5ecb81718633ed40e262c90f?source=copy_link)에 정리했습니다.

### 4. 로드맵 생성 폼 필드 검증 및 에러 복구 플로우 설계

#### 왜 문제였는가

로드맵 생성이 실패하면 사용자에게 보여주는 건 에러 토스트 하나가 전부였고, 어떤 값이 잘못됐는지, 어디로 가서 무엇을 다시 선택해야 하는지는 알려주지 않았습니다. 백엔드도 필드별 에러 코드를 아직 정리해두지 않은 상태였고, Swagger로 스펙을 다시 확인해보니 일부 필드가 optional로 처리돼 있어 값이 비어 있어도 API가 200으로 응답한다는 사각지대까지 발견했습니다.

#### 어떻게 결정했는가

백엔드 코드가 확정되기 전이라 프론트엔드가 먼저 필드별 코드를 자체적으로 부여하고, 나중에 매핑 테이블 키만 바꾸면 되도록 설계했습니다. 처음에는 각 페이지 "다음" 버튼마다 전체 필드를 재검증하는 가드를 뒀는데, 정상 플로우에서도 이 가드가 오작동할 수 있다는 걸 발견해 제거하고 페이지 버튼의 활성화 조건으로 역할을 옮겼습니다. API 호출 전 자체 검증(사각지대 방어)과 호출 후 에러 처리를 역할별로 분리하고, 에러로 이동한 페이지에서 값을 고친 뒤 원래 페이지로 바로 복귀하는 파라미터를 추가했습니다.

#### 결과

에러 발생 시 문제 필드로 자동 이동 + 해당 필드만 선택적으로 초기화하는 복구 플로우를 구현했고, 백엔드가 아직 처리하지 못하는 케이스까지 프론트엔드에서 방어했습니다.

> 통합 에러 라우팅 맵, 사전 검증·사후 처리 코드, `returnTo` 복귀 메커니즘은 [노션 문서](https://ahead-bug-3de.notion.site/39db438e5ecb8181bf05f5eadd3a38ef?source=copy_link)에 정리했습니다.

### 5. React Hook Form + Zod로 폼 유효성 로직 중앙화

#### 무엇이 문제였는가

회원가입·프로필 입력 등 폼이 늘어나면서 두 가지 문제가 동시에 보였습니다. 첫째, 유효성 검사 로직이 컴포넌트마다 흩어져 있어서 같은 규칙(이메일 형식, 비밀번호 복잡도 등)이 중복 작성되고 일관성이 없었습니다. 둘째, 제어 컴포넌트 방식으로 구현하다 보니 한 글자 입력할 때마다 useState가 갱신되면서 폼 전체가 리렌더링되고 있었습니다.

#### 왜 React Hook Form이었는가

제어 방식의 리렌더링 문제를 풀려면 입력값을 매 키 입력마다 state로 들고 있지 않아야 했습니다. React Hook Form은 ref 기반 비제어 방식이라 입력 중에는 리렌더링이 발생하지 않고, 제출·blur 시점에만 값을 읽습니다. 이 동작 방식 자체가 제가 가진 리렌더링 문제의 정확한 해법이었습니다.

#### 왜 Zod를 함께 썼는가

검증 로직 분산 문제는 "규칙을 한 곳에서 선언적으로 관리"해야 풀립니다. Zod는 스키마로 유효성 규칙을 한 번 정의하면 그걸 폼 검증에 그대로 쓸 수 있고, 무엇보다 스키마에서 TypeScript 타입이 자동 생성됩니다. 즉 "검증 규칙"과 "타입"을 따로 관리하다 어긋나는 일이 없어집니다. 검증 중앙화와 타입 안전성을 한 번에 얻을 수 있어서 React Hook Form의 resolver로 Zod를 연결했습니다.

```tsx
export const profileSchema = z.object({
  name: z
    .string()
    .min(1, "이름을 입력해 주세요.")
    .min(2, "이름은 최소 2자 이상 입력해 주세요.")
    .max(20, "이름은 최대 20자까지 입력할 수 있습니다."),
  phone: z
    .string()
    .refine(
      (val) => {
        if (!val || val === "") return true; // 빈 값은 통과
        if (val.length > 30) return false;
        return /^[0-9-]+$/.test(val);
      },
      (val) => {
        if (val.length > 30) {
          return {
            message: "휴대전화 번호는 최대 30자까지 입력할 수 있습니다.",
          };
        }
        return { message: "숫자만 입력 가능합니다." };
      }
    )
    .optional()
    .or(z.literal("")),

  email: z
    .string()
    .refine(
      (val) => {
        if (!val || val === "") return true; // 빈 값은 통과
        if (val.length > 100) return false;

        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!emailRegex.test(val)) return false;

        const domain = val.split("@")[1];
        return ALLOWED_EMAIL_DOMAINS.includes(domain);
      },
      (val) => {
        if (val.length > 100) {
          return {
            message: "이메일은 최대 100자까지 입력할 수 있습니다.",
          };
        }
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!emailRegex.test(val)) {
          return { message: "올바른 이메일 주소를 입력해주세요." };
        }
        return {
          message:
            "허용된 이메일 도메인이 아닙니다 (naver.com, gmail.com 등만 가능)",
        };
      }
    )
    .optional()
    .or(z.literal("")),

  gender: z.string().optional().or(z.literal("")),

  roadAddress: z.string().optional().or(z.literal("")),
  detailAddress: z
    .string()
    .refine(
      (val) => {
        if (!val || val === "") return true;
        return val.length <= 100;
      },
      {
        message: "상세 주소는 최대 100자까지 입력할 수 있습니다.",
      }
    )
    .optional()
    .or(z.literal("")),
  zipCode: z.string().optional().or(z.literal("")),

  birthDate: z.string().optional().or(z.literal("")),
  imagePath: z.union([z.custom<File>(), z.string()]).optional(),
});
```

#### 결과

입력 중 불필요한 리렌더링을 제거했고, 검증 규칙을 스키마로 중앙화하면서 타입 안전성까지 확보했습니다.

### 6. 퀴즈 역량 분석 추천 신뢰도 문제 제기

#### 무엇이 걸렸는가

퀴즈 응답으로 사용자의 부족 역량을 분석해 학습 로드맵을 추천하는 게 서비스의 핵심이었습니다. 그런데 초기 설계를 보니 퀴즈와 역량이 1:1로 매핑돼 있었습니다. 문제 하나를 틀리면 그 역량이 곧바로 "부족 역량"으로 판정되는 구조였습니다.

#### 왜 이게 위험하다고 봤는가

이 구조의 문제는 추천 서비스에서 가장 중요한 "신뢰도"를 깨뜨릴 수 있다는 점이었습니다. 사용자가 희망 직군에서 비중이 낮은 역량의 문제를 어쩌다 하나 틀렸을 때, 시스템이 그걸 곧장 "이 사람의 약점"으로 판정하고 중요도 낮은 학습 콘텐츠를 로드맵 상단에 올릴 수 있었습니다. 단일 오답이 과대 반영되는 거죠. 추천이 한 번이라도 엉뚱하면 사용자는 서비스 전체를 신뢰하지 않게 됩니다.

#### 어떻게 접근했는가

이건 제가 단독으로 구조를 바꿀 사안이 아니라, 데이터 설계와 백엔드까지 얽힌 문제였습니다. 그래서 제 판단을 결론으로 들이밀기보다, 팀 회의에서 "오답 하나가 곧바로 역량 부족으로 직결되는 게 추천 신뢰도에 위험하지 않을까"라는 질문으로 제기했습니다. 거기서 퀴즈와 역량을 1:N으로 매핑하고 각 퀴즈에 가중치를 부여해서, 여러 문제의 가중치 합으로 역량 수준을 판정하는 방향으로 팀 논의가 이어졌습니다.

#### 결과

단일 오답의 과대 반영을 제거하는 방향으로 설계가 정리됐고, 추천이 사용자의 실제 약점에 더 가깝게 정렬되도록 개선됐습니다.



## 📸 화면

<!-- 본인이 구현한 주요 화면 스크린샷 추가 -->

| 페이지 | 설명 |
|---|---|
| (스크린샷) | 퀴즈 화면 |
| (스크린샷) | 역량 분석 결과 |
| (스크린샷) | 커리어 로드맵 추천 |
