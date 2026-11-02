# Docker 실습 명령 2부 (Dockerfile · 포트 매핑 · 바인드 마운트 · 볼륨)

작성 2026-11-02. Ubuntu 창(`msong@...:~$`)에서 한 줄씩 친다. 1부와 같은 방식.
이번엔 **브라우저 캡처 2장**이 필요하다 (3절). 주소창에 포트 번호가 보이게 찍는다.

---

## 0. Dockerfile 직접 작성 (VSCode 에서)

과제 제약: "Dockerfile 은 직접 작성해야 한다". 그래서 이 파일만은 사장님이 만든다.

1. VSCode 탐색기에서 `app` 폴더 → 마우스 오른쪽 → **New File** → 이름 `Dockerfile` (확장자 없음, 대문자 D)
2. 아래 내용을 **한 줄씩 타이핑**한다 (복붙해도 되지만 치면서 뜻을 보는 게 시험에 남는다)
3. 저장 (Ctrl+S)

```dockerfile
FROM nginx:alpine
LABEL org.opencontainers.image.title="my-web"
LABEL org.opencontainers.image.authors="Frost0313z"
ENV APP_ENV=dev
COPY site/ /usr/share/nginx/html/
EXPOSE 80
HEALTHCHECK --interval=10s --timeout=3s CMD wget -qO- http://localhost/ || exit 1
```

### 각 줄의 목적 (README 에 그대로 쓸 내용)

| 줄 | 커스텀 포인트 | 목적 |
|---|---|---|
| `FROM nginx:alpine` | 베이스 이미지 선택 | 웹 서버가 이미 들어 있는 공식 이미지. alpine 판이라 작다 |
| `LABEL ...` | 이미지 메타데이터 | 누가 만든 무슨 이미지인지 `docker inspect` 로 볼 수 있게 |
| `ENV APP_ENV=dev` | 환경 변수 | 설정을 코드 밖에 둔다. 실행 시 `-e` 로 덮어쓸 수 있다 |
| `COPY site/ ...` | 정적 콘텐츠 교체 | nginx 기본 환영 페이지 대신 내 HTML 을 내보낸다 |
| `EXPOSE 80` | 포트 문서화 | "이 이미지는 80 번을 쓴다"는 표시. 실제 연결은 `-p` 가 한다 |
| `HEALTHCHECK` | 상태 검사 | 10초마다 자기 페이지를 받아 보고 실패하면 unhealthy 표시 |

---

## 1. 녹화 켜고 빌드

```bash
script ~/codyssey/logs/05-docker-커스텀.log

cd /mnt/c/Users/msong/orca/projects/Setting/app   # 저장소의 app 폴더 (Windows 쪽)
ls -la                                             # Dockerfile, site/ 가 보여야 함
cat Dockerfile                                     # 방금 쓴 내용 확인
docker build -t my-web:1.0 .                       # 마지막 점(.)은 "현재 폴더를 빌드 재료로"
docker images                                      # my-web 1.0 이 생김
docker inspect my-web:1.0 --format '{{.Config.Labels}} {{.Config.Env}}'   # LABEL 과 ENV 확인
```

---

## 2. 포트 매핑 2회 실행

```bash
docker run -d -p 8080:80 --name my-web-8080 my-web:1.0
docker run -d -p 8081:80 --name my-web-8081 my-web:1.0
docker ps                              # 둘 다 Up, PORTS 에 0.0.0.0:8080->80, 8081->80. 30초 뒤엔 (healthy)
curl http://localhost:8080             # Hello from my-web 이 든 HTML
curl http://localhost:8081             # 같은 내용. 같은 이미지로 컨테이너 2개
```

---

## 3. 브라우저 캡처 (Windows 쪽에서)

Windows 브라우저(Edge/Chrome)를 열고 아래 주소로 간다. **주소창과 페이지가 같이 보이게** 캡처한다.

1. `http://localhost:8080` → 캡처 → 저장 이름 `포트-8080.png`
2. `http://localhost:8081` → 캡처 → 저장 이름 `포트-8081.png`

