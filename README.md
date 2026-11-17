# Development Workstation Lab

Linux CLI, Docker, Git/GitHub를 활용해 **재현 가능한 개발 워크스테이션**을 구축하고, 명령어와 출력 결과로 기능을 검증한 실습 프로젝트입니다.

> **실습일:** 2026-11-17  
> **제출 저장소:** https://github.com/Samdasoo3/dev-workstation-lab  
> **실행 환경:** Windows + WSL2 Ubuntu + Docker Desktop  
> **안내:** 본 문서의 결과는 실습 당시 기록입니다. 동일한 명령을 실행해도 컨테이너 ID, 실행 시각 등은 달라집니다.

## 1. 목표 및 실행 환경

이 프로젝트는 파일·디렉터리와 권한을 CLI로 관리하고, Docker 이미지와 컨테이너의 차이를 이해하며, 웹 서버를 이미지로 빌드해 포트와 스토리지를 연결하고, Git/GitHub로 결과를 공유하는 것을 목표로 합니다.

| 구분 | 실습 환경 |
|---|---|
| 호스트 OS | Windows |
| 리눅스 실행 환경 | WSL2, Ubuntu 26.04.1 LTS |
| 셸 / 터미널 | Bash (`/bin/bash`), Windows Terminal / VS Code 통합 터미널 |
| Docker | Docker Desktop 연동, Docker CLI 29.9.3 (build 5457199) |
| Git | 2.53.0 |
| GitHub CLI | 2.104.0 |
| 편집기 | VS Code (WSL: Ubuntu) |

**실제로 실행한 확인 명령과 출력(발췌)**

```bash
$ lsb_release -ds
Ubuntu 26.04.1 LTS
$ echo "$SHELL"
/bin/bash
$ docker --version
Docker version 29.9.3, build 5457199
$ git --version
git version 2.53.0
```

[환경 확인 스크린샷](evidence/environment-check.png) · [WSL2 정상 실행](evidence/wsl-success.png) · [Docker 엔진 정보](evidence/docker-info.png)

## 2. 수행 체크리스트

- [x] 현재 경로·숨김 파일 목록 확인, 이동·생성·복사·이름 변경·삭제, 파일 내용 확인
- [x] 파일·디렉터리 권한 확인 및 변경
- [x] Docker 설치·버전·데몬 동작 확인
- [x] 이미지 목록, 컨테이너 실행·중지·목록, 로그·리소스 확인
- [x] `hello-world`, `ubuntu:24.04` 실행 및 컨테이너 내부 명령 수행
- [x] NGINX 기반 Dockerfile 직접 작성 및 커스텀 이미지 빌드
- [x] 포트 매핑(8088, 8090) 및 브라우저/`curl` 접속
- [x] 바인드 마운트 파일 변경 반영 검증
- [x] 이름 있는 Docker 볼륨 데이터 영속성 검증
- [x] Git 작성자·기본 브랜치 설정 및 커밋
- [x] VS Code–WSL–GitHub 로그인/원격 저장소 연결
- [x] 트러블슈팅 기록과 재현 절차 문서화

## 3. 프로젝트 구성

```text
dev-workstation-lab/
├── README.md                 # 과제 설명·명령어·결과·증거
├── Dockerfile                # NGINX 기반 커스텀 이미지
├── site/
│   └── index.html            # 웹 페이지
├── practice/
│   └── sample.txt            # CLI·권한 실습 파일
└── evidence/                 # 실행 결과 스크린샷
```

`permissions-dir`처럼 비어 있는 폴더는 Git에서 추적되지 않을 수 있습니다. 필요 시 아래의 권한 실습 명령으로 다시 만듭니다.

## 4. Linux CLI: 경로·파일 조작·권한

### 4-1. 현재 경로와 기본 파일 조작

```bash
$ cd ~
$ mkdir -p dev-workstation-lab
$ cd dev-workstation-lab
$ mkdir practice site evidence
$ pwd
/home/user/dev-workstation-lab
$ ls -la
# practice, site, evidence 등을 확인

$ cd practice
$ touch sample.txt
$ echo "Hello Linux" > sample.txt
$ cat sample.txt
Hello Linux
$ cp sample.txt copy.txt
$ mv copy.txt renamed.txt
$ ls -la
# sample.txt, renamed.txt 확인
$ mkdir temp-dir
$ rmdir temp-dir
$ rm renamed.txt
$ ls -la
# sample.txt만 남음
```

