# Zero Install이 뭐임?
yarn install 없이 git clone만 해도 바로 실행 가능한 상태를 만드는 것이다.

```bash
git clone https://github.com/team/project
cd project
yarn dev  # yarn install 없이 바로 실행가능
```

.yarn/cache에 있는 zip 파일들을 git에 같이 올려서 이런게 가능함

# 그럼 걍 pnp 안쓰고 node_modules를 올려도 되는거 아님?
- 용량 문제: 기존 node_modules는 용량이 너무 커서 이걸 git에 다 올리기에는 커밋 속도도 엄청 느려지고 저장소 용량이 폭발한다.
	pnp를 쓰면 패키지당 zip 파일 하나여서 용량이 압도적으로 적음
- OS 문제: 또 일부 패키지는 설치할 때 OS에 맞는 바이너리를 컴파일해서 맥에서 올린 node_modules를 리눅스 CI에서 쓰면 동작 안 할 수 있다.
	zip은 그냥 패키지 파일을 압축한 거라 OS랑 상관없음

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

그래서

```
.yarn/*
!.yarn/cache
.yarn/unplugged   # unplugged는 git에 안 올림
```

하지만 git clone 받고 바로 실행하면 esbuild 같은 게 없어서 오류가 난다. 이 경우엔 yarn install을 한 번 해줘야 한다. yarn install을 하면 .yarn/cache는 이미 git clone해서 받아왔으니까 .yarn/unplugged 폴더만 채워준다.

```bash
git clone https://github.com/team/project
cd project
yarn install  # unplugged 패키지만 설치 (다운로드 없이 빠름).
yarn dev      # 실행
```
여기서 의문점이 생긴다. zero Install은 yarn install을 안해도 걍 바로 실행 가능한거 아녔음?

맞는 말이다. 공식문서를 보면 unplugged도 git에 올리라고 되어있지만 앞에서 말한 이유로 보통 사람들이 잘 안올린다.
완전한 Zero Install은 아니지만, 네트워크 없이도 yarn install이 끝난다는 점,  .yarn/unplugged 폴더만 깔면되서 속도가 빨라진다는 점에서 의미가 있다.

# 실제로 얼마나 빨라지냐
CI랑 배포 환경, 개발환경 모두에서 효과가 있다.
### CI

```
Zero Install 없을때
git clone     → 몇 초
yarn install  → 1~3분 (패키지 다운로드)
빌드/테스트 시작

Zero Install 있을때
git clone     → 수십 초 (zip 포함이라 좀 더 걸림)
yarn install  → 0초 (unplugged을 깔아야하면 몇 초)
빌드/테스트 시작
```

### 배포
배포 서버도 CI랑 똑같이 코드 받고 → 패키지 설치하고 → 빌드하는 흐름인데, Zero Install이 있으면 설치 단계가 없어지거나 극도로 짧아져서 배포 시간이 줄어든다.

# 단점

1. 저장소 용량이 커짐.
	.yarn/cache에 zip 파일들이 다 들어가니까 저장소 자체가 무거워져서 패키지가 많은 프로젝트면 수백 MB가 될 수 있다.
2. 패키지 추가/업데이트할 때 커밋이 커짐.
	 패키지를 깔거나 수정하면 zip 파일이 추가되거나 수정되니까 그 커밋에 바이너리 파일이 포함되서 PR이 지저분해 보일 수 있다.
3. 팀 합의가 필요함
	팀 전체가 .yarn/cache를 git에 올리기로 동의해야 한다. 중간에 누가 .gitignore를 수정해버리면 꼬인다. 

# 정리

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

팀 상황에 따라 PnP만 쓰고 Zero Install은 안 쓰는 경우도 많긴함.
