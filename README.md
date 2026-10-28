# HANOK3D GO 출시 패키지

웹대국·친구초대 창의 **333 · 355 · 277** 버튼, 세 판을 모두 판정하는 대국 서버, 구글 플레이(AAB)·PC 설치판·포터블 빌드 설정을 한데 모았습니다.

## 폴더 구성

| 경로 | 내용 |
|---|---|
| `hanok.config.json` | **여기 한 곳만 고치면 됩니다** — 서버 주소, 패키지 이름, 버전 |
| `game/HANOK3D_GO.html` | 수정한 게임 원본 (서버 주소 자리표시자 `__HANOK_SERVER_URL__` 포함) |
| `build/build.js` | 서버 주소를 게임에 넣어 안드로이드·PC·서버 세 곳에 복사, 규칙 엔진 추출 |
| `server/` | 대국 서버 v2 (`server.js`, `rules.js`, `test.js`, `privacy.html`, `public/sw.js`) |
| `android/` | Capacitor 8 안드로이드 프로젝트 (compile/target SDK 36), `configure.js` |
| `desktop/` | Electron PC판 (`main.js`, `preload.js`) |
| `web/` | 소개 페이지 `index.html`(영상·3D 체험 포함, 자체 완결형), 원본 `index.template.html`, 조립 스크립트 `build_web.py` |
| `.github/workflows/release.yml` | GitHub에서 AAB와 윈도우 설치판을 자동으로 만드는 설정 |

## 바뀐 점

1. **게임 종류 버튼**: 웹대국 탭과 친구초대 탭에 333·355·277 버튼. 누르면 그 판으로 바로 매칭하거나 초대 링크를 보냅니다. 판마다 기다리는 인원이 표시됩니다.
2. **세 판 온라인 지원**: 원래는 3×3×3만 온라인이 가능했습니다(정육면체 판만 허용). 이제 층 수(`l`)를 함께 보내고 서버가 세 판을 모두 판정합니다.
3. **옛 서버 감지**: 옛 server.js에 연결되면 355·277 버튼이 "서버 업데이트 필요"로 잠깁니다.
4. **서버 자동 지정**: 앱(플레이스토어·PC 설치판·포터블)은 `hanok.config.json`의 서버로 자동 연결되고 서버 설정 칸이 숨겨집니다. 웹은 페이지를 보여 준 서버를 그대로 씁니다.
5. **버그 수정**: 초대 링크로 들어온 사람의 판이 3×3×3으로 덮어써지던 문제를 고쳤습니다(화면 준비 전에 대국이 시작되던 경쟁 상태).

## 출시 순서

### 1단계 — 서버 주소 정하기

`hanok.config.json`을 엽니다.

```json
{
  "appName": "HANOK3D GO",
  "appId": "com.hanok3d.go",
  "version": "1.0.0",
  "versionCode": 1,
  "server": "https://내-서버-주소",
  "assetlinksSha256": []
}
```

- `server`: 반드시 `https://`. 앱은 보안 주소에만 연결됩니다.
- `appId`: 플레이 콘솔에 **이미 등록한 앱이 있으면 그 패키지 이름과 똑같이** 적어야 업데이트로 올라갑니다.

```bash
node build/build.js
```

### 2단계 — 대국 서버 배포

Node.js 18 이상이 도는 곳이면 어디든 됩니다(Render, Railway, Fly.io, 클라우드 VM 등). 웹소켓을 지원해야 합니다.

```bash
cd server
npm install
npm test        # 333·355·277 매칭·초대·착수 검증 시험
npm start       # PORT 환경 변수 (기본 3000)
```

- 배포 후 `https://서버/health`에 `"v":2`와 판 3종이 나오면 성공입니다.
- `https://서버/`로 들어가면 웹판 게임이 열립니다(웹대국도 이 주소로 자동 연결).
- 레이팅은 `server/data/ratings.json`에 저장됩니다. 서버를 다시 만들 때 지워지지 않게 하려면 `DATA_DIR`을 영구 디스크로 지정하세요.

### 3단계 — 구글 플레이 AAB

AAB는 업로드 키로 서명해야 하고, 빌드에 Android SDK·Gradle이 필요합니다. 두 가지 방법이 있습니다.

**방법 A: GitHub Actions (컴퓨터에 아무것도 설치하지 않음)**

1. 업로드 키를 한 번 만듭니다(JDK의 keytool). **이 파일과 비밀번호를 잃어버리면 앱을 업데이트하기 어려워지니 안전하게 보관하세요.**
   ```bash
   keytool -genkeypair -v -keystore upload.jks -alias upload -keyalg RSA -keysize 2048 -validity 10000
   base64 -w0 upload.jks > upload.b64     # 윈도우: certutil -encode upload.jks upload.b64
   ```