- **절대 경로**: `/home/user/dev-workstation-lab/practice`처럼 `/`부터 시작하며 현재 위치와 무관합니다.
- **상대 경로**: `practice/sample.txt`처럼 현재 작업 디렉터리를 기준으로 해석합니다.
- `pwd`는 현재 경로, `ls -la`는 숨김 파일 포함 상세 목록, `touch`는 빈 파일 생성, `cat`은 내용 확인, `cp`는 복사, `mv`는 이동/이름 변경, `rm`은 파일 삭제, `rmdir`는 빈 디렉터리 삭제에 사용했습니다.

[디렉터리 생성·경로 확인](evidence/linux-directories.png) · [파일 복사·이동·삭제 로그](evidence/linux-files.png)

### 4-2. 파일과 디렉터리 권한

```bash
$ ls -l sample.txt
-rw-r--r-- ... sample.txt
$ chmod 600 sample.txt
$ ls -l sample.txt
-rw------- ... sample.txt

$ mkdir permissions-dir
$ ls -ld permissions-dir
drwxr-xr-x ... permissions-dir
$ chmod 700 permissions-dir
$ ls -ld permissions-dir
drwx------ ... permissions-dir
```

권한은 **소유자 / 그룹 / 기타 사용자** 순서로 기록합니다. `r`(읽기)=4, `w`(쓰기)=2, `x`(실행·디렉터리 접근)=1입니다. `644`는 `rw- / r-- / r--`, `600`은 `rw- / --- / ---`, `755`는 `rwx / r-x / r-x`, `700`은 `rwx / --- / ---`입니다.

**검증:** 파일 644 → 600, 디렉터리 755 → 700으로 실제 변경했습니다. [권한 변경 전·후 증거](evidence/permissions.png)

## 5. Docker 설치 및 기본 운영

### 5-1. 엔진 연결 및 hello-world

Windows의 Docker Desktop에서 Ubuntu에 대한 **WSL Integration**을 활성화한 뒤 Ubuntu에서 다음을 실행했습니다.