캡처는 `Win + Shift + S`. 저장은 `C:\Users\msong\orca\projects\Setting\docs\images\` 에. 또는 이 대화창에 붙여 주면 내가 저장한다.

---

## 4. 바인드 마운트 (호스트 변경 → 즉시 반영)

빌드한 이미지는 HTML 이 **이미지 안에 구워져** 있어서 호스트 파일을 고쳐도 안 바뀐다. 바인드 마운트는 호스트 폴더를 그대로 연결해서 바로 반영된다.

```bash
docker run -d -p 8082:80 --name my-web-bind -v "$PWD/site:/usr/share/nginx/html" nginx:alpine
curl -s http://localhost:8082 | grep 버전        # 변경 전: 버전: 1.0

sed -i 's/버전: 1.0/버전: 1.1 (바인드 마운트로 수정)/' site/index.html   # 호스트 파일 수정
cat site/index.html | grep 버전                  # 호스트 파일이 바뀌었는지

curl -s http://localhost:8082 | grep 버전        # 변경 후: 1.1 로 바뀜. 재시작 없이
curl -s http://localhost:8080 | grep 버전        # 비교: 이미지에 구운 쪽은 여전히 1.0
```

마지막 두 줄의 대비가 README 의 "바인드 마운트 vs 이미지" 증거다.

---

## 5. 볼륨 영속성 (컨테이너 삭제 후에도 데이터 유지)

```bash
docker volume create mydata
docker volume ls                                        # mydata 보임
docker volume inspect mydata                            # Docker 가 관리하는 경로

docker run -d --name vol-test -v mydata:/data ubuntu sleep infinity
docker exec vol-test sh -c 'echo "survive me" > /data/hello.txt && cat /data/hello.txt'   # survive me
docker rm -f vol-test                                   # 컨테이너 삭제
docker ps -a | grep vol-test                            # 없음

docker run -d --name vol-test2 -v mydata:/data ubuntu sleep infinity   # 새 컨테이너에 같은 볼륨
docker exec vol-test2 cat /data/hello.txt               # survive me. 데이터가 살아 있다
```

비교용으로 볼륨 없이 해 보면 차이가 보인다.

```bash
docker run -d --name no-vol ubuntu sleep infinity
docker exec no-vol sh -c 'echo "gone" > /data.txt && cat /data.txt'
docker rm -f no-vol
docker run -d --name no-vol2 ubuntu sleep infinity
docker exec no-vol2 cat /data.txt                       # No such file. 컨테이너와 함께 사라짐
```

---

## 6. 운영 명령 한 번 더 (컨테이너 여러 개일 때)

```bash
docker ps                               # my-web-8080, 8081, bind, vol-test2, no-vol2, web
docker stats --no-stream                # 6개의 CPU/메모리
docker logs my-web-8080                 # curl 과 브라우저 접속 기록 (GET / 200)
```

---

## 7. 정리 후 녹화 끄기

```bash
docker rm -f my-web-8081 my-web-bind vol-test2 no-vol2   # my-web-8080 은 캡처 못 했을 때 대비해 남김
docker ps -a
docker volume ls                        # mydata 는 남아 있다. 컨테이너와 수명이 다르다
exit
cp ~/codyssey/logs/05-docker-커스텀.log /mnt/c/Users/msong/orca/projects/Setting/logs/
```

---

## 자주 나는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `failed to read dockerfile` | app 폴더가 아닌 곳에서 build, 또는 파일명이 `Dockerfile.txt` | `pwd` 확인, VSCode 에서 파일명 확인 |
| `port is already allocated` | 8080 을 다른 프로그램이 씀 | `-p 8090:80` 처럼 다른 번호로. 그게 트러블슈팅 1건 |
| `curl: (7) Failed to connect` | 컨테이너가 안 떴거나 포트 번호 틀림 | `docker ps` 로 PORTS 열 확인, `docker logs 이름` |
| 브라우저에서 안 열림 | Docker Desktop 이 Windows 포트로 전달 안 됨 | 30초 기다렸다 새로고침. 안 되면 `docker ps` 확인 |
| 바인드 마운트 후 페이지가 403 | site/ 경로가 틀려 빈 폴더가 마운트됨 | `ls site/` 로 index.html 있는지, `$PWD` 가 app 인지 |
| sed 가 한글을 못 바꿈 | 파일 인코딩 | VSCode 에서 직접 1.0 → 1.1 로 고치고 저장해도 같은 실험 |
