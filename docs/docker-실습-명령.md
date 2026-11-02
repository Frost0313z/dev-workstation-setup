# Docker 실습 명령 1부 (운영 명령 · 컨테이너 진입 · attach vs exec)

작성 2026-11-02. Ubuntu 창(`msong@...:~$`)에서 **한 줄씩** 친다. `#` 뒤는 설명.
Docker Desktop 이 켜져 있어야 한다 (트레이에 고래 아이콘).

---

## 0. 녹화 켜기

```bash
script ~/codyssey/logs/04-docker-실습.log
```

---

## 1. 설치 점검

```bash
docker --version          # Docker version 29.x
docker info | head -20    # Server Version 줄이 보이면 데몬 정상. 앞 20줄만
```

---

## 2. 이미지: 다운로드 / 목록

```bash
docker pull ubuntu        # Docker Hub 에서 ubuntu 이미지 받기
docker pull nginx:alpine  # 작은 웹 서버 이미지 (다음 실습에서 베이스로 씀)
docker images             # 받은 이미지 목록. hello-world, ubuntu, nginx 가 보인다
```

---

## 3. 컨테이너: 실행 / 목록 / 중지

```bash
docker run hello-world    # 실행하고 바로 끝나는 컨테이너
docker ps                 # 실행 중인 것만. hello-world 는 이미 끝나서 안 보인다
docker ps -a              # 전체. hello-world 가 Exited 상태로 보인다
```

이 차이가 중요하다. **컨테이너는 메인 프로세스가 끝나면 같이 끝난다.** hello-world 는 메시지 출력이 끝나서 종료됐다.

```bash
docker run -d --name web nginx:alpine   # -d: 백그라운드. 웹 서버는 계속 도니까 살아 있다
docker ps                               # web 이 Up 상태
docker stop web                         # 중지
docker ps                               # 안 보인다
docker ps -a                            # Exited 로 보인다
docker start web                        # 다시 시작
docker ps                               # 다시 Up
```

---

## 4. 운영: 로그 / 리소스

```bash
docker logs web                 # nginx 가 출력한 시작 로그
docker exec web wget -qO- http://localhost   # 컨테이너 안에서 자기 자신에게 요청 → HTML 출력
docker logs web                 # 방금 요청이 접속 로그 한 줄로 추가됨 (GET / 200)
docker stats --no-stream        # CPU/메모리 사용량 한 번만 출력. --no-stream 없으면 실시간이라 안 끝남
```

---

## 5. ubuntu 컨테이너 진입 (exec 방식)

```bash
docker run -d --name u1 ubuntu sleep infinity   # sleep infinity 로 살려 둔다
docker ps                                       # u1 Up
docker exec -it u1 bash                         # 안으로 들어간다. 프롬프트가 root@...:/# 로 바뀜
```

이제부터는 **컨테이너 안**이다. 몇 개 쳐 본다.

```bash
ls /                      # 컨테이너의 루트. 내 Ubuntu 와 다른 파일 시스템
echo "inside container"
cat /etc/os-release | head -2   # 어떤 리눅스인지
whoami                    # root
hostname                  # 컨테이너 ID 가 호스트 이름
touch /tmp/mark.txt       # 흔적 남기기
exit                      # 빠져나온다. 프롬프트가 msong@ 로 돌아옴
```

```bash
docker ps                 # u1 이 **아직 Up**. exec 로 들어갔다 나와도 컨테이너는 산다
docker exec u1 ls /tmp    # mark.txt 가 남아 있다
```

---

## 6. attach 방식과 비교

```bash
docker run -dit --name u2 ubuntu bash   # 메인 프로세스가 bash 인 컨테이너. -it 로 터미널 유지
docker ps                               # u2 Up
docker attach u2                        # 메인 프로세스(bash)에 **붙는다**. 프롬프트 root@
```

컨테이너 안에서:

```bash
echo "attached"
exit                      # 메인 프로세스 bash 가 종료됨
```

```bash
docker ps                 # u2 가 **없다**
docker ps -a              # u2 가 Exited. 메인 프로세스가 끝나서 컨테이너도 끝났다
```

### 정리 (README 에 적을 내용)

| | exec | attach |
|---|---|---|
| 무엇에 연결 | 새 프로세스를 하나 더 띄움 | 이미 도는 메인 프로세스(PID 1)에 붙음 |
| exit 하면 | 새 프로세스만 끝남, 컨테이너 유지 | 메인 프로세스가 끝남, 컨테이너 종료 |
| 쓰는 때 | 실무 대부분. 들어가서 확인하고 나올 때 | 메인 프로세스 화면을 직접 봐야 할 때 |

---

## 7. 정리 후 녹화 끄기

```bash
docker rm -f u1 u2        # 실습용 컨테이너 삭제 (-f: 실행 중이어도)
docker ps -a              # web 과 hello-world 만 남음. web 은 다음 실습에서 쓴다
exit                      # script 녹화 종료
cp ~/codyssey/logs/04-docker-실습.log /mnt/c/Users/msong/orca/projects/Setting/logs/
```

---

## 자주 나는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker Desktop 꺼짐 | 시작 메뉴에서 Docker Desktop 실행 후 30초 대기 |
| `docker stats` 가 안 끝남 | 실시간 모드 | `Ctrl+C` 로 빠져나온 뒤 `--no-stream` 붙여 다시 |
| `Conflict. The container name "/web" is already in use` | 같은 이름이 이미 있음 | `docker rm -f web` 후 다시 |
| 컨테이너 안에서 `curl: command not found` | ubuntu 이미지는 깡통 | 이 문서는 curl 대신 nginx 안의 `wget` 을 씀 |
| exec 했는데 프롬프트가 안 바뀜 | `-it` 빠짐 | `docker exec -it 이름 bash` |