```bash
$ docker --version
Docker version 29.9.3, build 5457199
$ docker info
# Client와 Server 정보 정상 출력
$ docker run hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

[WSL Integration 설정](evidence/docker-wsl-integration.png) · [Docker 엔진 확인](evidence/docker-info.png) · [hello-world 성공](evidence/docker-hello-world.png)

### 5-2. 이미지와 컨테이너 목록 / Ubuntu 내부 명령

```bash
$ docker images
# hello-world:latest, ubuntu:24.04, web-lab:1.0 등 확인
$ docker ps -a
# 실행·종료 상태의 컨테이너 목록 확인
$ docker run -it --name lab-ubuntu ubuntu:24.04 bash
root@<container-id>:/# pwd
/
root@<container-id>:/# ls /
# bin, boot, dev, etc, home, ... 확인
root@<container-id>:/# echo "Hello from Ubuntu container"
Hello from Ubuntu container
root@<container-id>:/# exit
exit
$ docker ps
# 실행 중인 컨테이너 없음
$ docker ps -a
# lab-ubuntu가 Exited (0) 상태
$ docker logs lab-ubuntu
# 앞서 실행한 pwd, ls, echo 등의 출력 확인
```

`docker run -it`는 대화형 컨테이너에 연결합니다. `exit`로 PID 1의 셸을 종료하자 해당 컨테이너는 종료되었지만 `docker ps -a` 목록에는 남았습니다. **이미지**는 실행 템플릿이고 **컨테이너**는 이미지를 바탕으로 생성된 실행 인스턴스입니다.

[이미지·컨테이너 목록](evidence/docker-images-ps.png) · [Ubuntu 컨테이너 내부 실행](evidence/docker-ubuntu.png) · [컨테이너 로그](evidence/docker-logs.png)

### 5-3. 백그라운드 실행·exec·stats·stop

```bash
$ docker run -d --name lab-background ubuntu:24.04 sleep infinity
# 컨테이너 ID 출력
$ docker ps
# lab-background가 Up 상태
$ docker exec lab-background echo "Background container is running"
Background container is running
$ docker stats --no-stream lab-background
# CPU %, MEM USAGE / LIMIT, NET I/O 등의 표 출력
$ docker stop lab-background
lab-background
$ docker ps -a
# lab-background가 Exited (137) 상태로 표시됨
```

`-d`는 백그라운드 실행, `docker exec`는 **실행 중인 컨테이너**에서 별도의 명령 실행, `--no-stream`은 `docker stats` 정보를 한 번만 표시합니다. `Exited (137)`은 종료 과정에 SIGKILL이 사용되었음을 나타냅니다. 이번 사례에서 중지 명령과 상태 확인은 정상 수행되었습니다.

[운영·리소스 사용량·종료 로그](evidence/docker-stats.png)

## 6. Dockerfile 기반 NGINX 웹 서버

### 6-1. 베이스 이미지와 커스텀 포인트

직접 작성한 [Dockerfile](Dockerfile)은 기존의 `nginx:alpine` 이미지를 베이스로 사용합니다.

```dockerfile
FROM nginx:alpine
LABEL org.opencontainers.image.title="my-first-docker-web"
COPY site/ /usr/share/nginx/html/
EXPOSE 80
```

| 항목 | 목적 |
|---|---|
| `FROM nginx:alpine` | NGINX가 설치된 경량 베이스 이미지 사용 |
| `LABEL` | 이미지 식별용 설명 메타데이터 추가 |
| `COPY site/ ...` | 직접 만든 HTML 파일을 NGINX 정적 페이지 경로로 복사 |
| `EXPOSE 80` | 컨테이너가 웹 서버 포트 80을 사용함을 문서화 (`-p` 없이 외부 공개되지 않음) |

[Dockerfile 직접 작성 증거](evidence/dockerfile-create.png) · [웹페이지 원본](site/index.html)

### 6-2. 빌드·실행·포트 매핑

```bash
$ cd ~/dev-workstation-lab
$ docker build -t web-lab:1.0 .
# Building ... FINISHED
# naming to docker.io/library/web-lab:1.0
$ docker images
# web-lab:1.0 목록 확인
$ docker run -d --name lab-web -p 127.0.0.1:8088:80 web-lab:1.0
# 컨테이너 ID 출력
$ docker ps
# lab-web: Up, 127.0.0.1:8088->80/tcp
$ curl http://localhost:8088
# HTML 문서 출력 (Docker Web Server Success!)
```

`127.0.0.1:8088:80`은 **호스트 로컬 주소의 8088 포트**를 컨테이너의 **80 포트**에 연결합니다. 포트 매핑이 있어야 브라우저가 컨테이너 내부 NGINX에 접근할 수 있습니다. `127.0.0.1`을 지정해 다른 장비로의 불필요한 노출을 줄였습니다.

[이미지 빌드 성공](evidence/docker-build.png) · [컨테이너 실행·curl](evidence/docker-web-run.png) · [주소창 포함 브라우저 접속 성공](evidence/web-success.png)

> **빌드 시점 차이:** `8088`의 이미지에는 최초 빌드 당시 웹 문구가 들어 있습니다. 이후 바인드 마운트 실습에서 호스트 HTML을 수정했습니다. 현재 저장소 파일을 새로 빌드하면 수정된 문구가 이미지에 포함됩니다.

## 7. 바인드 마운트 변경 반영

프로젝트의 `site/` 폴더를 컨테이너의 NGINX 페이지 폴더에 읽기 전용으로 연결했습니다.

```bash
$ cd ~/dev-workstation-lab
$ docker run -d \
    --name lab-bind \
    -p 127.0.0.1:8090:80 \
    --mount type=bind,source="$(pwd)/site",target=/usr/share/nginx/html,readonly \
    web-lab:1.0
