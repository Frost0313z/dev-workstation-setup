# Docker 실습 명령 3부 (보너스: Compose · 멀티 컨테이너 · 운영 명령 · 환경 변수)

작성 2026-11-02. Ubuntu 창에서 한 줄씩. 위치는 2부와 같은 `app` 폴더.

---

## 0. Compose 가 뭔가

2부에서 친 긴 `docker run -d -p 8080:80 --name ... my-web:1.0` 을 **파일에 적어 두고** `docker compose up` 한 번으로 실행하는 도구.
명령이 "문서화된 실행 설정"이 돼서 누가 해도 같은 구성이 뜬다. 컨테이너가 여러 개면 한 파일에 전부 적는다.

### `app/docker-compose.yml` 읽는 법

```yaml
services:
  web:                              # 서비스 1: 2부에서 만든 웹 서버
    build: .                        # 이 폴더의 Dockerfile 로 빌드
    image: my-web:compose
    ports:
      - "${WEB_PORT:-8083}:80"      # 환경 변수 WEB_PORT, 없으면 8083
    environment:
      APP_ENV: ${APP_ENV:-dev}      # 환경 변수 APP_ENV, 없으면 dev

  checker:                          # 서비스 2: 보조. 5초마다 web 에 요청해 로그에 남김
    image: alpine
    depends_on:
      - web                         # web 이 먼저 뜬 뒤에 시작
    command: sh -c "while true; do echo \"$(date +%T) $(wget -qO- http://web/ | grep -o '<h1>.*</h1>')\"; sleep 5; done"
```

핵심은 checker 의 `http://web/`. IP 가 아니라 **서비스 이름**으로 접속한다. Compose 가 두 컨테이너를 같은 네트워크에 넣고 이름을 주소로 쓸 수 있게 해 준다 (서비스 디스커버리).

---

## 1. 녹화 켜고 단일 서비스 실행

```bash
script ~/codyssey/logs/06-compose.log
cd /mnt/c/Users/msong/orca/projects/Setting/app
cat docker-compose.yml
docker compose config                  # 파일 문법 검사 + 변수 치환된 최종 설정 출력
docker compose up -d web               # web 서비스만 실행 (단일 서비스)
docker compose ps                      # web Up, 8083->80
curl http://localhost:8083
```

---

## 2. 멀티 컨테이너 + 네트워크 통신

```bash
docker compose up -d                   # 이번엔 전부 (checker 추가됨)
docker compose ps                      # web, checker 둘 다 Up
sleep 10
docker compose logs checker            # 5초마다 "<h1>Hello from my-web</h1>" 이 찍힘 = checker 가 web 이름으로 접속 성공
docker compose exec checker ping -c 2 web      # 이름이 IP 로 풀리는 것을 직접 확인
docker compose exec checker wget -qO- http://web/ | grep h1
```

---

## 3. 운영 명령 루틴 (up · ps · logs · down)

```bash
docker compose ps                      # 상태
docker compose logs --tail 3 web       # web 의 최근 로그 3줄 (checker 의 접속 기록이 보임)
docker compose logs --tail 3 checker
docker compose down                    # 전부 정지 + 삭제 + 네트워크 제거
docker compose ps                      # 비어 있음
docker ps -a | grep app-               # 컨테이너도 없음
```

---

## 4. 환경 변수로 포트 · 모드 바꾸기

같은 파일, 같은 명령인데 환경 변수만 다르게 준다.

```bash
docker compose up -d                   # 기본값
docker compose ps                      # 8083
docker compose exec web env | grep APP_ENV      # APP_ENV=dev
docker compose down

WEB_PORT=8084 APP_ENV=prod docker compose up -d    # 변수를 앞에 붙여 실행
docker compose ps                      # 8084
curl -s http://localhost:8084 | grep h1
docker compose exec web env | grep APP_ENV      # APP_ENV=prod
docker compose down
```

`.env` 파일에 적어 두는 방법도 있다 (파일이 있으면 자동으로 읽는다):

```bash
printf 'WEB_PORT=8085\nAPP_ENV=staging\n' > .env
cat .env
docker compose up -d
docker compose ps                      # 8085
docker compose exec web env | grep APP_ENV      # staging
docker compose down
rm .env                                # 실습 후 제거 (.env 는 보통 저장소에 안 올린다)
```

---

## 5. 녹화 끄기

```bash
docker compose ps                      # 비어 있는지 마지막 확인
exit
cp ~/codyssey/logs/06-compose.log /mnt/c/Users/msong/orca/projects/Setting/logs/
```

---

## 배움 포인트 (README 에 쓸 것)

| 항목 | 한 줄 |
|---|---|
| Compose 기초 | `docker run` 옵션 뭉치가 `docker-compose.yml` 한 파일로 바뀐다. 실행 설정이 코드처럼 저장소에 남는다 |
| 멀티 컨테이너 | 한 파일에 서비스를 여러 개 적으면 `up` 한 번에 다 뜬다 |
| 서비스 디스커버리 | 같은 Compose 안에서는 서비스 이름(`web`)이 곧 주소다. IP 를 몰라도 된다 |
| 운영 루틴 | `up -d` → `ps` → `logs` → `down`. 상태를 보는 습관 |
| 설정과 코드의 분리 | 포트·모드를 파일이 아니라 환경 변수(`WEB_PORT`, `APP_ENV`)로 바꾼다. 코드를 안 고치고 환경별로 다르게 띄운다 |

---

## 자주 나는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `port is already allocated` | 2부의 my-web-8080 등이 포트를 잡고 있음 | 이 문서는 8083~8085 를 써서 피함. 그래도 나면 `docker ps` 로 범인 확인 후 `docker rm -f` |
| `docker-compose: command not found` | 옛 명령(하이픈) | `docker compose` (공백) |
| `no configuration file provided` | app 폴더가 아님 | `pwd` 확인 |
| checker 로그가 비어 있음 | 5초 주기라 아직 안 찍힘 | `sleep 10` 후 다시 `logs` |