2. 이 폴더 전체를 GitHub **비공개** 저장소에 올립니다.
3. 저장소 Settings › Secrets and variables › Actions에 등록합니다.
   - `HANOK_SERVER` — 서버 주소
   - `ANDROID_KEYSTORE_BASE64` — upload.b64 내용
   - `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`(upload), `ANDROID_KEY_PASSWORD`
4. Actions 탭 › release › Run workflow. 끝나면 Artifacts에서 `HANOK3D_GO_android_aab`(AAB)와 `HANOK3D_GO_windows`(설치판·포터블)를 받습니다.
   - versionCode는 실행 번호로 자동 증가합니다.

**방법 B: 내 컴퓨터 (Android Studio)**

```bash
node build/build.js
cd android
npm install
node configure.js
npx cap sync android
npx cap open android      # Android Studio › Build › Generate Signed App Bundle
```

### 4단계 — 초대 링크로 앱 바로 열기 (선택)

1. 플레이 콘솔 › 테스트 및 출시 › 앱 무결성 › 앱 서명 키 인증서의 SHA-256을 복사합니다.
2. `hanok.config.json`의 `assetlinksSha256`에 넣습니다.
3. `node build/build.js` 실행 후 서버를 재배포하고, AAB를 다시 빌드합니다.

### 5단계 — PC판

`desktop/dist/`(또는 GitHub Actions 결과)에 두 파일이 생깁니다.

- `HANOK3D_GO_Setup_1.0.0.exe` — 설치판 (설치 위치 선택, 바탕화면·시작 메뉴 바로가기)
- `HANOK3D_GO_Portable_1.0.0.exe` — 설치 없이 바로 실행

```bash
node build/build.js
cd desktop && npm install && npm run dist:win
```

## 함께 드린 실행 파일에 대해

이 패키지와 함께 드린 PC 설치판·포터블은 **서버 주소가 비어 있는 상태로 빌드한 시험판**입니다. 실행하면 웹대국 창에 "서버 설정" 칸이 보이며, 주소를 직접 넣으면 연결됩니다. 정식 배포판은 서버 주소를 정한 뒤 위 5단계로 다시 빌드하세요.

- 코드 서명이 없어 윈도우에서 "Windows의 PC 보호" 경고가 뜰 수 있습니다(추가 정보 › 실행). 경고를 없애려면 코드 서명 인증서가 필요합니다.
- macOS·리눅스판은 `npm run dist:mac` / `npm run dist:linux`로 만들 수 있습니다(맥판은 맥에서 빌드).

## 확인한 것 / 직접 확인이 필요한 것

**이 환경에서 확인함**
- 서버 시험 63개 항목 통과: 세 판 × (랜덤 매칭·초대) × (착수 전달·불법 수 거부·판 밖 착수 거부·재접속·기권), 옛 앱 안내, 초대 링크 페이지
- 실제 브라우저 두 개로 333·355·277 웹대국 연결과 2×7×7 초대 링크 입장
- 윈도우 실행 파일 생성(PE 형식 확인)

**직접 확인이 필요함**
- AAB 빌드 자체: 이 작업 환경에서는 Gradle·구글 Maven 접속이 막혀 있어 빌드하지 못했습니다. GitHub Actions 또는 Android Studio에서 처음 빌드할 때 오류가 나면 로그를 보내 주세요.
- 실제 안드로이드 기기와 윈도우 PC에서의 실행
- Android 16 이상: 화면이 시스템 막대 아래까지 그려지는(edge-to-edge) 동작. 게임 CSS에 안전 영역 처리가 있지만 기기에서 위·아래가 가려지지 않는지 확인하세요.

## 웹사이트 (소개 페이지 + 게임)

서버를 배포하면 주소만으로 열립니다.

| 주소 | 열리는 페이지 |
|---|---|
| `https://서버/` | 소개 페이지 (web/index.html) |
| `https://서버/game.html` | 게임 |
| `https://서버/j/1234` | 초대 링크 → 게임이 열리고 바로 입장 |

`node build/build.js`가 `web/index.html`을 `server/public/index.html`로, 게임을 `server/public/game.html`로 복사합니다.
소개 페이지의 구글 플레이 주소와 PC판 내려받기 주소는 `web/index.html` 아래쪽 `CONFIG`에서 넣으면 "준비 중" 표시가 단추로 바뀝니다.