$ curl http://localhost:8090
# This is my first Docker website.
$ sed -i 's/This is my first Docker website./This page was updated through a bind mount./' site/index.html
$ curl http://localhost:8090
# This page was updated through a bind mount.
```

이때 **이미지를 다시 빌드하거나 컨테이너를 재시작하지 않았습니다.** 호스트의 파일을 직접 연결했기 때문에 NGINX가 변경된 파일을 읽었습니다. `readonly`는 컨테이너 쪽에서 수정하는 것을 제한하며 호스트 파일 수정을 막지는 않습니다.

[변경 전 실행 명령과 `curl` 출력](evidence/bind-before.png) · [변경 후 주소창 포함 화면](evidence/bind-after.png)

## 8. Docker 볼륨 영속성 검증

Docker가 관리하는 이름 있는 볼륨 `lab-data`를 만들어 첫 컨테이너에서 파일을 저장한 뒤, **첫 컨테이너를 삭제하고** 동일한 볼륨에 연결한 새 컨테이너에서 내용을 확인했습니다.

```bash
$ docker volume create lab-data
lab-data
$ docker volume ls
DRIVER    VOLUME NAME
local     lab-data
$ docker run -d --name lab-vol1 \
    --mount type=volume,source=lab-data,target=/data \
    ubuntu:24.04 sleep infinity
# 컨테이너 ID 출력
$ docker exec lab-vol1 sh -c 'echo "data-kept" > /data/test.txt'
$ docker exec lab-vol1 cat /data/test.txt
data-kept
$ docker stop lab-vol1
lab-vol1
$ docker rm lab-vol1
lab-vol1
$ docker volume ls
DRIVER    VOLUME NAME
local     lab-data
$ docker run -d --name lab-vol2 \
    --mount type=volume,source=lab-data,target=/data \
    ubuntu:24.04 sleep infinity
