# Zero Install이 뭐임?
yarn install 없이 git clone만 해도 바로 실행 가능한 상태를 만드는 것이다.

```bash
git clone https://github.com/team/project
cd project
yarn dev  # yarn install 없이 바로 실행
```

.yarn/cache에 있는 zip 파일들을 git에 같이 올려서 이런게 가능함

# 그럼 걍 pnp 안쓰고 node_modules를 올려도 되는거 아님?
- 용량 문제: 기존 node_modules는 용량이 너무 커서 이걸 git에 다 올리기에는 커밋 속도도 엄청 느려지고 저장소 용량이 폭발한다.
	pnp를 쓰면 패키지당 zip 파일 하나여서 용량이 압도적으로 적음.
- OS 문제: 또 일부 패키지는 설치할 때 OS에 맞는 바이너리를 컴파일해서 맥에서 올린 node_modules를 리눅스 CI에서 쓰면 동작 안 할 수 있다.
	zip은 그냥 패키지 파일을 압축한 거라 OS랑 상관없음.

# Zero Install 설정 방법
걍 .gitignore에서 .yarn/cache를 git에 올리도록 설정하면 끝이다.

```
.yarn/*
!.yarn/cache      # cache는 올림
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions
.pnp.*            # 자동 생성되니까 올려도 되고 안 올려도 됨
```

# unplugged 패키지

esbuild 같은 패키지는 설치할 때 OS에 맞는 바이너리를 컴파일한다. 그래서 zip으로 보관할 수 없어서 .yarn/unplugged/ 폴더에 풀려서 설치된다.

근데 이 바이너리 파일들은
- OS마다 달라서 git에 올리면 안 됨
- 용량이 커서 저장소에 넣기엔 살짝 애매함

그래서 `.gitignore`에 이렇게 설정해:

```
.yarn/*
!.yarn/cache
.yarn/unplugged   # unplugged는 git에 안 올림
```

결과적으로 `git clone` 받고 바로 실행하면 esbuild 같은 게 없어서 오류가 나. 이 경우엔 `yarn install`을 한 번 해줘야 해. `yarn install`이 `.yarn/cache`는 이미 있으니까 다운로드는 스킵하고 unplugged 폴더만 채워줘.

bash

```bash
git clone https://github.com/team/project
cd project
yarn install  # unplugged 패키지만 설치 (다운로드 없이 빠름)
yarn dev      # 실행
```

완전한 Zero Install은 아니지만, 네트워크 없이도 `yarn install`이 끝난다는 점에서 의미가 있어.

---

# 실제로 얼마나 빨라지냐

CI랑 배포 환경 모두에서 효과가 있어.

CI (GitHub Actions 같은 곳)

```
Zero Install 없음:
git clone     → 몇 초
yarn install  → 1~3분 (패키지 다운로드)
빌드/테스트 시작

Zero Install 있음:
git clone     → 수십 초 (zip 포함이라 좀 더 걸림)
yarn install  → 거의 0초 (unplugged만 있으면 몇 초)
빌드/테스트 시작
```

배포

배포 서버도 CI랑 똑같이 매번 새로 시작해. 코드 받고 → 패키지 설치하고 → 빌드하는 흐름인데, Zero Install이 있으면 설치 단계가 없어지거나 극도로 짧아져서 배포 시간이 줄어들어.

push할 때마다 반복되는 환경일수록 효과가 커.

---

# 단점은 없어?

저장소 용량이 커져

`.yarn/cache`에 zip 파일들이 다 들어가니까 저장소 자체가 무거워져. 패키지 많은 프로젝트면 수백 MB가 될 수 있어.

패키지 추가/업데이트할 때 커밋이 커져

`yarn add some-package` 하면 zip 파일이 추가되니까 그 커밋에 바이너리 파일이 포함돼. diff가 의미없어지고 PR이 지저분해 보일 수 있어.

팀 합의가 필요해

Zero Install 쓰려면 팀 전체가 `.yarn/cache`를 git에 올리기로 동의해야 해. 중간에 누가 `.gitignore`에 추가해버리면 꼬여.

---

# Zero Install 안 써도 PnP는 쓸 수 있어

Zero Install이 PnP의 필수 조건은 아니야.

```
PnP만 쓰는 경우:
→ node_modules 없애고 zip으로 관리
→ .yarn/cache는 git에 안 올림
→ yarn install은 여전히 필요

PnP + Zero Install:
→ node_modules 없애고 zip으로 관리
→ .yarn/cache도 git에 올림
→ yarn install 스킵 가능 (unplugged 있으면 한 번은 필요)
```

팀 상황에 따라 PnP만 쓰고 Zero Install은 안 쓰는 경우도 많아.