## 왜 npm 하나로 안 끝나냐

npm은 Node.js 설치하면 딸려오는 기본 패키지 매니저다. 근데 현업이나 오픈소스 프로젝트 보면 pnpm이나 yarn을 쓰는 경우가 훨씬 많다. 이유가 뭔지 궁금해서 정리했다.

---

## 세 줄 요약

|              | npm                 | yarn (classic) | pnpm                        |
| ------------ | ------------------- | -------------- | --------------------------- |
| 출시           | 2010                | 2016           | 2017                        |
| 속도           | 느림                  | 빠름             | 가장 빠름                       |
| 디스크          | 중복 많음               | 중복 많음          | 중복 없음 (content-addressable) |
| lockfile     | `package-lock.json` | `yarn.lock`    | `pnpm-lock.yaml`            |
| node_modules | flat                | flat           | 비선형 (symlink)               |

---

## npm

가장 오래됐고 생태계가 크다. 근데 초기 버전(v1~v2)은 의존성을 **트리 구조**로 그대로 설치해서 같은 패키지가 여러 depth에 중복으로 깔리는 문제가 있었다.

v3부터 **flat하게** 설치하는 방식으로 바뀌었는데, 이게 또 다른 문제를 만들었다. 내가 `package.json`에 명시하지 않은 패키지도 `node_modules` 최상위에 올라와서 `require()`로 접근할 수 있게 됐다. 이걸 **유령 의존성(phantom dependency)** 이라고 한다.

```
node_modules/
├── react/          # 내가 설치한 것
├── loose-envify/   # react가 쓰는 건데 내가 직접 import 가능해짐 ← 문제
└── ...
```

> 지금 당장은 동작하지만, react가 loose-envify를 안 쓰는 버전으로 업그레이드되면 내 코드가 터진다.

---

## yarn (classic, v1)

Facebook이 npm의 느린 속도와 비결정적 설치 문제를 해결하려고 2016년에 만들었다.

핵심 기여:

- **lockfile 도입** — `yarn.lock`으로 어디서 설치해도 동일한 버전 보장
- **병렬 설치** — npm은 순차적으로 설치했는데 yarn은 병렬로 처리
- **오프라인 캐시** — 한 번 받은 패키지는 네트워크 없어도 설치 가능

근데 node_modules 구조는 npm이랑 똑같이 flat해서 유령 의존성 문제는 그대로다.

---

## pnpm

"performant npm"의 약자. 속도도 빠르지만 핵심은 **디스크 효율**이다.

### content-addressable store

패키지를 글로벌 저장소(`~/.pnpm-store`)에 **딱 한 번만** 저장하고, 프로젝트의 `node_modules`에는 **심링크(symlink)** 로 연결한다.

```
~/.pnpm-store/
└── v3/
    └── files/
        └── ab/cd1234...  ← react@18.2.0의 실제 파일 (전 세계 딱 한 벌)

my-project/node_modules/
└── react → ~/.pnpm-store/v3/files/ab/cd1234  ← 심링크
```

react를 쓰는 프로젝트가 10개면 npm/yarn은 react를 10번 복사하지만, pnpm은 1번만 저장하고 10개 프로젝트에서 같은 파일을 가리킨다.

### 유령 의존성 차단

pnpm은 `node_modules` 구조를 flat하게 만들지 않는다. 내가 `package.json`에 명시한 패키지만 최상위에 노출되고, 그 패키지의 의존성은 격리된다.

```
node_modules/
├── react/          ← 내가 설치한 것만
└── .pnpm/
    └── loose-envify@1.4.0/  ← react 의존성은 여기 격리
```

덕분에 `loose-envify`를 직접 `import`하면 에러가 난다. 처음엔 불편하게 느껴질 수 있는데, 사실 이게 **올바른 동작**이다.

---

## 그래서 뭘 쓰면 돼?

- 빠르게 시작하는 개인 프로젝트 → **npm** (그냥 있으니까)
- 디스크 아끼고 싶고, 의존성 엄격하게 관리하고 싶으면 → **pnpm**
- 모노레포, Zero Install, PnP 같은 고급 기능 필요하면 → **yarn berry**

다음 편에서는 node_modules 내부 구조랑 hoisting이 구체적으로 어떻게 동작하는지 파본다.