# 새로운 컨테이너 ID 출력
$ docker exec lab-vol2 cat /data/test.txt
data-kept
$ docker volume inspect lab-data
# Driver: local, Name: lab-data, Mountpoint: /var/lib/docker/volumes/lab-data/_data
```

**결과:** `lab-vol1`을 삭제했는데도 `lab-vol2`에서 `data-kept`가 출력되었습니다. 따라서 컨테이너의 수명과 볼륨의 데이터 수명이 별개임을 확인했습니다.

[볼륨 생성·삭제 전/후 전체 검증 증거](evidence/volume-persistence.png)

**바인드 마운트 vs 볼륨:** 바인드 마운트는 **사용자가 지정한 호스트 폴더**를 연결해 파일 수정이 바로 반영되도록 활용했고, Docker 볼륨은 **Docker가 관리하는 저장소**를 연결해 컨테이너 삭제 후에도 데이터를 유지하도록 활용했습니다.

## 9. Git / GitHub / VS Code 연동

### 9-1. Git 설정과 최초 커밋

```bash
$ git --version
git version 2.53.0
$ git config --global init.defaultBranch main
$ git config --global user.name "Samdasoo3"
$ git config --global user.email "<GitHub에서 제공한 noreply 주소>"
$ git config --list
init.defaultbranch=main
user.name=Samdasoo3
user.email=<GitHub noreply 주소>
$ git init
# Initialized empty Git repository .../.git/
$ git add README.md Dockerfile site/index.html practice/
$ git commit -m "Initial development workstation setup"
# 최초 커밋 생성
$ git status
On branch main
nothing to commit, working tree clean
```

> **개인정보:** 첫 Git 설정 시 개인 이메일을 사용했다가, 공개 업로드 전에 `noreply` 이메일로 수정하고 `git commit --amend --reset-author --no-edit`로 최초 커밋의 작성자 정보도 교체했습니다. 위 설정의 이메일은 의도적으로 공개하지 않았습니다.

[Git 초기화](evidence/git-init.png) · [Git 최초 커밋](evidence/git-first-commit.png) · [이메일 보호 확인](evidence/git-email-privacy.png)

### 9-2. GitHub CLI 로그인·원격 연결·Push

```bash
$ gh --version
# gh version 2.104.0
$ gh auth status
# Logged in to github.com account Samdasoo3
# Active account: true
$ gh repo create dev-workstation-lab --public --source=. --remote=origin --push
# Added remote ...
# Pushed commits to https://github.com/Samdasoo3/dev-workstation-lab.git
$ git remote -v
origin  https://github.com/Samdasoo3/dev-workstation-lab.git (fetch)
origin  https://github.com/Samdasoo3/dev-workstation-lab.git (push)
$ git status
On branch main
Your branch is up to date with 'origin/main'.
nothing to commit, working tree clean
```

**Git**은 로컬 변경 이력을 기록하는 도구이고 **GitHub**는 원격 저장소에 코드를 공유하고 협업하는 플랫폼입니다. VS Code는 WSL Ubuntu 프로젝트를 열어 로컬 Git 저장소 및 GitHub 로그인을 확인했습니다.

[VS Code·WSL·Git](evidence/vscode-wsl-git.png) · [GitHub CLI 로그인 증거](evidence/github-login-success.png) · [VS Code GitHub 및 원격 저장소 연결](evidence/vscode-github-remote.png) · [GitHub 원격 Push](evidence/github-push.png) · [저장소 브라우저 화면](evidence/github-repository.png)

**보안:** 인증 토큰·비밀번호·일회용 인증 코드는 저장소에 기록하지 않았습니다. 인증 상태는 민감한 토큰 값이 마스킹된 출력만 사용했습니다.

## 10. 트러블슈팅

### 문제 1 — WSL에서 Ubuntu 배포판을 찾지 못함

| 단계 | 기록 |
|---|---|
| 문제 | PowerShell에서 `wsl -l -v` 실행 시 설치된 리눅스 배포판이 없다는 메시지 표시 |
| 원인 가설 | WSL 실행 환경은 준비되어 있으나 Ubuntu 배포판 자체가 설치되지 않았음 |
| 확인 | `wsl -l -v` 결과에 Ubuntu가 없음 |
| 조치 | 관리자 PowerShell에서 `wsl --install -d Ubuntu` 실행 후 사용자 계정·비밀번호 설정 |
| 해결 확인 | `wsl -l -v`에서 `Ubuntu`, `Running`, `VERSION 2` 표시 |

[설치 전 상태](evidence/wsl-before.png) · [설치 과정](evidence/wsl-install.png) · [정상 설치 결과](evidence/wsl-success.png)

### 문제 2 — Docker 소켓 접근 권한 거부

| 단계 | 기록 |
|---|---|
| 문제 | `docker info`, `docker run hello-world` 실행 시 `permission denied ... /var/run/docker.sock` |
| 원인 가설 | Docker Desktop WSL 연결 또는 현재 세션의 그룹 권한이 아직 정상 반영되지 않음 |
| 확인 | `ls -l /var/run/docker.sock`의 그룹이 `docker`, `getent group docker`에는 `user`가 등록되어 있으나 `id -nG`의 현재 활성 그룹에는 `docker`가 없었음 |
| 조치 | Docker Desktop의 WSL Integration 상태를 확인하고 WSL 세션 종료·재시작 후 재검증 |
| 해결 확인 | 새 Ubuntu 세션에서 `docker run hello-world` 실행 시 `Hello from Docker!` 출력 |

**주의:** 그룹 등록은 이미 확인되었고, 새 세션에서 성공했습니다. 따라서 특정 조치 하나만이 유일한 원인이었다고 단정하지 않습니다.

[권한 오류](evidence/docker-permission-error.png) · [그룹 상태 확인](evidence/docker-error-before.png) · [정상 실행](evidence/docker-hello-world.png)

### 참고 — WSL에서 브라우저 자동 실행 실패

`gh repo view --web`는 WSL 내부에 브라우저 실행 도구(`xdg-open` 등)가 없어서 자동으로 열지 못했습니다. 하지만 GitHub 저장소 생성·Push는 정상적으로 완료되었으며, **저장소 URL을 Windows 브라우저에 직접 입력**해 확인했습니다.

## 11. 재현 방법

다른 평가자가 **Docker Desktop(WSL2 연동)과 Git이 준비된 Linux/WSL 터미널**에서 프로젝트를 내려받아 실행하는 방법입니다. 실습 당시 사용한 PC의 `/home/user` 경로에 의존하지 않도록 `$(pwd)`를 사용합니다.

### 11-1. 저장소 내려받기와 웹 서버 실행

```bash
git clone https://github.com/Samdasoo3/dev-workstation-lab.git
cd dev-workstation-lab
docker info
docker build -t web-lab:1.0 .
docker run -d --name verify-web -p 127.0.0.1:8088:80 web-lab:1.0
curl http://localhost:8088
```

브라우저에서 `http://localhost:8088`을 열면 직접 작성한 HTML이 표시됩니다. **이미 다른 프로그램이 8088을 사용 중이면** 호스트 포트를 `8089` 등 사용 가능한 번호로 바꿉니다. 실행 이름이 이미 존재하면 이름을 변경하거나 기존 테스트 컨테이너를 정리합니다.

