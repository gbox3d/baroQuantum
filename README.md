# baroQuantum — CCTV PTZ 시뮬레이터 심플 프레임워크 (UE 5.8)

주차장 CCTV PTZ 카메라를 언리얼 안에서 **실기(Hucoms)처럼 행세**시키는 시뮬레이터를,
빈 프로젝트 + 간단한 데모 레벨로 보여주는 **경량 시연/시작용 프레임워크**.
`baro_calory`(Node/Python AI 에이전트)가 실기 IP 대신 이 시뮬레이터에 붙어 동일 프로토콜로 동작한다.

핵심 C++ 는 [`baroCCTVSimulator`](Plugins/baroCCTVSimulator) 플러그인에 있고, 이 프로젝트는 그것을
**git submodule 로 소비**한다. 무거운 실사 개발은 `baro_unreal`(별도 30GB 프로젝트)에서 하고,
**팀 공유·데모·신규 합류자 온보딩**은 이 baroQuantum 을 쓴다.

## 목차

- [무엇인가](#무엇인가)
- [저장소 구조](#저장소-구조)
- [요구 사항](#요구-사항)
- [처음 셋업 (클론 → 빌드)](#처음-셋업-클론--빌드)
- [실행법](#실행법)
  - [A. 에디터 PIE (정상 화면)](#a-에디터-pie-정상-화면)
  - [B. Standalone (검은 화면 — 의도됨)](#b-standalone-검은-화면--의도됨)
  - [C. 동작 확인 (curl)](#c-동작-확인-curl)
- [데모 레벨과 포트](#데모-레벨과-포트)
- [설정 (config)](#설정-config)
- [왜 standalone 은 검은 화면인가](#왜-standalone-은-검은-화면인가)
- [플러그인 업데이트 (submodule 동기화)](#플러그인-업데이트-submodule-동기화)
- [풀빌드 (전체 재빌드)](#풀빌드-전체-재빌드)
- [트러블슈팅](#트러블슈팅)

## 무엇인가

- **인엔진 Hucoms HTTP CGI 서버**: 레벨의 각 PTZ 카메라를 자기 포트에 독립 서버로 노출.
  `ptzf_status.cgi` / `ptz_centering.cgi` / `capabilityptz.cgi` / `jpeg.cgi` / `mjpeg.cgi`.
- **연속 MJPEG 스트리밍**: TCP multipart 스트림(MediaMTX/ffmpeg 로 RTSP 재송출 가능).
- **PTZ 짐벌 + 캡처**: Pan/Tilt/Zoom 카메라 + SceneCapture→JPEG (메인 뷰포트 렌더와 독립).
- **standalone 카메라 서버 게임모드**: `ABaroSimGameMode` (헤드리스풍 실행 — 아래 참조).

> 프로토콜/플러그인 내부 상세는 [`Plugins/baroCCTVSimulator/README.md`](Plugins/baroCCTVSimulator/README.md) 참조.

## 저장소 구조

```
baroQuantum/
├─ baroQuantum.uproject        # UE 5.8, baroCCTVSimulator 플러그인 활성
├─ Source/baroQuantum/         # 게임 부트 모듈만 (실 로직은 플러그인)
├─ Content/
│  ├─ simple_sample/simpleMap  # 데모 레벨 (PTZ 카메라 2대)
│  └─ simulator/               # 시뮬 관련 에셋
├─ Config/
│  ├─ DefaultEngine.ini        # GlobalDefaultGameMode + CoreRedirects
│  └─ DefaultGame.ini          # Hucoms 서버 설정(포트/fps/캡처)
├─ Plugins/
│  └─ baroCCTVSimulator/       # ← git submodule (단일 소스)
└─ _forAI/                     # AI 작업 인수용 문서 세트
```


## 요구 사항

- **Unreal Engine 5.8**
- **Visual Studio 2022** (C++, ".NET desktop" + "Game development with C++" 워크로드)
- **Git** (submodule 지원)

## 처음 셋업 (클론 → 빌드)

```bash
# 1) submodule 까지 함께 클론
git clone --recurse-submodules <baroQuantum-repo-url>
cd baroQuantum

# 이미 --recurse 없이 클론했다면:
git submodule update --init --recursive
```

> **submodule 이 비어 있으면** (`Plugins/baroCCTVSimulator` 가 빈 폴더) 위 `update --init` 을 반드시 실행.
> 플러그인 없이는 빌드/실행이 안 된다.

```
2) baroQuantum.uproject 우클릭 → "Generate Visual Studio project files"
3) baroQuantum.sln 열고 Development Editor / Win64 로 빌드 (또는 .uproject 더블클릭 시 빌드 프롬프트 → 예)
4) 에디터가 simpleMap 을 열며 시작
```

> `Binaries/ Intermediate/ Saved/ DerivedDataCache/` 및 `.sln`/`.vs` 는 git 에서 제외된다(재생성됨).
> 클론 직후 이 폴더들이 없는 게 정상이다.

## 실행법

게임모드가 프로젝트 전역(`GlobalDefaultGameMode`)으로 붙어 있어 **레벨에 별도 설정 없이** 서버가 뜬다.
PIE 와 standalone 의 화면이 다른데, **의도된 설계**다.

### A. 에디터 PIE (정상 화면)

- 에디터 툴바 **▶ Play**.
- 메인 뷰포트 **정상 렌더**, Hucoms/ MJPEG 서버 기동.
- 개발·디버깅·씬 확인용으로 이걸 쓴다.

### B. Standalone (검은 화면 — 의도됨)

- 에디터 툴바 Play 옆 **▾ → "Standalone Game"** (엔진 경로 몰라도 됨).
- 또는 CLI: `"<UE5.8>\Engine\Binaries\Win64\UnrealEditor.exe" "<path>\baroQuantum.uproject" -game`
- 메인 창은 **검은 화면**(헤드리스 카메라 서버 목적) — [아래 설명](#왜-standalone-은-검은-화면인가).
  검은 것은 메인 창뿐이고 **CCTV 프레임(jpeg/mjpeg)은 정상 출력**된다.
- **ESC** 로 종료.

### C. 동작 확인 (curl)

PIE 든 standalone 이든 서버가 뜨면 데모 레벨 기준 아래로 확인:

```bash
# 카메라 1 — PTZ 위치
curl "http://127.0.0.1:8081/cgi-bin/control/ptzf_status.cgi?action=getptzfpos"

# 카메라 1 — 실렌더 스냅샷
curl "http://127.0.0.1:8081/cgi-bin/image/jpeg.cgi" -o cam1.jpg

# 카메라 2
curl "http://127.0.0.1:8082/cgi-bin/image/jpeg.cgi" -o cam2.jpg
```

연속 MJPEG 스트림은 `8091` / `8092` (TCP, RTSP 브리지 입력).

## 데모 레벨과 포트

데모 레벨 `Content/simple_sample/simpleMap` 에는 PTZ 카메라 **2대**가 배치돼 있다.
두 카메라 모두 포트를 **0(자동)** 으로 두어, config 의 `BaseHttpPort`/`BaseMjpegPort` 에서
카메라 인덱스를 더한 값으로 배정된다:

| 카메라 | HTTP(CGI) | MJPEG(스트림) |
|---|---|---|
| 카메라 1 (index 0) | **8081** | 8091 |
| 카메라 2 (index 1) | **8082** | 8092 |

> 포트 규약: 카메라의 `HucomsHttpPort`/`HucomsMjpegPort` 가 `>0` 이면 그 값, `0` 이면
> `BaseHttpPort`/`BaseMjpegPort + 카메라 인덱스` 자동. 특정 포트를 고정하고 싶을 때만 카메라에
> 값을 명시하고, 평소엔 0(자동)으로 둔다. `baro_calory` 의 `devices.list[].port` 와 1:1 로 맞출 것.

## 설정 (config)

`Config/DefaultGame.ini` — Hucoms 서버 기본값(모듈명이 섹션 경로에 들어감):

```ini
[/Script/baroCCTVSimulator.HucomsServerSubsystem]
BaseHttpPort=8081       ; 카메라가 포트를 0으로 두면 이 값+인덱스로 자동배정
BaseMjpegPort=8091
StreamFps=30            ; 연속 MJPEG 목표 fps
StreamWidth=1280
StreamHeight=720
CaptureContrast=1.2     ; 캡처 대비(하이키 클리핑 회피값)
CaptureExposureBias=-0.7
```

`Config/DefaultEngine.ini` — 게임모드 + CoreRedirects(구 `BaroSim` 경로 참조 호환):

```ini
[/Script/EngineSettings.GameMapsSettings]
GameDefaultMap=/Game/simple_sample/simpleMap
GlobalDefaultGameMode=/Script/baroCCTVSimulator.BaroSimGameMode
```

## 왜 standalone 은 검은 화면인가

`ABaroSimGameMode` 는 "CCTV 카메라 서버"가 목적이라, **standalone(`-game`)에서만** 메인 뷰포트
렌더를 끈다(`bDisableWorldRendering=true`). PIE 는 건드리지 않는다.

| 실행 | WorldType | 메인 창 |
|---|---|---|
| PIE (에디터 ▶) | `PIE` | **정상 렌더** |
| Standalone (`-game`) | `Game` | **검은 화면** (헤드리스) |

- **CCTV 프레임은 검은 화면과 무관하게 정상 출력**된다 — SceneCapture 가 메인 뷰포트와 별개로 자체 렌더.
- **부수효과**: standalone 은 메인 월드 렌더가 꺼져 Lumen GI 가 캡처에 덜 반영될 수 있다(면이 어둡게).
  조명이 중요한 캡처는 PIE 를 쓰거나 라이팅을 베이크한다.
- 자세한 배경/우선순위는 [플러그인 README](Plugins/baroCCTVSimulator/README.md#게임모드와-실행-모드-pie-vs-standalone).

## 플러그인 업데이트 (submodule 동기화)

CCTV 시뮬 C++ 는 `baroCCTVSimulator` 리포에서만 고친다. 이 프로젝트는 그 플러그인의 **특정 커밋 하나에
핀(pin) 고정**된 상태로 소비한다(서브모듈의 gitlink). 그래서 플러그인을 버전업해도 baroQuantum 이
**자동으로 최신을 따라가지 않는다** — 중간에 "핀을 올리는" 명시적 단계가 필요하다.

> **핵심**: `git pull` 한 방으로는 최신 플러그인이 안 붙는다. 상황을 A/B 로 나눠서 본다.

### 상황 A — 내가 플러그인을 버전업하고 최신으로 올릴 때

플러그인 쪽에서 먼저 커밋·push:

```bash
cd ../baroCCTVSimulator
# ...C++/에셋 수정...
git commit -am "feat: 개선"
git push
```

그다음 baroQuantum 에서 **핀을 최신으로 올리고 그 사실을 커밋**한다:

```bash
cd ../baroQuantum
git submodule update --remote Plugins/baroCCTVSimulator   # 플러그인 origin/main 최신을 받아 체크아웃
git add Plugins/baroCCTVSimulator
git commit -m "chore: baroCCTVSimulator 최신으로 갱신"
git push
```

> `--remote` 가 핵심. "핀 무시하고 브랜치(main) 최신으로 당겨라"는 뜻이다. 이 커밋이 새 핀을 기록한다.

### 상황 B — 남(또는 다른 PC)이 A 를 해서 올린 핀을 받을 때

`pull` + **서브모듈 체크아웃** 2단계다:

```bash
git pull                                  # baroQuantum 의 새 핀(포인터)을 받음
git submodule update --init --recursive   # 플러그인을 그 핀 커밋으로 체크아웃 ← 빠지면 옛 파일 그대로
```

### pull 한 방으로 끝내고 싶으면 (편의 설정)

한 번만 설정하면 이후 `pull`/`checkout` 이 서브모듈까지 자동으로 따라온다:

```bash
git config submodule.recurse true         # 이후 pull/checkout 이 서브모듈 자동 update
# 또는 매번:  git pull --recurse-submodules
```

단, 이건 **상황 B**(올라간 핀 받기)만 자동화한다. **상황 A 의 "핀 올리기+커밋"은 여전히 수동**이다
— 어느 플러그인 버전에 고정할지는 프로젝트가 결정하는 git 의 의도된 명시적 단계다.

### ⚠️ 받은 뒤 — UE 리빌드

플러그인 **C++ 가 바뀐 버전**을 받았으면 파일 갱신만으로 끝이 아니다. **에디터를 닫고** 프로젝트를
리빌드해야 반영된다(Live Coding 으로는 모듈 핫스왑 불가). 에셋/콘텐츠만 바뀐 버전이면 리빌드 없이
에디터 재시작만으로 된다. → 절차는 아래 [풀빌드 (전체 재빌드)](#풀빌드-전체-재빌드).

## 풀빌드 (전체 재빌드)

**언제 필요한가**: ① 처음 셋업 때, ② 플러그인 C++ 를 새로 받았을 때, ③ `.uproject` 열 때 "모듈이
오래됨" 프롬프트가 뜰 때.

> **에디터 안의 컴파일 버튼(Live Coding)으로 때우지 말 것.** Live Coding 은 함수 본문 정도만 반영하고
> **헤더/구조 변경(UPROPERTY 추가, 새 클래스, WorldSubsystem 등)은 놓친다** — "성공"이라 떠도 옛 코드가
> 돌 수 있다. 플러그인 버전업 뒤엔 반드시 아래 풀빌드로 간다.

### 0단계 (필수) — 에디터를 완전히 닫는다

에디터가 열려 있으면 DLL 이 잠겨 빌드가 실패하거나 반쪽만 적용된다. **먼저 언리얼 에디터를 종료**한다.

### 1단계 (파일이 추가/삭제된 버전일 때만) — 프로젝트 파일 재생성

플러그인에 새 `.cpp/.h` 가 생겼거나 지워졌으면, 빌드 전에 한 번:

- `baroQuantum.uproject` **우클릭 → "Generate Visual Studio project files"**

(기존 파일 내용만 바뀐 버전이면 이 단계는 건너뛴다.)

### 2단계 — 셋 중 하나로 빌드

**방법 A. Visual Studio (가장 무난)**

1. `baroQuantum.sln` 을 연다 (`Automation_...` 말고)
2. 상단 구성 드롭다운을 **`Development Editor`** / **`Win64`** 로 맞춘다
3. **Build → Build Solution** (`Ctrl+Shift+B`)
   - 더 빠르게: 솔루션 탐색기에서 **`baroQuantum` 프로젝트 우클릭 → Build**

**방법 B. .uproject 리빌드 프롬프트 (가장 간단)**

- `baroQuantum.uproject` 더블클릭 → "모듈이 오래됨, 다시 빌드?" → **예**
- 빌드가 끝나면 에디터가 자동으로 열린다

**방법 C. 명령줄 (VS 없이 / 자동화용)**

```bat
"<UE_5.8 설치 경로>\Engine\Build\BatchFiles\Build.bat" baroQuantumEditor Win64 Development -Project="baroQuantum.uproject" -WaitMutex
```

- `baroQuantumEditor` = 빌드 타깃, `Win64` = 플랫폼, `Development` = 구성
- 엔진 설치 경로는 각자 환경에 맞게 

### 증상이 이상하면 — 완전 클린 빌드

빌드가 꼬였을 때만: 에디터 닫고 프로젝트의 **`Binaries/` 와 `Intermediate/` 폴더를 삭제** → 1단계(재생성)
→ 2단계(빌드). 둘 다 git 에서 제외돼 재생성되므로 지워도 안전하다.

## 트러블슈팅

- **`Plugins/baroCCTVSimulator` 가 비어 있음** → `git submodule update --init --recursive`.
- **`.uproject` 열 때 모듈 리빌드 물음** → "예". 엔진 버전(5.8) 불일치면 우클릭 → Switch Unreal Engine Version.
- **curl 이 응답 없음** → 서버는 Play/standalone 실행 중에만 뜬다. 포트는 기본 8081/8082(자동) 인지 확인.
- **standalone 이 검은 화면** → 정상(설계). CCTV 프레임은 curl 로 확인.
- **레벨의 PTZ 카메라가 안 붙음 / 참조 깨짐** → `DefaultEngine.ini` 의 `[CoreRedirects]` 블록 존재 확인
  (구 `BaroSim` 경로 → `baroCCTVSimulator` 매핑).
