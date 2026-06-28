## Yarn Berry가 뭐임?
yarn v2부터를 부르는 이름이다. yarn v1(classic)이랑 완전히 다른 코드베이스로 재작성됐어서 사실상 별개의 도구라고 할수있다.

```
yarn v1       → yarn classic  (레거시, 유지보수만 됨)
yarn v2, v3, v4 → yarn berry
```

"berry"는 v2 개발할 때 yarn 팀이 붙인 공식 코드명인데 그대로 굳어졌다. 내부 구조, 설정 파일(`.yarnrc` → `.yarnrc.yml`), 플러그인 시스템 전부 새로 만든 거라 yarn classic에서 berry로 넘어가는 건 버전 업그레이드가 아니라 도구 교체 수준이다.

## Yarn Berry가 나온 이유
yarn v1(classic)은 npm이랑 구조가 똑같다. flat + hoisting이라서 유령 의존성 문제도 그대로고, node_modules가 무겁다는 문제도 그대로다.

yarn 팀은 "구지 node_modules가 있어야 함?" 이라는 근본적인 질문을 던졌다.

node_modules가 있으면 생기는 문제들:
- 패키지 수백 개면 파일이 수십만 개 → 설치 느림
- 용량이 수 GB → 디스크 낭비
- hoisting 때문에 유령 의존성 발생
- node_modules폴더 자체를 git에 올릴 수 없음 → CI마다 매번 설치

그래서 yarn v2(berry)에서 node_modules를 아예 없애는 방식을 도입했는데 이게 PnP(Plug'n'Play)다.

---

## PnP가 동작하는 방식
### 기존 방식
```
npm install
→ node_modules에 파일 수십만 개 복사
→ Node.js가 위로 올라가며 파일 탐색
```

### PnP 방식
```
yarn install
→ node_modules 안 만듦
→ .yarn/cache에 패키지를 zip 파일로 저장
→ .pnp.cjs 파일 생성
```

.pnp.cjs파일에 모든 패키지의 위치가 다 적혀있다

```js
// .pnp.cjs (단순화한 예시)
{
  "react": {
    "version": "18.2.0",
    "location": ".yarn/cache/react-npm-18.2.0.zip"
  },
  "loose-envify": {
    "version": "1.4.0",
    "location": ".yarn/cache/loose-envify-npm-1.4.0.zip"
  }
}
```

`import react` 하면 Node.js가 위로 올라가며 탐색하는 게 아니라, .pnp.cjs를 보고 바로 위치를 찾아간다. 파일 탐색이 없어지니까 훨씬 빠르다.

---

## zip 파일을 어떻게 실행함?

.yarn/cache에 있는 건 압축 파일이다. 근데 Node.js는 원래 zip을 직접 읽지 못한다.
그래서 yarn berry는 Node.js의 require()나 import가 실행되기 전에 끼어들어서 .pnp.cjs를 먼저 확인하고, .yarn/cache에 있는 파일들을 읽어서 Node,js에 넘겨줌. 
개발자 입장에선 그냥 평소처럼 import 하면 됨.

---

## 유령 의존성이 원천 차단되는 이유
기존엔 Node.js가 node_modules를 위로 올라가며 탐색하니까 호이스팅된 패키지에 접근이 됐다.
하지만 PnP는 .pnp.cjs에 내가 쓸 수 있는 패키지 목록이 명시돼있어서 목록에 없는 패키지를 `import` 하면
```
import looseEnvify from 'loose-envify'
// → Error: loose-envify isn't declared in your dependencies
```

파일이 .yarn/cache에 있어도 .pnp.cjs에 없으면 접근 자체가 안됨.

---

## PnP 단점
### IDE 호환성 문제

VSCode 같은 에디터는 node_modules 폴더를 직접 읽어서 타입 정보, 자동완성을 제공하는데, PnP는 node_modules가 없어서 기본적으론 이게 안됨.

그래서 이걸 해결하려고 SDK를 제공함

```bash
yarn dlx @yarnpkg/sdks vscode
```

이 명령어 실행하면 .yarn/sdks 폴더가 생기고, VSCode가 zip 파일에서 타입 정보를 읽을 수 있게 연결해줌.

### 패키지 호환성 문제
일부 패키지는 내부적으로 node_modules 경로를 하드코딩해놔서 PnP 환경에서 동작을 안함. 이 경우엔 해당 패키지만 node_modules에 설치하도록 예외 처리할 수 있음.

---

## PnP 도입 전후 비교

| 🥑      | 기존 (node_modules) | PnP                |
| ------- | ----------------- | ------------------ |
| 설치 결과물  | 파일 수십만 개          | zip 파일 몇 백 개       |
| 패키지 탐색  | 위로 올라가며 탐색        | .pnp.cjs 보고 바로 접근  |
| 유령 의존성  | 발생 가능             | 발생 안함              |
| IDE 지원  | 기본 지원             | SDK 설치 필요          |
| 패키지 호환성 | 대부분 됨             | 일부 패키지 문제 있음       |
| git 추적  | 불가                | 가능 -> Zero Install |
|         |                   |                    |
