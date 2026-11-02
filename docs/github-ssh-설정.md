# 보너스: GitHub SSH 키 설정 (PowerShell 에서)

작성 2026-11-02. 이번엔 **Ubuntu 가 아니라 PowerShell** 이다. 이 저장소의 git 은 Windows 쪽에서 쓰기 때문.
시작 메뉴 → "PowerShell" (관리자 아님). 프롬프트가 `PS C:\Users\msong>` 이면 맞다.

---

## 0. 왜 SSH 인가

지금은 HTTPS 로 연결돼 있고, GitHub CLI 가 발급받은 토큰으로 인증한다. SSH 는 **키 쌍**으로 인증한다.
- 개인키 (`id_ed25519`): 내 PC 에만 둔다. 절대 밖으로 안 나간다.
- 공개키 (`id_ed25519.pub`): GitHub 에 등록한다. 공개돼도 괜찮다.
GitHub 는 "이 공개키와 짝인 개인키를 가진 사람"만 통과시킨다. 토큰처럼 복사해서 돌려 쓸 수 있는 문자열이 없다.

---

## 1. 녹화 켜고 키 만들기

PowerShell 은 `script` 가 없고 `Start-Transcript` 가 같은 일을 한다.

```powershell
cd C:\Users\msong\orca\projects\Setting
Start-Transcript -Path logs\07-ssh.txt

ssh-keygen -t ed25519 -C "BOOK-laptop"
```

질문이 세 번 나온다. 전부 **Enter 만** 치면 된다.
1. `Enter file in which to save the key (C:\Users\msong/.ssh/id_ed25519):` → Enter (기본 위치)
2. `Enter passphrase (empty for no passphrase):` → Enter (비밀번호 없음. 실무에선 넣는 게 좋지만 실습이라 생략)
3. `Enter same passphrase again:` → Enter

```powershell
Get-ChildItem ~\.ssh            # id_ed25519 (개인키), id_ed25519.pub (공개키) 두 개가 생김
Get-Content ~\.ssh\id_ed25519.pub   # 공개키 내용. ssh-ed25519 AAAA... BOOK-laptop 한 줄
```

**개인키(`id_ed25519`)는 절대 cat 하지 않는다.** 녹화에 들어가면 안 된다.

---

## 2. GitHub 에 공개키 등록

```powershell
gh auth refresh -h github.com -s admin:public_key
```
브라우저가 열리고 8자리 코드를 넣으라고 한다. 터미널에 보이는 코드를 입력하고 승인. (키 등록 권한을 한 번 더 받는 것)

```powershell
gh ssh-key add $HOME\.ssh\id_ed25519.pub --title "BOOK-laptop"    # ~ 는 외부 프로그램(gh)에 안 통한다. $HOME 사용
gh ssh-key list                 # BOOK-laptop 이 보이면 등록 완료
```

---

## 3. 동작 확인

```powershell
ssh -T git@github.com
```
처음 한 번은 `Are you sure you want to continue connecting (yes/no/[fingerprint])?` 라고 묻는다. `yes` 입력.
그 뒤 이렇게 나오면 성공:
```
Hi Frost0313z! You've successfully authenticated, but GitHub does not provide shell access.
```
"does not provide shell access" 는 에러가 아니다. "로그인은 됐는데 터미널은 안 열어 준다"는 뜻.

---

## 4. 저장소 연결을 SSH 로 바꾸기

```powershell
git remote -v                   # 변경 전: https://github.com/Frost0313z/dev-workstation-setup.git
git remote set-url origin git@github.com:Frost0313z/dev-workstation-setup.git
git remote -v                   # 변경 후: git@github.com:...
git fetch origin                # SSH 로 실제 통신. 에러 없이 끝나면 성공
Stop-Transcript
```

푸시는 나중에 커밋할 때 SSH 로 나간다. 그게 최종 증거가 된다.

---

## 자주 나는 실수

| 증상 | 원인 | 해결 |
|---|---|---|
| `Permission denied (publickey)` | 공개키가 GitHub 에 안 올라감 | `gh ssh-key list` 로 확인, 없으면 2절 다시 |
| `gh ssh-key add` 가 404 / scope 에러 | 권한 갱신 안 됨 | `gh auth refresh -h github.com -s admin:public_key` 먼저 |
| `ssh-keygen` 이 "already exists" | 키가 이미 있음 | 덮어쓰지 말고 `Get-ChildItem ~\.ssh` 로 확인 후 그대로 사용 |
| `Could not resolve hostname` | 네트워크 | 인터넷 확인 |