### 11-2. 현재 저장소 버전으로 바인드 마운트 재검증

현재 저장소의 `site/index.html`에는 이미 바인드 마운트 실습 후의 문구가 반영되어 있습니다. 따라서 원래 실습의 변경 전 문구를 다시 치환하기보다 다음처럼 다른 문구로 변경해 검증합니다.

```bash
docker run -d --name verify-bind \
  -p 127.0.0.1:8090:80 \
  --mount type=bind,source="$(pwd)/site",target=/usr/share/nginx/html,readonly \
  web-lab:1.0
curl http://localhost:8090
sed -i 's/This page was updated through a bind mount./Bind mount verified again./' site/index.html
curl http://localhost:8090
# 확인 후 원본 파일로 복구
git restore site/index.html
```

`git restore` 후에도 바인드 마운트로 연결된 파일은 즉시 원래 문구를 다시 보여줍니다.

### 11-3. 새 이름의 볼륨으로 영속성 재검증

```bash
docker volume create verify-data
docker run -d --name verify-vol1 \
  --mount type=volume,source=verify-data,target=/data \
  ubuntu:24.04 sleep infinity
docker exec verify-vol1 sh -c 'echo persistence-ok > /data/check.txt'
docker exec verify-vol1 cat /data/check.txt
docker rm -f verify-vol1
docker run -d --name verify-vol2 \
  --mount type=volume,source=verify-data,target=/data \
  ubuntu:24.04 sleep infinity
docker exec verify-vol2 cat /data/check.txt
# persistence-ok가 다시 출력되면 성공
```

실습이 끝나면 필요에 따라 `docker rm -f verify-web verify-bind verify-vol2`로 테스트 컨테이너를 제거할 수 있습니다. **`docker volume rm verify-data`를 실행하면 저장된 실습 데이터도 삭제**되므로 필요할 때만 실행합니다.

## 12. 증거 파일 바로가기

각 절에 명령어와 관련 증거를 연결했습니다. 주요 자료를 다시 찾으려면 아래 링크를 사용할 수 있습니다.

| 영역 | 증거 |
|---|---|
| 환경 | [환경](evidence/environment-check.png), [WSL 설치](evidence/wsl-success.png) |
| CLI·권한 | [파일 조작](evidence/linux-files.png), [권한 변경](evidence/permissions.png) |
| Docker 기본 | [정보](evidence/docker-info.png), [이미지·목록](evidence/docker-images-ps.png), [Ubuntu 내부](evidence/docker-ubuntu.png), [로그](evidence/docker-logs.png), [통계](evidence/docker-stats.png) |
| 웹 서버 | [Dockerfile](evidence/dockerfile-create.png), [빌드](evidence/docker-build.png), [실행](evidence/docker-web-run.png), [브라우저](evidence/web-success.png) |
| 스토리지 | [바인드 전](evidence/bind-before.png), [바인드 후](evidence/bind-after.png), [볼륨 삭제 전·후](evidence/volume-persistence.png) |
| Git·GitHub | [커밋](evidence/git-first-commit.png), [VS Code 연동](evidence/vscode-github-remote.png), [로그인](evidence/github-login-success.png), [Push](evidence/github-push.png) |
| 문제 해결 | [WSL 전](evidence/wsl-before.png), [Docker 권한 오류](evidence/docker-permission-error.png), [해결](evidence/docker-hello-world.png) |

---

**최종 검증 요약:** Linux 파일/권한 실습, Docker 이미지·컨테이너, NGINX 커스텀 빌드, `8088`/`8090` 포트, 바인드 마운트 변경 반영, 볼륨 삭제 전·후 데이터 유지, Git/GitHub 연동을 직접 실행하여 확인했습니다.
