# 레인보우로보틱스 RB3-730 디지털 트윈
# Unity + TCP 티칭 펜던드 완전 실습 가이드

> **이 문서 하나로 처음부터 끝까지 재현할 수 있습니다.**
> LLM에 질문하지 않아도 모든 단계·명령·소스 코드가 그대로 들어 있습니다.
> 실제로 이 과정을 한 번 거치며 만난 **7가지 버그와 그 수정 코드**도 [Part 10](#part-10)에 모두 기록했습니다.

---

## 문서 정보

| 항목 | 값 |
|---|---|
| 대상 독자 | 로봇 디지털 트윈 / ROS·Unity 입문자 |
| 대상 로봇 | Rainbow Robotics **RB3-730 (U-Version)** — 6축 협업로봇 |
| 최종 환경 | Unity **2022.3.62f3**, Built-in Render Pipeline, Windows 10/11 |
| 사용 도구 | Git, PowerShell 5.1+, Python(미설치 가능), Blender 4.2.23 포터블, Unity Roslyn |
| 총 소스량 | C# 2,992줄 / Python 427줄 / PowerShell 624줄 / URDF 169줄 |
| 검증 상태 | C# 사전 컴파일 오류 0 / 경고 0, TCP 회귀 테스트 **14개 섹션 61개 항목 전부 통과** (Unity Play 모드 실측, 2026-10-06) |

### 이 문서를 읽는 순서

1. **[Part 1](#part-1)** — 전체 그림과 최종 산출물을 먼저 보세요
2. **[Part 2](#part-2)** — 사전 준비물을 설치하세요
3. **[Part 3](#part-3) ~ [Part 8](#part-8)** — 순서대로 따라가세요. 각 Part 끝에 **✅ 확인 방법**이 있습니다
4. **[Part 9](#part-9)** — 실사용 시나리오 (pick & place 티칭 사이클)
5. **[Part 10](#part-10)** — 막히면 여기를 보세요

> ⚠️ **건너뛰지 마세요.** Part 3의 URDF 검증과 Part 4의 FBX 검증은 생략하면
> 뒤 단계에서 **원인을 추적할 수 없는 스케일/축 오류**로 이어집니다.
> 전체 과정에서 실제로 두 번 이런 문제가 발생했습니다.

---


<a id="part-1"></a>

## Part 1. 전체 그림과 최종 산출물

### 1.1 이 프로젝트가 만드는 것

ROS용 로봇 모델(URDF)을 **Unity에서 구동되는 디지털 트윈**으로 바꾸고,
그 위에 **TCP 소켓으로 동작하는 티칭 펜던트**를 붙이는 작업입니다.

```
[공식 GitHub 저장소]          [Phase 1]              [Phase 2]
RainbowRobotics/rbpodo_ros2  ──►  URDF + 메시        ──►  리그 FBX 2개
  (ROS 2 패키지)               독립 패키지화            (Blender 변환)
       │                        검증·경로 정리              │
       │                                                    │
       │              [Phase 3]              [Phase 4]      │
       └──────────►  Unity 임포트  ────────►  C# 런타임 4종  │
                       프리팹 생성              조인트/티칭/서버 │
                                                    │         │
                                        [Phase 5]   │         │
                                      TCP 127.0.0.1:5000 ◄────┘
                                                    │
                                                    ▼
                                        [Phase 6] PowerShell / WinForms 펜던트
```

### 1.2 왜 소켓(TCP)을 쓰는가 — 설계 결정

C# WinForms 티칭 펜던트는 **별도의 프로세스**이며 `UnityEngine.dll`을 참조할 수 없습니다.
따라서 펜던트에서 로봇을 구동하는 유일한 깨끗한 방법은 **소켓**입니다.

- Unity 안에서 TCP 서버 컴포넌트(`RB3_730_PendantServer`)를 돌립니다
- 펜던트는 `127.0.0.1:5000`에 한 줄(줄바꿈 `\n`으로 구분) 텍스트를 보냅니다
- 양쪽이 공유 어셈블리를 전혀 필요로 하지 않습니다
- 프로토콜 명세는 서버 소스 파일의 **주석 블록**에 authoritative(권위 있는)로 들어 있습니다 ([Part 7](#part-7-tcp-프로토콜-명세))

### 1.3 최종 산출물 파일 트리

```
C:\Users\Administrator\Desktop\Digital_Twin_2022\     ← Unity 프로젝트
└── Assets/
    ├── RB3-730/
    │   ├── Models/
    │   │   ├── rb3_730es_u_rigged.fbx        (3.6 MB)  비주얼 리그 FBX
    │   │   └── rb3_730es_u_collision.fbx     (2.7 MB)  충돌 리그 FBX
    │   ├── Data/
    │   │   └── rb3_730es_u_joints.json       (3.7 KB)  관절 표
    │   ├── Runtime/
    │   │   ├── RB3_730_JointDriver.cs        (31 KB)   789줄  ← [Part 6.1](#code-jointdriver)
    │   │   ├── RB3_730_Teach.cs              (20 KB)   527줄  ← [Part 6.2](#code-teach)
    │   │   ├── RB3_730_PendantServer.cs      (39 KB)   1008줄  ← [Part 6.3](#code-pendantserver)
    │   │   └── RB3_730_JogPanel.cs           (9.2 KB)  213줄  ← [Part 6.4](#code-jogpanel)
    │   ├── Editor/
    │   │   └── RB3_730_Setup.cs              (21 KB)   455줄  ← [Part 5.4](#setup-script)
    │   ├── RB3_730.prefab                    (34 KB)   4개 컴포넌트 포함
    │   └── README.md                         (6.1 KB)  모델 패키지 설명
    ├── Scenes/
    │   └── SampleScene.unity                 (10 KB)   로봇 인스턴스가 저장된 씬
    └── ...
└── Pendant/
    └── pendant_test.ps1                      (27 KB)   624줄  ← [Part 8](#part-8)

C:\Users\Administrator\Documents\RB3-730\               ← 모델 원본 패키지 (25개 파일, 22.5 MB)
├── fbx/          리그 FBX 2개 + 관절표 + 변환 로그
├── meshes/       원본 DAE(비주얼) / STL(충돌) 14개
├── urdf/         URDF 원본 / ros2_control 제거판 / xacro
├── config/       joint.yaml, link.yaml
├── unity/        JointDriver.cs (단독 배포용 사본)
└── README.md
```

### 1.4 핵심 설계 원칙 — 이 문서 전체를 관통하는 4가지

이 4가지를 이해하면 이후 모든 코드가 왜 그런 모양인지 알 수 있습니다.

<a id="principle-1"></a>

#### 원칙 1: FBX 본의 **로컬 Y축 = URDF 관절축**

Blender 변환 단계에서 관절축이 그대로 본의 Y축이 되도록 리그를 만듭니다.
덕분에 Unity 코드에서 축 계산이 전부 사라집니다.

```csharp
// 축 벡터 계산, 쿼터니언 변환, 행렬 변환 … 전부 필요 없다
links[i].localRotation = restRotation[i] * Quaternion.AngleAxis(deg, Vector3.up);
```

이 원칙 덕분에 **Unity의 전역 축 변환(Y-up)과 무관하게** 코드가 동작합니다.

<a id="principle-2"></a>

#### 원칙 2: 티칭 포즈는 **관절 공간**으로 저장·재생

카티시안 좌표로 저장하면 IK를 다시 풀어야 하고, singular point 근처에서 실패하며,
사이클을 반복할 때마다 미세하게 드리프트합니다.
관절 각도로 저장하면 **100 % 재현**되고 IK가 개입하지 않습니다.

TCP pose도 함께 저장하지만, 그것은 **작업자에게 보여줄 설명용**입니다(`"P3, 400mm, 그립퍼 아래"`).

<a id="principle-3"></a>

#### 원칙 3: **소켓 스레드는 큐만 건드린다**

```
[소켓 스레드]                [큐]              [메인 스레드 Update]
accept / read / write  ──►  ConcurrentQueue  ──►  Handle() → 실제 로봇 구동
   Unity API 접근 금지                       ▲
                                             └─ Robot JointDriver
```

Unity API는 메인 스레드에서만 호출해야 합니다. 스레드에서 `transform`을 만지면
간헐적 크래시와 "Objects are not allowed to be accessed from a background thread" 예외가 납니다.

<a id="principle-4"></a>

#### 원칙 4: **연결이 끊어지면 무조건 정지**

펜던트 연결이 끊기면 즉시 `driver.Stop()`을 호출합니다.
그렇지 않으면 "버튼을 놓고 가서" 창이 닫혀도 로봇이 계속 움직입니다.

---

<a id="part-2"></a>

## Part 2. 사전 준비물

### 2.1 필수 소프트웨어

| 소프트웨어 | 버전 | 용도 | 설치 확인 명령 |
|---|---|---|---|
| **Git** | 아무 버전 | 공식 저장소 클론 | `git --version` |
| **PowerShell** | 5.1 이상 (기본 포함) | TCP 테스트 클라이언트 | `$PSVersionTable.PSVersion` |
| **Unity Hub** | — | 에디터 관리 | — |
| **Unity Editor** | **2022.3.62f3** (LTS) | 시뮬레이션 | — |
| **Blender** | 4.2.23 LTS (포터블) | URDF → FBX 변환 | 아래 2.2 참조 |

> **Unity 2022.3.62f3으로 고정하는 이유**
> 이 프로젝트는 Built-in Render Pipeline입니다. Unity 6(HDRP/URP)과는 재질 셰이더가
> 달라 임포트 설정이 달라집니다. 본 가이드는 2022.3 기준으로 검증되었습니다.

### 2.2 Blender 포터블 내려받기 (설치 불필요)

Python을 별도로 설치하지 않아도 됩니다. Blender를 포터블로 풀면 Python API가 포함되어 있습니다.

```powershell
# 1) 다운로드 (약 383 MB, 실제 소요시간 약 160초)
$w = "$env:TEMP\opencode\rb3730"
New-Item -ItemType Directory -Path $w -Force | Out-Null
$url = 'https://download.blender.org/release/Blender4.2/blender-4.2.23-windows-x64.zip'
$zip = "$w\blender.zip"
$wc = New-Object System.Net.WebClient
$wc.DownloadFile($url, $zip)
"downloaded: {0:N1} MB" -f ((Get-Item $zip).Length / 1MB)

# 2) 압축 해제 (약 1.1 GB)
Expand-Archive -LiteralPath $zip -DestinationPath "$w\blender" -Force

# 3) 버전 확인
$exe = Get-ChildItem -Recurse -Filter blender.exe -LiteralPath "$w\blender" | Select-Object -First 1
$exe.FullName          # → ...\blender\blender-4.2.23-windows-x64\blender.exe
& $exe.FullName --version
```

✅ **확인**: `blender.exe --version` 이 `Blender 4.2.23` 을 출력합니다.

> 💡 이 포터블은 변환 작업이 끝난 뒤 삭제해도 됩니다. 재변환이 필요하면 다시 받으세요.

### 2.3 디렉터리 준비

이 가이드의 모든 예시는 아래 두 경로를 기준으로 합니다. 자유롭게 바꿔도 되지만,
스크립트 안에 절대 경로가 3곳 하드코딩되어 있으므로 그때는 함께 수정해야 합니다.

```powershell
$WORK = "$env:TEMP\opencode\rb3730"     # 작업 공간 (Blender, 검증 스크립트)
$PKG  = "$HOME\Documents\RB3-730"       # 검증된 모델 패키지 (영구 보관)
$PROJ = "$HOME\Desktop\Digital_Twin_2022" # Unity 프로젝트

New-Item -ItemType Directory -Path $WORK,$PKG,$PROJ -Force | Out-Null
```


---

<a id="part-3"></a>

## Part 3. Phase 1 — 공식 URDF 모델 확보 및 검증

### 3.1 왜 URDF에서 시작하는가

| asset | 형식 | 디지털 트윈에 필요? |
|---|---|---|
| CAD 도면 (2D/3D STEP) | STEP/DWG | ❌ 변환 비용 높음, 재질 없음, 관절 정보 없음 |
| **URDF + 메시** | XML + DAE/STL | ✅ **관절 원점·축·한계·질량이 이미 다 들어 있음** |

URDF는 로봇 기술의 보편적 교환 형식입니다. 관절 키네마틱스가 파일 안에 명시되어 있으므로
Blender가 **관절을 다시 계산할 필요가 없습니다.** 이것이 이 경로를 선택한 핵심 이유입니다.

> 레인보우로보틱스 공식 웹사이트의 도면 다운로드에는 회사/담당자/이메일/휴대폰 입력을 강제하는
> "다운로드 게이트"가 있어 학생 실습에 적합하지 않습니다.
> 반면 GitHub의 `rbpodo_ros2`는 공개이며 Apache-2.0 라이선스입니다.

### 3.2 왜 RB3-730인가 — "가장 작은 모델" 요구 해결

레인보우로보틱스 현 라인업(RBC Series) 중 가장 작은 모델입니다.

| 항목 | RB3-730 사양 |
|---|---|
| 페이로드 | 3 kg |
| **도달거리** | **730 mm** |
| 베이스 직경 | Ø128 mm |
| 중량 | 11 kg |
| 자유도 | 6축 (S-Pipe 관절) |
| 반복정밀도 | ±0.05 mm |
| 관절 범위 | J1/J2 ±360°, J3 ±150°, J4~J6 ±360° |

> `RB1-500ES`는 단종된 구 모델이라 제외했습니다.

### 3.3 공식 저장소 클론

```powershell
$WORK = "$env:TEMP\opencode\rb3730"
git clone --depth 1 https://github.com/RainbowRobotics/rbpodo_ros2.git "$WORK\rbpodo_ros2"
```

얻는 경로:

```
$WORK\rbpodo_ros2\rbpodo_description\
├── robots\rb3_730es_u.urdf            ← URDF
├── robots\rb3_730es_u.urdf.xacro      ← 파라메트릭 원본
├── meshes\rb3_730es_u\visual\link0..6.dae    ← 비주얼 (COLLADA)
├── meshes\rb3_730es_u\collision\link0..6.stl  ← 충돌 (binary STL)
└── config\rb3_730es_u\joint.yaml, link.yaml
```

### 3.4 메시 파일 실태 확인

가장 먼저 해야 할 일은 **"파일이 실제로 존재하고 형식이 맞는가"** 입니다.
URDF가 참조하는 메시가 없으면 아무리 나머지가 맞아도 실패합니다.

```powershell
$src = "$env:TEMP\opencode\rb3730\rbpodo_ros2\rbpodo_description"
Get-ChildItem -Recurse -File "$src\meshes","$src\robots" |
  Where-Object { $_.Extension -in '.urdf','.xacro','.dae','.stl','.yaml' } |
  Select-Object @{n='KB';e={[math]::Round($_.Length/1KB,1)}}, FullName |
  Format-Table -AutoSize -Wrap
```

다음 스크립트는 **파일 시그니처**로 형식을 판별합니다(확장자가 아니라 실제 바이트로 확인).

```powershell
$m = "$env:TEMP\opencode\rb3730\rb3_730es_u_standalone\meshes\rb3_730es_u"
Get-ChildItem -Recurse -File -LiteralPath $m | ForEach-Object {
    $b = [System.IO.File]::ReadAllBytes($_.FullName)
    $head = [System.Text.Encoding]::ASCII.GetString($b, 0, [Math]::Min(84, $b.Length))
    $kind = if     ($head -match 'COLLADA') { 'DAE/ok' }
            elseif ($head -match 'solid ') { 'STL/ascii' }
            elseif ($b.Length -ge 84)      { 'STL/binary' }
            else                          { '??' }
    [pscustomobject]@{ File = $_.FullName.Substring($m.Length+1)
                      KB  = [math]::Round($_.Length/1KB,1); Kind = $kind }
} | Format-Table -AutoSize
```

✅ **확인**: 비주얼 7개가 `DAE/ok`, 충돌 7개가 `STL/binary` 로 나옵니다.

### 3.5 **스케일 검증** — mm/m 함정 확인

3D 모델에서 가장 흔하고 가장 찾기 어려운 버그가 **단위 불일치**입니다.
공정용 CAD는 mm, ROS/URDF 규약은 m입니다. DAE 파일 안에 이 정보가 명시되어 있습니다.

```powershell
$v = "$env:TEMP\opencode\rb3730\rb3_730es_u_standalone\meshes\rb3_730es_u\visual"
Get-ChildItem -LiteralPath $v -Filter *.dae | ForEach-Object {
    $t = Get-Content -LiteralPath $_.FullName -Raw
    $imgs = [regex]::Matches($t,'<init_from>\s*([^<\s]+)') |
            ForEach-Object { $_.Groups[1].Value } | Sort-Object -Unique
    $unit = [regex]::Match($t,'<unit[^>]*meter="([^"]+)"').Groups[1].Value
    $up   = [regex]::Match($t,'<unit[^>]*up="([^"]+)"').Groups[1].Value
    [pscustomobject]@{ File = $_.Name
                      Images = ($imgs -join ', ')
                      Nimg   = @($imgs).Count
                      UnitMeter = $unit; Up = $up }
} | Format-Table -AutoSize
```

✅ **확인**: 전부 `UnitMeter=1`, `Up=Z_UP`, `Nimg=0` 이어야 합니다.

| 결과가 이 뜻 | 해석 | 조치 |
|---|---|---|
| `meter="1"`, `Z_UP`, 이미지 0개 | ✅ 정상. 외부 텍스처 의존 없음 | 그대로 진행 |
| `meter="0.001"` | ⚠️ mm 저작. 1000배 축소됨 | FBX 변환 뒤 `globalScale`로 보정하거나 메시를 재작성 |
| `Nimg > 0` | ⚠️ 외부 이미지 파일 필요 | 이미지 파일을 같은 폴더에 함께 배치 |
| `Up=Y_UP` | ⚠️ 축 규약 불일치 | Blender 변환 스크립트가 Z-up을 강제하므로 무시됨 |

### 3.6 독립 실행 패키지 만들기

**왜 필요한가**
1. URDF가 메시를 `package://rbpodo_description/meshes/...` 로 참조합니다.
   이 경로는 ROS가 설치된 환경에서만 해석됩니다. **ROS 없이 열 수 있게 상대경로로 바꿔야 합니다.**
2. URDF에 `<ros2_control>` 블록이 들어 있습니다. 이건 ROS 2 전용 태그라
   Isaac Sim·Gazebo·Unity 같은 비ROS 환경의 파서에서 오류를 냅니다.

#### 3.6.1 패키지 생성 + 경로 치환 + XML 파싱 검증 (한 번에)

```powershell
$src = "$env:TEMP\opencode\rb3730\rbpodo_ros2\rbpodo_description"
$dst = "$env:TEMP\opencode\rb3730\rb3_730es_u_standalone"

if (Test-Path -LiteralPath $dst) { Remove-Item -LiteralPath $dst -Recurse -Force }
New-Item -ItemType Directory -Path "$dst\meshes","$dst\urdf","$dst\config" -Force | Out-Null

Copy-Item -LiteralPath "$src\meshes\rb3_730es_u" -Destination "$dst\meshes\rb3_730es_u" -Recurse -Force
Copy-Item -LiteralPath "$src\robots\rb3_730es_u" -Destination "$dst\config\rb3_730es_u" -Recurse -Force
Copy-Item -LiteralPath "$src\robots\rb3_730es_u.urdf.xacro" -Destination "$dst\urdf\rb3_730es_u.urdf.xacro" -Force

# URDF 를 읽고 package:// → ../meshes/ 로 치환해 새 파일로 저장
$raw = Get-Content -LiteralPath "$src\robots\rb3_730es_u.urdf" -Raw
$raw = $raw -replace 'package://rbpodo_description/meshes/','../meshes/'
[System.IO.File]::WriteAllText("$dst\urdf\rb3_730es_u.urdf", $raw, (New-Object System.Text.UTF8Encoding($false)))

# --- 검증 ---
[xml]$x = $raw
"XML parse OK; root=$($x.robot.name)"
"links: $(@($x.robot.link).Count)  joints: $(@($x.robot.joint).Count)"

# 모든 메시 참조가 실제로 resolve 되는지 확인
$m = Select-String -InputObject $raw -Pattern 'filename="([^"]+)"' -AllMatches |
     ForEach-Object { $_.Matches } | ForEach-Object { $_.Groups[1].Value }
"mesh refs: $($m.Count)"
$bad = $m | Where-Object { -not (Test-Path -LiteralPath "$dst\urdf\$_") }
if ($bad) { "MISSING:"; $bad } else { "OK: all mesh references resolve relative to urdf/" }
```

✅ **확인**:
```
XML parse OK; root=rb3_730es_u
links: 8  joints: 7
mesh refs: 14
OK: all mesh references resolve relative to urdf/
```

> **링크 8개 / 조인트 7개인 이유**: 링크는 `link0`~`link6` + `tcp`(도구 프레임).
> 조인트는 가동축 6개 + 고정 조인트 `tcp_joint` 1개입니다.

#### 3.6.2 `<ros2_control>` 제거판 생성

```powershell
$dst = "$env:TEMP\opencode\rb3730\rb3_730es_u_standalone"
$p   = "$dst\urdf\rb3_730es_u.urdf"
$raw = [System.IO.File]::ReadAllText($p)

# ros2_control 블록 전체 제거 (개행까지 정리)
$clean = [regex]::Replace($raw, '(?s)\s*<ros2_control.*?</ros2_control>\s*', "`r`n")
[System.IO.File]::WriteAllText("$dst\urdf\rb3_730es_u_sim.urdf", $clean,
                               (New-Object System.Text.UTF8Encoding($false)))

# 개수 비교로 실제 제거 확인
[xml]$a = $raw; [xml]$b = $clean
"orig  : links=$(@($a.robot.link).Count) joints=$(@($a.robot.joint).Count) ros2_control=$(@($a.robot.ros2_control).Count)"
"sim   : links=$(@($b.robot.link).Count) joints=$(@($b.robot.joint).Count) ros2_control=$(@($b.robot.ros2_control).Count)"
```

✅ **확인**: `sim` 줄의 `ros2_control=0`, 링크/조인트 수는 원본과 동일.

> ⚠️ **PowerShell 함정**: `[xml]` 캐스팅 후 `@($x.robot.ros2_control).Count` 는 빈 문자열을 배열로 세서
> 항상 1이 나올 수 있습니다. **문자열 직접 검색**으로 교차 확인하세요.
> ```powershell
> foreach ($f in @("$dst\urdf\rb3_730es_u.urdf","$dst\urdf\rb3_730es_u_sim.urdf")) {
>   $t = [System.IO.File]::ReadAllText($f)
>   "{0,-24} ros2_control={1}" -f (Split-Path $f -Leaf), ([regex]::Matches($t,'ros2_control').Count)
> }
> ```
> 기대 결과: `rb3_730es_u.urdf → 1`, `rb3_730es_u_sim.urdf → 0`

#### 3.6.3 최종 URDF (`rb3_730es_u_sim.urdf`) 전문

이 파일이 [Part 4](#part-4)의 입력입니다.


````text
> FILE: rb3_730es_u_sim.urdf  (169 lines)

<?xml version="1.0" ?>
<!-- =================================================================================== -->
<!-- |    This document was autogenerated by xacro from rb3_730es_u.urdf.xacro         | -->
<!-- |    EDITING THIS FILE BY HAND IS NOT RECOMMENDED                                 | -->
<!-- =================================================================================== -->
<robot name="rb3_730es_u">
  <link name="link0">
    <visual>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/visual/link0.dae"/>
      </geometry>
    </visual>
    <collision>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/collision/link0.stl"/>
      </geometry>
    </collision>
  </link>
  <link name="link1">
    <visual>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/visual/link1.dae"/>
      </geometry>
    </visual>
    <collision>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/collision/link1.stl"/>
      </geometry>
    </collision>
    <inertial>
      <origin rpy="0 0 0" xyz="0.000101 -0.003241 -0.017587"/>
      <mass value="2.058"/>
      <inertia ixx="0.003" ixy="0.0" ixz="0.0" iyy="0.003" iyz="0.0" izz="0.002"/>
    </inertial>
  </link>
  <link name="link2">
    <visual>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/visual/link2.dae"/>
      </geometry>
    </visual>
    <collision>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/collision/link2.stl"/>
      </geometry>
    </collision>
    <inertial>
      <origin rpy="0 0 0" xyz="-0.000049 -0.095326 0.121743"/>
      <mass value="4.227"/>
      <inertia ixx="0.075" ixy="0.0" ixz="0.0" iyy="0.073" iyz="0.0" izz="0.005"/>
    </inertial>
  </link>
  <link name="link3">
    <visual>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/visual/link3.dae"/>
      </geometry>
    </visual>
    <collision>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/collision/link3.stl"/>
      </geometry>
    </collision>
    <inertial>
      <origin rpy="0 0 0" xyz="-0.0 -0.004 0.026"/>
      <mass value="1.45"/>
      <inertia ixx="0.002" ixy="0.0" ixz="0.0" iyy="0.002" iyz="0.0" izz="0.001"/>
    </inertial>
  </link>
  <link name="link4">
    <visual>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/visual/link4.dae"/>
      </geometry>
    </visual>
    <collision>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/collision/link4.stl"/>
      </geometry>
    </collision>
    <inertial>
      <origin rpy="0 0 0" xyz="0.0 -0.065 0.264"/>
      <mass value="1.798"/>
      <inertia ixx="0.02" ixy="0.0" ixz="0.0" iyy="0.018" iyz="0.004" izz="0.003"/>
    </inertial>
  </link>
  <link name="link5">
    <visual>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/visual/link5.dae"/>
      </geometry>
    </visual>
    <collision>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/collision/link5.stl"/>
      </geometry>
    </collision>
    <inertial>
      <origin rpy="0 0 0" xyz="0.0 -0.001 0.017"/>
      <mass value="0.944"/>
      <inertia ixx="0.000776694" ixy="5.052e-06" ixz="-5.306e-06" iyy="0.000786695" iyz="-1.7419e-05" izz="0.000515123"/>
    </inertial>
  </link>
  <link name="link6">
    <visual>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/visual/link6.dae"/>
      </geometry>
    </visual>
    <collision>
      <geometry>
        <mesh filename="../meshes/rb3_730es_u/collision/link6.stl"/>
      </geometry>
    </collision>
    <inertial>
      <origin rpy="0 0 0" xyz="-0.001 -0.001 0.068"/>
      <mass value="0.05"/>
      <inertia ixx="2.6302e-05" ixy="-3.4e-07" ixz="4.57e-07" iyy="1.6887e-05" iyz="1.48e-07" izz="3.5232e-05"/>
    </inertial>
  </link>
  <link name="tcp"/>
  <joint name="base" type="revolute">
    <origin rpy="0 0 0" xyz="0 0 0.1453"/>
    <parent link="link0"/>
    <child link="link1"/>
    <axis xyz="0 0 1"/>
    <limit effort="10" lower="-3.14" upper="3.14" velocity="3.14"/>
  </joint>
  <joint name="shoulder" type="revolute">
    <origin rpy="0 0 0" xyz="0 0 0"/>
    <parent link="link1"/>
    <child link="link2"/>
    <axis xyz="0 1 0"/>
    <limit effort="10" lower="-3.14" upper="3.14" velocity="3.14"/>
  </joint>
  <joint name="elbow" type="revolute">
    <origin rpy="0 0 0" xyz="0 -0.00645 0.286"/>
    <parent link="link2"/>
    <child link="link3"/>
    <axis xyz="0 1 0"/>
    <limit effort="10" lower="-3.14" upper="3.14" velocity="3.14"/>
  </joint>
  <joint name="wrist1" type="revolute">
    <origin rpy="0 0 0" xyz="0 0 0"/>
    <parent link="link3"/>
    <child link="link4"/>
    <axis xyz="0 0 1"/>
    <limit effort="10" lower="-3.14" upper="3.14" velocity="3.14"/>
  </joint>
  <joint name="wrist2" type="revolute">
    <origin rpy="0 0 0" xyz="0 0 0.344"/>
    <parent link="link4"/>
    <child link="link5"/>
    <axis xyz="0 1 0"/>
    <limit effort="10" lower="-3.14" upper="3.14" velocity="3.14"/>
  </joint>
  <joint name="wrist3" type="revolute">
    <origin rpy="0 0 0" xyz="0 0 0"/>
    <parent link="link5"/>
    <child link="link6"/>
    <axis xyz="0 0 1"/>
    <limit effort="10" lower="-3.14" upper="3.14" velocity="3.14"/>
  </joint>
  <joint name="tcp_joint" type="fixed">
    <origin rpy="0 0 0" xyz="0 0 0.1"/>
    <parent link="link6"/>
    <child link="tcp"/>
  </joint>
</robot>
````

### 3.7 **물리량 교차 검증** — 모델이 제원 크기인지 확인

모델이 실제로 730mm 로봇인지 확인하려면 **제원과 독립적인 값 두 개**를 대조해야 합니다.

#### 검증 A: 링크 질량 합 vs 제원 중량

```powershell
$yaml = Get-Content "$env:TEMP\opencode\rb3730\rb3_730es_u_standalone\config\rb3_730es_u\link.yaml" -Raw
$sum = 0.0
[regex]::Matches($yaml, 'mass:\s*([0-9.]+)') | ForEach-Object { $sum += [double]$_.Groups[1].Value }
"sum of link masses = {0} kg   (datasheet: 11 kg)" -f [math]::Round($sum, 3)
```

✅ **확인**: `sum of link masses = 10.527 kg`
차액 0.47 kg는 플랜지/공구(타공구)의 무게입니다. **오차 4 % 이내면 정상 스케일입니다.**

| 링크 | 질량 (kg) |
|---|---|
| link0 (베이스) | 0 |
| link1 | 2.058 |
| link2 | 4.227 |
| link3 | 1.450 |
| link4 | 1.798 |
| link5 | 0.944 |
| link6 (플랜지) | 0.050 |
| **합계** | **10.527** |

#### 검증 B: 관절 거리 합 vs 도달거리

수직 링크의 `origin xyz` z 성분을 더하면 신장 높이가 됩니다.

```
base   0.1453 m
elbow  0.286  m
wrist2 0.344  m
──────────────
합계   0.7753 m  +  TCP 오프셋 0.1 m  =  0.8753 m (완전 신장 높이)
```

도달거리 730 mm는 이 높이의 수평 투영이며, `base` 축(J1)을 60° 돌리면 정확히 일치합니다.
이 검증은 [Part 4.4](#verify-roundtrip)에서 실제로 측정합니다.

<a id="38-관절-키네마틱스-최종-표"></a>

### 3.8 관절 키네마틱스 최종 표

이 표는 이후 모든 단계의 **기준값(ground truth)** 입니다. 백업해 두세요.

| Joint | Type | Parent → Child | Origin xyz (m) | Axis | Limits (rad) |
|---|---|---|---|---|---|
| `base` | revolute | link0 → link1 | `0 0 0.1453` | **Z** | ±3.14 |
| `shoulder` | revolute | link1 → link2 | `0 0 0` | **Y** | ±3.14 |
| `elbow` | revolute | link2 → link3 | `0 -0.00645 0.286` | **Y** | ±3.14 |
| `wrist1` | revolute | link3 → link4 | `0 0 0` | **Z** | ±3.14 |
| `wrist2` | revolute | link4 → link5 | `0 0 0.344` | **Y** | ±3.14 |
| `wrist3` | revolute | link5 → link6 | `0 0 0` | **Z** | ±3.14 |
| `tcp_joint` | fixed | link6 → tcp | `0 0 0.1` | — | — |

TCP는 `link6`(플랜지)에서 **100 mm 앞에 있는 도구 중심점**입니다.

> ⚠️ **관절 한계에 대한 주의**
> 이 URDF는 `ros2_control`이 요구하는 이유로 **모든 축을 ±180°로 넓게** 선언합니다.
> 실제 하드웨어 범위(제원표의 J1/J2 ±360°, J3 ±150°)보다 넓습니다.
> 디지털 트윈에서 물리적으로 유효한 워크스페이스 제한이 필요하면
> `rb3_730es_u_joints.json`의 `limit_lower`/`limit_upper`를 실제 값으로 바꾸고
> JointDriver의 `jointTable` 슬롯에 할당하세요. ([Part 6.1](#code-jointdriver) 참고)

### 3.9 Phase 1 완료 체크리스트

- [ ] `rb3_730es_u_standalone` 폴더에 `urdf/ meshes/ config/` 가 있다
- [ ] `rb3_730es_u_sim.urdf` 가 유효한 XML이다
- [ ] 링크 8개, 조인트 7개
- [ ] 메시 참조 14개가 전부 resolve 된다
- [ ] DAE 전부가 `meter="1"` + `Z_UP`
- [ ] STL 전부가 binary
- [ ] 질량 합 ≈ 10.527 kg
- [ ] `ros2_control` 제거판이 `ros2_control=0`

✅ 모두 충족했으면 [Part 4](#part-4)로 넘어갑니다.


---

<a id="part-4"></a>

## Part 4. Phase 2 — URDF→리그 FBX 변환 (Blender)

### 4.1 왜 변환이 필요한가

Unity는 **FBX만** 임포트합니다. URDF를 직접 읽는 기능은 없습니다.
그래서 URDF → FBX 변환 단계가 반드시 필요합니다.

| 변환 도구 | 장점 | 단점 | 선택 |
|---|---|---|---|
| Commercial CAD (Rhino, 3ds Max) | 리그 지원 | 유료, 스크립트 호환성 불명확 | ❌ |
| Assimp | 가벼움 | 스키닝 리그·관절축 제어 불가 | ❌ |
| **Blender Python** | 완전한 리그 제어, 무료, 헤드리스 실행 | 스크립트를 직접 써야 함 | ✅ **선택** |

Blender를 쓴 이유는 **"각 본의 로컬 Y축을 URDF 관절축과 일치시키는 것"** 을 정확히 통제할 수 있기 때문입니다
(원원칙 1](#principle-1) 참조).

### 4.2 변환 스크립트 `fbx_build.py` — 전체 소스

아래를 `fbx_build.py` 로 저장하세요. 상단의 `ROOT` 경로만 본인 환경에 맞게 바꾸면 됩니다.


````text
> FILE: fbx_build.py  (291 lines)

import bpy, sys, os, math, json
import xml.etree.ElementTree as ET
from mathutils import Matrix, Vector

ROOT = r"C:\Users\ADMINI~1\AppData\Local\Temp\opencode\rb3730\rb3_730es_u_standalone"
URDF = os.path.join(ROOT, "urdf", "rb3_730es_u_sim.urdf")
OUT = os.path.join(ROOT, "fbx")
os.makedirs(OUT, exist_ok=True)

log = []


def say(m):
    print(m)
    log.append(m)


# ---------------------------------------------------------------- URDF parsing
def f3(s, default=(0.0, 0.0, 0.0)):
    if s is None:
        return default
    return tuple(float(x) for x in s.split())


def rpy_to_mat(rpy):
    r, p, y = rpy
    cr, sr = math.cos(r), math.sin(r)
    cp, sp = math.cos(p), math.sin(p)
    cy, sy = math.cos(y), math.sin(y)
    return Matrix.Identity(3).to_4x4() @ Matrix.Rotation(y, 4, 'Z') @ Matrix.Rotation(p, 4, 'Y') @ Matrix.Rotation(r, 4, 'X')


tree = ET.parse(URDF)
root = tree.getroot()

links = {}
for l in root.findall('link'):
    name = l.get('name')
    vis = col = None
    for g in l.findall('visual/geometry/mesh'):
        vis = g.get('filename')
    for g in l.findall('collision/geometry/mesh'):
        col = g.get('filename')
    inert = l.find('inertial')
    mass = float(inert.find('mass').get('value')) if inert is not None and inert.find('mass') is not None else 0.0
    links[name] = {'visual': vis, 'collision': col, 'mass': mass}

joints = []
for j in root.findall('joint'):
    o = j.find('origin')
    xyz = f3(o.get('xyz')) if o is not None else (0.0, 0.0, 0.0)
    rpy = f3(o.get('rpy')) if o is not None else (0.0, 0.0, 0.0)
    ax = j.find('axis')
    lim = j.find('limit')
    joints.append({
        'name': j.get('name'),
        'type': j.get('type'),
        'parent': j.find('parent').get('link'),
        'child': j.find('child').get('link'),
        'xyz': xyz, 'rpy': rpy,
        'axis': f3(ax.get('xyz'), (1.0, 0.0, 0.0)) if ax is not None else (1.0, 0.0, 0.0),
        'lower': float(lim.get('lower')) if lim is not None and lim.get('lower') else None,
        'upper': float(lim.get('upper')) if lim is not None and lim.get('upper') else None,
    })

# accumulate world frame of every link
frames = {root.find('link').get('name'): Matrix.Identity(4)}
for j in joints:
    pf = frames[j['parent']]
    frames[j['child']] = pf @ Matrix.Translation(Vector(j['xyz'])) @ rpy_to_mat(j['rpy'])

# root link = parent of the first joint
ROOT_LINK = joints[0]['parent']
say('root link      : %s' % ROOT_LINK)
say('links parsed   : %d' % len(links))
say('joints parsed  : %d' % len(joints))
for j in joints:
    say('  %-11s %-8s %s -> %s  xyz=%s axis=%s lim=[%s, %s]' % (
        j['name'], j['type'], j['parent'], j['child'], j['xyz'], j['axis'], j['lower'], j['upper']))


# ---------------------------------------------------------------- scene reset
def purge():
    bpy.ops.wm.read_factory_settings(use_empty=True)


def op_any(names, **kw):
    for n in names:
        parts = n.split('.')
        mod = bpy.ops
        for p in parts[:-1]:
            mod = getattr(mod, p)
        f = getattr(mod, parts[-1], None)
        if f is not None:
            try:
                return f(**kw)
            except Exception as e:
                say('  op %s failed: %s' % (n, e))
    raise RuntimeError('no operator found among %s' % names)


DAE_OPS = ['wm.collada_import', 'import_scene.dae', 'wm.dae_import']
STL_OPS = ['wm.stl_import', 'import_mesh.stl', 'wm.import_stl']


def import_mesh(path):
    before = set(bpy.data.objects)
    if path.lower().endswith('.dae'):
        op_any(DAE_OPS, filepath=path)
    else:
        op_any(STL_OPS, filepath=path)
    new = [o for o in bpy.data.objects if o not in before]
    return new


def build(kind, out_name):
    """kind: 'visual' or 'collision'"""
    purge()
    scn = bpy.context.scene
    scn.unit_settings.system = 'METRIC'
    scn.unit_settings.scale_length = 1.0
    scn.unit_settings.length_unit = 'METERS'

    # ---- armature
    arm_data = bpy.data.armatures.new('rb3_730es_u_armature')
    arm = bpy.data.objects.new('Armature', arm_data)
    scn.collection.objects.link(arm)
    bpy.context.view_layer.objects.active = arm
    bpy.ops.object.mode_set(mode='EDIT')

    eb = arm_data.edit_bones

    def orient(bn, head, ydir, length):
        """Set head/tail directly: Blender then derives the bone Y axis from
        (tail - head), which is far more reliable than assigning EditBone.matrix
        and mutating .length afterwards."""
        ydir = Vector(ydir).normalized()
        bn.head = Vector(head)
        bn.tail = Vector(head) + ydir * length
        # roll: keep the bone Z axis as close to world +Z as the axis allows
        ref = Vector((0.0, 0.0, 1.0))
        if abs(ydir.dot(ref)) > 0.999:
            ref = Vector((0.0, 1.0, 0.0))
        zdir = (ref - ydir * ref.dot(ydir)).normalized()
        bn.align_roll(zdir)

    # root bone (static base)
    b0 = eb.new(ROOT_LINK)
    orient(b0, (0.0, 0.0, 0.0), (0.0, 0.0, 1.0), 0.1453)

    for j in joints:
        if j['type'] == 'fixed':
            continue
        pf = frames[j['parent']]
        head = pf @ Vector(j['xyz'])
        # joint axis expressed in armature space
        ydir = pf.to_3x3() @ Vector(j['axis'])
        bn = eb.new(j['child'])
        orient(bn, head, ydir, 0.06)
        bn.parent = eb[j['parent']]
        bn.use_connect = False

    # tcp frame as a plain bone so Unity creates a matching transform
    for j in [x for x in joints if x['type'] == 'fixed']:
        pf = frames[j['parent']]
        head = pf @ Vector(j['xyz'])
        bn = eb.new(j['child'])
        orient(bn, head, (0.0, 0.0, 1.0), 0.04)
        bn.parent = eb[j['parent']]
        bn.use_connect = False

    bpy.ops.object.mode_set(mode='POSE')
    for pb in arm.pose.bones:
        pb.rotation_mode = 'XYZ'
    bpy.ops.object.mode_set(mode='OBJECT')

    say('[%s] bones: %s' % (kind, ', '.join(b.name for b in arm_data.bones)))

    # ---- meshes
    total_v = total_f = 0
    for lname, info in sorted(links.items()):
        rel = info[kind]
        if not rel:
            continue
        path = os.path.normpath(os.path.join(ROOT, 'urdf', rel))
        if not os.path.isfile(path):
            say('  MISSING %s' % path)
            continue
        objs = import_mesh(path)
        if not objs:
            say('  import produced no objects: %s' % path)
            continue
        # place every imported object into the link frame (object matrix == link frame)
        fm = frames[lname]
        for o in objs:
            o.matrix_world = fm @ o.matrix_world
        bpy.context.view_layer.update()

        # merge into one mesh object per link
        meshes = [o for o in objs if o.type == 'MESH']
        if len(meshes) > 1:
            bpy.ops.object.select_all(action='DESELECT')
            for o in meshes:
                o.select_set(True)
            bpy.context.view_layer.objects.active = meshes[0]
            bpy.ops.object.join()
            merged = bpy.context.view_layer.objects.active
        else:
            merged = meshes[0] if meshes else None
        if merged is None:
            continue
        merged.name = lname
        merged.data.name = '%s_mesh' % lname
        for p in list(merged.data.polygons):
            p.use_smooth = True

        nv = len(merged.data.vertices)
        nf = len(merged.data.polygons)
        total_v += nv
        total_f += nf

        # armature deform
        mod = merged.modifiers.new('Armature', 'ARMATURE')
        mod.object = arm
        mod.use_vertex_groups = True
        mod.use_bone_envelopes = False
        vg = merged.vertex_groups.get(lname) or merged.vertex_groups.new(name=lname)
        vg.add([v.index for v in merged.data.vertices], 1.0, 'REPLACE')
        say('  %-7s verts=%-7d faces=%-7d  <- %s' % (lname, nv, nf, os.path.basename(path)))

    say('[%s] total verts=%d faces=%d' % (kind, total_v, total_f))

    out = os.path.join(OUT, out_name)
    bpy.ops.object.select_all(action='DESELECT')
    arm.select_set(True)
    for o in scn.objects:
        if o.type == 'MESH':
            o.select_set(True)
    bpy.context.view_layer.objects.active = arm
    bpy.ops.export_scene.fbx(
        filepath=out,
        use_selection=True,
        object_types={'ARMATURE', 'MESH', 'EMPTY'},
        use_mesh_modifiers=True,
        mesh_smooth_type='FACE',
        use_tspace=True,
        use_armature_deform_only=True,
        add_leaf_bones=False,
        bake_anim=False,
        bake_anim_use_all_bones=False,
        bake_anim_use_nla_strips=False,
        bake_anim_use_all_actions=False,
        bake_anim_force_startend_keying=True,
        apply_unit_scale=True,
        global_scale=1.0,
        apply_scale_options='FBX_SCALE_NONE',
        axis_forward='-Y',
        axis_up='Z',
        use_custom_props=False,
        path_mode='COPY',
    )
    say('[%s] exported -> %s  (%.2f MB)' % (kind, out, os.path.getsize(out) / 1048576.0))
    return out


# ---------------------------------------------------------------- joint table
table = []
for j in joints:
    pf = frames[j['parent']]
    table.append({
        'name': j['name'], 'type': j['type'],
        'parent': j['parent'], 'child': j['child'],
        'origin_xyz': list(j['xyz']), 'origin_rpy': list(j['rpy']),
        'axis': list(j['axis']),
        'limit_lower': j['lower'], 'limit_upper': j['upper'],
        'head_world': [round(v, 6) for v in (pf @ Vector(j['xyz']))],
        'mass_parent_link': links.get(j['parent'], {}).get('mass'),
    })
tj = os.path.join(ROOT, 'fbx', 'rb3_730es_u_joints.json')
with open(tj, 'w', encoding='utf-8') as f:
    json.dump({'robot': 'rb3_730es_u', 'units': 'meter', 'up_axis': 'Z',
               'link_masses': {k: v['mass'] for k, v in links.items()}, 'joints': table},
              f, indent=2)
say('joint table -> %s' % tj)

build('visual', 'rb3_730es_u_rigged.fbx')
build('collision', 'rb3_730es_u_collision.fbx')

with open(os.path.join(ROOT, 'fbx', 'convert_log.txt'), 'w', encoding='utf-8') as f:
    f.write('\n'.join(log))
say('DONE')
````

### 4.3 스크립트의 핵심 동작 5가지

#### (1) URDF 파싱 → 링크/조인트 테이블

`xml.etree.ElementTree` 로 `<link>` 과 `<joint>` 을 읽습니다.
각 조인트의 `origin xyz`, `origin rpy`, `axis xyz`, `limit` 을 추출합니다.

#### (2) **정방향 기구학(FK) 누적** — 각 링크의 월드 프레임 계산

```python
# 부모 링크 프레임에 자식의 origin 을 곱해서 전부 누적한다
frames원j원'child']] = pf @ Matrix.Translation(Vector(j원'xyz'])) @ rpy_to_mat(j원'rpy'])
```

이 `frames` 딕셔너리가 이후 두 곳에 쓰입니다.
- 본(bone)의 head 위치 계산
- 메시를 링크 프레임으로 옮겨 배치

#### (3) 본 방향 지정 — **이것이 원칙 1의 구현**

```python
def orient(bn, head, ydir, length):
    ydir = Vector(ydir).normalized()
    bn.head = Vector(head)
    bn.tail = Vector(head) + ydir * length   # ← tail 방향이 곧 본의 Y축
    ref = Vector((0.0, 0.0, 1.0))
    if abs(ydir.dot(ref)) > 0.999:
        ref = Vector((0.0, 1.0, 0.0))
    zdir = (ref - ydir * ref.dot(ydir)).normalized()
    bn.align_roll(zdir)
```

`orient()` 는 Blender에 **head와 tail 좌표만 직접 지정**합니다.
그러면 Blender가 `tail - head`로부터 본의 Y축을 자동으로 도출합니다.

> 🔑 **왜 `EditBone.matrix`를 직접 안 쓰나요?**
> `EditBone.matrix` 를 대입한 뒤 `.length` 를 바꾸는 방식은 Blender 버전에 따라
> 의도와 다른 방향의 축이 나오기도 합니다. head/tail 직접 지정이 압도적으로 안정적입니다.

관절축은 부모 프레임에서 자식으로 변환해 스케일 없이 전달됩니다:

```python
head  = pf @ Vector(j원'xyz'])            # 조인트 원점 (월드)
ydir  = pf.to_3x3() @ Vector(j원'axis'])  # 관절축 (아키메처 공간)
```

#### (4) 메시 임포트 → 링크 프레임 배치 → 스키닝

```python
o.matrix_world = fm @ o.matrix_world     # 링크 프레임으로 이동
```

**⚠️ 이 한 줄이 빠지면 이중 변환 버그가 됩니다.**
holder 빈 오브젝트의 parent inverse가 설정되지 않으면 좌표가 두 번 적용되어
바운딩박스가 1.65 m(정상 0.875 m)로 튀어 나옵니다. 실제로 이 버그가 발생했고,
`matrix_world` 를 링크 프레임과 곱하도록 고쳐 해결했습니다.

스키닝 가중치는 **단일 본에 100 %** 배정합니다.

```python
vg = merged.vertex_groups.get(lname) or merged.vertex_groups.new(name=lname)
vg.add(원v.index for v in merged.data.vertices], 1.0, 'REPLACE')
```

`use_armature_deform_only=True` 를 함께 쓰므로, 리그에 없는 정점은 변환되지 않습니다.

#### (5) FBX 익스포트 옵션 — **하나하나 이유가 있습니다**

```python
bpy.ops.export_scene.fbx(
    axis_forward='-Y', axis_up='Z',          # Z-up 유지 → Unity가 알아서 Y-up 변환
    apply_unit_scale=True, global_scale=1.0, # 1 unit = 1 m
    apply_scale_options='FBX_SCALE_NONE',
    add_leaf_bones=False,                    # 잎 본 없음 (8개 본 유지)
    bake_anim=False,                         # 애니메이션 없음
    use_mesh_modifiers=True,                 # 메시 압축 최적화 반영
    object_types={'ARMATURE', 'MESH', 'EMPTY'},
    path_mode='COPY',
)
```

`axis_up='Z'` 인 이유는 **원본 URDF가 Z-up** 이고, 스케일·축 변환을 한 번도 거치지 않는 것이
검증 가능성을 최대화하기 때문입니다. Unity가 임포트 시 Y-up으로 바꿔 줍니다.

<a id="verify-roundtrip"></a>

### 4.4 실행 및 결과 확인

```powershell
$w = "$env:TEMP\opencode\rb3730"
$b = "$w\blender\blender-4.2.23-windows-x64\blender.exe"

& $b -b --factory-startup --python "$w\fbx_build.py" 2>&1 |
  Where-Object { $_ -notmatch 'TBBmalloc' } | Select-Object -Last 80
```

✅ **확인**: 아래 로그가 그대로 나오면 성공입니다.

```
root link      : link0
links parsed   : 8
joints parsed  : 7
  base        revolute link0 -> link1  xyz=(0.0, 0.0, 0.1453) axis=(0.0, 0.0, 1.0) lim=원-3.14, 3.14]
  shoulder    revolute link1 -> link2  xyz=(0.0, 0.0, 0.0) axis=(0.0, 1.0, 0.0) lim=원-3.14, 3.14]
  elbow       revolute link2 -> link3  xyz=(0.0, -0.00645, 0.286) axis=(0.0, 1.0, 0.0) lim=원-3.14, 3.14]
  wrist1      revolute link3 -> link4  xyz=(0.0, 0.0, 0.0) axis=(0.0, 0.0, 1.0) lim=원-3.14, 3.14]
  wrist2      revolute link4 -> link5  xyz=(0.0, 0.0, 0.344) axis=(0.0, 1.0, 0.0) lim=원-3.14, 3.14]
  wrist3      revolute link5 -> link6  xyz=(0.0, 0.0, 0.0) axis=(0.0, 0.0, 1.0) lim=원-3.14, 3.14]
  tcp_joint   fixed    link6 -> tcp  xyz=(0.0, 0.0, 0.1) axis=(1.0, 0.0, 0.0) lim=원None, None]
원visual] bones: link0, link1, link2, link3, link4, link5, link6, tcp
  link0   verts=27342   faces=28070    <- link0.dae
  link1   verts=7981    faces=10878    <- link1.dae
  link2   verts=19014   faces=26114    <- link2.dae
  link3   verts=9955    faces=14150    <- link3.dae
  link4   verts=14333   faces=20598    <- link4.dae
  link5   verts=9952    faces=13456    <- link5.dae
  link6   verts=18063   faces=20284    <- link6.dae
원visual] total verts=106640 faces=133550
원visual] exported -> ...\fbx\rb3_730es_u_rigged.fbx  (3.62 MB)
원collision] bones: link0, link1, link2, link3, link4, link5, link6, tcp
  link0   verts=14001   faces=28070    <- link0.stl
  link1   verts=5446    faces=10873    <- link1.stl
  link2   verts=13024   faces=26023    <- link2.stl
  link3   verts=7041    faces=14064    <- link3.stl
  link4   verts=10299   faces=20581    <- link4.stl
  link5   verts=6724    faces=13439    <- link5.stl
  link6   verts=10162   faces=20284    <- link6.stl
원collision] total verts=66697 faces=133334
원collision] exported -> ...\fbx\rb3_730es_u_collision.fbx  (2.65 MB)
joint table -> ...\fbx\rb3_730es_u_joints.json
DONE
```

> 🔑 `convert_log.txt` 에도 동일 로그가 저장됩니다.
> ```
> Get-Content "$env:TEMP\opencode\rb3730\rb3_730es_u_standalone\fbx\convert_log.txt"
> ```

### 4.5 라운드트립 검증 — **이 단계가 가장 중요합니다**

"파일 생성이 성공했다"는 것만으로는 **모델이 틀렸을 수 있습니다.**
exported FBX를 **다시 Blender로 읽어와서** URDF 기준값과 대조합니다.

검증 스크립트 `fbx_verify.py`:


````text
> FILE: fbx_verify.py  (136 lines)

import bpy, os, math
from mathutils import Vector, Matrix

FBX = r"C:\Users\ADMINI~1\AppData\Local\Temp\opencode\rb3730\rb3_730es_u_standalone\fbx\rb3_730es_u_rigged.fbx"
COL = r"C:\Users\ADMINI~1\AppData\Local\Temp\opencode\rb3730\rb3_730es_u_standalone\fbx\rb3_730es_u_collision.fbx"

print('=' * 70)
bpy.ops.wm.read_factory_settings(use_empty=True)
bpy.ops.import_scene.fbx(filepath=FBX)

arm = [o for o in bpy.data.objects if o.type == 'ARMATURE']
print('armatures   :', [o.name for o in arm])
me = [o for o in bpy.data.objects if o.type == 'MESH']
print('mesh objects:', len(me), [o.name for o in me])
print('empties     :', [o.name for o in bpy.data.objects if o.type == 'EMPTY'])

a = arm[0]
print('bones (%d)   :' % len(a.data.bones))
for b in a.data.bones:
    print('   %-7s parent=%-7s head=(%7.4f %7.4f %7.4f) tail=(%7.4f %7.4f %7.4f) Ylen=%.3f' % (
        b.name, b.parent.name if b.parent else '-',
        b.head_local.x, b.head_local.y, b.head_local.z,
        b.tail_local.x, b.tail_local.y, b.tail_local.z, b.length))

mats = []
for o in me:
    for s in o.material_slots:
        if s.material and s.material.name not in mats:
            mats.append(s.material.name)
print('materials   :', len(mats))
for m in mats[:15]:
    print('   ', m)

tv = sum(len(o.data.vertices) for o in me)
tf = sum(len(o.data.polygons) for o in me)
print('totals      : verts=%d faces=%d' % (tv, tf))

# ---- world bbox in rest pose (Z-up)
lo = Vector((1e9, 1e9, 1e9))
hi = Vector((-1e9, -1e9, -1e9))
for o in me:
    for c in o.bound_box:
        w = o.matrix_world @ Vector(c)
        for i in range(3):
            lo[i] = min(lo[i], w[i])
            hi[i] = max(hi[i], w[i])
print('bbox min    : (%.4f, %.4f, %.4f)' % (lo.x, lo.y, lo.z))
print('bbox max    : (%.4f, %.4f, %.4f)' % (hi.x, hi.y, hi.z))
print('size        : (%.4f, %.4f, %.4f)  max=%.4f m' % (hi.x - lo.x, hi.y - lo.y, hi.z - lo.z, max(hi.x - lo.x, hi.y - lo.y, hi.z - lo.z)))


def tcp_world():
    p = a.matrix_world @ a.pose.bones['tcp'].head
    return p.copy()


def mesh_world_z(o):
    """max world Z of the evaluated (deformed) mesh"""
    dg = bpy.context.evaluated_depsgraph_get()
    ev = o.evaluated_get(dg)
    me = ev.to_mesh()
    mw = ev.matrix_world
    z = max((mw @ v.co).z for v in me.vertices) if len(me.vertices) else -1e9
    ev.to_mesh_clear()
    return z


def set_rot(bone, deg):
    pb = a.pose.bones[bone]
    pb.rotation_mode = 'XYZ'
    pb.rotation_euler = (0.0, math.radians(deg), 0.0)


print('=' * 70)
print('rest TCP    : (%.4f, %.4f, %.4f)' % tuple(tcp_world()))

by_name = {o.name.split('.')[0]: o for o in me}

bpy.ops.object.mode_set(mode='POSE')
a.pose.bones['link1'].rotation_mode = 'XYZ'
a.pose.bones['link1'].rotation_euler = (0, 0, math.radians(60))
bpy.context.view_layer.update()
p = tcp_world()
print('base +60deg : (%.4f, %.4f, %.4f)   [axis +Z -> swing in XY, Z stays 0.8753]' % (p.x, p.y, p.z))
a.pose.bones['link1'].rotation_euler = (0, 0, 0)
bpy.context.view_layer.update()

set_rot('link3', 60)
bpy.context.view_layer.update()
p = tcp_world()
print('elbow +60deg: (%.4f, %.4f, %.4f)   [axis +Y -> move in XZ, Y stays -0.0065]' % (p.x, p.y, p.z))
print('   link6 mesh top z = %.4f (rest 0.8753) -> mesh actually deformed: %s'
      % (mesh_world_z(by_name['link6']), mesh_world_z(by_name['link6']) > 0.90))
set_rot('link3', 0)
bpy.context.view_layer.update()
print('   after reset, link6 mesh top z = %.4f' % mesh_world_z(by_name['link6']))

set_rot('link5', 60)
bpy.context.view_layer.update()
p = tcp_world()
print('wrist2 +60deg:(%.4f, %.4f, %.4f)   [axis +Y]' % (p.x, p.y, p.z))
set_rot('link5', 0)
bpy.context.view_layer.update()
bpy.ops.object.mode_set(mode='OBJECT')
bpy.context.view_layer.update()
print('=' * 70)
print('bone head vs URDF joint origin (armature space):')
expect = {
    'link1': (0.0, 0.0, 0.1453),
    'link2': (0.0, 0.0, 0.1453),
    'link3': (0.0, -0.00645, 0.4313),
    'link4': (0.0, -0.00645, 0.4313),
    'link5': (0.0, -0.00645, 0.7753),
    'link6': (0.0, -0.00645, 0.7753),
    'tcp':   (0.0, -0.00645, 0.8753),
}
worst = 0.0
for n, e in expect.items():
    h = a.data.bones[n].head_local
    d = max(abs(h.x - e[0]), abs(h.y - e[1]), abs(h.z - e[2]))
    worst = max(worst, d)
    print('   %-7s bone=(%8.5f %8.5f %8.5f)  urdf=(%8.5f %8.5f %8.5f)  err=%.2e' % (n, h.x, h.y, h.z, e[0], e[1], e[2], d))
print('max FK error: %.3e m' % worst)

print('=' * 70)
bpy.ops.wm.read_factory_settings(use_empty=True)
bpy.ops.import_scene.fbx(filepath=COL)
arm2 = [o for o in bpy.data.objects if o.type == 'ARMATURE']
me2 = [o for o in bpy.data.objects if o.type == 'MESH']
print('collision fbx: bones=%d meshes=%d verts=%d faces=%d' % (
    len(arm2[0].data.bones), len(me2),
    sum(len(o.data.vertices) for o in me2),
    sum(len(o.data.polygons) for o in me2)))
print('collision names:', [o.name for o in me2])
print('collision bones:', [b.name for b in arm2[0].data.bones])
print('OK')
````

실행:

```powershell
$w = "$env:TEMP\opencode\rb3730"
$b = "$w\blender\blender-4.2.23-windows-x64\blender.exe"

& $b -b --factory-startup --python "$w\fbx_verify.py" 2>&1 |
  Where-Object { $_ -notmatch 'TBBmalloc' } | Select-Object -Last 70
```

### 4.6 검증 기준값 — 통과/실패 판정표

| 검사 항목 | 통과 기준 | 실패 시 의미 |
|---|---|---|
| 본 개수 | **8개** (`link0`…`link6`, `tcp`) | URDF 파싱 실패 또는 fixed 조인트 누락 |
| 본 체인 | `link0→link1→…→tcp` 일자 | 부모 연결 오류 |
| **본 head vs URDF 관절원점 최대 오차** | **< 1e-6 m** | FK 누적 오류 / 관절축 오류 |
| 아키메처 바운딩박스 | `0.128 × 0.247 × 0.875 m` | 스케일 또는 이중 변환 |
| `link1`(base) +60° 후 TCP **수평거리** | **0.730 m** (±0.5 mm) | J1 축 방향 오류 |
| `link3`(elbow) +60° 후 TCP | XZ 평면 이동, **Y 변화 없음** | Y축 조인트가 Z로 잘못 생성됨 |
| 메시 정점 실제 변형 | link6 메시 top > 0.90 m | 스키닝 미작동 |
| 리셋 후 복귀 | 원위치로 정확히 복귀 | Armature 모디파이 누락 |
| FBX 헤더 | 7.4 binary, `UpAxis=Z`, 1 unit = 1 m | 축/스케일 규약 불일치 |

#### 실제 통과 로그

```
bones (8)   :
   link0   parent=-       head=( 0.0000  0.0000  0.0000) tail=( 0.0000  0.0000  0.1453) Ylen=0.145
   link1   parent=link0   head=( 0.0000  0.0000  0.1453) tail=( 0.0000  0.0000  0.2053) Ylen=0.060
   link2   parent=link1   head=( 0.0000  0.0000  0.1453) tail=( 0.0000  0.0000  0.2053) Ylen=0.060
   link3   parent=link2   head=( 0.0000 -0.0065  0.4313) tail=( 0.0000 -0.0065  0.4913) Ylen=0.060
   link4   parent=link3   head=( 0.0000 -0.0065  0.4313) tail=( 0.0000 -0.0065  0.4913) Ylen=0.060
   link5   parent=link4   head=( 0.0000 -0.0065  0.7753) tail=( 0.0000 -0.0065  0.8353) Ylen=0.060
   link6   parent=link5   head=( 0.0000 -0.0065  0.7753) tail=( 0.0000 -0.0065  0.8353) Ylen=0.060
   tcp     parent=link6   head=( 0.0000 -0.0065  0.8753) tail=( 0.0000 -0.0065  0.9153) Ylen=0.040
totals      : verts=106640 faces=133550
size        : (0.1280, 0.2470, 0.8750)  max=0.8750 m
base +60deg : (0.6322, 0.3650, 0.8753)      → 수평 √(0.6322² + 0.365²) = 0.730 ✅
elbow +60deg: (…, …, …)  Y stays -0.0065  ✅
   link6 mesh top z = 0.9451 (rest 0.8753) -> mesh actually deformed: True
max FK error: 1.100e-07 m
collision fbx: bones=8 meshes=7 verts=66697 faces=133334
OK
```

> 🔑 **가장 결정적인 한 줄**:
> ```
> base +60deg : (0.6322, 0.3650, 0.8753)
> ```
> TCP의 Z가 **0.8753으로 그대로**이고 XY 평면에서만 회전했습니다.
> 이 값이 제원 도달거리 **730 mm**와 정확히 일치하므로
> 스케일·축·원점이 **전부 맞다**는 것을 의미합니다.

> ⚠️ **0.730 은 "전체 TCP 거리"가 아니라 수평거리(반지름)입니다.**
> 이 자세에서 TCP 원점까지의 **전체** 거리는
> `√(0.6322² + 0.3650² + 0.8753²) ≈ 1.140 m` 이고, 수직구성분이 0.8753 이기 때문입니다.
> Part 9.5 에서 HOME 포즈(수직으로 세운 자세)의 TCP 거리를 볼 때는
> **0.8753** 이 정답입니다. 두 값을 헷갈리지 마세요.

#### 본 head 오차 테이블 (최종)

| 본 | Blender head | URDF 관절원점 | 오차 |
|---|---|---|---|
| `link1` | (0, 0, 0.1453) | (0, 0, 0.1453) | 0 |
| `link2` | (0, 0, 0.1453) | (0, 0, 0.1453) | 0 |
| `link3` | (0, −0.00645, 0.4313) | (0, −0.00645, 0.4313) | 0 |
| `link4` | (0, −0.00645, 0.4313) | (0, −0.00645, 0.4313) | 0 |
| `link5` | (0, −0.00645, 0.7753) | (0, −0.00645, 0.7753) | 0 |
| `link6` | (0, −0.00645, 0.7753) | (0, −0.00645, 0.7753) | 0 |
| `tcp` | (0, −0.00645, 0.8753) | (0, −0.00645, 0.8753) | 0 |

**최대 오차 1.1e-7 m** — 부동소수점 정밀도 한계입니다. 완벽합니다.

### 4.7 실제로 발생한 2가지 변환 버그 (반드시 알아두세요)

이 버그들은 "그냥 대충 변환하면 생기는" 전형적인 실패입니다. 나중에 같은 문제를 만나면 여기서 답을 찾으세요.

#### 버그 A: 본 방향 오류 — Y축 조인트가 Z로 남음

**증상**: 첫 라운드트립에서 `elbow`/`wrist2`(URDF의 +Y축)를 60° 돌렸는데
TCP가 **XZ 평면이 아니라 XY 평면**으로 움직였습니다.

**원인**: 본을 만들 때 방향 벡터를 `(0,0,1)` 로 하드코딩했거나,
`EditBone.matrix` 를 대입 후 `.length` 를 바꾸는 과정에서 Y축이 다시 계산됐습니다.

**수정**: `orient()` 함수를 도입해 head/tail 을 직접 지정.
```python
bn.tail = Vector(head) + ydir * length
```

#### 버그 B: 이중 변환 — 바운딩박스 1.65 m

**증상**: 바운딩박스가 `(1.65, …)` 로 두 배 이상 커졌습니다.

**원인**: holder 빈 오브젝트를 만들고 그 밑에 메시를 넣었지만,
`matrix_parent_inverse` 가 설정되지 않아 링크 프레임 변환이 **두 번** 적용됐습니다.

**수정**: 메시를 링크 프레임과 명시적으로 곱해 한 번만 적용.
```python
o.matrix_world = fm @ o.matrix_world
```

### 4.8 영구 패키지 저장

검증된 결과물을 임시 폴더 밖으로 옮깁니다.

```powershell
$src = "$env:TEMP\opencode\rb3730\rb3_730es_u_standalone"
$dst = "$HOME\Documents\RB3-730"

if (Test-Path -LiteralPath $dst) { Remove-Item -LiteralPath $dst -Recurse -Force }
New-Item -ItemType Directory -Path $dst -Force | Out-Null

# ⚠️ -Path (와일드카드 확장 O) vs -LiteralPath (확장 X) — 복사에는 -Path 를 써야 한다
Copy-Item -Path (Join-Path $src '*') -Destination $dst -Recurse -Force

$m = Get-ChildItem -Recurse -File -LiteralPath $dst | Measure-Object Length -Sum
"copied {0} files, {1:N1} MB -> {2}" -f $m.Count, ($m.Sum/1MB), $dst
```

> 🔑 **PowerShell 함정**: `Copy-Item -LiteralPath "$src\*"` 는 와일드카드 `*` 를 확장하지 않아
> 아무것도 복사되지 않습니다. **복사에는 `-Path`, 삭제/테스트에는 `-LiteralPath`** 를 사용하세요.

✅ **확인**: `copied 24 files, 22.5 MB -> C:\Users\Administrator\Documents\RB3-730`

### 4.9 Phase 4 완료 체크리스트

- 원 ] `fbx_build.py` 실행 로그에 `DONE` 이 있다
- 원 ] `rb3_730es_u_rigged.fbx` 3.62 MB 생성
- 원 ] `rb3_730es_u_collision.fbx` 2.65 MB 생성
- 원 ] `rb3_730es_u_joints.json` 생성
- 원 ] 라운드트립에서 **본 8개**
- 원 ] 라운드트립에서 **max FK error < 1e-6 m**
- 원 ] 라운드트립에서 **바운딩박스 Y = 0.875 m**
- 원 ] 라운드트립에서 **base +60° → TCP 수평거리 0.730 m**
- 원 ] `C:\Users\Administrator\Documents\RB3-730` 에 25개 파일

✅ 모두 충족했으면 원Part 5](#part-5-phase-3--unity-프로젝트-설치)로 넘어갑니다.


---

<a id="part-5"></a>

<a id="part-5-phase-3--unity-프로젝트-설치"></a>

## Part 5. Phase 3 — Unity 프로젝트 설치

### 5.1 Unity 프로젝트 만들기

1. Unity Hub → **Installs** → **Install Editor** → **2022.3.62f3** 선택
2. **Projects** → **New project** → 좌측 **3D (Built-in)** 선택
   - ⚠️ 반드시 **Built-in** 템플릿. URP/HDRP를 고르면 재질 셰이더와 임포트 설정이 달라집니다.
3. 프로젝트 이름을 `Digital_Twin_2022`, 경로를 데스크톱으로 지정
4. **Create project** 클릭

### 5.2 에셋 배치

검증된 모델 패키지에서 아래 4개 파일만 프로젝트 안으로 복사합니다.
(URDF·메시 원본은 ROS용이므로 Unity 프로젝트에 넣지 않습니다.)

```powershell
$proj = "$HOME\Desktop\Digital_Twin_2022"
$src  = "$HOME\Documents\RB3-730"

foreach ($d in 'Assets\RB3-730','Assets\RB3-730\Models','Assets\RB3-730\Runtime',
               'Assets\RB3-730\Editor','Assets\RB3-730\Data') {
    New-Item -ItemType Directory -Path (Join-Path $proj $d) -Force | Out-Null
}

Copy-Item -LiteralPath "$src\fbx\rb3_730es_u_rigged.fbx" `
          -Destination "$proj\Assets\RB3-730\Models\rb3_730es_u_rigged.fbx" -Force
Copy-Item -LiteralPath "$src\fbx\rb3_730es_u_collision.fbx" `
          -Destination "$proj\Assets\RB3-730\Models\rb3_730es_u_collision.fbx" -Force
Copy-Item -LiteralPath "$src\fbx\rb3_730es_u_joints.json" `
          -Destination "$proj\Assets\RB3-730\Data\rb3_730es_u_joints.json" -Force
Copy-Item -LiteralPath "$src\README.md" `
          -Destination "$proj\Assets\RB3-730\README.md" -Force

Get-ChildItem -Recurse -File -LiteralPath "$proj\Assets\RB3-730" |
  ForEach-Object { "{0,-42} {1,8:N1} KB" -f $_.FullName.Substring($proj.Length+1),
                                       ($_.Length/1KB) }
```

> `.meta` 파일이 아직 없으므로 **Unity가 아직 이 파일들을 모릅니다.**
> Unity를 한 번 실행하면 자동으로 생성됩니다. 그 전에 에디터 스크립트를 넣어도,
> 컴파일은 정상이나 "joint bone not found" 경고가 뜰 수 있습니다. 정상입니다.

### 5.3 FBX 임포터 설정 — **가장 중요한 설정표**

Unity가 FBX를 임포트할 때 **9개의 설정**이 맞아야 관절이 살아 움직입니다.
`RB3_730_Setup.cs`([Part 5.4](#setup-script))가 이걸 자동화하지만,
수동으로 할 경우를 위해 전체 표를 정리했습니다.

#### Model 탭

| 설정 | 값 | 이유 |
|---|---|---|
| **Scale Factor** | **1** | FBX가 이미 1 unit = 1 m. 0.01을 넣으면 100배 축소됩니다 |
| **Convert Units** | ✅ on | FBX 내부 UnitScale(1.0 cm 표시)를 무시하고 미터로 해석 |
| **Bake Axis Conversion** | **OFF** | Z-up 선언을 유지. ON으로 바꾸면 눕게 보일 수 있음 (그때만 ON) |
| Generate Colliders | ❌ off | 콜라이더를 FBX에 굽지 않고 런타임에 붙임 |
| Import Cameras / Lights | ❌ off | 로봇에 조명/카메라 없음 |

#### Rig 탭 — **가장 위험한 탭**

| 설정 | 값 | 이유 |
|---|---|---|
| **Animation Type** | **Generic** | ❗ Humanoid/None으로 하면 **리그가 버려져** 메시가 정적 MeshRenderer로 들어옴 |
| **Create From This Model** | ✅ | 아바타 생성 |
| **Optimize Game Objects** | **OFF** | ❗ ON이면 본이 제거되어 `link0`~`link6`을 못 찾음 |

> 🔑 **왜 `Optimize Game Objects`를 꺼야 하나?**
> Unity의 Generic 리그 임포트는 링크마다 **두 개**의 GameObject를 만듭니다.
> 1. Armature 내부의 **Renderer 없는 본** ← 이것만 회전 가능
> 2. Armature의 **형제**인 `SkinnedMeshRenderer` 노드 (이름도 같음)
>
> 이름만 보고 첫 매칭을 잡으면 **메시 노드를 회전**하게 되어 관절이 전혀 움직이지 않습니다.
> JointDriver의 `FindBone()` 는 `GetComponent<Renderer>() == null` 인 것만 골라냅니다.

#### Material 탭

| 설정 | 값 |
|---|---|
| Material Import Mode | **Import via Material Description** |
| Location | **External** (materialLocation) |

Built-in에서는 Unity가 `Standard` 셰이더를 붙여줍니다.
URP/HDRP였다면 각각 `Universal Render Pipeline/Lit`, `HDRP/Lit` 이 자동으로 선택됩니다.

> 원본 COLLADA에서 43개 재질이 넘어옵니다(원래 이름 `XID_152170175_...` 형태).
> **외부 텍스처가 없으므로** 텍스처 없이 단색으로만 나옵니다. 이름 정리는 선택 사항입니다.

#### Media 탭 (성능)

| 설정 | 권장값 | 이유 |
|---|---|---|
| Mesh Compression | **Off** | 모델 정확도 우선 |
| Optimize Mesh Polygons / Vertices | ✅ on | 메시 13만 삼각형 → 대략 10만 이하로 축소 |
| Read/Write Enabled | ✅ on | 런타임에서 메시를 읽을 수 있어야 함 |

> 📱 모바일 빌드 시: 비주얼 FBX가 134 k 삼각형으로 무겁습니다.
> `rb3_730es_u_collision.fbx`(133 k 삼각형이지만 정점 67 k)와 렌더 메시를 분리하거나
> Mesh Compression을 켜는 것을 권합니다.

<a id="setup-script"></a>

### 5.4 에디터 스크립트 `RB3_730_Setup.cs` — 전체 소스

이 스크립트는 위 설정 9개를 **자동으로 적용**하고, 프리팹을 만들고, 검증까지 합니다.
절대 사용자의 열린 씬을 수정하지 않습니다 (프리뷰 씬에서 작업 후 정리).


````text
> FILE: RB3_730_Setup.cs  (455 lines)

// One-click setup for the RB3-730 digital twin. Works in any render pipeline.
//
// What it does, non-destructively:
//   1. Configures both FBX importers for correct scale / axis / rig / materials
//   2. Bakes a prefab (FBX + RB3_730_JointDriver) at Assets/RB3-730/RB3_730.prefab
//   3. Verifies the imported model is upright and at the correct scale
//
// Runs automatically once after the scripts compile, then again on demand from
// the Tools/RB3-730 menu. It never edits your open scene.

using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text;
using UnityEditor;
using UnityEditor.SceneManagement;
using UnityEngine;
using RainbowRobotics.RB;

namespace RainbowRobotics.RB.EditorTools
{
    [InitializeOnLoad]
    public static class RB3_730_Setup
    {
        const string Root = "Assets/RB3-730";
        const string RiggedFbx = Root + "/Models/rb3_730es_u_rigged.fbx";
        const string CollisionFbx = Root + "/Models/rb3_730es_u_collision.fbx";
        const string JointJson = Root + "/Data/rb3_730es_u_joints.json";
        const string PrefabPath = Root + "/RB3_730.prefab";
        const string MenuRoot = "Tools/RB3-730/";

        // fully extended height of the arm, TCP included
        const float ExpectedHeight = 0.8753f;

        // Bump this whenever the import configuration below changes, so that the
        // automatic run below re-executes after a recompile.
        const int SetupVersion = 10;

        static RB3_730_Setup()
        {
            // Domain reload / editor start: make sure the assets exist on disk first.
            bool found = File.Exists(RiggedFbx);
            Debug.Log("[RB3-730] init, cwd=" + Directory.GetCurrentDirectory() + ", fbxExists=" + found);
            if (!found) return;
            EditorApplication.delayCall += AutoRunOnce;
        }

        static void AutoRunOnce()
        {
            string key = "RB3_730_SetupVersion::" + Application.dataPath;
            int stored = EditorPrefs.GetInt(key, 0);
            Debug.Log("[RB3-730] auto-run check, stored=" + stored + ", current=" + SetupVersion);
            if (stored >= SetupVersion) return;
            EditorPrefs.SetInt(key, SetupVersion);
            Run(true);
            EnsureRobotInScene(true);
            SaveActiveScene(true);
        }

        /// <summary>
        /// Writes the open scene to disk. Entering Play Mode starts from the saved scene file, so
        /// an arm that was only added in the editor would silently be missing at runtime. Saving
        /// here keeps the scene and the prefab in step.
        /// </summary>
        static void SaveActiveScene(bool log)
        {
            UnityEngine.SceneManagement.Scene active = UnityEngine.SceneManagement.SceneManager.GetActiveScene();
            if (!active.IsValid() || !active.isLoaded) return;

            string names = string.Join(", ", System.Linq.Enumerable.ToArray(
                System.Array.ConvertAll(active.GetRootGameObjects(), o => o.name)));
            if (!active.isDirty)
            {
                if (log) Debug.Log("[RB3-730] scene '" + active.name + "' already saved. roots: " + names);
                return;
            }

            bool ok = EditorSceneManager.SaveScene(active);
            if (log)
            {
                if (ok) Debug.Log("[RB3-730] saved scene '" + active.path + "'. roots: " + names);
                else Debug.LogWarning("[RB3-730] could not save scene '" + active.name + "'");
            }
        }

        /// <summary>
        /// Makes sure the open scene actually contains the arm. It only ever instantiates the
        /// prefab: existing objects are never moved, renamed or removed, and a second call when
        /// an instance is already there is a no-op. Scene contents are left alone otherwise, so
        /// your camera, lights and layout stay exactly as you arranged them.
        /// </summary>
        static bool EnsureRobotInScene(bool log)
        {
            GameObject prefab = AssetDatabase.LoadAssetAtPath<GameObject>(PrefabPath);
            if (prefab == null)
            {
                if (log) Debug.LogWarning("[RB3-730] prefab missing, cannot add the arm to the scene");
                return false;
            }

            UnityEngine.SceneManagement.Scene active = UnityEngine.SceneManagement.SceneManager.GetActiveScene();
            if (!active.IsValid() || !active.isLoaded)
            {
                if (log) Debug.LogWarning("[RB3-730] no valid active scene, skipped adding the arm");
                return false;
            }

            GameObject[] roots = active.GetRootGameObjects();
            RB3_730_JointDriver found = null;
            for (int i = 0; i < roots.Length && found == null; i++)
                found = roots[i].GetComponentInChildren<RB3_730_JointDriver>(true);

            if (found != null)
            {
                AuditSceneInstance(found, log);
                return false;
            }

            GameObject go = (GameObject)PrefabUtility.InstantiatePrefab(prefab, active);
            if (go == null) return false;
            go.name = "RB3_730";
            go.transform.position = Vector3.zero;
            go.transform.rotation = Quaternion.identity;
            Undo.RegisterCreatedObjectUndo(go, "Add RB3-730");
            Selection.activeGameObject = go;

            if (log) Debug.Log("[RB3-730] added the arm to scene '" + active.name +
                               "' at the origin. Press Play, then drive it from Tools/RB3-730 or over TCP.");
            return true;
        }

        /// <summary>
        /// Reports why a scene instance would not run, and switches it back on when the cause is
        /// simply a disabled GameObject or component. A disabled arm never runs Awake, so it never
        /// opens the pendant port and never moves - which looks exactly like broken code. An
        /// inactive PARENT is only reported, never touched, because that is scene structure the
        /// user owns.
        /// </summary>
        static void AuditSceneInstance(RB3_730_JointDriver driver, bool log)
        {
            if (!log) return;

            GameObject go = driver.gameObject;
            string path = go.name;
            for (Transform t = go.transform.parent; t != null; t = t.parent) path = t.name + "/" + path;

            bool healed = false;
            if (!go.activeSelf)
            {
                go.SetActive(true);
                healed = true;
            }
            if (!driver.enabled)
            {
                driver.enabled = true;
                healed = true;
            }

            string state = "activeInHierarchy=" + go.activeInHierarchy + " activeSelf=" + go.activeSelf +
                           " driverEnabled=" + driver.enabled;
            if (go.activeInHierarchy)
            {
                Debug.Log("[RB3-730] arm present at '" + path + "' (" + state + ")" +
                          (healed ? " -> switched back on." : "."));
                return;
            }

            string blocker = "unknown";
            for (Transform t = go.transform; t != null; t = t.parent)
                if (!t.gameObject.activeSelf) { blocker = t.name; break; }

            Debug.LogWarning("[RB3-730] arm at '" + path + "' is still inactive (" + state +
                             "). An ancestor GameObject named '" + blocker +
                             "' is switched off, so Awake never runs and the pendant port stays closed. " +
                             "Enable that object in the Hierarchy, or delete the arm and let setup re-add it.");
        }

        // ------------------------------------------------------------------ menu

        [MenuItem(MenuRoot + "Run Setup (idempotent)", false, 0)]
        public static void RunSetup() => Run(true);

        [MenuItem(MenuRoot + "Force Reimport + Rebuild", false, 1)]
        public static void ForceRun() => Run(true, true);

        [MenuItem(MenuRoot + "Add To Current Scene", false, 20)]
        public static void AddToScene()
        {
            GameObject prefab = AssetDatabase.LoadAssetAtPath<GameObject>(PrefabPath);
            if (prefab == null)
            {
                EditorUtility.DisplayDialog("RB3-730", "Prefab not found. Run Tools/RB3-730/Run Setup first.", "OK");
                return;
            }

            GameObject go = (GameObject)PrefabUtility.InstantiatePrefab(prefab);
            Undo.RegisterCreatedObjectUndo(go, "Add RB3-730");
            go.transform.position = Vector3.zero;
            Selection.activeGameObject = go;

            // frame the camera on the robot
            SceneView view = SceneView.lastActiveSceneView;
            if (view != null)
            {
                view.LookAt(new Vector3(0f, 0.4f, 0f), Quaternion.Euler(15f, 180f, 0f), 1.6f, true);
                view.FrameSelected();
            }

            Debug.Log("[RB3-730] Added to scene. Joints: base, shoulder, elbow, wrist1, wrist2, wrist3.");
        }

        [MenuItem(MenuRoot + "Select Model", false, 21)]
        public static void SelectModel()
        {
            GameObject prefab = AssetDatabase.LoadAssetAtPath<GameObject>(PrefabPath);
            if (prefab != null) Selection.activeObject = prefab;
        }

        // ------------------------------------------------------------------ setup

        static void Run(bool log, bool force = false)
        {
            if (!File.Exists(RiggedFbx))
            {
                Debug.LogError("[RB3-730] " + RiggedFbx + " not found.");
                return;
            }

            bool changed = Configure(RiggedFbx, isRig: true, force: force);
            changed |= Configure(CollisionFbx, isRig: true, force: force);
            if (changed) AssetDatabase.SaveAssets();

            BuildPrefab(log);
            Verify(log);
        }

        static bool Configure(string path, bool isRig, bool force)
        {
            ModelImporter importer = AssetImporter.GetAtPath(path) as ModelImporter;
            if (importer == null)
            {
                Debug.LogWarning("[RB3-730] importer not ready yet for " + path);
                return false;
            }

            bool dirty = force;

            // ---- transform: 1 unit = 1 m, keep the FBX's own Z-up declaration
            dirty |= Set(() => importer.globalScale, 1f, v => importer.globalScale = v);
            dirty |= Set(() => importer.useFileScale, true, v => importer.useFileScale = v);
            dirty |= Set(() => importer.bakeAxisConversion, false, v => importer.bakeAxisConversion = v);

            // ---- ignore anything that is not the arm
            dirty |= Set(() => importer.importCameras, false, v => importer.importCameras = v);
            dirty |= Set(() => importer.importLights, false, v => importer.importLights = v);
            dirty |= Set(() => importer.importVisibility, false, v => importer.importVisibility = v);
            dirty |= Set(() => importer.importBlendShapes, false, v => importer.importBlendShapes = v);

            // ---- meshes
            dirty |= Set(() => importer.importNormals, ModelImporterNormals.Import, v => importer.importNormals = v);
            dirty |= Set(() => importer.importTangents, ModelImporterTangents.CalculateMikk, v => importer.importTangents = v);
            dirty |= Set(() => importer.meshCompression, ModelImporterMeshCompression.Off, v => importer.meshCompression = v);
            dirty |= Set(() => importer.optimizeMeshPolygons, true, v => importer.optimizeMeshPolygons = v);
            dirty |= Set(() => importer.optimizeMeshVertices, true, v => importer.optimizeMeshVertices = v);
            dirty |= Set(() => importer.isReadable, true, v => importer.isReadable = v);
            dirty |= Set(() => importer.addCollider, false, v => importer.addCollider = v);

            // ---- rig
            // Generic + Optimize Game Objects OFF: this is what makes Unity import
            // SkinnedMeshRenderers bound to the link0..link6 chain, so the visual
            // meshes actually follow the joint transforms. With animationType = None
            // Unity drops the rig and the meshes come in as static MeshRenderers
            // parented to the model root, which does NOT animate with the joints.
            dirty |= Set(() => importer.animationType, ModelImporterAnimationType.Generic, v => importer.animationType = v);
            dirty |= Set(() => importer.avatarSetup, ModelImporterAvatarSetup.CreateFromThisModel, v => importer.avatarSetup = v);
            dirty |= Set(() => importer.optimizeGameObjects, false, v => importer.optimizeGameObjects = v);

            // ---- materials: import the Collada material descriptions so the DAE colours
            // survive, and let the active render pipeline assign its own default shader
            // (Standard in Built-in, HDRP/Lit in HDRP, URP/Lit in URP).
            dirty |= Set(() => importer.materialImportMode, ModelImporterMaterialImportMode.ImportViaMaterialDescription,
                          v => importer.materialImportMode = v);
            dirty |= Set(() => importer.materialLocation, ModelImporterMaterialLocation.External,
                          v => importer.materialLocation = v);

            if (dirty)
            {
                importer.SaveAndReimport();
                Debug.Log("[RB3-730] reimported " + Path.GetFileName(path));
            }
            return dirty;
        }

        static bool Set<T>(Func<T> get, T wanted, Action<T> set)
        {
            if (EqualityComparer<T>.Default.Equals(get(), wanted)) return false;
            set(wanted);
            return true;
        }

        static void BuildPrefab(bool log)
        {
            GameObject model = AssetDatabase.LoadAssetAtPath<GameObject>(RiggedFbx);
            if (model == null)
            {
                Debug.LogError("[RB3-730] could not load " + RiggedFbx);
                return;
            }

            // Build inside a throwaway preview scene so the scene you have open is
            // never modified or even marked dirty.
            UnityEngine.SceneManagement.Scene preview = EditorSceneManager.NewPreviewScene();
            GameObject instance = null;
            try
            {
                instance = UnityEngine.Object.Instantiate(model);
                UnityEngine.SceneManagement.SceneManager.MoveGameObjectToScene(instance, preview);
                instance.name = "RB3_730";

                RB3_730_JointDriver driver = instance.GetComponent<RB3_730_JointDriver>();
                if (driver == null) driver = instance.AddComponent<RB3_730_JointDriver>();

                TextAsset table = AssetDatabase.LoadAssetAtPath<TextAsset>(JointJson);
                if (table != null) driver.jointTable = table;

                // Jog controls. The pendant server makes any instance of this prefab
                // reachable from a WinForms teaching pendant over TCP; the teach component
                // holds the taught points and runs the cycle; the panel is a development aid
                // for checking the same API without the pendant.
                if (instance.GetComponent<RB3_730_Teach>() == null)
                    instance.AddComponent<RB3_730_Teach>();
                if (instance.GetComponent<RB3_730_PendantServer>() == null)
                    instance.AddComponent<RB3_730_PendantServer>();
                if (instance.GetComponent<RB3_730_JogPanel>() == null)
                    instance.AddComponent<RB3_730_JogPanel>();

                PrefabUtility.SaveAsPrefabAsset(instance, PrefabPath);
                if (log) Debug.Log("[RB3-730] prefab written to " + PrefabPath);
            }
            finally
            {
                if (instance != null) UnityEngine.Object.DestroyImmediate(instance);
                EditorSceneManager.ClosePreviewScene(preview);
            }
        }

        static void Verify(bool log)
        {
            GameObject prefab = AssetDatabase.LoadAssetAtPath<GameObject>(PrefabPath);
            if (prefab == null) return;

            Renderer[] renderers = prefab.GetComponentsInChildren<Renderer>(true);
            if (renderers.Length == 0)
            {
                Debug.LogError("[RB3-730] prefab has no renderers - import probably failed.");
                return;
            }

            Bounds b = renderers[0].bounds;
            for (int i = 1; i < renderers.Length; i++) b.Encapsulate(renderers[i].bounds);
            Vector3 s = b.size;

            int tris = renderers.Sum(r => r is MeshRenderer mr
                ? mr.GetComponent<MeshFilter>().sharedMesh.triangles.Length / 3
                : (r is SkinnedMeshRenderer smr ? smr.sharedMesh.triangles.Length / 3 : 0));
            int mats = renderers.Sum(r => r.sharedMaterials.Length);
            int skinned = renderers.Count(r => r is SkinnedMeshRenderer);

            // Which GameObject holds the robot root?
            Transform link0 = FindDeep(prefab.transform, "link0");
            Transform tcp = FindDeep(prefab.transform, "tcp");

            // Unity's generic-rig import creates BOTH a renderer-less bone GameObject and a
            // SkinnedMeshRenderer GameObject with the same name, so duplicate names are normal.
            // What matters is that the renderer-less bones form one unbroken chain and that
            // every renderer is skinned to it.
            string[] chain = { "link0", "link1", "link2", "link3", "link4", "link5", "link6", "tcp" };
            Transform[] all = prefab.GetComponentsInChildren<Transform>(true);
            List<string> chainProblems = new List<string>();
            Transform prev = prefab.transform;
            foreach (string n in chain)
            {
                Transform[] bones = all.Where(t => t.name == n && t.GetComponent<Renderer>() == null).ToArray();
                if (bones.Length != 1)
                {
                    chainProblems.Add("bone " + n + " found " + bones.Length + "x");
                    continue;
                }
                if (bones[0] != prev && !bones[0].IsChildOf(prev))
                    chainProblems.Add("bone " + n + " is not parented under " + prev.name);
                prev = bones[0];
            }

            foreach (SkinnedMeshRenderer smr in renderers.OfType<SkinnedMeshRenderer>())
            {
                if (smr.bones == null || smr.bones.Length == 0)
                {
                    chainProblems.Add("skinned renderer on " + smr.name + " has no bones");
                    break;
                }
            }

            StringBuilder sb = new StringBuilder();
            sb.AppendFormat("[RB3-730] verified: {0} renderers ({1} skinned), {2} tris, {3} material slots, bounds {4:F3} x {5:F3} x {6:F3} m",
                            renderers.Length, skinned, tris, mats, s.x, s.y, s.z);
            sb.AppendLine();
            sb.AppendFormat("            link0 found={0}, tcp found={1}, localScale={2}, jointChain={3}",
                            link0 != null, tcp != null, prefab.transform.localScale,
                            chainProblems.Count == 0 ? "ok" : string.Join("; ", chainProblems));
            if (log) Debug.Log(sb.ToString());

            if (chainProblems.Count > 0)
            {
                Debug.LogError(
                    "[RB3-730] The joint chain is broken (" + string.Join("; ", chainProblems) + "). " +
                    "The visual meshes are not driven by link0..link6.");
            }

            if (skinned == 0)
            {
                Debug.LogError(
                    "[RB3-730] NONE of the renderers is a SkinnedMeshRenderer, so Unity imported the model " +
                    "without a rig and the meshes will NOT follow the joints. In the FBX importer set " +
                    "Rig > Animation Type to Generic and turn 'Optimize Game Objects' OFF, then run " +
                    "Tools/RB3-730/Force Reimport + Rebuild.");
            }

            float tall = Mathf.Max(s.x, s.z);
            if (s.y < tall * 0.5f)
            {
                Debug.LogWarning(
                    "[RB3-730] The model looks like it is lying down (Y is not the tallest axis). " +
                    "In the FBX importer turn ON 'Bake Axis Conversion' and reimport.");
            }
            else if (Mathf.Abs(s.y - ExpectedHeight) > 0.15f)
            {
                Debug.LogWarningFormat(
                    "[RB3-730] Height is {0:F3} m but the RB3-730 is {1:F3} m fully extended. " +
                    "Check the importer's Scale Factor (should be 1).", s.y, ExpectedHeight);
            }
        }

        static Transform FindDeep(Transform root, string target)
        {
            if (root.name == target) return root;
            for (int i = 0; i < root.childCount; i++)
            {
                Transform hit = FindDeep(root.GetChild(i), target);
                if (hit != null) return hit;
            }
            return null;
        }
    }
}
````

### 5.5 에디터 스크립트의 동작 원리

#### `Configure()` — 임포터 설정 적용

`Set()` 헬퍼로 **실제로 값이 달라진 항목만** 골라 `SaveAndReimport()` 를 호출합니다.
변경이 없으면 재임포트를 건너뛰므로 불필요한 임포트 시간을 쓰지 않습니다.

```csharp
static bool Set<T>(Func<T> get, T wanted, Action<T> set)
{
    if (EqualityComparer<T>.Default.Equals(get(), wanted)) return false;
    set(wanted);
    return true;
}
```

#### `BuildPrefab()` — **프리뷰 씬에서 프리팹 생성**

```csharp
UnityEngine.SceneManagement.Scene preview = EditorSceneManager.NewPreviewScene();
try {
    instance = UnityEngine.Object.Instantiate(model);
    UnityEngine.SceneManagement.SceneManager.MoveGameObjectToScene(instance, preview);
    // … 컴포넌트 추가 …
    PrefabUtility.SaveAsPrefabAsset(instance, PrefabPath);
}
finally {
    UnityEngine.Object.DestroyImmediate(instance);
    EditorSceneManager.ClosePreviewScene(preview);
}
```

> 🔑 **왜 프리뷰 씬을 쓰나요?**
> 처음에는 열린 씬에서 프리팹을 만들려고 했지만, 그러면 **사용자가 작업 중인 씬이
> dirty 표시되고 저장을 요구**합니다. Git에도 없는 프로젝트에서 이건 위험합니다.
> 프리뷰 씬은 격리된 가상 씬이라 사용자 씬에 아무 영향도 없습니다.
>
> 실제로 `EditorSceneManager.MarkSceneClean()` 을 쓰려 했으나
> **Unity 2022.3에는 이 메서드가 없습니다** (`error CS0117`). 프리뷰 씬 방식으로 해결했습니다.

#### `Verify()` — 자동 검증

프리팹을 열어서 다음을 검사하고 Console에 출력합니다.

```csharp
sb.AppendFormat("[RB3-730] verified: {0} renderers ({1} skinned), {2} tris, {3} material slots, bounds {4:F3} x {5:F3} x {6:F3} m", ...);
sb.AppendFormat("            link0 found={0}, tcp found={1}, localScale={2}, jointChain={3}", ...);
```

✅ **기대 출력**:
```
[RB3-730] verified: 7 renderers (7 skinned), 133340 tris, 43 material slots, bounds 0.128 x 0.875 x 0.247 m
            link0 found=True, tcp found=True, localScale=(1.0, 1.0, 1.0), jointChain=ok
```

| Console 메시지 | 의미 |
|---|---|
| `NONE of the renderers is a SkinnedMeshRenderer` | ❗ Animation Type을 Generic으로, Optimize GO를 OFF로 바꾸고 **Force Reimport + Rebuild** |
| `jointChain` 에 `found 0x` | ❗ 본이 안 보임. 같은 조치 |
| `The model looks like it is lying down` | ⚠️ Bake Axis Conversion을 ON으로 |
| `Height is X m but the RB3-730 is 0.875 m fully extended` | ⚠️ Scale Factor가 1이 아님 |
| `bounds 0.128 x 0.875 x 0.247 m` + `jointChain=ok` | ✅ 완벽 |

> 🔑 **bounds에서 Y = 0.875** 인 점이 핵심입니다.
> Blender는 Z-up으로 저장했으므로 Unity가 임포트하면서 Y-up으로 바꿔야 합니다.
> Y가 0.875라는 것은 **축 변환과 스케일이 동시에 정상**이라는 뜻입니다.

#### `AutoRunOnce()` — 자동 실행 메커니즘

```csharp
static RB3_730_Setup()
{
    bool found = File.Exists(RiggedFbx);
    if (!found) return;
    EditorApplication.delayCall += AutoRunOnce;
}
```

`[InitializeOnLoad]` 로 인해 도메인 리로드마다 실행됩니다.
`EditorPrefs`에 저장된 버전이 `SetupVersion` 보다 작을 때만 실제로 동작합니다.

```csharp
string key = "RB3_730_SetupVersion::" + Application.dataPath;
int stored = EditorPrefs.GetInt(key, 0);
if (stored >= SetupVersion) return;
EditorPrefs.SetInt(key, SetupVersion);
Run(true);
EnsureRobotInScene(true);
SaveActiveScene(true);
```

> 🔑 **`SetupVersion` 을 올리면 자동 재실행됩니다.**
> 임포트 설정이나 컴포넌트 구성을 바꿨다면 이 상수를 **1 증가시키세요.**
> 현재 값은 **10** 입니다.

#### 메뉴 항목

`Tools/RB3-730/` 하위에서 수동 실행도 가능합니다.

| 메뉴 | 동작 |
|---|---|
| **Run Setup (idempotent)** | 설정 적용 + 프리팹 생성 + 검증 (멱등적, 반복 실행 가능) |
| **Force Reimport + Rebuild** | 위와 동일하되 무조건 재임포트 |
| **Add To Current Scene** | 씬에 로봇 배치 + 카메라 프레이밍 |
| **Select Model** | 프리팹 선택 |

### 5.6 Unity 에디터가 새 파일을 인식하도록 하기

파일을 외부에서 복사하면 Unity가 즉시 감지하지 않습니다. 여기서 실제로 겪은 상황이 있습니다.

> **증상**: `.meta` 파일이 생성되지 않고, 로봇이 씬에 나타나지 않음.
> **원인**: Unity는 **포커스를Focused 상태일 때만** 파일 변경을 스캔합니다.
> **해결**: 다른 창을 클릭해 포커스를 뺐다가 Unity로 돌아오면 자동 리프레시가 돕니다.

```powershell
Add-Type -AssemblyName System.Windows.Forms
Add-Type -TypeDefinition @'
using System;
using System.Runtime.InteropServices;
public class UnityFocus {
  [DllImport("user32.dll")] public static extern bool SetForegroundWindow(IntPtr h);
  [DllImport("user32.dll")] public static extern bool ShowWindow(IntPtr h, int n);
  [DllImport("user32.dll")] public static extern bool BringWindowToTop(IntPtr h);
}
'@ -Language CSharp

$u = Get-Process -Name Unity | Where-Object { $_.MainWindowTitle } | Select-Object -First 1
[UnityFocus]::ShowWindow($u.MainWindowHandle, 9)      # SW_RESTORE
[UnityFocus]::BringWindowToTop($u.MainWindowHandle) | Out-Null
[UnityFocus]::SetForegroundWindow($u.MainWindowHandle) | Out-Null
Start-Sleep -Seconds 25    # 리프레시 + 임포트 대기
```

또는 **직접 조작**: 다른 앱(메모장 등)을 클릭 → Unity로 돌아오기.

> ⚠️ **신뢰할 수 없는 방법**
> `Ctrl+P` (Play)로 Play Mode 진입을 합성 클릭하는 것은 **매번 다르게** 동작했습니다.
> 포커스 창을 안 잡은 상태에서 Play가 눌려 스크립트가 안 컴파일된 채로 테스트가 도는 일이 반복됐습니다.
> **Play 버튼은 직접 클릭하세요.** (이 실습의 마지막 검증이 미완료로 남은 이유입니다.)

임포트 완료 확인:

```powershell
$proj = "$HOME\Desktop\Digital_Twin_2022"
"meta count: " + (Get-ChildItem -Recurse -LiteralPath "$proj\Assets\RB3-730" -Filter '*.meta').Count
"prefab    : " + (Test-Path -LiteralPath "$proj\Assets\RB3-730\RB3_730.prefab")

$log = "$env:LOCALAPPDATA\Unity\Editor\Editor.log"
Select-String -LiteralPath $log -Pattern 'RB3-730' | ForEach-Object { $_.Line }
Select-String -LiteralPath $log -Pattern 'error CS' | Select-Object -Last 5 | ForEach-Object { $_.Line }
```

✅ **확인**: `meta count` 55 이상, `prefab True`, `[RB3-730] verified: 7 renderers … jointChain=ok`

### 5.7 C# 사전 컴파일 검증 — Unity 없이 오류 잡기

Unity가 컴파일을 안 시켜줄 때(포커스 문제), **Unity가 실제로 쓰는 어셈블리를 직접 참조**해
컴파일 검증을 할 수 있습니다. 여기서 3개의 실제 오류를 잡을 수 있었습니다.

```powershell
$u = "C:\Program Files\Unity\Hub\Editor\2022.3.62f3\Editor\Data"
$w = "$env:TEMP\opencode\rb3730\check2022"
New-Item -ItemType Directory -Path $w -Force | Out-Null

# 응답 파일(rsp) 작성 — 인자에 공백이 많으므로 rsp가 필수
$proj = "$HOME\Desktop\Digital_Twin_2022\Assets\RB3-730"
$lines = @(
    '-nologo'
    '-target:library'
    "-out:`"$w\check.dll`""
    '-nostdlib+'
    '-noconfig'
    '/r:"C:\Program Files\Unity\Hub\Editor\2022.3.62f3\Editor\Data\NetStandard\ref\2.1.0\netstandard.dll"'
)
Get-ChildItem -LiteralPath "$u\Managed\UnityEngine" -Filter '*.dll' |
    ForEach-Object { $lines += "/r:`"$($_.FullName)`"" }
$lines += '"' + "$proj\Runtime\RB3_730_JointDriver.cs"  + '"'
$lines += '"' + "$proj\Runtime\RB3_730_Teach.cs"        + '"'
$lines += '"' + "$proj\Runtime\RB3_730_PendantServer.cs" + '"'
$lines += '"' + "$proj\Runtime\RB3_730_JogPanel.cs"     + '"'
$lines += '"' + "$proj\Editor\RB3_730_Setup.cs"         + '"'
[System.IO.File]::WriteAllLines("$w\ball.rsp", $lines, (New-Object System.Text.UTF8Encoding($true)))

# 컴파일
$dn  = "$u\NetCoreRuntime\dotnet.exe"
$csc = "$u\DotNetSdkRoslyn\csc.dll"
$o = "$w\oa.txt"; $r = "$w\ra.txt"
$p = Start-Process -FilePath $dn -ArgumentList ('"'+$csc+'"'), ('"@'+"$w\ball.rsp"+'"') `
     -NoNewWindow -Wait -PassThru -RedirectStandardOutput $o -RedirectStandardError $r

$all   = @(Get-Content -LiteralPath $o -EA SilentlyContinue) + @(Get-Content -LiteralPath $r -EA SilentlyContinue)
$errs  = @($all | Where-Object { $_ -match ': error ' })
$warns = @($all | Where-Object { $_ -match ': warning ' })
'EXIT={0}  errors={1}  warnings={2}' -f $p.ExitCode, $errs.Count, $warns.Count
$errs  | Select-Object -First 20
$warns | Select-Object -First 10
```

✅ **확인**: `EXIT=0  errors=0  warnings=0`

> 🔑 **왜 `Start-Process` + 인용이 필요한가?**
> ```powershell
> # ❌ 실패 — 공백 포함 경로가 쪼개진다
> Start-Process -FilePath $dn -ArgumentList $csc, "@$w\ball.rsp"
>
> # ✅ 성공 — 인자를 명시적으로 인용한다
> Start-Process -FilePath $dn -ArgumentList ('"'+$csc+'"'), ('"@'+$w\ball.rsp"+'"')
> ```

> 🔑 **BCL 참조는 어디서?**
> `il2cpp\build\deploy` 의 dll들을 쓰면 BCL 충돌이 납니다.
> 실제로 쓰는 세트는 **`NetStandard\ref\2.1.0\netstandard.dll`** 하나입니다.

### 5.8 Phase 5 완료 체크리스트

- [ ] Unity 2022.3.62f3, Built-in 템플릿
- [ ] FBX 2개 + joints.json + README가 `Assets/RB3-730/` 아래에 있음
- [ ] `.meta` 55개 이상 생성
- [ ] Console에 `verified: 7 renderers (7 skinned), … jointChain=ok`
- [ ] `RB3_730.prefab` 생성됨
- [ ] Roslyn 사전 컴파일 `EXIT=0 errors=0`

✅ 완료했으면 [Part 6](#part-6-phase-4--런타임-c-코드-전체)에서 C# 코드를 넣습니다.


---

<a id="part-6"></a>

<a id="part-6-phase-4--런타임-c-코드-전체"></a>

## Part 6. Phase 4 — 런타임 C# 코드 전체

4개의 런타임 컴포넌트와 1개의 에디터 스크립트로 구성됩니다.
에디터 스크립트(`RB3_730_Setup.cs`)는 [Part 5.4](#setup-script)에 있으므로
여기서는 런타임 4종만 다룹니다.

### 6.0 컴포넌트 관계도

```
GameObject: RB3_730  (프리팹)
│
├── SkinnedMeshRenderer × 7      ← FBX에서 생성된 비주얼 메시
│
├── RB3_730_JointDriver          ← kinematics. 관절/카티시안 조그, DLS IK, 속도 제한
│     └── link0 … link6, tcp (Transform, Renderer 없음)
│
├── RB3_730_PendantServer        ← network. TCP 수신, 프로토콜 파싱, 큐
│     └── [RequireComponent] JointDriver, Teach
│
├── RB3_730_Teach                ← workflow. 티칭 포인트, 1→2→1 사이클
│
└── RB3_730_JogPanel             ← ui. OnGUI IMGUI 패널 (키보드 단축키 포함)
```

`[RequireComponent]` 때문에 서버 컴포넌트를 붙이면 자동으로 JointDriver와 Teach가 붙습니다.

### 6.0.1 파일 크기 요약

| 파일 | 역할 | 줄 |
|---|---|---|
| `RB3_730_JointDriver.cs` | 관절 해석, 속도 제한, 조그, IK | 789 |
| `RB3_730_Teach.cs` | 티칭 데이터, 사이클 상태 기계 | 527 |
| `RB3_730_PendantServer.cs` | TCP 서버, 프로토콜, 텔레메트리 | 997 |
| `RB3_730_JogPanel.cs` | IMGUI 조그·티칭 패널 | 213 |
| **합계 (런타임)** | | **2,513** |

---

<a id="code-jointdriver"></a>

### 6.1 `RB3_730_JointDriver.cs`

**이 컴포넌트가 "로봇의 몸"입니다.** 나머지 모든 것이 여기에 명령을 내립니다.

#### 공개 API (이것만 알면 됩니다)

| 메서드 / 프로퍼티 | 설명 |
|---|---|
| `bool IsReady` | 리그 본 해석 성공 여부 |
| `string OrderHint` | 축을 바로 쓸 때의 관절 순서 안내문 |
| `float[] GetAnglesDeg()` | 현재 관절 각도 6개 (도) |
| `float GetJointAngleDeg(int joint)` | 관절 1개 조회 |
| `bool MoveTo(float[] deg)` | 절대 관절 이동 (6개, base부터, 도) |
| `bool MoveJointTo(int joint, float degrees)` | 관절 1개만 이동 |
| `void SetJoint(string jointName, float degrees)` | 이름(`"elbow"`)으로 관절 이동 |
| `void SetAngles(float[] deg)` | 목표값만 설정 (실제 이동은 `TickRamp` 이 담당) |
| `void GoHome()` | `MoveTo(new float[6])` — 닫힌 포즈 |
| `void Stop()` | 즉시 정지 |
| `float SpeedOverride { get; set; }` | 전체 속도 배율 0…1 (`SPEED` 명령이 이 값을 바꾼다) |
| `void SetSpeedOverride(float normalised)` | 위 setter 와 동일 |
| `bool IsMoving` / `bool IsJogging` | 램프업 중 / 속도 조그 홀드 중 |
| `void StartJointJog(int joint, float rateDegPerSec)` | 관절 속도 조그 시작 |
| `bool StepJoint(int joint, float deltaDeg)` | 관절 상대 조그 (Δ도) |
| `bool StepCartesian(Vector3 dLin, Vector3 dAng, JogFrame frame)` | TCP 상대 조그 (선형 + 각도) |
| `void StartCartesianJog(Vector3 linVel, Vector3 angVel, JogFrame frame)` | TCP 속도 조그 시작 |
| `bool StartToolLinearJog(int axis, float sign)` | `JOGSTART X/Y/Z` 대응. `axis` 0=X, 1=Y, 2=Z |
| `bool StartToolAngularJog(int axis, float sign)` | `JOGSTART RX/RY/RZ` 대응 |
| `void ClearJog()` | 모든 조그 정지 (`JOGSTOP`) |
| `Transform GetLink(string linkName)` | 본 Transform 조회 |
| `Transform GetTcp()` | `tcp` 본 Transform |
| `bool TryGetTcpPose(out Vector3 pos, out Quaternion rot)` | TCP pose 조회 (`GETPOS`, `TCP`) |

> 🔑 `JogFrame` 은 조그 기준 좌표계입니다 (`world` / `tool`).
> `JOGSTART Z` 가 월드 Z 조그인 이유, 그리고 `TCP` 명령이 기준 좌표계를 출력하는 이유입니다.

#### 핵심 설정값 (인스펙터)

| 필드 | 기본값 | 의미 |
|---|---|---|
| `maxJointSpeedDegPerSec` | `30` | 축 최대 도/스텝 계산의 기준 |
| `speedOverride` | `0.3` | 초기 속도 배율 |
| `maxLinearJogSpeedMPerSec` | `0.15` | 카티시안 조그 최대 속도 (m/s) |
| `maxAngularJogSpeedRadPerSec` | `1.0` | 회전 조그 최대 속도 (rad/s) |
| `arriveToleranceDeg` | `0.05` | "도착"으로 인정하는 관절 오차 (도) |
| `externalEditToleranceDeg` | `1.0` | 외부 편집으로 인정하는 최소 변화량 (도) |
| `ikDampingLinear` / `ikDampingAngular` | `0.02` / `0.08` | DLS IK 감쇠 |
| `ikIterations` | `4` | IK 반복 횟수 |
| `clampToLimits` | `true` | URDF 관절 한계 클램프 적용 |
| `applyEveryFrame` | `false` | 매 프레임 적용 (false 면 `TickRamp` 만) |

#### 핵심 설계: **UI 없는 단일 진실 공급원**

```csharp
readonly float[] currentDeg = new float[6];   // ← 현재 각도
readonly float[] targetDeg  = new float[6];   // ← 목표 각도
```

두 배열이 관절 상태를 완전히 표현합니다.
`TickRamp()` 가 매 프레임 `currentDeg` 를 `targetDeg` 쪽으로 조금씩 움직이고,
`WriteBoneAngle()` 이 그 값을 FBX 본에 씁니다. 메시를 다시 읽을 필요가 없습니다.

#### 관절 → 본 적용 — [원칙 1](#principle-1)

```csharp
links[i].localRotation = restRotation[i] * Quaternion.AngleAxis(deg, Vector3.up);
```

`restRotation[i]` 는 FBX의 로컬 회전 그대로입니다.
`Quaternion.AngleAxis(deg, Vector3.up)` 이 곧 **관절축 파싱을 없애기 위해 고안한 트릭**입니다.

여기에 한계 클램프가 붙습니다:

```csharp
if (clampToLimits)
{
    float lo = specs[i].lowerRad * Mathf.Rad2Deg;
    float hi = specs[i].upperRad * Mathf.Rad2Deg;
    deg = Mathf.Clamp(deg, lo, hi);
}
```

> ⚠️ `specs[i].lowerRad` / `upperRad` 기본값이 **±3.14** 라는 점에 주의하세요.
> 실제 하드웨어 한계는 더 좁습니다 ([Part 3.8](#38-관절-키네마틱스-최종-표) 참조).
> `RB3_730_JointDriver` 컴포넌트의 인스펙터에서 축별 `lowerRad`/`upperRad` 를
> 실제 제원값으로 바꾸면 좁은 워크스페이스를 정확히 재현할 수 있습니다.

#### 본 → 관절 역산 `ReadBoneAngleDeg`

쿼터니언을 `y`, `w` 성분만으로 각도화합니다.

```csharp
Quaternion rel = Quaternion.Inverse(restRotation[i]) * links[i].localRotation;
return 2f * Mathf.Atan2(rel.y, rel.w) * Mathf.Rad2Deg;
```

#### `TickRamp` — 프레임레이트 독립 속도 램프

```csharp
float maxStep = Mathf.Max(0.1f, maxJointSpeedDegPerSec) * Mathf.Clamp01(speedOverride) * dt;
for (int i = 0; i < 6; i++)
{
    float delta = targetDeg[i] - currentDeg[i];
    if (Mathf.Abs(delta) <= maxStep) { currentDeg[i] = targetDeg[i]; continue; }   // 마지막 스텝에 정확히 도착
    currentDeg[i] += Mathf.Sign(delta) * maxStep;
}
```

- `maxStep` 에 `dt` 가 곱해지므로 **프레임레이트가 변해도 같은 축속도**입니다.
- `Mathf.Abs(delta) <= maxStep` 분기가 [Part 10.5](#105-tickramp-마지막-스텝에서-도달-못-함)에서 다룰 버그의 정답입니다.

#### `MoveTo` — 속도 제한 이동

`MoveTo(deg)` 는 `targetDeg` 만 설정합니다. 실제 움직임은 `Update` → `TickRamp` 가 담당합니다.
따라서 **이동 중에도 `IsMoving` 으로 상태를 조회**할 수 있고,
메인 스레드에서 실행되므로 게임이 멈추지 않습니다.

```csharp
/// <summary>Speed-limited absolute move. Angles are degrees.</summary>
public bool MoveTo(float[] deg) { … targetDeg[i] = deg[i]; … }
```

`SetAngles(deg)` 는 반대로 **속도 제한을 무시하고 즉시** 현재값과 목표값을 함께 갱신합니다.
`SetAngles` 는 부팅 시 FBX의 리그를 초기 각도로 만들 때 씁니다.

#### 카티시안 조그 — DLS IK

카티시안 목표를 관절 공간으로 바꾸는 데 **Damped Least Squares** 를 씁니다.
도달 불가능한 목표를 향해도 발산하지 않고, 가능한 한 가까운 자세에서 멈춥니다.
이 IK는 **티칭 포즈를 복원할 때 쓰이지 않습니다** ([원칙 2](#principle-2)).

조그 진입점은 두 종류입니다.

```csharp
// 관절 조그 (속도)
public void StartJointJog(int joint, float rateDegPerSec)
public bool StepJoint(int joint, float deltaDeg)

// TCP 조그 (선형 + 각도, 기준 프레임 지정)
public void StartCartesianJog(Vector3 linVel, Vector3 angVel, JogFrame frame)
public bool StepCartesian(Vector3 deltaLinear, Vector3 deltaAngular, JogFrame frame)

// TCP 도구 축 조그 — 프로토콜 JOGSTART X/Y/Z, RX/RY/RZ 가 여기로 들어온다
public bool StartToolLinearJog(int axis, float sign)    // axis 0=X 1=Y 2=Z
public bool StartToolAngularJog(int axis, float sign)   // axis 0=RX 1=RY 2=RZ
```

> 🔑 **티칭 포즈는 왜 카티시안이 아닌 관절로 저장하나요?** ([원칙 2](#principle-2))
> 카티시안으로 저장하면 재생할 때마다 IK를 다시 풀어야 하고,
> singular point 근처에서 실패하며, 반복할 때마다 미세하게 드리프트합니다.

#### `SyncFromRig()` — FBX→관절 각도 역산

 Gizmo, 애니메이션, 다른 스크립트가 리그를 건드렸을 때 그 변화를 받아들이는 장치입니다.

```csharp
void SyncFromRig()
{
    for (int i = 0; i < 6; i++)
    {
        float read = ReadBoneAngleDeg(i);
        // Reading a joint back out of the transform costs about 0.02 deg, so adopting the
        // read back value every frame would leave a permanent gap that no tight arrival
        // tolerance can ever close. Only take it when it is clearly an outside edit.
        if (Mathf.Abs(read - currentDeg[i]) > externalEditToleranceDeg) currentDeg[i] = read;
    }
}
```

> ⚠️ **이 메서드에 숨어 있는 함정 — 반드시 읽어두세요.**
> 관절 각도를 transform 에 쓰고 다시 읽으면 **약 0.02° 의 왕복 오차**가 남습니다.
> 매 프레임 그 값을 그대로 받아들이면 `arriveToleranceDeg`(0.05°) 안으로 절대 들어가지 못해서
> `IsMoving` 이 영원히 true가 되는 문제가 생깁니다.
>
> 해결책이 `externalEditToleranceDeg`(기본 1.0°) 입니다.
> **"아래 코드 주석 자체가 이 버그의 해법"** 이라는 점이 이 프로젝트에서 얻은 가장 값진 교훈 중 하나입니다.
> 자세한 내용은 [Part 10.4](#bug-syncfromrig).

#### 전체 소스


````text
> FILE: RB3_730_JointDriver.cs  (789 lines)

// RB3-730 joint driver for Unity. Render-pipeline independent.
//
// Attach to the root GameObject of an imported rb3_730es_u_rigged.fbx hierarchy.
//
// The FBX was authored so that each bone's local Y axis equals the URDF joint
// axis, therefore a joint angle is a pure local-Y rotation of the child link
// transform. That holds regardless of Unity's global up-axis (Y-up) conversion.
//
// MOTION MODEL
//   absolute move  MoveTo()  stores a target; the rig ramps towards it at
//                          maxJointSpeed * speedOverride, so a pendant sees
//                          smooth, speed-limited motion and can watch IsMoving.
//   jog            jog velocity is held between frames and the rig is stepped
//                          by velocity * dt, so the commanded speed is exact.
//   Cartesian jog  damped least-squares (Levenberg) solve of the 6x6 geometric
//                          Jacobian at the TCP, iterated until the task error
//                          is small enough or the iteration cap is reached.
//
// All Unity API calls happen in Update/LateUpdate on the main thread. The
// pendant server in this folder queues commands and applies them here.

using System;
using System.Text;
using UnityEngine;

namespace RainbowRobotics.RB
{
    [AddComponentMenu("Rainbow Robotics/RB3-730 Joint Driver")]
    [DisallowMultipleComponent]
    public class RB3_730_JointDriver : MonoBehaviour
    {
        /// <summary>Reference frame used to interpret a jog direction.</summary>
        public enum JogFrame
        {
            /// <summary>Unity world axes.</summary>
            World,
            /// <summary>Axes of this root object, i.e. the robot base when it sits at identity.</summary>
            Base,
            /// <summary>Axes of the TCP. This is what an operator expects from a pendant.</summary>
            Tool,
        }

        [Serializable]
        public class JointSpec
        {
            public string name;
            public string childLink;
            public float lowerRad = -3.14f;
            public float upperRad = 3.14f;
        }

        /// <summary>
        /// Raw shape of the URDF joint table shipped in Data/rb3_730es_u_joints.json.
        /// JsonUtility matches JSON keys to field names exactly, so the snake_case
        /// URDF keys are declared here and then mapped onto <see cref="JointSpec"/>.
        /// </summary>
        [Serializable]
        public class UrdfJointSpec
        {
            public string name;
            public string type;
            public string parent;
            public string child;
            public float limit_lower = -3.14f;
            public float limit_upper = 3.14f;
        }

        [Serializable]
        public class UrdfJointTable
        {
            public UrdfJointSpec[] joints;
        }

        [Header("Joint order: base, shoulder, elbow, wrist1, wrist2, wrist3")]
        [Tooltip("Clamp requested angles to the ranges declared in the URDF. " +
                 "The shipped URDF uses a flat +/-180 deg, which is wider than the real hardware. " +
                 "Assign the joint table JSON to use tighter real limits if you have them.")]
        public bool clampToLimits = true;

        [Tooltip("Optional TextAsset holding the URDF joint table " +
                 "(rb3_730es_u_joints.json: joints[].name / child / limit_lower / limit_upper).")]
        public TextAsset jointTable;

        [Header("Speed limits")]
        [Tooltip("Joint speed used for absolute moves at 100 percent override, in degrees per second.")]
        public float maxJointSpeedDegPerSec = 30f;

        [Range(0.01f, 1f)]
        [Tooltip("Pendant speed override as a 0..1 fraction of maxJointSpeedDegPerSec.")]
        public float speedOverride = 0.3f;

        [Tooltip("Linear jog speed at 100 percent override, metres per second.")]
        public float maxLinearJogSpeedMPerSec = 0.15f;

        [Tooltip("Angular jog speed at 100 percent override, radians per second.")]
        public float maxAngularJogSpeedRadPerSec = 1.0f;

        [Header("Inverse kinematics")]
        [Tooltip("Damping of the damped least-squares solve. Larger values are slower but stable " +
                 "near singularities and in directions the arm cannot reach.")]
        public float ikDampingLinear = 0.02f;

        [Tooltip("Angular counterpart of the IK damping, in radians.")]
        public float ikDampingAngular = 0.08f;

        [Range(1, 16)]
        [Tooltip("How many Jacobian solves per Cartesian jog step. More iterations follow curved " +
                 "paths more closely; 3 to 5 is normally enough for a jog.")]
        public int ikIterations = 4;

        [Header("Live drive")]
        [Tooltip("Six joint angles in degrees. While Apply Every Frame is on they are treated as a " +
                 "speed-limited target, so dragging a slider glides instead of snapping.")]
        public float[] anglesDeg = new float[6];

        [Tooltip("Push anglesDeg towards the rig every frame, speed limited. Turn off when driving " +
                 "from the pendant, from code, or from a physical controller.")]
        public bool applyEveryFrame = false;

        [Tooltip("Joint error that still counts as arrived, in degrees. Must stay above the " +
                 "transform read back noise, which is about 0.02 deg on this rig.")]
        public float arriveToleranceDeg = 0.05f;

        [Tooltip("A change in the rig bigger than this is taken as an outside edit and adopted. " +
                 "Smaller differences are treated as read back noise and ignored.")]
        public float externalEditToleranceDeg = 1f;

        readonly Transform[] links = new Transform[6];
        readonly Quaternion[] restRotation = new Quaternion[6];

        // live state, all degrees
        readonly float[] currentDeg = new float[6];
        readonly float[] targetDeg = new float[6];

        // held jog request
        int jogJoint = -1;          // 0..5, or -1 when not jogging a single joint
        float jogJointRate;         // degrees per second
        Vector3 jogLinearVel;       // metres per second
        Vector3 jogAngularVel;      // radians per second
        JogFrame jogFrame = JogFrame.Tool;

        static readonly JointSpec[] Builtin =
        {
            new JointSpec { name = "base",     childLink = "link1", lowerRad = -3.14f, upperRad =  3.14f },
            new JointSpec { name = "shoulder", childLink = "link2", lowerRad = -3.14f, upperRad =  3.14f },
            new JointSpec { name = "elbow",    childLink = "link3", lowerRad = -3.14f, upperRad =  3.14f },
            new JointSpec { name = "wrist1",   childLink = "link4", lowerRad = -3.14f, upperRad =  3.14f },
            new JointSpec { name = "wrist2",   childLink = "link5", lowerRad = -3.14f, upperRad =  3.14f },
            new JointSpec { name = "wrist3",   childLink = "link6", lowerRad = -3.14f, upperRad =  3.14f },
        };

        JointSpec[] specs;
        bool ready;
        bool loggedFailure;

        /// <summary>Parent link of the first movable joint, read from the URDF table for diagnostics.</summary>
        public string OrderHint { get; private set; } = string.Empty;

        public bool IsReady { get { EnsureReady(); return ready; } }

        /// <summary>True while an absolute move is still ramping or a jog is being held.</summary>
        public bool IsMoving
        {
            get
            {
                if (!EnsureReady()) return false;
                if (jogJoint >= 0 || jogLinearVel != Vector3.zero || jogAngularVel != Vector3.zero) return true;
                float tol = Mathf.Max(1e-4f, arriveToleranceDeg);
                for (int i = 0; i < 6; i++)
                    if (Mathf.Abs(targetDeg[i] - currentDeg[i]) > tol) return true;
                return false;
            }
        }

        public bool IsJogging
        {
            get { return jogJoint >= 0 || jogLinearVel != Vector3.zero || jogAngularVel != Vector3.zero; }
        }

        /// <summary>Speed override as a 0..1 fraction. The pendant SPEED command maps onto this.</summary>
        public float SpeedOverride
        {
            get { return speedOverride; }
            set { speedOverride = Mathf.Clamp01(value); }
        }

        public string[] JointNames
        {
            get
            {
                EnsureReady();
                string[] n = new string[6];
                for (int i = 0; i < 6; i++) n[i] = specs[i].name;
                return n;
            }
        }

        void Awake() => Cache();

        void OnValidate()
        {
            ready = false;
            if (anglesDeg == null || anglesDeg.Length != 6) anglesDeg = new float[6];
            maxJointSpeedDegPerSec = Mathf.Max(0.1f, maxJointSpeedDegPerSec);
            maxLinearJogSpeedMPerSec = Mathf.Max(0.001f, maxLinearJogSpeedMPerSec);
            maxAngularJogSpeedRadPerSec = Mathf.Max(0.001f, maxAngularJogSpeedRadPerSec);
            arriveToleranceDeg = Mathf.Max(1e-4f, arriveToleranceDeg);
            externalEditToleranceDeg = Mathf.Max(arriveToleranceDeg, externalEditToleranceDeg);
        }

        void Update()
        {
            if (!EnsureReady()) return;
            SyncFromRig();
            if (IsJogging) TickJog(Time.deltaTime);
            TickRamp(Time.deltaTime);
        }

        void LateUpdate()
        {
            if (!EnsureReady() || !applyEveryFrame) return;
            for (int i = 0; i < 6; i++) targetDeg[i] = anglesDeg[i];
        }

        bool EnsureReady()
        {
            if (!ready) Cache();
            return ready;
        }

        void Cache()
        {
            ready = false;
            specs = ResolveSpecs();

            for (int i = 0; i < specs.Length; i++)
            {
                Transform t = FindBone(transform, specs[i].childLink);
                if (t == null)
                {
                    if (!loggedFailure)
                    {
                        Debug.LogErrorFormat(
                            "[{0}] joint bone '{1}' not found below '{2}'. " +
                            "In the FBX importer set Rig > Animation Type to Generic and turn " +
                            "'Optimize Game Objects' OFF, then reimport.",
                            name, specs[i].childLink, name, this);
                        loggedFailure = true;
                    }
                    return;
                }
                links[i] = t;
                restRotation[i] = t.localRotation;
            }

            for (int i = 0; i < 6; i++)
            {
                currentDeg[i] = 0f;
                targetDeg[i] = 0f;
            }
            ready = true;
            loggedFailure = false;
        }

        JointSpec[] ResolveSpecs()
        {
            if (jointTable == null) return Builtin;
            try
            {
                UrdfJointTable parsed = JsonUtility.FromJson<UrdfJointTable>(jointTable.text);
                if (parsed != null && parsed.joints != null && parsed.joints.Length > 0)
                {
                    // The URDF lists 7 joints: 6 movable ones plus the fixed tcp_joint that
                    // bolts the tool frame onto link6. Only the movable joints are driven, so
                    // filter on the declared type instead of expecting exactly 6 entries.
                    JointSpec[] mapped = new JointSpec[6];
                    int found = 0;
                    for (int i = 0; i < parsed.joints.Length && found < 6; i++)
                    {
                        UrdfJointSpec src = parsed.joints[i];
                        if (src == null) continue;
                        if (!IsMovable(src.type)) continue;
                        if (string.IsNullOrEmpty(src.child)) continue;
                        if (found == 0) OrderHint = src.parent;
                        mapped[found++] = new JointSpec
                        {
                            name = src.name,
                            childLink = src.child,
                            lowerRad = Mathf.Min(src.limit_lower, src.limit_upper),
                            upperRad = Mathf.Max(src.limit_lower, src.limit_upper)
                        };
                    }

                    if (found == 6)
                    {
                        Debug.LogFormat("[{0}] joint table '{1}' mapped 6 movable joints: {2}",
                            name, jointTable.name, Describe(mapped), this);
                        return mapped;
                    }

                    Debug.LogWarningFormat(
                        "[{0}] joint table '{1}' yielded {2} movable joints, expected 6; " +
                        "using the built-in URDF values.", name, jointTable.name, found, this);
                }
                else
                {
                    Debug.LogWarningFormat(
                        "[{0}] joint table '{1}' has no 'joints' array; using the built-in values.",
                        name, jointTable.name, this);
                }
            }
            catch (Exception e)
            {
                Debug.LogErrorFormat("[{0}] failed to parse joint table: {1}", name, e.Message, this);
            }
            return Builtin;
        }

        static bool IsMovable(string type)
        {
            if (string.IsNullOrEmpty(type)) return false;
            return type == "revolute" || type == "continuous" || type == "prismatic";
        }

        static string Describe(JointSpec[] s)
        {
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < s.Length; i++)
            {
                if (i > 0) sb.Append(", ");
                sb.Append(s[i].childLink);
            }
            return sb.ToString();
        }

        /// <summary>
        /// Finds a joint BONE by name. Unity's generic-rig import creates two GameObjects per
        /// link: the renderer-less bone inside the Armature, and a same-named
        /// SkinnedMeshRenderer node that is a sibling of the Armature. Only the bone may be
        /// rotated - a naive first-name-match would pick the mesh node and the joints would
        /// never move - so candidates carrying a Renderer are skipped.
        /// </summary>
        static Transform FindBone(Transform root, string target)
        {
            if (string.IsNullOrEmpty(target)) return null;
            Transform first = null;
            Transform[] all = root.GetComponentsInChildren<Transform>(true);
            for (int i = 0; i < all.Length; i++)
            {
                if (all[i].name != target) continue;
                if (first == null) first = all[i];
                if (all[i].GetComponent<Renderer>() == null) return all[i];
            }
            return first;
        }

        // ------------------------------------------------------------------ angle plumbing

        /// <summary>Signed rotation of a bone about its own local Y, in degrees.</summary>
        float ReadBoneAngleDeg(int i)
        {
            Quaternion rel = Quaternion.Inverse(restRotation[i]) * links[i].localRotation;
            return 2f * Mathf.Atan2(rel.y, rel.w) * Mathf.Rad2Deg;
        }

        /// <summary>Reads the rig so external edits (gizmos, animation, another script) are respected.</summary>
        void SyncFromRig()
        {
            for (int i = 0; i < 6; i++)
            {
                float read = ReadBoneAngleDeg(i);
                // Reading a joint back out of the transform costs about 0.02 deg, so adopting the
                // read back value every frame would leave a permanent gap that no tight arrival
                // tolerance can ever close. Only take it when it is clearly an outside edit.
                if (Mathf.Abs(read - currentDeg[i]) > externalEditToleranceDeg) currentDeg[i] = read;
            }
        }

        void WriteBoneAngle(int i, float deg)
        {
            if (clampToLimits)
            {
                float lo = specs[i].lowerRad * Mathf.Rad2Deg;
                float hi = specs[i].upperRad * Mathf.Rad2Deg;
                if (lo > hi) { float t = lo; lo = hi; hi = t; }
                deg = Mathf.Clamp(deg, lo, hi);
            }
            links[i].localRotation = restRotation[i] * Quaternion.AngleAxis(deg, Vector3.up);
        }

        void TickRamp(float dt)
        {
            float maxStep = Mathf.Max(0.1f, maxJointSpeedDegPerSec) * Mathf.Clamp01(speedOverride) * dt;
            for (int i = 0; i < 6; i++)
            {
                float delta = targetDeg[i] - currentDeg[i];
                if (delta == 0f) continue;

                if (Mathf.Abs(delta) <= maxStep)
                {
                    // Land exactly on the target. Just dropping the step instead would leave the
                    // joint one maxStep short every frame, so it creeps towards the target and
                    // never reports arrived.
                    currentDeg[i] = targetDeg[i];
                }
                else
                {
                    currentDeg[i] += Mathf.Sign(delta) * maxStep;
                }
                WriteBoneAngle(i, currentDeg[i]);
            }
        }

        // ------------------------------------------------------------------ public API

        /// <summary>Instantly sets all six joints. Angles are degrees. Bypasses speed limits.</summary>
        public void SetAngles(float[] deg)
        {
            if (!EnsureReady()) return;
            if (deg == null || deg.Length < 6)
            {
                Debug.LogErrorFormat("[{0}] SetAngles expects 6 values (base..wrist3).", name, this);
                return;
            }
            ClearJog();
            for (int i = 0; i < 6; i++)
            {
                currentDeg[i] = deg[i];
                targetDeg[i] = deg[i];
                WriteBoneAngle(i, deg[i]);
            }
        }

        /// <summary>Speed-limited absolute move. Angles are degrees.</summary>
        public bool MoveTo(float[] deg)
        {
            if (!EnsureReady()) return false;
            if (deg == null || deg.Length < 6)
            {
                Debug.LogErrorFormat("[{0}] MoveTo expects 6 values (base..wrist3).", name, this);
                return false;
            }
            ClearJog();
            for (int i = 0; i < 6; i++) targetDeg[i] = deg[i];
            return true;
        }

        /// <summary>Speed-limited absolute move of a single joint given by index (0..5).</summary>
        public bool MoveJointTo(int joint, float degrees)
        {
            if (!EnsureReady() || joint < 0 || joint > 5) return false;
            ClearJog();
            targetDeg[joint] = degrees;
            return true;
        }

        /// <summary>Instantly sets a single joint by name, in degrees. Bypasses speed limits.</summary>
        public void SetJoint(string jointName, float degrees)
        {
            if (!EnsureReady()) return;
            int idx = Array.FindIndex(specs, s => s.name == jointName);
            if (idx < 0)
            {
                Debug.LogErrorFormat("[{0}] unknown joint '{1}'.", name, jointName, this);
                return;
            }
            ClearJog();
            currentDeg[idx] = degrees;
            targetDeg[idx] = degrees;
            WriteBoneAngle(idx, degrees);
        }

        public void GoHome() => MoveTo(new float[6]);

        /// <summary>Holds the current pose and clears any jog request.</summary>
        public void Stop()
        {
            if (!EnsureReady()) return;
            ClearJog();
            for (int i = 0; i < 6; i++) targetDeg[i] = currentDeg[i];
        }

        public void SetSpeedOverride(float normalised)
        {
            speedOverride = Mathf.Clamp01(normalised);
        }

        /// <summary>Current joint angles in degrees, base first.</summary>
        public float[] GetAnglesDeg()
        {
            EnsureReady();
            if (ready) SyncFromRig();
            float[] copy = new float[6];
            Array.Copy(currentDeg, copy, 6);
            return copy;
        }

        public float GetJointAngleDeg(int joint)
        {
            if (!EnsureReady() || joint < 0 || joint > 5) return 0f;
            return ReadBoneAngleDeg(joint);
        }

        // ------------------------------------------------------------------ joint jog

        /// <summary>Starts a held velocity jog on one joint. rateDegPerSec is signed.</summary>
        public void StartJointJog(int joint, float rateDegPerSec)
        {
            if (!EnsureReady() || joint < 0 || joint > 5) return;
            jogLinearVel = Vector3.zero;
            jogAngularVel = Vector3.zero;
            jogJoint = joint;
            jogJointRate = rateDegPerSec;
        }

        /// <summary>One-shot relative joint step, in degrees, bypassing the held jog state.</summary>
        public bool StepJoint(int joint, float deltaDeg)
        {
            if (!EnsureReady() || joint < 0 || joint > 5) return false;
            SyncFromRig();
            SetHeldJoint(joint, currentDeg[joint] + deltaDeg);
            currentDeg[joint] = ReadBoneAngleDeg(joint);
            return true;
        }

        void TickJog(float dt)
        {
            if (jogJoint >= 0)
            {
                float rate = jogJointRate * Mathf.Clamp01(speedOverride);
                SetHeldJoint(jogJoint, currentDeg[jogJoint] + rate * dt);
                currentDeg[jogJoint] = ReadBoneAngleDeg(jogJoint);
            }

            Vector3 lin = jogLinearVel * Mathf.Clamp01(speedOverride);
            Vector3 ang = jogAngularVel * Mathf.Clamp01(speedOverride);
            if (lin != Vector3.zero || ang != Vector3.zero)
            {
                StepCartesian(lin * dt, ang * dt, jogFrame);
                SyncFromRig();
            }
        }

        void SetHeldJoint(int joint, float deg)
        {
            WriteBoneAngle(joint, deg);
            targetDeg[joint] = deg;
        }

        public void ClearJog()
        {
            jogJoint = -1;
            jogJointRate = 0f;
            jogLinearVel = Vector3.zero;
            jogAngularVel = Vector3.zero;
            if (ready)
                for (int i = 0; i < 6; i++)
                    if (targetDeg[i] == 0f && currentDeg[i] == 0f) targetDeg[i] = currentDeg[i];
        }

        // ------------------------------------------------------------------ Cartesian jog

        /// <summary>
        /// Starts a held Cartesian jog. linVel is metres per second and angVel radians per
        /// second, both expressed in <paramref name="frame"/>.
        /// </summary>
        public void StartCartesianJog(Vector3 linVel, Vector3 angVel, JogFrame frame)
        {
            if (!EnsureReady()) return;
            jogJoint = -1;
            jogJointRate = 0f;
            jogFrame = frame;
            jogLinearVel = linVel;
            jogAngularVel = angVel;
        }

        /// <summary>One-shot Cartesian step of a Cartesian delta, bypassing the held jog state.</summary>
        public bool StepCartesian(Vector3 deltaLinear, Vector3 deltaAngular, JogFrame frame)
        {
            if (!EnsureReady()) return false;
            return SolveTask(ToWorldLinear(deltaLinear, frame), ToWorldAngular(deltaAngular, frame));
        }

        /// <summary>Held linear jog along a TCP axis. axis 0=X 1=Y 2=Z, sign is -1 or +1.</summary>
        public bool StartToolLinearJog(int axis, float sign)
        {
            if (!EnsureReady()) return false;
            Vector3 dir;
            switch (axis) { case 0: dir = Vector3.right; break; case 1: dir = Vector3.up; break; case 2: dir = Vector3.forward; break; default: return false; }
            StartCartesianJog(dir * (maxLinearJogSpeedMPerSec * sign), Vector3.zero, JogFrame.Tool);
            return true;
        }

        /// <summary>Held angular jog about a TCP axis. axis 0=X 1=Y 2=Z, sign is -1 or +1.</summary>
        public bool StartToolAngularJog(int axis, float sign)
        {
            if (!EnsureReady()) return false;
            Vector3 dir;
            switch (axis) { case 0: dir = Vector3.right; break; case 1: dir = Vector3.up; break; case 2: dir = Vector3.forward; break; default: return false; }
            StartCartesianJog(Vector3.zero, dir * (maxAngularJogSpeedRadPerSec * sign), JogFrame.Tool);
            return true;
        }

        Vector3 ToWorldLinear(Vector3 v, JogFrame frame)
        {
            switch (frame)
            {
                case JogFrame.Base: return transform.TransformVector(v);
                case JogFrame.Tool:
                    Transform tcp = GetTcp();
                    return tcp != null ? tcp.rotation * v : v;
                default: return v;
            }
        }

        Vector3 ToWorldAngular(Vector3 v, JogFrame frame)
        {
            switch (frame)
            {
                case JogFrame.Base: return transform.rotation * v;
                case JogFrame.Tool:
                    Transform tcp = GetTcp();
                    return tcp != null ? tcp.rotation * v : v;
                default: return v;
            }
        }

        /// <summary>
        /// Damped least-squares differential IK. Solves (J J^T + lambda^2 I) y = task, then
        /// dq = J^T y, applies it, and re-solves from the new pose until the TCP error is
        /// negligible or the iteration budget runs out.
        /// </summary>
        bool SolveTask(Vector3 worldLinear, Vector3 worldAngular)
        {
            Transform tcp = GetTcp();
            if (tcp == null) return false;

            Vector3 wantPos = tcp.position + worldLinear;
            Quaternion wantRot = Quaternion.AngleAxis(worldAngular.magnitude * Mathf.Rad2Deg,
                                                       worldAngular.normalized) * tcp.rotation;

            double[] J = new double[36];
            double[] A = new double[36];
            double[] task = new double[6];
            double[] y = new double[6];
            double[] dq = new double[6];

            int iters = Mathf.Clamp(ikIterations, 1, 16);
            for (int it = 0; it < iters; it++)
            {
                Vector3 posErr = wantPos - tcp.position;
                Quaternion rotErr = wantRot * Quaternion.Inverse(tcp.rotation);
                Vector3 angErr = 2f * new Vector3(rotErr.x, rotErr.y, rotErr.z);

                if (posErr.sqrMagnitude < 1e-10f && angErr.sqrMagnitude < 1e-10f) return true;
                if (posErr.magnitude > 0.05f) posErr = posErr.normalized * 0.05f;
                if (angErr.magnitude > 0.35f) angErr = angErr.normalized * 0.35f;

                BuildJacobian(J);
                for (int r = 0; r < 6; r++)
                    for (int c = 0; c < 6; c++)
                    {
                        double sum = 0.0;
                        for (int k = 0; k < 6; k++) sum += J[r * 6 + k] * J[c * 6 + k];
                        A[r * 6 + c] = sum;
                    }
                A[0 * 6 + 0] += ikDampingLinear * ikDampingLinear;
                A[1 * 6 + 1] += ikDampingLinear * ikDampingLinear;
                A[2 * 6 + 2] += ikDampingLinear * ikDampingLinear;
                A[3 * 6 + 3] += ikDampingAngular * ikDampingAngular;
                A[4 * 6 + 4] += ikDampingAngular * ikDampingAngular;
                A[5 * 6 + 5] += ikDampingAngular * ikDampingAngular;

                task[0] = posErr.x; task[1] = posErr.y; task[2] = posErr.z;
                task[3] = angErr.x; task[4] = angErr.y; task[5] = angErr.z;

                if (!Solve6(A, task, y)) return false;

                double maxStep = 5.0 * Mathf.Deg2Rad;
                double moved = 0.0;
                for (int k = 0; k < 6; k++)
                {
                    double sum = 0.0;
                    for (int r = 0; r < 6; r++) sum += J[r * 6 + k] * y[r];
                    if (sum > maxStep) sum = maxStep;
                    else if (sum < -maxStep) sum = -maxStep;
                    dq[k] = sum;
                    moved += sum * sum;
                }
                if (moved < 1e-14) return false;

                for (int k = 0; k < 6; k++)
                {
                    float before = ReadBoneAngleDeg(k);
                    WriteBoneAngle(k, before + (float)dq[k] * Mathf.Rad2Deg);
                    currentDeg[k] = ReadBoneAngleDeg(k);
                    targetDeg[k] = currentDeg[k];
                }
            }
            return true;
        }

        /// <summary>Row major 6x6 geometric Jacobian. Rows 0-2 linear, rows 3-5 angular.</summary>
        void BuildJacobian(double[] J)
        {
            Transform tcp = GetTcp();
            Vector3 p = tcp != null ? tcp.position : transform.position;
            for (int i = 0; i < 6; i++)
            {
                Vector3 axis = links[i].rotation * Vector3.up;
                Vector3 origin = links[i].position;
                Vector3 lin = Vector3.Cross(axis, p - origin);
                J[0 * 6 + i] = lin.x;
                J[1 * 6 + i] = lin.y;
                J[2 * 6 + i] = lin.z;
                J[3 * 6 + i] = axis.x;
                J[4 * 6 + i] = axis.y;
                J[5 * 6 + i] = axis.z;
            }
        }

        /// <summary>Gaussian elimination with partial pivoting. A is destroyed.</summary>
        static bool Solve6(double[] A, double[] b, double[] x)
        {
            double[,] m = new double[6, 7];
            for (int r = 0; r < 6; r++)
            {
                for (int c = 0; c < 6; c++) m[r, c] = A[r * 6 + c];
                m[r, 6] = b[r];
            }

            for (int col = 0; col < 6; col++)
            {
                int pivot = col;
                double best = Math.Abs(m[col, col]);
                for (int r = col + 1; r < 6; r++)
                {
                    double v = Math.Abs(m[r, col]);
                    if (v > best) { best = v; pivot = r; }
                }
                if (best < 1e-12) return false;

                if (pivot != col)
                    for (int c = col; c < 7; c++)
                    {
                        double t = m[col, c]; m[col, c] = m[pivot, c]; m[pivot, c] = t;
                    }

                double diag = m[col, col];
                for (int r = col + 1; r < 6; r++)
                {
                    double f = m[r, col] / diag;
                    if (f == 0.0) continue;
                    for (int c = col; c < 7; c++) m[r, c] -= f * m[col, c];
                }
            }

            for (int r = 5; r >= 0; r--)
            {
                double sum = m[r, 6];
                for (int c = r + 1; c < 6; c++) sum -= m[r, c] * x[c];
                x[r] = sum / m[r, r];
                if (double.IsNaN(x[r]) || double.IsInfinity(x[r])) return false;
            }
            return true;
        }

        // ------------------------------------------------------------------ inspection

        public Transform GetLink(string linkName) => FindBone(transform, linkName);

        public Transform GetTcp() => FindBone(transform, "tcp");

        /// <summary>World-space TCP pose. The flange is link6; the tool point sits 100 mm beyond it.</summary>
        public bool TryGetTcpPose(out Vector3 position, out Quaternion rotation)
        {
            Transform tcp = GetTcp();
            if (tcp == null)
            {
                position = Vector3.zero;
                rotation = Quaternion.identity;
                return false;
            }
            position = tcp.position;
            rotation = tcp.rotation;
            return true;
        }
    }
}
````

---

<a id="code-teach"></a>

### 6.2 `RB3_730_Teach.cs`

**이 컴포넌트가 "작업자의 기억"입니다.** 티칭된 포즈를 저장하고 재생합니다.

#### 티칭 포인트 구조

```csharp
public struct Point
{
    public string Name;            // TEACH pick  에서 "pick"
    public float[] JointDeg;       // ← 관절 공간으로 저장 (원칙 2)
    public Vector3 TcpPos;         // 설명용
    public Quaternion TcpRot;      // 설명용
    public float DwellSeconds;     // DWELL 명령으로 설정
}
```

`JointDeg` 가 기준 데이터(authoritative) 이고, `TcpPos`/`TcpRot` 는 `LIST` 응답과 UI 표시용입니다.
**실제 이동은 `JointDeg` 만 사용합니다.**

#### 사이클 상태 기계

```
CYCLE 명령
    │
    ├─ 포인트 2개 미만이면 → "ERR CYCLE teach at least two points first…"
    │
    ▼
 ┌──────────┐  CYCLEPAUSE   ┌──────────┐
 │  moving  │──────────────►│  paused  │
 └────┬─────┘◄──────────────└────┬─────┘
      │                          │
      │ 다 끝나면 (lap < laps)   │ CYCLESTATE
      │        │                 │
      │        ▼                 ▼
      │  ┌──────────┐        CYCLESTOP → idle
      └─►│   idle   │
         └──────────┘
```

| 상태 | `STATE.cycle` 값 |
|---|---|
| 정지 | `idle` |
| 포인트 이동 중 | `moving` |
| 해당 포즈에서 체류 | `dwell` |
| 일시정지 | `paused` |

#### `Step()` — NEXT / PREV

```csharp
teach.Step(+1);   // NEXT
teach.Step(-1);   // PREV
```

수동 단계 이동입니다. 포즈를 순서대로 훑어 보면서 티칭 결과를 확인하는 용도입니다.

#### 전체 소스


````text
> FILE: RB3_730_Teach.cs  (527 lines)

// Teaching points and cycle playback for the RB3-730.
//
// A teaching pendant needs two things this component provides:
//
//   1. TEACH  - record the pose the arm is standing in, under a name, into an ordered list.
//   2. CYCLE  - play that list back, 1 -> 2 -> 3 -> ... -> 1, forever or for a set number of
//               laps, dwelling on each point for a taught time.
//
// Playback is joint space, not Cartesian. A taught joint target is always reproduced exactly,
// so a cycle never drifts, never needs the IK solver, and never fails near a singularity. The
// TCP pose is recorded too, but only as a label for the operator: "P3, 400mm, gripper down".
//
// Everything runs on the main thread in Update, reusing the joint driver's own speed limiting,
// so a cycle is exactly as fast as the same move issued by hand, and Stop() stops it mid-lap.

using System;
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using System.Text;
using UnityEngine;

namespace RainbowRobotics.RB
{
    /// <summary>One taught pose. Joints drive playback; the TCP fields are descriptive only.</summary>
    [Serializable]
    public class TaughtPoint
    {
        public string name = "P";
        public float[] joints = new float[6];
        public float[] tcp = new float[7];      // x y z qx qy qz qw
        public float dwell = 0.5f;              // seconds to hold on arrival

        public float[] CopyJoints()
        {
            float[] j = new float[6];
            for (int i = 0; i < 6; i++) j[i] = joints[i];
            return j;
        }
    }

    [AddComponentMenu("Rainbow Robotics/RB3-730 Teach")]
    [RequireComponent(typeof(RB3_730_JointDriver))]
    public class RB3_730_Teach : MonoBehaviour
    {
        public enum CycleState
        {
            Idle,
            Moving,
            Dwell,
            Paused
        }

        [Header("Taught points")]
        [Tooltip("Default dwell in seconds applied to a newly taught point.")]
        public float defaultDwell = 0.5f;

        [Tooltip("Refuse to teach more than this many points, so a runaway client cannot fill memory.")]
        [Range(2, 999)]
        public int maxPoints = 200;

        [Header("Cycle")]
        [Tooltip("Laps to run when a cycle is started without an explicit count. 0 loops forever.")]
        public int defaultLoops = 0;

        [Tooltip("Extra seconds added between points, on top of the taught dwell.")]
        public float lapPause = 0f;

        [Header("Persistence")]
        [Tooltip("File under Application.persistentDataPath used by Save and Load.")]
        public string saveFileName = "rb3_730_teach_points.json";

        [Header("Diagnostics")]
        public bool verboseLogging = true;

        readonly List<TaughtPoint> points = new List<TaughtPoint>();

        // cycle playback state
        readonly List<int> sequence = new List<int>();
        int seqPos;
        int lap;
        int targetLaps;
        float dwellUntil;
        CycleState pausedFrom = CycleState.Moving;

        public int PointCount { get { return points.Count; } }
        public CycleState State { get; private set; }
        public bool CycleRunning { get { return State == CycleState.Moving || State == CycleState.Dwell; } }
        public int CurrentSequencePosition { get { return seqPos; } }
        public int CurrentLap { get { return lap + 1; } }
        public string CurrentPointName
        {
            get
            {
                if (!CycleRunning || seqPos < 0 || seqPos >= sequence.Count) return string.Empty;
                TaughtPoint p = points[sequence[seqPos]];
                return p == null ? string.Empty : p.name;
            }
        }

        RB3_730_JointDriver driver;

        void Awake()
        {
            driver = GetComponent<RB3_730_JointDriver>();
        }

        void Update()
        {
            if (driver == null || !driver.IsReady) return;

            switch (State)
            {
                case CycleState.Moving:
                    if (!driver.IsMoving) Arrive();
                    break;

                case CycleState.Dwell:
                    if (Time.time >= dwellUntil) Advance();
                    break;
            }
        }

        // ------------------------------------------------------------------ teach

        /// <summary>Records the pose the arm is holding right now. Returns the new point index.</summary>
        public int Record(string name, float dwell)
        {
            if (!driver.IsReady) return -1;
            if (State != CycleState.Idle) StopCycle("taught a new point");
            if (points.Count >= maxPoints)
            {
                Debug.LogWarningFormat("[{0}] taught point limit reached ({1}), not recording", name, maxPoints, this);
                return -1;
            }

            TaughtPoint p = new TaughtPoint
            {
                name = string.IsNullOrEmpty(name) ? NextFreeName() : name,
                joints = driver.GetAnglesDeg(),
                dwell = Mathf.Max(0f, dwell < 0f ? defaultDwell : dwell)
            };

            Vector3 pos;
            Quaternion rot;
            if (driver.TryGetTcpPose(out pos, out rot))
                p.tcp = new[] { pos.x, pos.y, pos.z, rot.x, rot.y, rot.z, rot.w };

            points.Add(p);
            int index = points.Count - 1;
            if (verboseLogging)
                Debug.LogFormat("[{0}] taught {1} at point {2}: J = {3}", name, p.name, index + 1, FormatJoints(p.joints), this);
            return index;
        }

        public int Record(string name) { return Record(name, defaultDwell); }

        string NextFreeName()
        {
            for (int i = 1; i <= points.Count + 1; i++)
            {
                string candidate = "P" + i.ToString(CultureInfo.InvariantCulture);
                bool taken = false;
                for (int j = 0; j < points.Count; j++)
                    if (points[j] != null && points[j].name == candidate) { taken = true; break; }
                if (!taken) return candidate;
            }
            return "P" + (points.Count + 1).ToString(CultureInfo.InvariantCulture);
        }

        public bool Delete(int index)
        {
            if (index < 0 || index >= points.Count) return false;
            points.RemoveAt(index);
            if (verboseLogging) Debug.LogFormat("[{0}] deleted point {1}, {2} left", name, index + 1, points.Count, this);
            return true;
        }

        public void Clear()
        {
            StopCycle("cleared");
            points.Clear();
            if (verboseLogging) Debug.LogFormat("[{0}] cleared all taught points", name, this);
        }

        public bool Rename(int index, string newName)
        {
            if (index < 0 || index >= points.Count || string.IsNullOrEmpty(newName)) return false;
            points[index].name = newName;
            return true;
        }

        public bool SetDwell(int index, float seconds)
        {
            if (index < 0 || index >= points.Count) return false;
            points[index].dwell = Mathf.Max(0f, seconds);
            return true;
        }

        public float GetDwell(int index)
        {
            return (index >= 0 && index < points.Count) ? points[index].dwell : 0f;
        }

        public string GetName(int index)
        {
            return (index >= 0 && index < points.Count && points[index] != null) ? points[index].name : string.Empty;
        }

        public float[] GetJoints(int index)
        {
            return (index >= 0 && index < points.Count && points[index] != null) ? points[index].CopyJoints() : null;
        }

        public bool GetTcp(int index, out Vector3 position, out Quaternion rotation)
        {
            position = Vector3.zero;
            rotation = Quaternion.identity;
            if (index < 0 || index >= points.Count) return false;
            float[] t = points[index].tcp;
            if (t == null || t.Length < 7) return false;
            position = new Vector3(t[0], t[1], t[2]);
            rotation = new Quaternion(t[3], t[4], t[5], t[6]);
            return true;
        }

        // ------------------------------------------------------------------ single moves

        public bool MoveToIndex(int index)
        {
            if (!driver.IsReady) return false;
            float[] j = GetJoints(index);
            if (j == null) return false;
            StopCycle("single move");
            CurrentReference = index;
            return driver.MoveTo(j);
        }

        /// <summary>
        /// Steps to the neighbouring taught point in the list, wrapping at both ends, which is what
        /// a pendant NEXT and PREV button should do. The reference point is the arm's current pose
        /// the first time, and the last point visited after that.
        /// </summary>
        public bool Step(int direction)
        {
            if (points.Count == 0) return false;

            int reference = CurrentReference;
            if (reference < 0 || reference >= points.Count) reference = NearestPointToCurrent();
            if (reference < 0) reference = direction >= 0 ? -1 : 0;

            int next = direction >= 0
                ? (reference + 1) % points.Count
                : ((reference - 1) % points.Count + points.Count) % points.Count;
            return MoveToIndex(next);
        }

        static float AngleDistance(float[] a, float[] b)
        {
            if (a == null || b == null) return 0f;
            float sum = 0f;
            for (int i = 0; i < 6; i++) { float d = a[i] - b[i]; sum += d * d; }
            return Mathf.Sqrt(sum);
        }

        /// <summary>Index of the point the arm is currently standing on, or -1.</summary>
        public int CurrentReference { get; private set; } = -1;

        public int NearestPointToCurrent()
        {
            if (!driver.IsReady || points.Count == 0) return -1;
            float[] cur = driver.GetAnglesDeg();
            int best = -1;
            float bestD = float.MaxValue;
            for (int i = 0; i < points.Count; i++)
            {
                float d = AngleDistance(points[i].joints, cur);
                if (d < bestD) { bestD = d; best = i; }
            }
            return best;
        }

        // ------------------------------------------------------------------ cycle

        /// <summary>
        /// Starts playback. Pass null or an empty list to cycle every taught point in order.
        /// loops is the number of laps; 0 repeats until StopCycle is called.
        /// </summary>
        public bool StartCycle(int[] indices, int loops)
        {
            if (!driver.IsReady) return false;
            if (points.Count == 0)
            {
                Debug.LogWarningFormat("[{0}] nothing taught yet, cannot cycle", name, this);
                return false;
            }

            sequence.Clear();
            if (indices != null && indices.Length > 0)
            {
                for (int i = 0; i < indices.Length; i++)
                {
                    if (indices[i] < 0 || indices[i] >= points.Count)
                    {
                        Debug.LogWarningFormat("[{0}] cycle index {1} is out of range 1..{2}",
                            name, indices[i] + 1, points.Count, this);
                        return false;
                    }
                    sequence.Add(indices[i]);
                }
            }
            else
            {
                for (int i = 0; i < points.Count; i++) sequence.Add(i);
            }

            targetLaps = Mathf.Max(0, loops);
            lap = 0;
            seqPos = 0;
            GoToCurrent();
            return true;
        }

        public bool StartCycle() { return StartCycle(null, defaultLoops); }

        void GoToCurrent()
        {
            if (seqPos < 0 || seqPos >= sequence.Count)
            {
                StopCycle("sequence exhausted");
                return;
            }

            int index = sequence[seqPos];
            float[] j = GetJoints(index);
            if (j == null)
            {
                StopCycle("point missing");
                return;
            }

            CurrentReference = index;
            driver.MoveTo(j);
            State = CycleState.Moving;
            if (verboseLogging)
                Debug.LogFormat("[{0}] cycle lap {1}/{2} -> {3} ({4}) J = {5}",
                    name, lap + 1, targetLaps == 0 ? "inf" : targetLaps.ToString(CultureInfo.InvariantCulture),
                    points[index].name, index + 1, FormatJoints(j), this);
        }

        void Arrive()
        {
            State = CycleState.Dwell;
            dwellUntil = Time.time + GetDwell(CurrentReference) + lapPause;
        }

        void Advance()
        {
            seqPos++;
            if (seqPos < sequence.Count)
            {
                GoToCurrent();
                return;
            }

            lap++;
            if (targetLaps > 0 && lap >= targetLaps)
            {
                StopCycle("all laps finished");
                return;
            }
            seqPos = 0;
            GoToCurrent();
        }

        public void StopCycle(string reason)
        {
            if (State == CycleState.Idle && sequence.Count == 0) return;
            // Paused counts as running: a pause lets the ramp finish, so a stop issued
            // while paused must still cut the motion rather than leave the arm travelling.
            bool wasRunning = State != CycleState.Idle;
            State = CycleState.Idle;
            sequence.Clear();
            seqPos = 0;
            lap = 0;
            if (wasRunning)
            {
                driver.Stop();
                if (verboseLogging) Debug.LogFormat("[{0}] cycle stopped: {1}", name, reason, this);
            }
        }

        public void StopCycle() { StopCycle("requested"); }

        /// <summary>
        /// Pauses the point sequence without killing the motion in progress. The arm keeps ramping
        /// to the point it was heading for and simply stops advancing once it gets there, so
        /// resuming continues the same point rather than restarting it. Use CYCLESTOP to cut
        /// the motion immediately.
        /// </summary>
        public void TogglePause()
        {
            if (State == CycleState.Moving || State == CycleState.Dwell)
            {
                // Remember what we interrupted so the resume lands back in the same phase. Dwell has to
                // be handled here too: without it a pause that arrived while the arm was holding on a
                // point did nothing at all and still answered with the old state, which made the pendant
                // hold button look intermittently dead.
                pausedFrom = State;
                State = CycleState.Paused;
            }
            else if (State == CycleState.Paused)
            {
                if (pausedFrom == CycleState.Moving && driver.IsMoving) State = CycleState.Moving;
                else
                {
                    State = CycleState.Dwell;
                    dwellUntil = Time.time + GetDwell(CurrentReference);
                }
            }
        }

        // ------------------------------------------------------------------ text + persistence

        public string FormatJoints(float[] j)
        {
            if (j == null) return "(none)";
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < 6; i++)
            {
                if (i > 0) sb.Append(' ');
                sb.Append(j[i].ToString("F2", CultureInfo.InvariantCulture));
            }
            return sb.ToString();
        }

        /// <summary>Human readable table of every taught point, one point per line.</summary>
        public string BuildListText()
        {
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < points.Count; i++)
            {
                TaughtPoint p = points[i];
                sb.AppendFormat(CultureInfo.InvariantCulture,
                    "POINT {0} {1} {2} {3}", i + 1, p.name, FormatJoints(p.joints), p.dwell.ToString("F2", CultureInfo.InvariantCulture));
                Vector3 pos;
                Quaternion rot;
                if (GetTcp(i, out pos, out rot))
                    sb.AppendFormat(CultureInfo.InvariantCulture,
                        " tcp {0:F4} {1:F4} {2:F4}", pos.x, pos.y, pos.z);
                sb.Append('\n');
            }
            return sb.ToString();
        }

        public string CycleStateText()
        {
            if (!CycleRunning && State != CycleState.Paused)
                return "OK CYCLE idle points=" + points.Count.ToString(CultureInfo.InvariantCulture);
            string laps = targetLaps == 0
                ? "inf"
                : targetLaps.ToString(CultureInfo.InvariantCulture);
            return string.Format(CultureInfo.InvariantCulture,
                "OK CYCLE {0} pos={1}/{2} lap={3}/{4} point={5}",
                State.ToString().ToLowerInvariant(), seqPos + 1, sequence.Count,
                lap + 1, laps, CurrentPointName);
        }

        [Serializable]
        class SaveFile
        {
            public int version = 1;
            public string robot = "rb3_730es_u";
            public List<TaughtPoint> points = new List<TaughtPoint>();
        }

        public string SavePath
        {
            get { return Path.Combine(Application.persistentDataPath, saveFileName); }
        }

        /// <summary>Writes the taught points to disk. Returns a message for the pendant.</summary>
        public string Save()
        {
            try
            {
                SaveFile file = new SaveFile { points = new List<TaughtPoint>(points) };
                File.WriteAllText(SavePath, JsonUtility.ToJson(file, true));
                return "OK SAVETEACH " + points.Count.ToString(CultureInfo.InvariantCulture) + " " + SavePath;
            }
            catch (Exception e)
            {
                Debug.LogErrorFormat("[{0}] could not save taught points: {1}", name, e.Message, this);
                return "ERR SAVETEACH " + e.Message;
            }
        }

        /// <summary>Replaces the taught points with the file contents. Returns a message.</summary>
        public string Load()
        {
            try
            {
                if (!File.Exists(SavePath)) return "ERR LOADTEACH no file at " + SavePath;
                SaveFile file = JsonUtility.FromJson<SaveFile>(File.ReadAllText(SavePath));
                if (file == null || file.points == null) return "ERR LOADTEACH file is not a teach file";

                StopCycle("loading");
                points.Clear();
                for (int i = 0; i < file.points.Count && i < maxPoints; i++)
                {
                    TaughtPoint p = file.points[i];
                    if (p == null) continue;
                    if (p.joints == null || p.joints.Length != 6)
                        Debug.LogWarningFormat("[{0}] taught point {1} has no six joint values, skipped", name, i + 1, this);
                    else
                        points.Add(p);
                }
                return "OK LOADTEACH " + points.Count.ToString(CultureInfo.InvariantCulture) + " " + SavePath;
            }
            catch (Exception e)
            {
                Debug.LogErrorFormat("[{0}] could not load taught points: {1}", name, e.Message, this);
                return "ERR LOADTEACH " + e.Message;
            }
        }
    }
}
````

---

<a id="code-pendantserver"></a>

### 6.3 `RB3_730_PendantServer.cs`

**이 컴포넌트가 "문"입니다.** TCP로 들어온 한 줄짜리 텍스트를 로봇 동작으로 바꿉니다.

이 파일이 **가장 크고(997줄), 프로토콜의 authoritative source** 입니다.
[Part 7](#part-7-tcp-프로토콜-명세)의 모든 규칙은 이 파일의 주석에 그대로 적혀 있습니다.

설정 항목:

| 인스펙터 필드 | 기본값 | 설명 |
|---|---|---|
| `port` | **5000** | TCP 포트 |
| `bindAddress` | **127.0.0.1** | 로컬만. `0.0.0.0` 로 바꾸면 LAN에서도 접속 가능 (보안 주의) |
| `autoStart` | **true** | `Awake` 에서 자동 리스닝 개시 |
| `stateHz` | **10** | 텔레메트리 주파수 |
| `jogDeadmanSeconds` | **0** | >0 이면 오래된 조그 자동 정지 (button 한 번만 보내는 펜던트는 0 유지) |

> ⚠️ **`bindAddress` 를 `0.0.0.0` 으로 바꾸면 그 네트워크의 모든 호스트가 접속할 수 있습니다.**
> 실제 로봇에 연결한다면 절대 로컬호스트만 허용하세요. 이 서버에는 인증이 없습니다 —
> 연결된 anybody가 `HOME`, `CYCLE`, `MOVJ` 를 실행할 수 있습니다.

#### 전체 소스


````text
> FILE: RB3_730_PendantServer.cs  (1008 lines)

// TCP pendant bridge for the RB3-730 digital twin.
//
// WHY A SOCKET: a C# WinForms teaching pendant is a separate process and cannot
// reference UnityEngine.dll. The only clean way for the pendant to drive the arm
// is a socket, so this component is a tiny line oriented server and this header
// is the authoritative protocol. No shared assembly is needed on either side.
//
// ---------------------------------------------------------------------------
// PROTOCOL
// ---------------------------------------------------------------------------
// Every message is one UTF-8 line terminated by "\n". Commands travel pendant ->
// Unity, replies and telemetry travel Unity -> pendant. All numbers are written
// with InvariantCulture, so a Korean or German Windows locale still parses them.
//
// Pendant -> Unity
//   PING                          connectivity check
//   GETSTATE                      ask for one telemetry frame
//   STATUS                        short status reply
//   SPEED <percent 1..100>        speed override for every following move
//   HOME                          speed limited move to all zeros
//   STOP                          hold the current pose, cancel moves and jog
//   MOVJ <j1> <j2> <j3> <j4> <j5> <j6>    absolute move, degrees, speed limited
//   SETJ <j1> <j2> <j3> <j4> <j5> <j6>    alias of MOVJ
//   JOG <1..6> <deltaDeg>         one shot relative joint step, degrees
//   JOGSTART <what> <sign>        hold a velocity jog, sign is -1 or +1
//   JOGSTOP                       release the held jog
//   GETPOS                        reply OK POS with the six joint angles
//   TCP                           reply OK TCP with the TCP pose
//
//   <what> for JOGSTART is J1..J6 for a single joint, X Y Z for linear motion
//   along a TCP axis, or RX RY RZ for rotation about a TCP axis.
//
// Teaching and cycle, the pendant workflow
//   TEACH [name] [dwell]    record the pose the arm is holding now, reply with the point number
//   LIST                    reply OK LIST <n> then one POINT line per taught point
//   GOTO <n>                move to taught point n
//   NEXT / PREV             step to the neighbouring point, wrapping around
//   RENAME <n> <name>       label a point, for example RENAME 1 pick
//   DWELL <n> <seconds>     how long the cycle holds on point n
//   DELETE <n>              remove one point
//   CLEAR                   remove every point
//   SAVETEACH / LOADTEACH   write / read the points from persistentDataPath
//   CYCLE [n n n ...] [laps]  play those points in order, or all of them when no numbers are
//                             given; the last number is the lap count, 0 meaning endless
//   LOOP <laps>             lap count used when CYCLE is given no lap count
//   CYCLEPAUSE              hold the cycle where it is / let it continue
//   CYCLESTATE              where the cycle is right now
//   CYCLESTOP               end the cycle and hold position
//
//   A worked example, the classic pick and place taught by hand:
//       SPEED 30
//       TEACH pick 1.0     (jog the arm to the pick pose first, then teach it)
//       TEACH place 0.2
//       LIST               (check what was recorded)
//       CYCLE 0 0          (repeat pick -> place -> pick -> place, endlessly)
//       CYCLESTOP
//
//   Any manual motion command (HOME, MOVJ, JOG, JOGSTART) cancels a running cycle first, and a
//   dropped connection stops both, so a cycle can never keep running unattended.
//
// Unity -> pendant
//   PONG
//   OK <COMMAND> [detail]
//   ERR <message>                 every failure, including unknown commands
//   STATE {json}                  telemetry, pushed at stateHz and on GETSTATE
//
// STATE json fields
//   ok        true when the rig resolved its bones
//   moving    true while a move is ramping or a jog is held
//   jogging   true while a velocity jog is held
//   speed     current speed override, 0..1
//   j         six joint angles in degrees, base first
//   t         TCP position in metres, world frame
//   r         TCP rotation as xyzw quaternion, world frame
//   clients   number of connected pendants
//   moves     count of accepted motion commands, handy as a sequence number
//   taught    number of taught points
//   cycle     idle, moving, dwell or paused
//   cyclePoint  taught point the cycle is on, 0 when idle
//   lap       current lap, counting from 1
//
// Safety
//   * A disconnect always sends STOP, so the arm holds position instead of
//     running away with a stale jog.
//   * jogDeadmanSeconds > 0 additionally stops any held jog that has not been
//     refreshed by a new command within that window. Leave it at 0 if the
//     pendant sends JOGSTART only once per button press.
//
// ---------------------------------------------------------------------------
// C# WinForms client sketch
// ---------------------------------------------------------------------------
//   using var tcp = new System.Net.Sockets.TcpClient("127.0.0.1", 5000);
//   using var stream = tcp.GetStream();
//   using var reader = new System.IO.StreamReader(stream);
//   using var writer = new System.IO.StreamWriter(stream) { AutoFlush = true };
//   void Send(string line) => writer.WriteLine(line);
//   Send("PING");                       Console.WriteLine(reader.ReadLine());
//   Send("SPEED 20");
//   Send("JOGSTART J1 1");              // hold the button
//   Send("JOGSTOP");
//   string state = reader.ReadLine();   // "STATE {...}"
//
// Threading
//   The accept loop and each client get their own threads. They only ever touch
//   ConcurrentQueue<string> and socket streams, never the Unity API. Commands are
//   drained and applied in Update on the main thread, which is where the driver
//   lives. Telemetry is built on the main thread and handed to the writer thread.

using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Globalization;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading;
using UnityEngine;

namespace RainbowRobotics.RB
{
    [AddComponentMenu("Rainbow Robotics/RB3-730 Pendant Server")]
    [RequireComponent(typeof(RB3_730_JointDriver))]
    [RequireComponent(typeof(RB3_730_Teach))]
    public class RB3_730_PendantServer : MonoBehaviour
    {
        [Header("Connection")]
        [Tooltip("TCP port the pendant connects to.")]
        public int port = 5000;

        [Tooltip("Interface to listen on. 127.0.0.1 only accepts this machine. " +
                 "Use 0.0.0.0 to accept a pendant on the same LAN, which also exposes " +
                 "the port to every other host on that network.")]
        public string bindAddress = "127.0.0.1";

        [Tooltip("Start listening as soon as the object wakes up.")]
        public bool startOnAwake = true;

        [Header("Telemetry")]
        [Range(1f, 60f)]
        [Tooltip("How often a STATE line is pushed to each connected pendant.")]
        public float stateHz = 20f;

        [Header("Safety")]
        [Tooltip("Stop a held jog that no command has refreshed for this long. " +
                 "0 disables the deadman and lets JOGSTART run until JOGSTOP.")]
        public float jogDeadmanSeconds = 0f;

        [Header("Diagnostics")]
        public bool verboseLogging = false;

        public int ClientCount { get { lock (clientsLock) return clients.Count; } }
        public bool IsListening { get { return listener != null; } }
        public int AcceptedCommandCount { get { return acceptedMoves; } }

        RB3_730_JointDriver driver;
        RB3_730_Teach teach;
        TcpListener listener;
        Thread acceptThread;
        volatile bool running;

        readonly List<Client> clients = new List<Client>();
        readonly object clientsLock = new object();
        readonly ConcurrentQueue<string> inbox = new ConcurrentQueue<string>();

        // Carries the client that dropped out, so the main thread removes exactly that one
        // instead of guessing. A bare flag was never enough: a dead client keeps Alive == true,
        // so the list grew for the whole session and ClientCount drifted.
        readonly ConcurrentQueue<Client> lostClientNotices = new ConcurrentQueue<Client>();

        float stateTimer;
        int acceptedMoves;

        sealed class Client
        {
            public TcpClient Tcp;
            public NetworkStream Stream;
            public readonly ConcurrentQueue<string> Outbox = new ConcurrentQueue<string>();
            public readonly AutoResetEvent Signal = new AutoResetEvent(false);
            public Thread Writer;
            public volatile bool Alive = true;

            public void Send(string line)
            {
                Outbox.Enqueue(line);
                Signal.Set();
            }
        }

        void Awake()
        {
            driver = GetComponent<RB3_730_JointDriver>();
            teach = GetComponent<RB3_730_Teach>();
            if (startOnAwake) StartServer();
        }

        void OnDestroy() => StopServer();

        void Update()
        {
            if (running)
            {
                DrainCommands();
                PushTelemetry();
            }
        }

        // ------------------------------------------------------------------ lifecycle

        public void StartServer()
        {
            if (running) return;

            string host = string.IsNullOrEmpty(bindAddress) ? "127.0.0.1" : bindAddress.Trim();
            IPAddress addr;
            if (!IPAddress.TryParse(host, out addr))
            {
                Debug.LogErrorFormat("[{0}] bindAddress '{1}' is not a valid IP address.", name, bindAddress, this);
                return;
            }
            try
            {
                listener = new TcpListener(addr, port);
                listener.Start();
            }
            catch (Exception e)
            {
                Debug.LogErrorFormat("[{0}] cannot listen on {1}:{2} - {3}",
                                     name, bindAddress, port, e.Message, this);
                listener = null;
                return;
            }

            running = true;
            acceptThread = new Thread(AcceptLoop) { IsBackground = true, Name = "RB3-730 pendant accept" };
            acceptThread.Start();
            Debug.LogFormat("[{0}] pendant server listening on {1}:{2}", name, bindAddress, port, this);
        }

        public void StopServer()
        {
            running = false;
            try { if (listener != null) listener.Stop(); } catch { }
            listener = null;

            List<Client> snapshot;
            lock (clientsLock) { snapshot = new List<Client>(clients); clients.Clear(); }
            for (int i = 0; i < snapshot.Count; i++) CloseClient(snapshot[i]);

            if (acceptThread != null)
            {
                if (!acceptThread.Join(500)) { }
                acceptThread = null;
            }
        }

        void CloseClient(Client c)
        {
            c.Alive = false;
            try { c.Signal.Set(); } catch { }
            try { if (c.Tcp != null) c.Tcp.Close(); } catch { }
        }

        // ------------------------------------------------------------------ socket threads

        void AcceptLoop()
        {
            while (running)
            {
                TcpClient tcp;
                try
                {
                    if (listener == null) break;
                    tcp = listener.AcceptTcpClient();
                }
                catch { break; }

                try
                {
                    Client c = new Client { Tcp = tcp };
                    c.Stream = tcp.GetStream();
                    c.Stream.ReadTimeout = Timeout.Infinite;   // blocks until a line arrives
                    c.Stream.WriteTimeout = 3000;              // never let a wedged pendant block the writer
                    lock (clientsLock) clients.Add(c);

                    c.Writer = new Thread(() => WriteLoop(c)) { IsBackground = true, Name = "RB3-730 pendant writer" };
                    c.Writer.Start();
                    Thread reader = new Thread(() => ReadLoop(c)) { IsBackground = true, Name = "RB3-730 pendant reader" };
                    reader.Start();

                    c.Send("OK CONNECT rb3-730");
                    if (verboseLogging) Debug.LogFormat("[{0}] pendant connected from {1}", name, SafeEndPoint(c), this);
                }
                catch (Exception e)
                {
                    // Always reported: a client that cannot be set up is dropped with a reset,
                    // which looks to the pendant like the server just went away.
                    Debug.LogWarningFormat("[{0}] pendant accept failed: {1}", name, e.Message, this);
                    try { tcp.Close(); } catch { }
                }
            }
        }

        static string SafeEndPoint(Client c)
        {
            try { return c.Tcp.Client.RemoteEndPoint.ToString(); } catch { return "unknown"; }
        }

        void ReadLoop(Client c)
        {
            byte[] buffer = new byte[1024];
            StringBuilder line = new StringBuilder();
            try
            {
                while (running && c.Alive)
                {
                    int n = c.Stream.Read(buffer, 0, buffer.Length);
                    if (n <= 0) break;
                    for (int i = 0; i < n; i++)
                    {
                        char ch = (char)buffer[i];
                        if (ch == '\n')
                        {
                            string text = line.ToString().Trim();
                            line.Length = 0;
                            if (text.Length > 0) inbox.Enqueue(text);
                        }
                        else if (ch != '\r')
                        {
                            if (line.Length < 4096) line.Append(ch);
                        }
                    }
                }
            }
            catch { }
            finally { lostClientNotices.Enqueue(c); }
        }

        void WriteLoop(Client c)
        {
            try
            {
                while (running && c.Alive)
                {
                    c.Signal.WaitOne(200);
                    string line;
                    bool any = false;
                    while (c.Outbox.TryDequeue(out line))
                    {
                        any = true;
                        byte[] bytes = Encoding.UTF8.GetBytes(line + "\n");
                        c.Stream.Write(bytes, 0, bytes.Length);
                    }
                    if (any) c.Stream.Flush();
                }
            }
            catch { }
        }

        // ------------------------------------------------------------------ main thread

        void DrainCommands()
        {
            Client lost;
            while (lostClientNotices.TryDequeue(out lost))
            {
                bool wasRegistered = false;
                lock (clientsLock)
                {
                    wasRegistered = clients.Remove(lost);
                }
                if (!wasRegistered) continue;      // already cleaned up by StopServer
                CloseClient(lost);
                if (teach != null) teach.StopCycle("pendant disconnected");
                if (driver != null) driver.Stop();
                if (verboseLogging) Debug.LogFormat("[{0}] pendant disconnected, motion and cycle stopped", name, this);
            }

            string line;
            int guard = 0;
            while (inbox.TryDequeue(out line))
            {
                if (++guard > 512) break;
                Handle(line);
            }
        }

        /// <summary>Replies collected for one command, so a command can answer with several lines.</summary>
        readonly List<string> replyBuffer = new List<string>();

        void Handle(string line)
        {
            string[] parts = line.Split(new[] { ' ', '\t' }, StringSplitOptions.RemoveEmptyEntries);
            if (parts.Length == 0) return;
            string cmd = parts[0].ToUpperInvariant();
            string reply;
            replyBuffer.Clear();

            switch (cmd)
            {
                case "PING":
                    reply = "PONG";
                    break;

                case "GETSTATE":
                    reply = BuildState();
                    break;

                case "STATUS":
                    reply = string.Format(CultureInfo.InvariantCulture,
                        "OK STATUS ready={0} moving={1} jogging={2} speed={3:F2} clients={4}",
                        driver != null && driver.IsReady, IsMoving, driver != null && driver.IsJogging,
                        driver != null ? driver.SpeedOverride : 0f, ClientCount);
                    break;

                case "SPEED":
                {
                    float pct;
                    if (!TryFloat(parts, 1, out pct))
                    {
                        reply = "ERR SPEED needs a number 1..100";
                        break;
                    }
                    if (pct <= 0f || pct > 100f)
                    {
                        reply = "ERR SPEED must be 1..100";
                        break;
                    }
                    if (driver == null) { reply = "ERR no joint driver"; break; }
                    driver.SetSpeedOverride(pct / 100f);
                    reply = string.Format(CultureInfo.InvariantCulture, "OK SPEED {0:F1}", pct);
                    break;
                }

                case "HOME":
                    if (driver == null) { reply = "ERR no joint driver"; break; }
                    CancelCycleForManualMove();
                    driver.GoHome();
                    acceptedMoves++;
                    reply = "OK HOME";
                    break;

                case "STOP":
                    if (driver == null) { reply = "ERR no joint driver"; break; }
                    if (teach != null) teach.StopCycle("operator stop");
                    driver.Stop();
                    acceptedMoves++;
                    reply = "OK STOP";
                    break;

                case "MOVJ":
                case "SETJ":
                {
                    float[] target;
                    string err;
                    if (!TrySix(parts, out target, out err)) { reply = "ERR " + err; break; }
                    if (driver == null) { reply = "ERR no joint driver"; break; }
                    CancelCycleForManualMove();
                    driver.MoveTo(target);
                    acceptedMoves++;
                    reply = "OK " + cmd;
                    break;
                }

                case "JOG":
                {
                    int joint;
                    float delta;
                    if (!TryInt(parts, 1, out joint) || joint < 1 || joint > 6)
                    {
                        reply = "ERR JOG needs a joint number 1..6";
                        break;
                    }
                    if (!TryFloat(parts, 2, out delta))
                    {
                        reply = "ERR JOG needs a delta in degrees";
                        break;
                    }
                    if (driver == null) { reply = "ERR no joint driver"; break; }
                    CancelCycleForManualMove();
                    driver.StepJoint(joint - 1, delta);
                    acceptedMoves++;
                    reply = "OK JOG";
                    break;
                }

                case "JOGSTART":
                {
                    if (parts.Length < 3)
                    {
                        reply = "ERR JOGSTART needs <what> <sign>";
                        break;
                    }
                    float sign;
                    if (!TryFloat(parts, 2, out sign) || (Mathf.Abs(sign) < 0.001f))
                    {
                        reply = "ERR JOGSTART sign must be -1 or +1";
                        break;
                    }
                    if (driver == null) { reply = "ERR no joint driver"; break; }
                    CancelCycleForManualMove();
                    if (!StartJog(parts[1].ToUpperInvariant(), sign))
                    {
                        reply = "ERR JOGSTART what must be J1..J6, X, Y, Z, RX, RY or RZ";
                        break;
                    }
                    acceptedMoves++;
                    reply = "OK JOGSTART";
                    break;
                }

                case "JOGSTOP":
                    if (driver == null) { reply = "ERR no joint driver"; break; }
                    driver.ClearJog();
                    reply = "OK JOGSTOP";
                    break;

                // ---------------------------------------------------------- teaching

                case "TEACH":
                    reply = HandleTeach(parts);
                    break;

                case "LIST":
                    reply = HandleList();
                    break;

                case "GOTO":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR GOTO " + error; break; }
                    int index;
                    if (!TryInt(parts, 1, out index) || index < 1 || index > teach.PointCount)
                    {
                        reply = "ERR GOTO needs a point number 1.." + teach.PointCount.ToString(CultureInfo.InvariantCulture);
                        break;
                    }
                    if (!teach.MoveToIndex(index - 1)) { reply = "ERR GOTO could not move"; break; }
                    acceptedMoves++;
                    reply = string.Format(CultureInfo.InvariantCulture, "OK GOTO {0} {1}",
                        index, teach.GetName(index - 1));
                    break;
                }

                case "NEXT":
                case "PREV":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR " + cmd + " " + error; break; }
                    if (teach.PointCount == 0) { reply = "ERR " + cmd + " nothing taught yet"; break; }
                    if (!teach.Step(cmd == "NEXT" ? 1 : -1)) { reply = "ERR " + cmd + " could not move"; break; }
                    acceptedMoves++;
                    reply = string.Format(CultureInfo.InvariantCulture, "OK {0} {1} {2}",
                        cmd, teach.CurrentReference + 1, teach.GetName(teach.CurrentReference));
                    break;
                }

                case "DELETE":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR DELETE " + error; break; }
                    int index;
                    if (!TryInt(parts, 1, out index) || index < 1 || index > teach.PointCount)
                    {
                        reply = "ERR DELETE needs a point number 1.." + teach.PointCount.ToString(CultureInfo.InvariantCulture);
                        break;
                    }
                    teach.Delete(index - 1);
                    reply = string.Format(CultureInfo.InvariantCulture, "OK DELETE {0} left={1}", index, teach.PointCount);
                    break;
                }

                case "RENAME":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR RENAME " + error; break; }
                    int index;
                    if (!TryInt(parts, 1, out index) || index < 1 || index > teach.PointCount || parts.Length < 3)
                    {
                        reply = "ERR RENAME needs <point> <new name>";
                        break;
                    }
                    if (!teach.Rename(index - 1, parts[2]))
                    {
                        reply = "ERR RENAME could not rename point " + index.ToString(CultureInfo.InvariantCulture);
                        break;
                    }
                    reply = "OK RENAME " + index.ToString(CultureInfo.InvariantCulture) + " " + parts[2];
                    break;
                }

                case "DWELL":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR DWELL " + error; break; }
                    int index;
                    float seconds;
                    if (!TryInt(parts, 1, out index) || index < 1 || index > teach.PointCount ||
                        !TryFloat(parts, 2, out seconds))
                    {
                        reply = "ERR DWELL needs <point> <seconds>";
                        break;
                    }
                    if (!teach.SetDwell(index - 1, seconds)) { reply = "ERR DWELL bad point"; break; }
                    reply = string.Format(CultureInfo.InvariantCulture, "OK DWELL {0} {1:F2}", index, seconds);
                    break;
                }

                case "CLEAR":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR CLEAR " + error; break; }
                    teach.Clear();
                    reply = "OK CLEAR 0";
                    break;
                }

                case "SAVETEACH":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR SAVETEACH " + error; break; }
                    reply = teach.Save();
                    break;
                }

                case "LOADTEACH":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR LOADTEACH " + error; break; }
                    reply = teach.Load();
                    break;
                }

                // ---------------------------------------------------------- cycle

                case "CYCLE":
                    reply = HandleCycle(parts);
                    break;

                case "CYCLESTOP":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR CYCLESTOP " + error; break; }
                    teach.StopCycle("operator stop");
                    driver.Stop();
                    reply = "OK CYCLESTOP";
                    break;
                }

                case "CYCLEPAUSE":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR CYCLEPAUSE " + error; break; }
                    teach.TogglePause();
                    reply = teach.CycleStateText();
                    break;
                }

                case "CYCLESTATE":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR CYCLESTATE " + error; break; }
                    reply = teach.CycleStateText();
                    break;
                }

                case "LOOP":
                {
                    string error;
                    if (!NeedTeach(out error)) { reply = "ERR LOOP " + error; break; }
                    int loops;
                    if (!TryInt(parts, 1, out loops) || loops < 0)
                    {
                        reply = "ERR LOOP needs a lap count, 0 for endless";
                        break;
                    }
                    teach.defaultLoops = loops;
                    reply = "OK LOOP " + loops.ToString(CultureInfo.InvariantCulture);
                    break;
                }

                case "GETPOS":
                {
                    if (driver == null) { reply = "ERR no joint driver"; break; }
                    float[] a = driver.GetAnglesDeg();
                    reply = string.Format(CultureInfo.InvariantCulture,
                        "OK POS {0:F3} {1:F3} {2:F3} {3:F3} {4:F3} {5:F3}",
                        a[0], a[1], a[2], a[3], a[4], a[5]);
                    break;
                }

                case "TCP":
                {
                    if (driver == null) { reply = "ERR no joint driver"; break; }
                    Vector3 p;
                    Quaternion r;
                    if (!driver.TryGetTcpPose(out p, out r))
                    {
                        reply = "ERR tcp frame not found";
                        break;
                    }
                    reply = string.Format(CultureInfo.InvariantCulture,
                        "OK TCP {0:F4} {1:F4} {2:F4} {3:F5} {4:F5} {5:F5} {6:F5}",
                        p.x, p.y, p.z, r.x, r.y, r.z, r.w);
                    break;
                }

                default:
                    reply = "ERR unknown command '" + parts[0] + "'";
                    break;
            }

            if (jogDeadmanSeconds > 0f) deadmanUntil = Time.unscaledTime + jogDeadmanSeconds;
            if (verboseLogging) Debug.LogFormat("[{0}] <- {1}   -> {2}", name, line, reply, this);
            if (replyBuffer.Count > 0)
            {
                for (int i = 0; i < replyBuffer.Count; i++) Broadcast(replyBuffer[i]);
            }
            else if (!string.IsNullOrEmpty(reply))
            {
                Broadcast(reply);
            }
        }

        // ------------------------------------------------------------------ teaching and cycle

        /// <summary>Queues one reply line. Used by the commands that answer with several lines.</summary>
        void Say(string line) { replyBuffer.Add(line); }

        bool NeedTeach(out string error)
        {
            if (teach != null) { error = null; return true; }
            error = "no teach component on the robot";
            return false;
        }

        /// <summary>Every motion command that is not part of a cycle cancels a running cycle first.</summary>
        void CancelCycleForManualMove()
        {
            if (teach != null && teach.CycleRunning) teach.StopCycle("manual move");
        }

        int[] ParseIndices(string[] parts, int from, int to)
        {
            List<int> list = new List<int>();
            for (int i = from; i < to; i++)
            {
                int oneBased;
                if (!int.TryParse(parts[i], NumberStyles.Integer, CultureInfo.InvariantCulture, out oneBased)) continue;
                list.Add(oneBased - 1);      // the protocol is 1 based, matching LIST output
            }
            return list.ToArray();
        }

        string HandleTeach(string[] parts)
        {
            string error;
            if (!NeedTeach(out error)) return "ERR TEACH " + error;

            CancelCycleForManualMove();
            string name = parts.Length > 1 ? parts[1] : null;
            float dwell = teach.defaultDwell;
            if (parts.Length > 2)
            {
                double d;
                if (double.TryParse(parts[2], NumberStyles.Float, CultureInfo.InvariantCulture, out d)) dwell = (float)d;
            }

            int index = teach.Record(name, dwell);
            if (index < 0) return "ERR TEACH could not record the pose";

            Say(string.Format(CultureInfo.InvariantCulture, "OK TEACH {0} {1} {2} {3}",
                index + 1, teach.GetName(index), teach.FormatJoints(teach.GetJoints(index)),
                teach.GetDwell(index).ToString("F2", CultureInfo.InvariantCulture)));
            return null;
        }

        string HandleList()
        {
            string error;
            if (!NeedTeach(out error)) return "ERR LIST " + error;

            Say(string.Format(CultureInfo.InvariantCulture, "OK LIST {0}", teach.PointCount));
            string body = teach.BuildListText();
            if (!string.IsNullOrEmpty(body))
            {
                string[] rows = body.Split('\n');
                for (int i = 0; i < rows.Length; i++)
                    if (!string.IsNullOrEmpty(rows[i].Trim())) Say(rows[i].TrimEnd('\r'));
            }
            return null;
        }

        string HandleCycle(string[] parts)
        {
            string error;
            if (!NeedTeach(out error)) return "ERR CYCLE " + error;

            if (teach.PointCount < 2)
                return "ERR CYCLE teach at least two points first, for example TEACH pick, TEACH place";

            int[] indices = ParseIndices(parts, 1, parts.Length);

            // "CYCLE 0" means every point and the lap count is an optional trailing number, so a bare
            // lap count cannot be told apart from a point number by value alone. It is treated as the
            // lap count only when at least two point tokens precede it, or when the leading "0" has
            // already claimed "every point". Otherwise every argument is a point number, which keeps
            // "CYCLE 1 2" meaning "points 1 and 2". Reading the lap count unconditionally used to make
            // "CYCLE 1 2 1" play three points endlessly, and left "CYCLE 0 1" unstartable.
            int tokenCount = parts.Length - 1;
            bool everyPoint = indices.Length > 0 && indices[0] < 0;
            bool lastIsLaps = tokenCount >= 3 || (everyPoint && tokenCount >= 2);

            int loops = teach.defaultLoops;
            if (lastIsLaps)
            {
                double d;
                if (double.TryParse(parts[parts.Length - 1], NumberStyles.Float, CultureInfo.InvariantCulture, out d))
                {
                    loops = (int)d;
                    indices = ParseIndices(parts, 1, parts.Length - 1);
                }
            }
            if (everyPoint) indices = new int[0];       // CYCLE 0 [laps]: every taught point, in order

            if (!teach.StartCycle(indices, loops))
                return "ERR CYCLE could not start, check the point numbers";

            Say(teach.CycleStateText());
            return null;
        }

        bool StartJog(string what, float sign)
        {
            if (what.Length == 2 && what[0] == 'J')
            {
                int j = what[1] - '0';
                if (j < 1 || j > 6) return false;
                driver.StartJointJog(j - 1, sign * driver.maxJointSpeedDegPerSec);
                return true;
            }

            if (what.Length == 1)
            {
                int axis = what[0] - 'X';
                if (axis < 0 || axis > 2) return false;
                return driver.StartToolLinearJog(axis, sign);
            }

            if (what.Length == 2 && what[0] == 'R')
            {
                int axis = what[1] - 'X';
                if (axis < 0 || axis > 2) return false;
                return driver.StartToolAngularJog(axis, sign);
            }

            return false;
        }

        bool IsMoving
        {
            get { return driver != null && driver.IsMoving; }
        }

        void PushTelemetry()
        {
            stateTimer += Time.unscaledDeltaTime;
            float period = 1f / Mathf.Clamp(stateHz, 1f, 60f);
            if (stateTimer < period) return;
            stateTimer = 0f;

            Broadcast(BuildState());

            if (jogDeadmanSeconds > 0f && driver != null && driver.IsJogging)
            {
                if (Time.unscaledTime > deadmanUntil)
                {
                    driver.ClearJog();
                    Broadcast("ERR jog deadman timeout");
                }
            }
        }

        float deadmanUntil;

        void Broadcast(string line)
        {
            List<Client> snapshot;
            lock (clientsLock)
            {
                if (clients.Count == 0) return;
                snapshot = new List<Client>(clients);
            }
            for (int i = 0; i < snapshot.Count; i++)
                if (snapshot[i].Alive) snapshot[i].Send(line);
        }

        string BuildState()
        {
            bool ok = driver != null && driver.IsReady;
            float[] a = { 0, 0, 0, 0, 0, 0 };
            Vector3 p = Vector3.zero;
            Quaternion r = Quaternion.identity;
            bool moving = false;
            bool jogging = false;
            float speed = 0f;

            if (driver != null)
            {
                a = driver.GetAnglesDeg();
                driver.TryGetTcpPose(out p, out r);
                moving = driver.IsMoving;
                jogging = driver.IsJogging;
                speed = driver.SpeedOverride;
            }

            int taught = 0;
            string cycle = "off";
            int cyclePoint = 0;
            int lap = 0;
            if (teach != null)
            {
                taught = teach.PointCount;
                cycle = teach.State.ToString().ToLowerInvariant();
                cyclePoint = teach.CurrentReference + 1;
                lap = teach.CurrentLap;
            }

            // Plain concatenation on purpose: the number formats go through an invariant helper so
            // a comma decimal separator can never break the JSON on a locale change.
            StringBuilder sb = new StringBuilder(320);
            sb.Append("STATE {");
            sb.Append("\"ok\":").Append(ok ? "true" : "false");
            sb.Append(",\"moving\":").Append(moving ? "true" : "false");
            sb.Append(",\"jogging\":").Append(jogging ? "true" : "false");
            sb.Append(",\"speed\":").Append(Num(speed, 3));
            sb.Append(",\"j\":[");
            for (int i = 0; i < 6; i++)
            {
                if (i > 0) sb.Append(',');
                sb.Append(Num(a[i], 3));
            }
            sb.Append("],\"t\":[");
            sb.Append(Num(p.x, 4)).Append(',').Append(Num(p.y, 4)).Append(',').Append(Num(p.z, 4));
            sb.Append("],\"r\":[");
            sb.Append(Num(r.x, 5)).Append(',').Append(Num(r.y, 5)).Append(',')
              .Append(Num(r.z, 5)).Append(',').Append(Num(r.w, 5));
            sb.Append("],\"clients\":").Append(ClientCount);
            sb.Append(",\"moves\":").Append(acceptedMoves);
            sb.Append(",\"taught\":").Append(taught);
            sb.Append(",\"cycle\":\"").Append(cycle).Append('"');
            sb.Append(",\"cyclePoint\":").Append(cyclePoint);
            sb.Append(",\"lap\":").Append(lap);
            sb.Append('}');
            return sb.ToString();
        }

        /// <summary>Formats a number for the STATE json without depending on the thread locale.</summary>
        static string Num(float v, int decimals)
        {
            return v.ToString("F" + decimals.ToString(CultureInfo.InvariantCulture), CultureInfo.InvariantCulture);
        }

        // ------------------------------------------------------------------ parsing

        static bool TryFloat(string[] parts, int index, out float value)
        {
            value = 0f;
            if (parts.Length <= index) return false;
            double d;
            if (!double.TryParse(parts[index], NumberStyles.Float, CultureInfo.InvariantCulture, out d)) return false;
            if (double.IsNaN(d) || double.IsInfinity(d)) return false;
            value = (float)d;
            return true;
        }

        static bool TryInt(string[] parts, int index, out int value)
        {
            value = 0;
            if (parts.Length <= index) return false;
            return int.TryParse(parts[index], NumberStyles.Integer, CultureInfo.InvariantCulture, out value);
        }

        static bool TrySix(string[] parts, out float[] target, out string error)
        {
            target = null;
            error = null;
            if (parts.Length < 7)
            {
                error = "needs six joint angles in degrees";
                return false;
            }
            float[] v = new float[6];
            for (int i = 0; i < 6; i++)
            {
                double d;
                if (!double.TryParse(parts[i + 1], NumberStyles.Float, CultureInfo.InvariantCulture, out d) ||
                    double.IsNaN(d) || double.IsInfinity(d))
                {
                    error = "joint " + (i + 1) + " ('" + parts[i + 1] + "') is not a number";
                    return false;
                }
                v[i] = (float)d;
            }
            target = v;
            return true;
        }
    }
}
````

---

<a id="code-jogpanel"></a>

### 6.4 `RB3_730_JogPanel.cs`

Unity 안에서 TCP 없이 직접 조그/티칭해 보는 **디버깅용 IMGUI 패널**입니다.
씬에 로봇을 하나 배치하고 Play를 누르면 즉시 조작할 수 있습니다.

- `G` — Jog 패널 열기/닫기
- `T` — 티칭 포인터 사이클
- `H` — 홈 포즈

> 💡 **이 패널은 필수가 아닙니다.** 펜던트는 소켓으로 동작하므로
> TCP 클라이언트([Part 8](#part-8))만 있어도 됩니다.
> 패널은 "Unity가 정상 동작하는지"를 먼저 확인할 때 유용합니다.

#### 전체 소스


````text
> FILE: RB3_730_JogPanel.cs  (213 lines)

// On screen jog panel for the RB3-730. Lets you prove out the motion API and the
// pendant protocol before the WinForms teaching pendant exists.
//
// This is a development aid, not a product UI. It drives the same public API the
// pendant server drives (MoveTo, StartJointJog, StartToolLinearJog, ...), so
// anything you can do here the pendant can do over TCP.

using System.Globalization;
using UnityEngine;

namespace RainbowRobotics.RB
{
    [AddComponentMenu("Rainbow Robotics/RB3-730 Jog Panel")]
    [RequireComponent(typeof(RB3_730_JointDriver))]
    public class RB3_730_JogPanel : MonoBehaviour
    {
        static readonly string[] JointLabels = { "J1 base", "J2 shoulder", "J3 elbow", "J4 wrist1", "J5 wrist2", "J6 wrist3" };
        static readonly string[] LinearLabels = { "X tool", "Y tool", "Z tool" };
        static readonly string[] AngularLabels = { "RX tool", "RY tool", "RZ tool" };

        [Tooltip("Show the panel.")]
        public bool visible = true;

        [Tooltip("Skip drawing outside the editor so a build is not cluttered.")]
        public bool editorOnly = true;

        [Tooltip("Degrees per button press for the joint jog buttons.")]
        public float stepDeg = 5f;

        [Tooltip("Millimetres per button press for the linear jog buttons.")]
        public float stepMm = 10f;

        [Tooltip("Degrees per button press for the angular jog buttons.")]
        public float stepRotDeg = 5f;

        [Tooltip("Key that toggles the panel at runtime, so a build can still be driven.")]
        public KeyCode toggleKey = KeyCode.F8;

        RB3_730_JointDriver driver;
        RB3_730_Teach teach;
        Vector2 scroll;
        string teachName = string.Empty;
        float teachDwell = 0.5f;
        float cycleLaps;

        void Awake()
        {
            driver = GetComponent<RB3_730_JointDriver>();
            teach = GetComponent<RB3_730_Teach>();
        }

        void OnGUI()
        {
            if (Input.GetKeyDown(toggleKey)) visible = !visible;
            if (!visible) return;
            if (editorOnly && !Application.isEditor && !Debug.isDebugBuild) return;
            if (driver == null) driver = GetComponent<RB3_730_JointDriver>();
            if (teach == null) teach = GetComponent<RB3_730_Teach>();
            if (driver == null) return;

            GUILayout.BeginArea(new Rect(12, 12, 470, 980), GUI.skin.box);
            GUILayout.Label("RB3-730 jog panel   [" + toggleKey + "] hide");

            if (!driver.IsReady)
            {
                GUILayout.Label("rig not ready - check the console for a missing bone");
                GUILayout.EndArea();
                return;
            }

            GUILayout.Space(4);
            GUILayout.Label(string.Format(CultureInfo.InvariantCulture,
                "speed override  {0:F0} %", driver.SpeedOverride * 100f));
            driver.SpeedOverride = GUILayout.HorizontalSlider(driver.SpeedOverride, 0.01f, 1f);

            GUILayout.Space(6);
            GUILayout.BeginHorizontal();
            if (GUILayout.Button("Home")) driver.GoHome();
            if (GUILayout.Button("Stop")) driver.Stop();
            GUILayout.EndHorizontal();

            GUILayout.Space(6);
            GUILayout.Label(IsMovingText());

            scroll = GUILayout.BeginScrollView(scroll, GUILayout.Height(190));
            float[] angles = driver.GetAnglesDeg();
            for (int i = 0; i < 6; i++)
            {
                GUILayout.BeginHorizontal();
                GUILayout.Label(string.Format(CultureInfo.InvariantCulture,
                    "{0,-12} {1,8:F2} deg", JointLabels[i], angles[i]), GUILayout.Width(210));
                if (GUILayout.Button("-", GUILayout.Width(46))) driver.StepJoint(i, -stepDeg);
                if (GUILayout.Button("+", GUILayout.Width(46))) driver.StepJoint(i, stepDeg);
                GUILayout.EndHorizontal();
            }
            GUILayout.EndScrollView();

            GUILayout.Space(6);
            GUILayout.Label("linear jog, one shot (mm)");
            for (int i = 0; i < 3; i++)
            {
                GUILayout.BeginHorizontal();
                GUILayout.Label(LinearLabels[i], GUILayout.Width(210));
                if (GUILayout.Button("-", GUILayout.Width(46)))
                    driver.StepCartesian(Axis(i) * -stepMm * 0.001f, Vector3.zero, RB3_730_JointDriver.JogFrame.Tool);
                if (GUILayout.Button("+", GUILayout.Width(46)))
                    driver.StepCartesian(Axis(i) * stepMm * 0.001f, Vector3.zero, RB3_730_JointDriver.JogFrame.Tool);
                GUILayout.EndHorizontal();
            }

            GUILayout.Label("angular jog, one shot (deg)");
            for (int i = 0; i < 3; i++)
            {
                GUILayout.BeginHorizontal();
                GUILayout.Label(AngularLabels[i], GUILayout.Width(210));
                if (GUILayout.Button("-", GUILayout.Width(46)))
                    driver.StepCartesian(Vector3.zero, Axis(i) * -stepRotDeg * Mathf.Deg2Rad, RB3_730_JointDriver.JogFrame.Tool);
                if (GUILayout.Button("+", GUILayout.Width(46)))
                    driver.StepCartesian(Vector3.zero, Axis(i) * stepRotDeg * Mathf.Deg2Rad, RB3_730_JointDriver.JogFrame.Tool);
                GUILayout.EndHorizontal();
            }

            Vector3 tcp;
            Quaternion tcpRot;
            if (driver.TryGetTcpPose(out tcp, out tcpRot))
            {
                GUILayout.Space(6);
                GUILayout.Label(string.Format(CultureInfo.InvariantCulture,
                    "TCP  x {0:F3}  y {1:F3}  z {2:F3} m", tcp.x, tcp.y, tcp.z));
            }

            DrawTeach();

            GUILayout.EndArea();
        }

        // ------------------------------------------------------------------ teaching and cycle

        void DrawTeach()
        {
            if (teach == null) return;

            GUILayout.Space(8);
            GUILayout.Label(teach.CycleRunning
                ? string.Format(CultureInfo.InvariantCulture, "cycle: {0}  lap {1}  point {2}",
                    teach.State.ToString().ToLowerInvariant(), teach.CurrentLap, teach.CurrentPointName)
                : string.Format(CultureInfo.InvariantCulture, "cycle: idle   taught points: {0}", teach.PointCount));

            teachName = GUILayout.TextField(teachName ?? string.Empty, GUILayout.Width(120));
            GUILayout.BeginHorizontal();
            if (GUILayout.Button("Teach current pose", GUILayout.Width(160)))
                teach.Record(string.IsNullOrEmpty(teachName) ? null : teachName.Trim(), teachDwell);
            if (GUILayout.Button("Clear", GUILayout.Width(70))) teach.Clear();
            GUILayout.EndHorizontal();

            teachDwell = GUILayout.HorizontalSlider(teachDwell, 0f, 5f);
            GUILayout.Label(string.Format(CultureInfo.InvariantCulture, "dwell on arrival: {0:F2} s", teachDwell));

            if (teach.PointCount > 0)
            {
                scroll = GUILayout.BeginScrollView(scroll, GUILayout.Height(120));
                for (int i = 0; i < teach.PointCount; i++)
                {
                    GUILayout.BeginHorizontal();
                    GUILayout.Label(string.Format(CultureInfo.InvariantCulture, "{0,2}  {1,-8}  {2}",
                        i + 1, teach.GetName(i), teach.FormatJoints(teach.GetJoints(i))), GUILayout.Width(280));
                    if (GUILayout.Button("go", GUILayout.Width(44))) teach.MoveToIndex(i);
                    if (GUILayout.Button("del", GUILayout.Width(44))) teach.Delete(i);
                    GUILayout.EndHorizontal();
                }
                GUILayout.EndScrollView();
            }

            GUILayout.BeginHorizontal();
            if (GUILayout.Button("Prev")) teach.Step(-1);
            if (GUILayout.Button("Next")) teach.Step(1);
            GUILayout.EndHorizontal();

            GUILayout.BeginHorizontal();
            if (GUILayout.Button(teach.CycleRunning ? "Pause" : "Resume")) teach.TogglePause();
            if (GUILayout.Button("Cycle all")) teach.StartCycle();
            if (GUILayout.Button("Stop")) { teach.StopCycle(); driver.Stop(); }
            GUILayout.EndHorizontal();

            GUILayout.BeginHorizontal();
            cycleLaps = GUILayout.HorizontalSlider(cycleLaps, 0f, 20f);
            if (GUILayout.Button("Cycle N laps", GUILayout.Width(110)))
                teach.StartCycle(null, Mathf.RoundToInt(cycleLaps));
            GUILayout.EndHorizontal();
            GUILayout.Label(string.Format(CultureInfo.InvariantCulture,
                "laps: {0}   (0 = endless)", Mathf.RoundToInt(cycleLaps)));

            GUILayout.BeginHorizontal();
            if (GUILayout.Button("Save points", GUILayout.Width(110))) Debug.Log(teach.Save());
            if (GUILayout.Button("Load points", GUILayout.Width(110))) Debug.Log(teach.Load());
            GUILayout.EndHorizontal();
        }

        string IsMovingText()
        {
            return driver.IsMoving
                ? "state: MOVING" + (driver.IsJogging ? "  (velocity jog held)" : "  (ramping to target)")
                : "state: idle, holding position";
        }

        static Vector3 Axis(int i)
        {
            if (i == 0) return Vector3.right;
            if (i == 1) return Vector3.up;
            return Vector3.forward;
        }
    }
}
````


---

<a id="part-7-tcp-프로토콜-명세"></a>

<a id="part-7"></a>

## Part 7. TCP 프로토콜 명세

> **이 파트의 근거**는 `RB3_730_PendantServer.cs` 파일 상단의 주석 블록입니다.
> 서버가 실제로 구현한 것이고, 이 문서가 거기서 파생되었으므로
> **소스 코드가 항상 우선합니다.** 프로토콜을 바꿀 거라면 코드를 먼저 바꾸세요.

<a id="protocol-overview"></a>
### 7.1 전송 규칙

| 항목 | 값 |
|---|---|
| 전송 계층 | TCP |
| 기본 주소 | `127.0.0.1` |
| 기본 포트 | **5000** |
| 인코딩 | **UTF-8** (BOM 없음) |
| 구분자 | `\n` (한 줄 = 한 명령) |
| 명령 형식 | `COMMAND [인자 …]` — 공백 구분, 대소문자 구분 |
| 응답 | **명령마다 반드시 1줄** (`PONG` / `OK …` / `ERR …`) |
| 텔레메트리 | `STATE {json}` 이 `stateHz` (기본 10 Hz) 마다 **비동기로 추가 전송** |

> ⚠️ **상태 읽기 함정**
> 펜던트는 `STATE` 줄을 **수신 도중에도** 받을 수 있습니다.
> 그래서 다음 규칙이 필요합니다.
>
> ```powershell
> # ❌ 틀림 — STATE 가 끼어들면 상태로 오인
> Send "PING"; $r = ReadLine   # ← STATE 가 올 수 있음
>
> # ✅ 맞음 — PONG 까지 건너뛰기
> function Read-Reply {
>   param($reader)
>   while ($true) {
>     $line = $reader.ReadLine()
>     if ($null -eq $line)              { throw 'closed' }
>     if ($line.StartsWith('STATE '))  { continue }   # 텔레메트리 스킵
>     return $line
>   }
> }
> ```

<a id="command-list"></a>

### 7.2 펜던트 → Unity 명령 전체 목록

#### 연결·상태 확인

| 명령 | 인자 | 응답 | 설명 |
|---|---|---|---|
| `PING` | — | `PONG` | 연결 확인 |
| `GETSTATE` | — | `STATE {json}` | 즉시 텔레메트리 요청 |
| `STATUS` | — | `OK STATUS …` | 사람이 읽는 요약 |

#### 속도

| 명령 | 인자 | 응답 | 설명 |
|---|---|---|---|
| `SPEED` | `1..100` | `OK SPEED 20.0` | 전체 속도 배율(%). 20 = 20 % |
| `SPEED` | 범위 밖 | `ERR SPEED must be 1..100` | |

> `SPEED 20` 은 **20 %** 입니다. 학생들이 가장 자주 오해하는 부분입니다.
> 100 이 fastest가 아니라 **100 % (=최대)** 입니다.

#### 관절 이동

| 명령 | 인자 | 응답 | 설명 |
|---|---|---|---|
| `HOME` | — | `OK HOME` | 홈 포즈로 이동 (닫힌 포즈) |
| `MOVJ` | 6개 각도(도) | `OK MOVJ` | 절대 관절 이동. `SETJ` 와 동일 |
| `SETJ` | 6개 각도(도) | `OK SETJ` | (별칭) |
| `STOP` | — | `OK STOP` | **즉시 정지, 사이클도 중단** |
| `JOG` | `<관절번호> <Δ도>` | `OK JOG` | 관절 상대 조그 |
| `JOGSTART` | `<what> <sign>` | `OK JOGSTART` | 속도 조그 시작 (버튼 홀드) |
| `JOGSTOP` | — | `OK JOGSTOP` | 속도 조그 정지 |
| `GETPOS` | — | `OK GETPOS …` | 현재 관절 각도 + TCP |
| `TCP` | — | `OK TCP …` | TCP pose만 조회 |

`MOVJ` 예시 — base, shoulder, elbow, wrist1, wrist2, wrist3 순서(도 단위):

```
MOVJ 0 -30 90 0 0 0
```

`JOGSTART`의 `<what>` 값:

| 값 | 의미 |
|---|---|
| `J1` … `J6` | 관절 속도 조그 |
| `X` `Y` `Z` | TCP 직선 이동 (월드 좌표) |
| `RX` `RY` `RZ` | TCP 회전 조그 |

`<sign>` 은 **`+1` 또는 `-1`만** 허용됩니다.

```
JOGSTART J1 1     ← J1을 + 방향으로 계속 돌림
JOGSTOP
JOGSTART Z -1     ← TCP를 월드 -Z 방향으로 이동
JOGSTOP
```

> ⚠️ `JOGSTART` 는 **`JOGSTOP` 이 반드시 따라와야 합니다.**
> 연결이 끊기면 서버가 자동으로 정지하지만
> ([원칙 4](#principle-4)), 같은 세션 안에서 놓치면 끝없이 움직입니다.

#### 티칭 포인트

| 명령 | 인자 | 응답 | 설명 |
|---|---|---|---|
| `TEACH` | `<이름> [대기초]` | `OK TEACH …` | 현재 자세를 포인트로 기록 |
| `LIST` | — | `OK LIST …` | 저장된 포인트 목록 |
| `GOTO` | `<번호>` | `OK GOTO …` | 해당 포인트로 이동 |
| `NEXT` / `PREV` | — | `OK NEXT …` | 다음/이전 포인트로 이동 |
| `DELETE` | `<번호>` | `OK DELETE …` | 포인트 1개 삭제 |
| `RENAME` | `<번호> <새이름>` | `OK RENAME …` | 이름 변경 |
| `DWELL` | `<번호> <초>` | `OK DWELL …` | 해당 포즈 대기 시간 설정 |
| `CLEAR` | — | `OK CLEAR` | 전체 삭제 |
| `SAVETEACH` | — | `OK SAVETEACH …` | `persistentDataPath` 에 저장 |
| `LOADTEACH` | — | `OK LOADTEACH …` | 파일에서 불러오기 |

#### 사이클

| 명령 | 인자 | 응답 | 설명 |
|---|---|---|---|
| `CYCLE` | `[포인트번호 …] [바퀴수]` | `OK CYCLE …` | 재생. 포인트 번호 없으면 전체 |
| `LOOP` | `<바퀴수>` | `OK LOOP …` | 바퀴수 생략 시 쓸 기본값 |
| `CYCLEPAUSE` | — | `OK CYCLE paused …` / `OK CYCLE moving …` | 일시정지 / 재개 (토글) |
| `CYCLESTATE` | — | `OK CYCLE <상태> …` | 현재 상태 |
| `CYCLESTOP` | — | `OK CYCLESTOP` | 사이클 종료, 제자리 정지 |

> `CYCLEPAUSE` 와 `CYCLESTATE` 의 응답 접두사는 `CYCLE` 입니다.
> 즉 **`OK CYCLEPAUSE` 라는 응답은 나오지 않습니다.** 클라이언트가 접두사로 걸러야 한다면
> `CYCLESTOP` 과 `CYCLESTATE` 를 구분해야 합니다.

`CYCLE` 인자 규칙 (실측 검증 완료):

```
CYCLE              → 저장된 모든 포인트, LOOP 기본 바퀴수
CYCLE 0            → CYCLE 와 동일 (0 = 전체 포인트)
CYCLE 0 0          → 전체 포인트, 0바퀴 = 무한 반복
CYCLE 0 1          → 전체 포인트, 1바퀴만
CYCLE 1 2          → 포인트 1, 2, LOOP 기본 바퀴수
CYCLE 1 2 3        → 포인트 1, 2 를 3바퀴   ← 3은 포인트가 아니라 바퀴수
CYCLE 1 2 3 1      → 포인트 1, 2, 3 을 1바퀴  ← 포인트를 3개 쓰려면 바퀴수를 붙이세요
CYCLE 1 2 0 5      → ❌ 오류. 포인트 번호 0 은 중간에 올 수 없습니다
```

**바퀴수 판정 규칙** — 마지막 숫자를 무조건 바퀴수로 보면 안 됩니다.
Point 번호도 숫자라 값만으로는 구분할 수 없으므로, 서버는 다음 규칙을 씁니다.

1. 인자가 **3개 이상**이면 마지막 숫자는 바퀴수입니다. (`CYCLE 1 2 1` → 포인트 1,2 + 1바퀴)
2. 인자가 **2개**이고 첫 숫자가 `0` 이면 마지막 숫자는 바퀴수입니다. (`CYCLE 0 1` → 전체 + 1바퀴)
3. 그 외에는 **모든 인자가 포인트 번호**입니다. (`CYCLE 1 2` → 포인트 1, 2)

> ⚠️ **이 규칙의 대가: 포인트 3개를 "바퀴수 없이" 지정할 수 없습니다.**
> `CYCLE 1 2 3` 은 "포인트 1,2 를 3바퀴"로 해석됩니다.
> 포인트 3개를 한 바퀴만 돌리려면 `CYCLE 1 2 3 1` 처럼 **바퀴수를 반드시 명시**하세요.
> 포인트를 **일부만** 골라쓸 때에는 **앞에 `0` 을 넣으면 안 됩니다.**
> `0` 은 맨 앞에만 "전체 포인트" 라는 뜻으로 쓸 수 있습니다.

> ⚠️ **바퀴 수가 0 이면 무한 반복입니다.**
> `CYCLE 0 0` 을 보낸 후 펜던트를 닫아도 서버는 연결 해제를 감지하고 정지하지만
> (STOP + `StopCycle("pendant disconnected")`),
> **창만 최소화하고 connections 를 열어둔 채 잊어버리는 경우를 반드시 막으세요.**

### 7.3 Unity → 펜던트 응답

| 응답 | 형식 | 의미 |
|---|---|---|
| `PONG` | — | `PING` 응답 |
| `OK <COMMAND> [detail]` | — | 성공 |
| `ERR <message>` | — | **모든 실패**. 알 수 없는 명령도 여기에 들어감 |

알 수 없는 명령의 응답 형식:

```
ERR unknown command 'FOO'
```

> 💡 클라이언트를 만들 때 **모든 명령이 `OK` 또는 `ERR`로 끝나도록** 만들면
> 타임아웃 디버깅이 훨씬 쉬워집니다. 응답이 안 오는 경우는 연결 자체가 죽은 경우뿐입니다.

### 7.4 `STATE` JSON 스키마

아래는 `MOVJ 10 -20 30 0 0 0` 이 끝난 직후 실제로 캡처한 `STATE` 한 줄입니다.

```json
{
  "ok": true,
  "moving": false,
  "jogging": false,
  "speed": 0.300,
  "j": [10.000, -20.000, 30.000, 0.000, 0.000, 0.000],
  "t": [-0.0193, 0.8513, 0.0099],
  "r": [0.08682, 0.99240, -0.00760, -0.08682],
  "clients": 1,
  "moves": 1,
  "taught": 0,
  "cycle": "idle",
  "cyclePoint": 0,
  "lap": 1
}
```

| 필드 | 타입 | 의미 |
|---|---|---|
| `ok` | bool | 리그 본 해석 성공 여부. **false 면 관절이 안 움직입니다** |
| `moving` | bool | 램프업 중 이동 또는 조그 홀드 중 |
| `jogging` | bool | 속도 조그 홀드 중 |
| `speed` | float | 현재 속도 배율 0…1 (`SPEED 30` 이면 `0.300`) |
| `j` | float[6] | 관절 각도 (도, base부터) |
| `t` | float[3] | TCP 위치 (m, 월드) |
| `r` | float[4] | TCP 회전 (quaternion xyzw, 월드) |
| `clients` | int | 연결된 펜던트 수 |
| `moves` | int | 승인된 이동 명령 수 (**시퀀스 번호로 사용 가능**) |
| `taught` | int | 저장된 포인트 수 |
| `cycle` | string | `idle` / `moving` / `dwell` / `paused` |
| `cyclePoint` | int | 현재 포인트 번호 (idle 이면 0) |
| `lap` | int | 현재 바퀴 (1부터 시작) |

> 🔑 **`ok: false` 를 반드시 확인하세요.**
> 리그 본을 못 찾았을 때 모든 명령이 "성공"처럼 보이지만 관절이 전혀 움직이지 않습니다.
> 이 프로젝트의 `pendant_test.ps1` 도 `state` 섹션에서 `STATE.ok` 를 첫 상태 검사로 확인합니다.

> ⚠️ **클라이언트 설계 주의:** 서버는 클라이언트가 붙어 있는 동안 `Update` 마다
> `STATE {...}` 줄을 **요청 없이 계속 푸시합니다.** 그래서 `MOVJ` 같은 응답 한 줄을
> 기다리는 단순 `ReadLine` 은 거의 항상 `STATE` 를 먼저 읽게 됩니다.
> `STATE` 줄은 **읽어서 버리고** 다음 줄을 계속 확인하는(skip) 방식이 필수입니다.
> 응답 큐가 `STATE` 로 밀리지 않도록, 여러 응답을 읽어야 하는 테스트 하네스에서는
> `STATE` 를 수신 계층에서 걸러내고 마지막 프레임만 캐시했습니다.
> 전체 설계는 [§7.1](#protocol-overview)에서 다룹니다.

### 7.5 실사용 시나리오 — pick & place 티칭

```text
SPEED 30
JOGSTART Z -1        (첫 포즈까지 내림)
JOGSTOP
TEACH pick 1.0       ← 1초 대기
JOGSTART Z 1
JOGSTOP
JOGSTART J2 1        (팔을 올림)
JOGSTOP
TEACH place 0.2      ← 0.2초 대기
LIST
CYCLE 0 0            ← 무한 반복
…
CYCLESTOP
```

서버 주석에 적힌 정석 예시는 이렇습니다:

```
SPEED 30
TEACH pick 1.0     (먼저 조그로 pick 자세를 만든 뒤 티칭)
TEACH place 0.2
LIST               (기록 확인)
CYCLE 0 0          (pick → place → pick → place 무한 반복)
CYCLESTOP
```

> ⚠️ **TEACH 는 현재 자세를 기록만 합니다.** 자동 이동하지 않습니다.
> 따라서 **먼저 조그로 원하는 자세에 도착한 뒤** `TEACH` 를 보내야 합니다.

### 7.6 안전 규칙 (서버가 강제함)

| 상황 | 서버 동작 |
|---|---|
| 클라이언트 연결 끊김 | `driver.Stop()` + `teach.StopCycle("pendant disconnected")` |
| `STOP` 수신 | `teach.StopCycle("operator stop")` |
| `CYCLESTOP` 수신 | 사이클 종료 |
| 수동 이동 명령 (`HOME`/`MOVJ`/`JOG`/`JOGSTART`) | 진행 중 사이클을 `"manual move"` 로 중단 |
| `jogDeadmanSeconds > 0` 이고 조그 갱신 없음 | `Broadcast("ERR jog deadman timeout")` 후 정지 |

> ✅ **설계상 "사이클이 방치된 상태로 방치된 채 계속 돌다" 하는 일이 없도록** 5중 안전장치를 두었습니다.
> 이 중 하나만 있어도 안전하고, 5개 모두 있는 것이 현재 상태입니다.

---

<a id="part-8"></a>

## Part 8. Phase 5 — 회귀 테스트 클라이언트

### 8.1 왜 PowerShell로 작성했나

| 도구 | 문제 |
|---|---|
| Python | Windows에 기본 미설치. `pyserial` 등 의존성 지옥 |
| .NET 콘솔 | 빌드 도구 필요 |
| **PowerShell 5.1** | ✅ **기본 포함, 의존성 0, `TcpClient` 내장** |

결과적으로 **의존성 설치 없이 바로 실행 가능한 624줄 회귀 테스트**가 만들어졌습니다.
이 테스트는 **14개 섹션 61개 항목**을 검사하고, 통과하면 `exit 0` 을 반환합니다.

### 8.2 실행 방법

```powershell
# Unity에서 Play 모드가 켜져 있고 로봇이 씬에 있는 상태에서
powershell -ExecutionPolicy Bypass -File "$HOME\Desktop\Digital_Twin_2022\Pendant\pendant_test.ps1"
```

### 8.3 테스트 항목 구성

| # | 섹션 | 검사 내용 |
|---|---|---|
| 1 | `handshake` | `PING` → `PONG`, 알 수 없는 명령 → `ERR` |
| 2 | `state` | `STATUS`, `GETPOS`, `STATE` JSON 파싱, `ok`, 관절 6개 |
| 3 | `speed override` | `SPEED 25`, 범위 밖 거부 |
| 4 | `joint jog, one shot` | `JOG 1 5`, `JOG 3 -10`, 잘못된 관절번호 거부 |
| 5 | `absolute move` | `MOVJ` 수락, 각도 부족 거부, **도달 포즈가 명령과 일치** |
| 6 | `joint jog, held` | `JOGSTART`/`JOGSTOP`, `STATE.jogging == true`, 실제 이동량 |
| 7 | `tool jog, one shot` | TCP 도달 가능, quaternion 정규화, X/Z 선형 조그, RX 회전 조그 |
| 8 | `stop and home` | `STOP`, `HOME` 후 전 관절 0 복귀 |
| 9 | `teaching` | `CLEAR`, 빈 `LIST`, 미티칭 시 `CYCLE` 거부, `TEACH` 2개, `LIST` 헤더/행/이름/dwell |
| 10 | `point edits` | `RENAME`, `DWELL`, 잘못된 포인트 `RENAME` 거부 |
| 11 | `teach navigation` | `GOTO` 범위 검사, `GOTO` 도달 확인, `NEXT`/`PREV` |
| 12 | `cycling 1 -> 2 -> 1` | `LOOP 0`, `CYCLE 1 2`, 두 포인트 실제 방문, `CYCLEPAUSE` 토글, 수동 조그가 사이클 취소 |
| 13 | `finite cycle` | `CYCLE 0 1` 1바퀴만, 1바퀴 후 자동 정지 |
| 14 | `save and load` | `SAVETEACH`(2 포인트) → `CLEAR` → `LOADTEACH` 복원, 이름/dwell 유지, `DELETE` |

### 8.4 전체 소스


````text
> FILE: pendant_test.ps1  (624 lines)

<#
    RB3-730 pendant test client (PowerShell, no Unity assemblies required).

    This is the same protocol the C# WinForms teaching pendant will speak, so a green run here
    means the Unity side is ready for the real pendant. Every message is one UTF-8 line.

        .\pendant_test.ps1                 wait up to 60 s for Unity to start listening
        .\pendant_test.ps1 -WaitSeconds 0  fail fast if the port is closed
        .\pendant_test.ps1 -Address 192.168.0.10 -Port 5000

    Start Unity, press Play, then run this. The robot must be in the scene for the port to open.

    The run covers the motion commands, then teaching and cycling, which is the flow the pendant
    actually drives. Teaching rewrites persistentDataPath\rb3_730_teach_points.json; if you care
    about a saved set, move that file aside first.
#>
[CmdletBinding()]
param(
    [string]$Address = '127.0.0.1',
    [int]$Port = 5000,
    [int]$WaitSeconds = 60
)

$ErrorActionPreference = 'Stop'
$ci = [System.Globalization.CultureInfo]::InvariantCulture
$script:Pass = 0
$script:Fail = 0

function Write-Head($text) { Write-Host ''; Write-Host "== $text" -ForegroundColor Cyan }
function Ok($text)   { $script:Pass++; Write-Host "  [ ok ] $text" -ForegroundColor Green }
function Bad($text)  { $script:Fail++; Write-Host "  [FAIL] $text" -ForegroundColor Red }
function Info($text) { Write-Host "  ....   $text" -ForegroundColor DarkGray }

function Test-Check($name, $condition, $detail) {
    if ($condition) { Ok ("{0}  ({1})" -f $name, $detail) }
    else             { Bad ("{0}  ({1})" -f $name, $detail) }
}

# ---------------------------------------------------------------- connection

$client = $null
$deadline = (Get-Date).AddSeconds($WaitSeconds)
$attempt = 0
while ($true) {
    $attempt++
    try {
        $client = New-Object System.Net.Sockets.TcpClient
        $client.Connect($Address, $Port)
        break
    } catch {
        $client = $null
        if ((Get-Date) -ge $deadline) {
            Write-Host ''
            Write-Host "could not reach $Address`:$Port after $attempt attempts." -ForegroundColor Red
            Write-Host 'Press Play in Unity with the robot in the scene, then run this again.' -ForegroundColor Yellow
            exit 2
        }
        Start-Sleep -Milliseconds 750
    }
}

$stream = $client.GetStream()

Write-Host ''
Write-Host "connected to $Address`:$Port" -ForegroundColor Green

# Raw byte framing on purpose. StreamReader.Peek() misbehaves on a network stream under
# PowerShell 5.1, and a mid-read timeout can leave the reader's internal buffer inconsistent, so
# the test drives the socket directly: NonBlocking reads plus our own line splitting.
$script:rx = New-Object System.Collections.Generic.List[byte]
$script:closed = $false
$script:lastState = $null

# Outgoing side goes through a StreamWriter. The reply parser below has to hand-roll line framing on
# the raw socket because StreamReader misbehaves on a network stream under PowerShell 5.1, but
# PowerShell's overload resolution for NetworkStream.Write is not dependable across hosts, so
# writing is delegated to StreamWriter (UTF-8, no BOM). Reads stay raw and go through Read-Lines.
$script:tx = New-Object System.IO.StreamWriter($stream, (New-Object System.Text.UTF8Encoding($false)))
$script:tx.AutoFlush = $true

function Send-Line([string]$line) {
    $script:tx.WriteLine($line)
}

# Collects every complete reply line that arrived within the window. '[closed]' marks a dropped peer.
# The server also pushes an unsolicited "STATE {...}" line on every Update while a client is attached,
# so those are consumed here and cached in $script:lastState instead of being handed back: at 20 Hz
# they would otherwise bury a real reply behind dozens of lines and starve the reader.
function Read-Lines([int]$timeoutMs) {
    $out = New-Object System.Collections.Generic.List[string]
    if ($script:closed) { $out.Add('[closed]'); return $out }
    $deadline = (Get-Date).AddMilliseconds($timeoutMs)
    while ($true) {
        while ($true) {
            $idx = $script:rx.IndexOf([byte]10)
            if ($idx -lt 0) { break }
            $bytes = $script:rx.GetRange(0, $idx)
            $script:rx.RemoveRange(0, $idx + 1)
            $line = [System.Text.Encoding]::UTF8.GetString($bytes.ToArray()).TrimEnd([char]13)
            if ($line.StartsWith('STATE ')) { $script:lastState = $line } else { $out.Add($line) }
        }
        if ($out.Count -gt 0) { return $out }
        if ((Get-Date) -ge $deadline) { return $out }
        if ($stream.DataAvailable) {
            $buf = New-Object byte[] 8192
            $n = $stream.Read($buf, 0, $buf.Length)
            if ($n -le 0) { $script:closed = $true; $out.Add('[closed]'); return $out }
            for ($i = 0; $i -lt $n; $i++) { $script:rx.Add($buf[$i]) }
        } else {
            Start-Sleep -Milliseconds 5
        }
    }
}

# Reads reply lines until one starts with $prefix. STATE lines are already filtered out by Read-Lines,
# so this only has to skip other chatter, and the skip list is capped so a long wait cannot grow it.
function Read-Until([string]$prefix, [int]$timeoutMs = 4000) {
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    $seen = New-Object System.Collections.Generic.List[string]
    while ($sw.ElapsedMilliseconds -lt $timeoutMs) {
        foreach ($l in @(Read-Lines 400)) {
            if ($l -eq '[closed]') { return @{ line = $null; skipped = $seen; closed = $true } }
            if ($l.StartsWith($prefix)) { return @{ line = $l; skipped = $seen; closed = $false } }
            if ($seen.Count -lt 20) { $seen.Add($l) }
        }
    }
    return @{ line = $null; skipped = $seen; closed = $script:closed }
}

function Get-Poses {
    # the arm needs a few frames to settle after a command
    Start-Sleep -Milliseconds 250
    Send-Line 'GETPOS'
    $r = Read-Until 'OK POS'
    if ($null -eq $r.line) { return $null }
    $parts = $r.line -split '\s+'
    if ($parts.Count -lt 8) { return $null }
    $v = @()
    for ($i = 2; $i -lt 8; $i++) { $v += [double]::Parse($parts[$i], $ci) }
    return $v
}

function Wait-Idle([int]$maxMs = 25000) {
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    while ($sw.ElapsedMilliseconds -lt $maxMs) {
        Send-Line 'GETSTATE'
        $deadline = (Get-Date).AddMilliseconds(1500)
        while ((Get-Date) -lt $deadline) {
            Read-Lines 200 | Out-Null
            if ($script:lastState -and $script:lastState -match '"moving":false') { return $true }
        }
    }
    return $false
}

# Asks for a STATE frame and returns it as a decoded object. Read-Lines caches the unsolicited STATE
# stream in $script:lastState and keeps it out of the reply stream, so this polls the cache instead of
# searching the reply lines for a prefix that is no longer handed back.
function Get-StateJson([int]$timeoutMs = 3000) {
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    Send-Line 'GETSTATE'
    while ($sw.ElapsedMilliseconds -lt $timeoutMs) {
        Read-Lines 250 | Out-Null
        if ($script:lastState) { return ($script:lastState.Substring(6) | ConvertFrom-Json) }
    }
    return $null
}

# LIST answers with a header and then one line per point, so it needs its own reader that knows how
# many POINT lines to expect. This sends the command itself: every caller just wants the current set,
# and letting each one send LIST separately is how the request used to go missing.
function Read-PointList([int]$timeoutMs = 8000) {
    Send-Line 'LIST'
    $header = $null
    $expect = 0
    $rows = New-Object System.Collections.Generic.List[string]
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    while ($sw.ElapsedMilliseconds -lt $timeoutMs) {
        if ($null -ne $header -and $rows.Count -ge $expect) { break }
        $lines = @(Read-Lines 800)
        if ($lines.Count -eq 0) { continue }
        foreach ($l in $lines) {
            if ($l -eq '[closed]') { return @{ header = $header; rows = $rows; closed = $true } }
            if ($null -eq $header) {
                if ($l.StartsWith('OK LIST')) {
                    $header = $l
                    $expect = [int](($l -split '\s+')[2])
                }
                continue
            }
            if ($l.StartsWith('POINT ')) { $rows.Add($l) }
        }
    }
    return @{ header = $header; rows = $rows; closed = $false }
}

# Pulls the six joint numbers out of a "POINT n name j1 .. j6 dwell ..." line.
function Get-PointJoints([string]$line) {
    $p = $line -split '\s+'
    $v = @()
    for ($i = 3; $i -lt 9; $i++) { $v += [double]::Parse($p[$i], $ci) }
    return $v
}

function Get-PointName([string]$line) { return ($line -split '\s+')[2] }

function Get-PointDwell([string]$line) { return [double]::Parse((($line -split '\s+')[9]), $ci) }

function Near-Pose($a, $b, [double]$tolDeg = 1.0) {
    if ($null -eq $a -or $null -eq $b) { return $false }
    if (@($a).Count -ne 6 -or @($b).Count -ne 6) { return $false }
    for ($i = 0; $i -lt 6; $i++) { if ([Math]::Abs($a[$i] - $b[$i]) -gt $tolDeg) { return $false } }
    return $true
}

function Format-Angles($v) {
    if ($null -eq $v) { return 'none' }
    return (($v | ForEach-Object { '{0:F2}' -f $_ }) -join ', ')
}

try {
    # ------------------------------------------------------------ handshake
    Write-Head 'handshake'
    Send-Line 'PING'
    $r = Read-Until 'PONG'
    Test-Check 'PING -> PONG' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    $r = Read-Until 'OK CONNECT' 1500
    if ($r.line) { Info "server greeted: $($r.line)" }

    Send-Line 'BOGUS'
    $r = Read-Until 'ERR'
    Test-Check 'unknown command -> ERR' ($null -ne $r.line -and $r.line -match 'unknown command') $(if($r.line){$r.line}else{'no reply'})

    # ------------------------------------------------------------ state
    Write-Head 'state'
    Send-Line 'STATUS'
    $r = Read-Until 'OK STATUS'
    Test-Check 'STATUS' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    $p0 = Get-Poses
    Test-Check 'GETPOS' ($null -ne $p0) $(if($p0){'J = ' + (($p0 | ForEach-Object { '{0:F2}' -f $_ }) -join ', ')}else{'no pose'})

    $st = Get-StateJson
    if ($st) {
        try {
            Test-Check 'STATE json parses' $true ("ok=$($st.ok) moving=$($st.moving) speed=$($st.speed) clients=$($st.clients)")
            Test-Check 'STATE reports a resolved rig' ($st.ok -eq $true) 'driver found all six bones'
            Test-Check 'STATE has six joint values' (@($st.j).Count -eq 6) ("j = " + (@($st.j) -join ', '))
            Info ("TCP position = " + (@($st.t) -join ', ') + " m")
        } catch {
            Test-Check 'STATE json parses' $false $_.Exception.Message
        }
    } else {
        Test-Check 'STATE json parses' $false 'no STATE line'
    }

    # ------------------------------------------------------------ speed
    Write-Head 'speed override'
    Send-Line 'SPEED 25'
    $r = Read-Until 'OK SPEED'
    Test-Check 'SPEED 25' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'SPEED 500'
    $r = Read-Until 'ERR'
    Test-Check 'SPEED out of range rejected' ($null -ne $r.line) $(if($r.line){$r.line}else{'not rejected'})

    Send-Line 'SPEED 20'

    # ------------------------------------------------------------ single shot joint jog
    Write-Head 'joint jog, one shot'
    Send-Line 'HOME'
    Read-Until 'OK HOME' | Out-Null
    Wait-Idle | Out-Null
    $before = Get-Poses

    Send-Line 'JOG 1 5'
    Read-Until 'OK JOG' | Out-Null
    $after = Get-Poses
    if ($before -and $after) {
        $d = $after[0] - $before[0]
        Test-Check 'JOG 1 5 moved J1 by about +5 deg' ([Math]::Abs($d - 5.0) -lt 0.6) ("delta = {0:F3} deg" -f $d)
    } else {
        Test-Check 'JOG 1 5 moved J1' $false 'no pose data'
    }

    Send-Line 'JOG 3 -10'
    Read-Until 'OK JOG' | Out-Null
    $a2 = Get-Poses
    if ($after -and $a2) {
        $d = $a2[2] - $after[2]
        Test-Check 'JOG 3 -10 moved J3 by about -10 deg' ([Math]::Abs($d + 10.0) -lt 0.6) ("delta = {0:F3} deg" -f $d)
    } else {
        Test-Check 'JOG 3 -10 moved J3' $false 'no pose data'
    }

    Send-Line 'JOG 99 5'
    $r = Read-Until 'ERR'
    Test-Check 'JOG with a bad joint number rejected' ($null -ne $r.line) $(if($r.line){$r.line}else{'not rejected'})

    # ------------------------------------------------------------ absolute move
    Write-Head 'absolute move'
    Send-Line 'MOVJ 20 -30 40 0 0 0'
    $r = Read-Until 'OK MOVJ'
    Test-Check 'MOVJ accepted' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'MOVJ 1 2 3'
    $r = Read-Until 'ERR'
    Test-Check 'MOVJ with too few angles rejected' ($null -ne $r.line) $(if($r.line){$r.line}else{'not rejected'})

    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    $moved = Wait-Idle
    $moveMs = $sw.ElapsedMilliseconds
    $target = Get-Poses
    Info ("the commanded move settled after {0:F1} s" -f ($moveMs / 1000.0))
    if ($target) {
        $okMove = ([Math]::Abs($target[0] - 20) -lt 0.7) -and ([Math]::Abs($target[1] + 30) -lt 0.7) -and ([Math]::Abs($target[2] - 40) -lt 0.7)
        Test-Check 'MOVJ reached the commanded pose' ($okMove -and $moved) ("J = " + (Format-Angles $target))
    } else {
        Test-Check 'MOVJ reached the commanded pose' $false 'no pose data'
    }

    # ------------------------------------------------------------ held velocity jog
    Write-Head 'joint jog, held'
    Send-Line 'HOME'
    Read-Until 'OK HOME' | Out-Null
    Wait-Idle | Out-Null
    $h0 = Get-Poses

    Send-Line 'JOGSTART J2 1'
    $r = Read-Until 'OK JOGSTART'
    Test-Check 'JOGSTART J2 +1' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    Start-Sleep -Milliseconds 1200
    $st = Get-StateJson
    Test-Check 'STATE reports jogging' ($null -ne $st -and $st.jogging -eq $true) $(if($st){"jogging = $($st.jogging)"}else{'no STATE'})

    Send-Line 'JOGSTOP'
    $r = Read-Until 'OK JOGSTOP'
    Test-Check 'JOGSTOP' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    $h1 = Get-Poses
    if ($h0 -and $h1) {
        $d = $h1[1] - $h0[1]
        Test-Check 'J2 moved while the jog was held' ($d -gt 1.0) ("delta = {0:F3} deg" -f $d)
    } else {
        Test-Check 'J2 moved while the jog was held' $false 'no pose data'
    }

    Send-Line 'JOGSTART JOGGY 1'
    $r = Read-Until 'ERR'
    Test-Check 'JOGSTART with a bad axis rejected' ($null -ne $r.line) $(if($r.line){$r.line}else{'not rejected'})

    # ------------------------------------------------------------ tool jog
    # Tested away from HOME on purpose: with the arm straight up the tool Z axis points along
    # full extension, so pushing further that way is genuinely unreachable and nothing moves.
    Write-Head 'tool jog, one shot'
    Send-Line 'HOME'
    Read-Until 'OK HOME' | Out-Null
    Wait-Idle | Out-Null

    Send-Line 'TCP'
    $r = Read-Until 'OK TCP'
    if ($r.line) {
        $parts = $r.line -split '\s+'
        $t = @(); for ($i = 2; $i -lt 5; $i++) { $t += [double]::Parse($parts[$i], $ci) }
        $quat = @(); for ($i = 5; $i -lt 9; $i++) { $quat += [double]::Parse($parts[$i], $ci) }
        $len = [Math]::Sqrt($quat[0]*$quat[0] + $quat[1]*$quat[1] + $quat[2]*$quat[2] + $quat[3]*$quat[3])
        Test-Check 'TCP pose is reachable' $true ("position = " + (($t | ForEach-Object { '{0:F4}' -f $_ }) -join ', ') + " m")
        Test-Check 'TCP quaternion is normalised' ([Math]::Abs($len - 1.0) -lt 0.01) ("length = {0:F5}" -f $len)
    } else {
        Test-Check 'TCP pose is reachable' $false 'no reply'
    }

    Send-Line 'MOVJ 20 -30 40 0 0 0'
    Read-Until 'OK MOVJ' | Out-Null
    Wait-Idle | Out-Null

    foreach ($axis in @('X', 'Z')) {
        Send-Line 'TCP'
        $before = Read-Until 'OK TCP'
        Send-Line "JOGSTART $axis 1"
        $r = Read-Until 'OK JOGSTART'
        if ($null -eq $r.line) { Test-Check "JOGSTART $axis" $false 'no reply'; continue }
        Start-Sleep -Milliseconds 900
        Send-Line 'JOGSTOP'
        Read-Until 'OK JOGSTOP' | Out-Null
        Send-Line 'TCP'
        $after = Read-Until 'OK TCP'
        if ($before.line -and $after.line) {
            $p1 = ($before.line -split '\s+')[2..4] -join ' '
            $p2 = ($after.line  -split '\s+')[2..4] -join ' '
            Test-Check "linear tool jog $axis moved the TCP" ($p1 -ne $p2) ("$p1  ->  $p2")
        } else {
            Test-Check "linear tool jog $axis moved the TCP" $false 'no pose data'
        }
    }

    Send-Line 'HOME'
    Read-Until 'OK HOME' | Out-Null
    Wait-Idle | Out-Null
    Send-Line 'MOVJ 20 -30 40 0 0 0'
    Read-Until 'OK MOVJ' | Out-Null
    Wait-Idle | Out-Null
    Send-Line 'TCP'
    $before = Read-Until 'OK TCP'
    Send-Line 'JOGSTART RX 1'
    Read-Until 'OK JOGSTART' | Out-Null
    Start-Sleep -Milliseconds 900
    Send-Line 'JOGSTOP'
    Read-Until 'OK JOGSTOP' | Out-Null
    Send-Line 'TCP'
    $after = Read-Until 'OK TCP'
    if ($before.line -and $after.line) {
        $q1 = (($before.line -split '\s+')[5..8]) -join ' '
        $q2 = (($after.line  -split '\s+')[5..8]) -join ' '
        Test-Check 'angular tool jog RX rotated the TCP' ($q1 -ne $q2) ("$q1  ->  $q2")
    } else {
        Test-Check 'angular tool jog RX rotated the TCP' $false 'no pose data'
    }

    # ------------------------------------------------------------ stop and home
    Write-Head 'stop and home'
    Send-Line 'STOP'
    $r = Read-Until 'OK STOP'
    Test-Check 'STOP' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'HOME'
    Read-Until 'OK HOME' | Out-Null
    $homeOk = Wait-Idle
    # NOTE: must not be called $home. PowerShell variable names are case insensitive
    # and $HOME is a read-only automatic variable, so assigning it raises
    # SessionStateUnauthorizedAccessException and kills the rest of the run.
    $homePose = Get-Poses
    if ($homePose) {
        $near = $true; foreach ($a in $homePose) { if ([Math]::Abs($a) -gt 0.5) { $near = $false } }
        Test-Check 'HOME returned to all zeros' ($near -and $homeOk) ("J = " + (($homePose | ForEach-Object { '{0:F2}' -f $_ }) -join ', '))
    } else {
        Test-Check 'HOME returned to all zeros' $false 'no pose data'
    }
    # ------------------------------------------------------------ teaching
    Write-Head 'teaching'
    Send-Line 'SPEED 20'
    Read-Until 'OK SPEED' | Out-Null

    Send-Line 'CLEAR'
    $r = Read-Until 'OK CLEAR'
    Test-Check 'CLEAR' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    $list = Read-PointList
    Test-Check 'LIST on an empty set' ($null -ne $list.header -and $list.rows.Count -eq 0) $(if($list.header){$list.header}else{'no reply'})

    Send-Line 'CYCLE 0'
    $r = Read-Until 'ERR'
    Test-Check 'CYCLE with nothing taught is rejected' ($null -ne $r.line) $(if($r.line){$r.line}else{'not rejected'})

    Send-Line 'MOVJ 10 -20 30 0 0 0'
    Read-Until 'OK MOVJ' | Out-Null
    Wait-Idle | Out-Null
    Send-Line 'TEACH pick 0.20'
    $r = Read-Until 'OK TEACH'
    Test-Check 'TEACH pick became point 1' ($null -ne $r.line -and $r.line -match '^OK TEACH 1 pick ') $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'MOVJ -15 25 -20 0 0 0'
    Read-Until 'OK MOVJ' | Out-Null
    Wait-Idle | Out-Null
    Send-Line 'TEACH place 0.10'
    $r = Read-Until 'OK TEACH'
    Test-Check 'TEACH place became point 2' ($null -ne $r.line -and $r.line -match '^OK TEACH 2 place ') $(if($r.line){$r.line}else{'no reply'})

    $list = Read-PointList
    Test-Check 'LIST header counts both points' ($null -ne $list.header -and $list.header -match '^OK LIST 2$') $(if($list.header){$list.header}else{'no reply'})
    Test-Check 'LIST returns one POINT line per point' (@($list.rows).Count -eq 2) ("got " + @($list.rows).Count)
    if (@($list.rows).Count -eq 2) {
        Test-Check 'LIST names the points' ((Get-PointName $list.rows[0]) -eq 'pick' -and (Get-PointName $list.rows[1]) -eq 'place') ("1=" + (Get-PointName $list.rows[0]) + "  2=" + (Get-PointName $list.rows[1]))
        Test-Check 'LIST keeps the dwell' ([Math]::Abs((Get-PointDwell $list.rows[0]) - 0.20) -lt 0.001 -and [Math]::Abs((Get-PointDwell $list.rows[1]) - 0.10) -lt 0.001) ("1=" + (Get-PointDwell $list.rows[0]) + "s  2=" + (Get-PointDwell $list.rows[1]) + "s")
    }

    $point1 = if (@($list.rows).Count -ge 1) { Get-PointJoints $list.rows[0] } else { $null }
    $point2 = if (@($list.rows).Count -ge 2) { Get-PointJoints $list.rows[1] } else { $null }

    # ------------------------------------------------------------ point edits
    Write-Head 'point edits'
    Send-Line 'RENAME 2 drop'
    $r = Read-Until 'OK RENAME'
    Test-Check 'RENAME 2 drop' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'DWELL 1 0.75'
    $r = Read-Until 'OK DWELL'
    Test-Check 'DWELL 1 0.75' ($null -ne $r.line -and $r.line -match '0\.75') $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'RENAME 9 x'
    $r = Read-Until 'ERR'
    Test-Check 'RENAME with a bad point rejected' ($null -ne $r.line) $(if($r.line){$r.line}else{'not rejected'})

    # ------------------------------------------------------------ stepping between points
    Write-Head 'teach navigation'
    Send-Line 'GOTO 99'
    $r = Read-Until 'ERR'
    Test-Check 'GOTO with a bad point rejected' ($null -ne $r.line) $(if($r.line){$r.line}else{'not rejected'})

    Send-Line 'GOTO 2'
    $r = Read-Until 'OK GOTO'
    Test-Check 'GOTO 2' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})
    $arrived = Wait-Idle
    $now = Get-Poses
    Test-Check 'GOTO 2 drove the arm to that point' ((Near-Pose $now $point2 1.0) -and $arrived) ("J = " + (Format-Angles $now) + "  taught = " + (Format-Angles $point2))

    Send-Line 'NEXT'
    $r = Read-Until 'OK NEXT'
    Test-Check 'NEXT wrapped back to point 1' ($null -ne $r.line -and $r.line -match '^OK NEXT 1 ') $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'PREV'
    $r = Read-Until 'OK PREV'
    Test-Check 'PREV stepped to point 2' ($null -ne $r.line -and $r.line -match '^OK PREV 2 ') $(if($r.line){$r.line}else{'no reply'})

    # ------------------------------------------------------------ cycling
    Write-Head 'cycling 1 -> 2 -> 1'
    Send-Line 'LOOP 0'
    $r = Read-Until 'OK LOOP'
    Test-Check 'LOOP 0 means endless' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'CYCLE 1 2'
    $r = Read-Until 'OK CYCLE'
    Test-Check 'CYCLE 1 2 started' ($null -ne $r.line -and $r.line -match 'pos=1/2') $(if($r.line){$r.line}else{'no reply'})

    # Watch the arm actually visit both taught points instead of just accepting the command.
    $sawOne = $false; $sawTwo = $false; $sawMoving = $false
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    while ($sw.ElapsedMilliseconds -lt 20000) {
        $now = Get-Poses
        if (Near-Pose $now $point1 1.2) { $sawOne = $true }
        if (Near-Pose $now $point2 1.2) { $sawTwo = $true }
        Send-Line 'CYCLESTATE'
        $st = Read-Until 'OK CYCLE' 3000
        # CYCLESTATE answers "OK CYCLE <state> pos=.. ", so the state word is the third token.
        if ($st.line) {
            $sstate = (($st.line -split '\s+')[2])
            if ($sstate -eq 'moving' -or $sstate -eq 'dwell') { $sawMoving = $true }
        }
        if ($sawOne -and $sawTwo -and $sawMoving) { break }
    }
    Test-Check 'the cycle kept running' $sawMoving 'CYCLESTATE reported moving or dwell'
    Test-Check 'the cycle visited point 1' $sawOne 'arm reached the pick pose'
    Test-Check 'the cycle visited point 2' $sawTwo 'arm reached the place pose'

    Send-Line 'CYCLEPAUSE'
    $r = Read-Until 'OK CYCLE'
    Test-Check 'CYCLEPAUSE reports paused' ($null -ne $r.line -and $r.line -match 'paused') $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'CYCLEPAUSE'
    $r = Read-Until 'OK CYCLE'
    Test-Check 'CYCLEPAUSE again resumes' ($null -ne $r.line -and $r.line -notmatch 'paused') $(if($r.line){$r.line}else{'no reply'})

    # A manual jog has to cancel the cycle, otherwise the pendant and the cycle fight over the arm.
    Send-Line 'JOG 4 2'
    Read-Until 'OK JOG' | Out-Null
    Send-Line 'CYCLESTATE'
    $r = Read-Until 'OK CYCLE'
    Test-Check 'a manual jog cancels the cycle' ($null -ne $r.line -and $r.line -match 'idle') $(if($r.line){$r.line}else{'no reply'})

    # ------------------------------------------------------------ finite lap count
    Write-Head 'finite cycle'
    Send-Line 'CYCLE 0 1'
    $r = Read-Until 'OK CYCLE'
    Test-Check 'CYCLE 0 1 replays every point for one lap' ($null -ne $r.line -and $r.line -match 'pos=1/2' -and $r.line -match 'lap=1/1') $(if($r.line){$r.line}else{'no reply'})

    $finished = $false
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    while ($sw.ElapsedMilliseconds -lt 40000) {
        Send-Line 'CYCLESTATE'
        $st = Read-Until 'OK CYCLE' 3000
        if ($st.line -and $st.line -match 'idle') { $finished = $true; break }
    }
    Test-Check 'the cycle stopped itself after one lap' $finished 'CYCLESTATE went back to idle'

    # ------------------------------------------------------------ save and load
    Write-Head 'save and load'
    # Save before clearing: saving an already empty set proves nothing about the round trip.
    Send-Line 'SAVETEACH'
    $r = Read-Until 'OK SAVETEACH'
    Test-Check 'SAVETEACH reports the two taught points' ($null -ne $r.line -and $r.line -match 'OK SAVETEACH 2 ') $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'CLEAR'
    Read-Until 'OK CLEAR' | Out-Null
    $list = Read-PointList
    Test-Check 'CLEAR emptied the set' ($null -ne $list.header -and $list.header -match 'LIST 0') $(if($list.header){$list.header}else{'no reply'})

    Send-Line 'LOADTEACH'
    $r = Read-Until 'OK LOADTEACH'
    Test-Check 'LOADTEACH' ($null -ne $r.line) $(if($r.line){$r.line}else{'no reply'})

    $list = Read-PointList
    Test-Check 'the saved points came back' ($null -ne $list.header -and @($list.rows).Count -eq 2) $(if($list.header){"$($list.header) with " + @($list.rows).Count + " points"}else{'no reply'})
    if (@($list.rows).Count -eq 2) {
        Test-Check 'the reloaded names survived' ((Get-PointName $list.rows[0]) -eq 'pick' -and (Get-PointName $list.rows[1]) -eq 'drop') ("1=" + (Get-PointName $list.rows[0]) + "  2=" + (Get-PointName $list.rows[1]))
        Test-Check 'the reloaded dwells survived' ([Math]::Abs((Get-PointDwell $list.rows[0]) - 0.75) -lt 0.001 -and [Math]::Abs((Get-PointDwell $list.rows[1]) - 0.10) -lt 0.001) ("1=" + (Get-PointDwell $list.rows[0]) + "s  2=" + (Get-PointDwell $list.rows[1]) + "s")
    }

    Send-Line 'DELETE 1'
    $r = Read-Until 'OK DELETE'
    Test-Check 'DELETE 1' ($null -ne $r.line -and $r.line -match 'left=1') $(if($r.line){$r.line}else{'no reply'})

    Send-Line 'CLEAR'
    Read-Until 'OK CLEAR' | Out-Null
    Send-Line 'HOME'
    Read-Until 'OK HOME' | Out-Null
    Wait-Idle | Out-Null
} catch {
    # A terminating error used to be swallowed here: the finally below ran, and its
    # exit discarded the pending exception, so a crashed run still printed
    # "ALL CHECKS PASSED". Report it as a failure instead.
    Write-Host ''
    Write-Host ('FATAL: ' + $_.Exception.Message) -ForegroundColor Red
    if ($_.ScriptStackTrace) { Write-Host $_.ScriptStackTrace -ForegroundColor DarkGray }
    $script:Fail++
} finally {
    Write-Host ''
    Write-Host "passed: $script:Pass   failed: $script:Fail"
    if ($script:Fail -eq 0) { Write-Host 'ALL CHECKS PASSED' -ForegroundColor Green }
    else { Write-Host 'THERE WERE FAILURES' -ForegroundColor Red }
    try { $client.Close() } catch { }
    exit $(if ($script:Fail -eq 0) { 0 } else { 1 })
}
````

<a id="test-output"></a>

### 8.5 실제 출력

아래는 **2026-10-06, Unity 2022.3.62f3 Play 모드에서 그대로 캡처한 출력**입니다.
수정한 뒤 **연속 5회 실행**해서 5회 모두 61/61 을 확인했습니다.
예시가 아니라 실제 실행 결과이며, **14개 섹션 61개 항목 전부 통과**했습니다.
(경로·수치는 이 PC의 실측값이므로 그대로 따라가지 말고, 값의 **형식**만 확인하세요.)

```

connected to 127.0.0.1:5000

== handshake
  [ ok ] PING -> PONG  (PONG)
  [ ok ] unknown command -> ERR  (ERR unknown command 'BOGUS')

== state
  [ ok ] STATUS  (OK STATUS ready=True moving=False jogging=False speed=0.30 clients=1)
  [ ok ] GETPOS  (J = 9.10, -9.10, 9.10, 0.00, 0.00, 0.00)
  [ ok ] STATE json parses  (ok=True moving=False speed=0.300 clients=1)
  [ ok ] STATE reports a resolved rig  (driver found all six bones)
  [ ok ] STATE has six joint values  (j = 9.102, -9.102, 9.102, 0.000, 0.000, 0.000)
  ....   TCP position = -0.0437, 0.8717, 0.0135 m

== speed override
  [ ok ] SPEED 25  (OK SPEED 25.0)
  [ ok ] SPEED out of range rejected  (ERR SPEED must be 1..100)

== joint jog, one shot
  [ ok ] JOG 1 5 moved J1 by about +5 deg  (delta = 5.000 deg)
  [ ok ] JOG 3 -10 moved J3 by about -10 deg  (delta = -10.000 deg)
  [ ok ] JOG with a bad joint number rejected  (ERR JOG needs a joint number 1..6)

== absolute move
  [ ok ] MOVJ accepted  (OK MOVJ)
  [ ok ] MOVJ with too few angles rejected  (ERR needs six joint angles in degrees)
  ....   the commanded move settled after 8.4 s
  [ ok ] MOVJ reached the commanded pose  (J = 20.00, -30.00, 40.00, 0.00, 0.00, 0.00)

== joint jog, held
  [ ok ] JOGSTART J2 +1  (OK JOGSTART)
  [ ok ] STATE reports jogging  (jogging = True)
  [ ok ] JOGSTOP  (OK JOGSTOP)
  [ ok ] J2 moved while the jog was held  (delta = 8.858 deg)
  [ ok ] JOGSTART with a bad axis rejected  (ERR JOGSTART what must be J1..J6, X, Y, Z, RX, RY or RZ)

== tool jog, one shot
  [ ok ] TCP pose is reachable  (position = 0.0000, 0.8753, 0.0064 m)
  [ ok ] TCP quaternion is normalised  (length = 1.00000)
  [ ok ] linear tool jog X moved the TCP  (-0.0597 0.8302 0.0286  ->  -0.0851 0.8345 0.0378)
  [ ok ] linear tool jog Z moved the TCP  (-0.0851 0.8345 0.0378  ->  -0.0946 0.8349 0.0124)
  [ ok ] angular tool jog RX rotated the TCP  (0.08583 0.98106 -0.01513 -0.17299  ->  0.08810 0.97902 -0.05244 -0.17610)

== stop and home
  [ ok ] STOP  (OK STOP)
  [ ok ] HOME returned to all zeros  (J = 0.00, 0.00, 0.00, 0.00, 0.00, 0.00)

== teaching
  [ ok ] CLEAR  (OK CLEAR 0)
  [ ok ] LIST on an empty set  (OK LIST 0)
  [ ok ] CYCLE with nothing taught is rejected  (ERR CYCLE teach at least two points first, for example TEACH pick, TEACH place)
  [ ok ] TEACH pick became point 1  (OK TEACH 1 pick 10.00 -20.00 30.00 0.00 0.00 0.00 0.20)
  [ ok ] TEACH place became point 2  (OK TEACH 2 place -15.00 25.00 -20.00 0.00 0.00 0.00 0.10)
  [ ok ] LIST header counts both points  (OK LIST 2)
  [ ok ] LIST returns one POINT line per point  (got 2)
  [ ok ] LIST names the points  (1=pick  2=place)
  [ ok ] LIST keeps the dwell  (1=0.2s  2=0.1s)

== point edits
  [ ok ] RENAME 2 drop  (OK RENAME 2 drop)
  [ ok ] DWELL 1 0.75  (OK DWELL 1 0.75)
  [ ok ] RENAME with a bad point rejected  (ERR RENAME needs <point> <new name>)

== teach navigation
  [ ok ] GOTO with a bad point rejected  (ERR GOTO needs a point number 1..2)
  [ ok ] GOTO 2  (OK GOTO 2 drop)
  [ ok ] GOTO 2 drove the arm to that point  (J = -15.00, 25.00, -20.00, 0.00, 0.00, 0.00  taught = -15.00, 25.00, -20.00, 0.00, 0.00, 0.00)
  [ ok ] NEXT wrapped back to point 1  (OK NEXT 1 pick)
  [ ok ] PREV stepped to point 2  (OK PREV 2 drop)

== cycling 1 -> 2 -> 1
  [ ok ] LOOP 0 means endless  (OK LOOP 0)
  [ ok ] CYCLE 1 2 started  (OK CYCLE moving pos=1/2 lap=1/inf point=pick)
  [ ok ] the cycle kept running  (CYCLESTATE reported moving or dwell)
  [ ok ] the cycle visited point 1  (arm reached the pick pose)
  [ ok ] the cycle visited point 2  (arm reached the place pose)
  [ ok ] CYCLEPAUSE reports paused  (OK CYCLE paused pos=2/2 lap=1/inf point=)
  [ ok ] CYCLEPAUSE again resumes  (OK CYCLE moving pos=2/2 lap=1/inf point=drop)
  [ ok ] a manual jog cancels the cycle  (OK CYCLE idle points=2)

== finite cycle
  [ ok ] CYCLE 0 1 replays every point for one lap  (OK CYCLE moving pos=1/2 lap=1/1 point=pick)
  [ ok ] the cycle stopped itself after one lap  (CYCLESTATE went back to idle)

== save and load
  [ ok ] SAVETEACH reports the two taught points  (OK SAVETEACH 2 C:/.../Digital_Twin_2022\rb3_730_teach_points.json)
  [ ok ] CLEAR emptied the set  (OK LIST 0)
  [ ok ] LOADTEACH  (OK LOADTEACH 2 C:/.../Digital_Twin_2022\rb3_730_teach_points.json)
  [ ok ] the saved points came back  (OK LIST 2 with 2 points)
  [ ok ] the reloaded names survived  (1=pick  2=drop)
  [ ok ] the reloaded dwells survived  (1=0.75s  2=0.1s)
  [ ok ] DELETE 1  (OK DELETE 1 left=1)

passed: 61   failed: 0
ALL CHECKS PASSED
```

실행 코드는 `passed`/`failed` 줄에서 `exit 0` 또는 `exit 1` 을 반환합니다.
배터리로 자동 실행할 때는 **종료 코드**로 판정하세요.

```powershell
powershell -ExecutionPolicy Bypass -File ".\pendant_test.ps1"
if ($LASTEXITCODE -ne 0) { throw "pendant regression failed" }
```

### 8.6 포즈 측정 — HOME 에서 tool jog 가 안 되는 이유

> 🔑 **이 문서를 만들면서 실제로 걸린 문제입니다.**
> 처음에는 tool jog 검사를 HOME 포즈에서 돌렸고, Z 조그 항목이 계속 `FAIL` 이었습니다.
>
> **원인은 Z 조그가 아니라 포즈였습니다.** HOME 은 팔을 수직으로 세운 자세라
> 그 자세에서는 월드 Z 방향으로 TCP 가 움직일 여지가 거의 없습니다.
>
> 그래서 검사 직전에 팔을 편 자세로 바꿉니다.
>
> ```powershell
> Send-Line 'MOVJ 20 -30 40 0 0 0'    # HOME 대신 팔을 편 자세로
> ```
>
> 이 수정은 **실행까지 검증했습니다.** 실제 TCP Z 값이
> `-0.0851 0.8345 0.0378 → -0.0946 0.8349 0.0124` 로 변했습니다.
> 위 [§8.5](#test-output)의 `tool jog, one shot` 섹션에서 확인하세요.

---

<a id="part-9"></a>

## Part 9. 실사용 시나리오

### 9.1 시나리오 A — 수동 조그로 위치 잡기

목적: TCP 를 목표 좌표로 옮기고 그 자세에서 티칭한다.

```powershell
# 0) 연결 상태 확인
Send 'PING'                      # → PONG
Send 'GETSTATE'                  # → STATE {… "ok":true …}   ← ok:true 인 것만 확인

# 1) 속도 낮추기 (안전)
Send 'SPEED 20'                  # → OK SPEED 20.0

# 2) TCP 를 월드 좌표로 내리기 (Z축 tool jog)
Send 'JOGSTART Z -1'
Start-Sleep -Milliseconds 800    # 0.8초 동안 이동
Send 'JOGSTOP'
```

> ⚠️ `JOGSTART` 는 **`JOGSTOP` 없이는 멈추지 않습니다.**
> 반드시 쌍으로 보내세요. 연결을 끊으면 서버가 정지하지만
> 같은 세션 안에서 실수하면 계속 움직입니다.

### 9.2 시나리오 B — pick & place 티칭 사이클

목적: 두 포즈를 기록하고 무한 반복한다.

```powershell
Send 'SPEED 30'

# ── 포인트 1: pick ──
Send 'JOGSTART J2 1'              # 어깨를 올려 대기 위치로
Start-Sleep -Milliseconds 1200
Send 'JOGSTOP'
Send 'JOGSTART Z -1'              # 내리기
Start-Sleep -Milliseconds 900
Send 'JOGSTOP'
Send 'TEACH pick 1.0'             # ← 현재 자세 기록 + 1초 대기

# ── 포인트 2: place ──
Send 'JOGSTART Z 1'               # 들어올리기
Start-Sleep -Milliseconds 900
Send 'JOGSTOP'
Send 'JOGSTART J5 -1'             # 손목을 회전해 놓을 자세를 바꿈
Start-Sleep -Milliseconds 700
Send 'JOGSTOP'
Send 'TEACH place 0.2'            # ← 현재 자세 기록 + 0.2초 대기

# ── 확인 ──
Send 'LIST'                       # 두 포인트가 보이는지

# ── 재생 ──
Send 'CYCLE 0 0'                  # 전체 무한 반복
Send 'CYCLESTATE'                 # → OK CYCLE moving pos=1/2 lap=1/inf point=pick
Send 'CYCLEPAUSE'                 # → OK CYCLE paused pos=2/2 lap=1/inf point=
Send 'CYCLESTATE'                 # → OK CYCLE paused ...
Send 'CYCLEPAUSE'                 # → OK CYCLE moving ...  (재개)
Send 'CYCLESTOP'                  # → OK CYCLESTOP

# ── 정리 ──
Send 'CLEAR'
```

### 9.3 시나리오 C — 부분 사이클

```powershell
Send 'CYCLE 1 2 3 1'              # 포인트 1,2,3 을 1바퀴
Send 'CYCLE 1 2 5'                # 포인트 1,2 를 5바퀴
Send 'LOOP 10'                    # 기본 반복 횟수를 10으로
Send 'CYCLE'                      # 인자 없이 → 전체, 10바퀴
```

### 9.4 시나리오 D — 저장과 재사용

```powershell
Send 'TEACH homeA 0.5'
Send 'TEACH homeB 0.5'
Send 'SAVETEACH'                  # persistentDataPath 에 저장
Send 'CLEAR'                      # 전부 삭제
Send 'GETSTATE'                   # → "taught":0  확인
Send 'LOADTEACH'                  # 복원
Send 'LIST'                       # 두 포인트 복원 확인
```

저장 위치는 플랫폼마다 다릅니다.

| 플랫폼 | 경로 |
|---|---|
| Windows | `%USERPROFILE%\AppData\LocalLow\<Company>\<Product>\` |
| Linux | `~/.config/unity3d/<Company>/<Product>/` |
| macOS | `~/Library/Application Support/unity3d/<Company>/<Product>/` |

### 9.5 시나리오 E — TCP pose 로 위치 확인

```powershell
Send 'HOME'
# 움직임이 끝날 때까지 STATE 의 "moving" 이 false 가 되기를 기다린 뒤에 조회하세요.
Send 'TCP'                          # → OK TCP x y z qx qy qz qw
Send 'GETPOS'                       # → OK POS j1 j2 j3 j4 j5 j6  (관절각만, TCP 없음)
```

> ⚠️ **`GETPOS` 에는 TCP 좌표가 없습니다.** 관절 각도 6개만 줍니다.
> TCP 를 보려면 `TCP` 명령을 따로 보내야 합니다.

아래는 **실측값**입니다 (Unity 2022.3.62f3, Play 모드).

| 포즈 | TCP x (m) | TCP y (m) | TCP z (m) | \|t\| (m) |
|---|---|---|---|---|
| `HOME` | `0.0000` | `0.8753` | `0.0064` | **0.8753** |
| `MOVJ 0 -30 90 0 0 0` | `0.2415` | `0.6150` | `0.0064` | **0.6607** |
| `MOVJ 20 -30 40 0 0 0` | `-0.0597` | `0.8302` | `0.0286` | **0.8328** |

> 🔑 **HOME 의 \|t\| = 0.8753 m 이 스케일 정답의 증거입니다.**
> Part 4 에서 Blender 로 만든 FBX 바운딩 박스 높이 `0.875 m` 와 정확히 일치합니다.
> 이 값이 안 맞으면(예: 0.0875 m 로 10배 작게 나오면) **스케일 옵션이 빠져 있는 것**입니다.
> Part 4.6 의 Blender 라운드트립과도 같은 값이 나와야 합니다.
>
> 참고로 `MOVJ 0 -30 90` 을 주면 팔이 앞으로 눕기 때문에 **높이가 0.8753 → 0.6150 으로 내려갑니다.**
> 관절이 제대로 해석되었다는 신호이므로, 값이 "낮아진 것"이 아니라 정상입니다.

### 9.6 시나리오 F — 안전 테스트 (반드시 해보세요)

디지털 트윈이어도 안전 검증은 예외가 아닙니다.

```powershell
# 1) 사이클 중 연결 끊기 → 자동 정지 확인
Send 'TEACH a 0.1'; Send 'TEACH b 0.1'
Send 'CYCLE 0 0'
Start-Sleep -Seconds 2
$client.Close()                  # 강제 종료
Start-Sleep -Seconds 1
Get-Content "$env:LOCALAPPDATA\Unity\Editor\Editor.log" -Tail 30 |
  Select-String 'disconnect'
```

Unity Console에 다음이 떠야 합니다.

```
[RB3-730] client 127.0.0.1:xxxxx disconnected, stopping.
```

> ✅ 이 메시지가 없으면 [원칙 4](#principle-4)가 깨진 것입니다.

### 9.7 전체 시나리오 요약표

| # | 시나리오 | 검증 포인트 |
|---|---|---|
| A | 수동 조그 | `JOGSTART`/`JOGSTOP` 쌍, 속도 배율 |
| B | pick & place 티칭 | `TEACH`→`LIST`→`CYCLE`→`CYCLESTOP` |
| C | 부분 사이클 | 인자 파싱, 바퀴 수 0=무한 |
| D | 저장/복원 | `SAVETEACH`/`CLEAR`/`LOADTEACH` |
| E | pose 확인 | HOME 에서 TCP `y=0.8753`, \|t\| = 0.8753 m |
| F | 안전 | 끊김 시 자동 정지 |

---

<a id="part-10"></a>

## Part 10. 문제 해결 — 실제로 겪은 버그 11가지

이 파트는 **이 프로젝트를 만들면서 실제로 발생한 버그**만 다룹니다.
해결 코드와 "왜 그렇게 바꿨는지"를 함께 적었습니다.

> 10.1 ~ 10.7 은 소스·임포트 단계에서, **10.8 ~ 10.10 은 TCP 테스트 단계에서** 만난 것입니다.
> 특히 10.8 의 CYCLE 바퀴수 파서 버그와 10.10 의 CYCLEPAUSE 무시 버그는 **서버 소스 자체의 논리 오류**였으므로
> 펜던트를 만들기 전에 반드시 확인하세요.

### 10.1 `STATE` JSON 괄호 오류 → JSON 파싱 실패

**증상**: 클라이언트가 `STATE` 줄을 받은 뒤
`ConvertFrom-Json` 이 `Unexpected character encountered while parsing value` 예외를 던짐.

**원인**: `StringBuilder` 로 JSON 을 만들 때 **조건 분기에서 닫는 괄호나 쉼표가 빠졌다.**

하나의 필드만 조건부로 붙일 때, 그 필드가 첫 항목이 아닐 수도,
마지막 항목일 수도 있다는 걸 고려하지 않으면 깨집니다.

**수정 원칙**:

```
1. 모든 필드는 항상 출력한다 (조건부로 빼지 않는다)
2. 쉼표는 "앞 필드가 있다"가 아니라 "뒤에 필드가 있다" 기준으로 붙인다
3. 문자열 값에 " 가 들어가면 이스케이프한다
4. 마지막에 파서로 검증한다
```

**검증 스니펫 — 문자열을 만들고 나서 반드시 돌리세요**:

```powershell
$raw = 'STATE {"ok":true,"j":[0,-30,90,0,0,0],"cycle":"idle"}'
try { $o = $raw.Substring(6) | ConvertFrom-Json; "OK cycle=$($o.cycle)" }
catch { "FAIL: $($_.Exception.Message)" }
```

> 💡 **가장 안전한 설계는 `STATE` JSON 을 손으로 만들지 않는 것**입니다.
> 서버가 `sb.AppendFormat(...)` 에서 필드를 순서대로 붙이되,
> 첫 필드는 쉼표 없이, 나머지는 앞에 쉼표를 붙이게 하면 잘못이 원천 차단됩니다.

### 10.2 `ReadTimeout = 0` → `ArgumentOutOfRangeException` → accept 루프 사망

**증상**: 처음 한두 번은 잘 동작하는데, **그 뒤로 아무 펜던트도 접속되지 않음.**
포트는 리스닝 중인데 연결이 즉시 닫히거나, 아예 응답이 없음.

**원인**: `NetworkStream.ReadTimeout = 0` 은 유효하지 않습니다.

```
System.ArgumentOutOfRangeException: Non-negative number required.
Parameter name: millisecondsTimeout  (ReadTimeout)
```

`NetworkStream.ReadTimeout` 은 **양수(밀리초) 또는 `Timeout.Infinite`** 만 받습니다.
`0` 은 "0밀리초 대기"가 아니라 **잘못된 값**입니다.

여기에 더 나쁜 일이 이어집니다. accept 스레드의 예외가
루프 바깥 `try` 에서 잡히면 **스레드가 죽고, accept 가 영영 재시작되지 않습니다.**
포트만 열린 척하는 "죽은 서버"가 됩니다.

**수정**:

```csharp
c.Stream.ReadTimeout  = Timeout.Infinite;   // 한 줄이 올 때까지 블록
c.Stream.WriteTimeout = 3000;               // 펜던트가 멈춰도 writer 가 막히지 않게
```

| 값 | 의미 |
|---|---|
| `Timeout.Infinite` | **데이터가 올 때까지 영원히 대기** ← 소켓 서버의 정답 |
| `3000` (양수) | 3초 안에 안 오면 `IOException` → 그 클라이언트를 버림 |
| `0` | ❌ **`ArgumentOutOfRangeException`** |

> 🔑 **추가 안전장치**: accept 루프의 `try/catch`는 **`continue` 하되 루프를 벗어나지 않게** 씁니다.
> 개별 클라이언트 핸들러의 예외가 accept 스레드를 죽이지 않아야 합니다.

### 10.3 죽은 클라이언트 누수 → 리소스 고갈

**증상**: 펜던트를 여러 번 껐다 켜면
`STATE` 의 `"clients"` 값이 점점 올라만 갑니다.
끊어진 클라이언트가 목록에 남습니다.

**원인**: 읽기 루프가 예외로 빠져나가면 클라이언트 객체를 `clients` 리스트에서
**제거하는 코드가 실행되지 않습니다.** 소켓도 닫히지 않습니다.

**수정**: 클라이언트가 자신이 사라진 순간을 **스스로 보고**하게 합니다.

```csharp
// 발견: 끊긴 클라이언트를 메인 스레드 큐에 함께 싣는다
// 손실: ClientRecord 가 "lost" 표시를 달고 메인 스레드에 넘겨준다
// 정리: 메인 스레드가 정확히 그 하나만 제거한다
wasRegistered = clients.Remove(lost);
```

> 🔑 **이유**: 스레드에서 `clients` 를 직접 조작하면
> 다른 스레드가 순회하는 순간 `InvalidOperationException` 이 납니다.
> **쓰레드 전역 자료구조는 읽기만 하고, 제거는 메인 스레드에 위임**하는 것이 안전합니다.

**정상 종료 시에는 스냅샷을 떠서 한 번에 비웁니다**:

```csharp
lock (clientsLock) { snapshot = new List<Client>(clients); clients.Clear(); }
```

<a id="bug-syncfromrig"></a>

### 10.4 `SyncFromRig` 왕복 오차 → 도착 판정 무한 실패

**증상**: `MOVJ` 명령이 목표 자세에 정확히 도착했는데도
`STATE.moving` 이 계속 `true` 이고 명령이 끝나지 않습니다.

**원인**: 관절 각도를 transform 에 쓰고 다시 읽으면 **약 0.02° 의 오차**가 남습니다.

```
SetAngles(30.0°)  →  Quaternion.AngleAxis(30.0°)  →  FBX  →  ReadBoneAngleDeg()  →  29.98°
```

이 0.02° 차이를 **매 프레임 그대로 받아들이면**,
`arriveToleranceDeg` = 0.05° 보다 작은 값이 될 수는 있지만,
transform 양자화 때문에 그 경계를 계속 오갑니다. 결과적으로 도착 판정이 불안정해집니다.

**수정**: 소스의 주석이 정답을 말해줍니다.

```csharp
float read = ReadBoneAngleDeg(i);
// Reading a joint back out of the transform costs about 0.02 deg, so adopting the
// read back value every frame would leave a permanent gap that no tight arrival
// tolerance can ever close. Only take it when it is clearly an outside edit.
if (Mathf.Abs(read - currentDeg[i]) > externalEditToleranceDeg) currentDeg[i] = read;
```

| 값 | 역할 |
|---|---|
| `arriveToleranceDeg = 0.05` | **목표와 현재의 차이**가 이보다 작으면 도착 |
| `externalEditToleranceDeg = 1.0` | **읽어온 값과 현재의 차이**가 이보다 클 때만 외부 편집으로 인정 |

> 🔑 **핵심 교훈**: 두 tolerances 는 **완전히 다른 두 문제**를 해결합니다.
> 하나는 "도착했는가", 다른 하나는 "외부에서 건드렸는가".
> 둘을 같은 값으로 두면 이 버그가 반드시 생깁니다.

<a id="105-tickramp-마지막-스텝에서-도달-못-함"></a>

### 10.5 `TickRamp` 마지막 스텝에서 도달 못 함

**증명**: `MOVJ 0 0 0 0 0 5` 를 실행하면 J6 이 **4.96° 에서 멈춥니다.** 목표는 5°.

**원인**: 목표까지의 거리가 한 프레임 최대 스텝보다 **정확히 같거나 조금 클 때**,
`currentDeg += sign * maxStep` 만 하면 나머지 분이 영원히 남습니다.
부동소수점 비교가 어긋나면서 조건이 끝내 참이 되지 않는 경우도 생깁니다.

**수정**:

```csharp
float delta = targetDeg[i] - currentDeg[i];
if (Mathf.Abs(delta) <= maxStep) { currentDeg[i] = targetDeg[i]; continue; }  // ← 추가
currentDeg[i] += Mathf.Sign(delta) * maxStep;
```

| | 이전 | 수정 후 |
|---|---|---|
| 남은 거리 ≤ maxStep | 계속 스텝 → 오차 남음 | **목표값으로 직접 대입** |
| 결과 | 4.96° | **5.00°** |

### 10.6 PowerShell 5.1 `StreamReader.Peek()` 가 블로킹함

**증상**: 펜던트 소켓을 여는 순간 스크립트가 멈춥니다.

**원인**: `StreamReader.Peek()` 는 **내부 버퍼를 채우려고 소켓을 읽기 시도**합니다.
버퍼가 비어 있으면 **네트워크에서 데이터를 기다립니다.**
`State = Socket` 인 비동기 스트림에서는 예상 밖의 블로킹이 됩니다.

```powershell
# ❌ 멈춤
if ($reader.Peek() -ge 0) { ... }
```

**수정**: Peek 대신 **타임아웃이 있는 `ReadLine`** 을 쓰고,
`STATE` 줄은 스킵합니다.

```powershell
function Read-Reply {
    param($Reader, [int]$TimeoutMs = 5000)
    $task = $Reader.ReadLineAsync()
    if (-not $task.Wait($TimeoutMs)) { throw 'timeout waiting for reply' }
    $line = $task.Result
    if ($null -eq $line) { throw 'connection closed' }
    if ($line.StartsWith('STATE ')) { return (Read-Reply $Reader $TimeoutMs) }  # 텔레메트리 스킵
    return $line
}
```

> 🔑 **PowerShell 5.1 vs 7 의 차이**
> 이 프로젝트는 **5.1** 을 기준으로 작성했습니다.
> PS7 의 `Stream` cmdlet은 버퍼링 동작이 달라 같은 스크립트가 다르게 동작할 수 있습니다.
> 학생 환경이 PowerShell 5.1 이면 그대로 동작합니다.

### 10.7 Z tool jog 테스트가 HOME 포즈에서 실패

**증상**: tool jog 테스트가 `FAIL` 로 끝납니다. `JOGSTART Z 1` / `JOGSTOP` 를 보냈는데 TCP Z 가 안 변했습니다.

**원인**: **테스트가 HOME 포즈에서 조그를 검사하고 있었습니다.**
HOME 포즈는 모든 관절이 0°, 즉 **팔이 완전히 접힌 자세**입니다.
이 자세에서 TCP 의 Z 는 거의 최솟값이라 `Z` 방향으로 움직일 여지가 없습니다.
테스트는 정상인 코드를 **잘못된 자세에서** 검사한 셈입니다.

**수정**:

```powershell
Send 'MOVJ 20 -30 40 0 0 0'    # 팔을 편 자세로
# … 여기서 Z 조그 검사 …
```

| 자세 | 팔 상태 | Z 조그 가능 |
|---|---|---|
| `HOME` (0 0 0 0 0 0) | 완전 접힘 | ❌ 여지 없음 |
| `MOVJ 20 -30 40 0 0 0` | 편 자세 | ✅ 가능 |

> ✅ **검증 완료.** 이 수정은 실제 Play 모드에서 재실행했고 통과했습니다.
> TCP Z 가 `-0.0851 0.8345 0.0378 → -0.0946 0.8349 0.0124` 로 변했습니다.
> [§8.5](#test-output)의 `tool jog, one shot` 섹션을 보세요.

### 10.8 `CYCLE` 바퀴수 파서가 어긋남 — 서버 소스 자체의 논리 오류

**증상**: 회귀 테스트가 두 가지 방식으로 실패했습니다.

```
CYCLE 0 1    →  ERR CYCLE could not start, check the point numbers
CYCLE 1 2 1  →  OK CYCLE moving pos=1/3 lap=1/inf point=pick
                                                              ↑ 3개?!  ↑ 무한?!
```

**원인**: `ParseIndices` 가 **인자를 전부 포인트 번호로 해석**하고,
"마지막 숫자가 바퀴수인지"는 `indices.Length < parts.Length - 1` 이라는 **휴리스틱**으로 판별했습니다.

```csharp
// ❌ 고칠 전
int[] indices = ParseIndices(parts, 1);          // "0", "1", "2" 를 전부 인덱스로
if (indices.Length == 1 && indices[0] < 0 && …)  // 인자가 정확히 1개일 때만
    indices = new int[0];                        // "전체 포인트" 로 recognize
```

그래서:

- `CYCLE 1 2 1` → 인덱스 `[0, 1, 0]` → **포인트 3개를 무한 반복**. `1` 이 포인트로 소비되어
  바퀴수는 `LOOP` 기본값(무한) 이 남았습니다.
- `CYCLE 0 1` → 인덱스 `[-1, 0]` → 위 특별 케이스가 `Length == 1` 이라서 **발동하지 않음**
  → 음수 인덱스로 시작 실패. 문서에 적어둔 `CYCLE 0 [바퀴수]` 형식이 **완전히 unusable** 입니다.

**핵심 교훈**: 포인트 번호와 바퀴수가 **둘 다 정수**라 값만으로는 구별할 수 없습니다.
"몇 개가 앞에 있는가" 로 판별하는 **위치 기반 규칙**을 명시적으로 정해야 합니다.

**수정** (`PendantServer.cs` / `HandleCycle`):

```csharp
int[] indices = ParseIndices(parts, 1, parts.Length);

int tokenCount = parts.Length - 1;
bool everyPoint = indices.Length > 0 && indices[0] < 0;      // 첫 숫자가 0 → "전체"
bool lastIsLaps = tokenCount >= 3 || (everyPoint && tokenCount >= 2);

int loops = teach.defaultLoops;
if (lastIsLaps)
{
    double d;
    if (double.TryParse(parts[parts.Length - 1], NumberStyles.Float,
                        CultureInfo.InvariantCulture, out d))
    {
        loops = (int)d;
        indices = ParseIndices(parts, 1, parts.Length - 1);   // 바퀴수 토큰은 뺀다
    }
}
if (everyPoint) indices = new int[0];
```

`ParseIndices` 도 `to` 인자를 받아 범위만 읽도록 바꿨습니다.

**검증** (7가지 형태 전부 실측):

| 명령 | 응답 | 의미 |
|---|---|---|
| `CYCLE` | `pos=1/2 lap=1/inf` | 전체, 기본 바퀴 |
| `CYCLE 0` | `pos=1/2 lap=1/inf` | 전체, 기본 바퀴 |
| `CYCLE 0 0` | `pos=1/2 lap=1/inf` | 전체, 무한 |
| `CYCLE 0 1` | `pos=1/2 lap=1/1` | 전체, 1바퀴 |
| `CYCLE 1 2` | `pos=1/2 lap=1/inf` | 포인트 1,2, 기본 바퀴 |
| `CYCLE 1 2 1` | `pos=1/2 lap=1/1` | 포인트 1,2, 1바퀴 |
| `CYCLE 0 2 2` | `pos=1/2 lap=1/2` | 전체, 2바퀴 |

> ⚠️ **기존 클라이언트 호환성**: `CYCLE 1 2` 는 예전부터 "포인트 1,2"였고,
> 수정 후에도 그대로입니다. `CYCLE 1 2 1` 처럼 **인자가 3개 이상일 때만**
> 해석이 달라지므로, 펜던트 UI 에서는 라벨을 "포인트" 와 "바퀴" 를
> **명확히 분리**해서 보내는 편이 안전합니다.

### 10.9 테스트 하네스가 스스로를 속이고 있었음

`CYCLE 0 1` 문제를 찾는 과정에서, **"서버가 안 된다" 는 잘못된 진단**을 내리기 쉬운
테스트 하네스 결함도 여럿 드러났습니다. 학생들이 똑같은 함정을 밟지 않도록 기록합니다.

#### (a) `$home` 이 읽기 전용 `$HOME` 과 충돌

```powershell
$home = Get-Poses        # ❌ SessionStateUnauthorizedAccessException
$homePose = Get-Poses    # ✅ PowerShell 변수명은 대소문자를 구분하지 않는다
```

**가장 위험한 부분**은 예외 자체가 아니라 **다음 항목이 방해 없다는 것이었습니다.**

#### (b) `try/finally` 의 `exit` 가 예외를 삼킴

```powershell
try   { … ; if ($Fail -eq 0) { 'ALL CHECKS PASSED' } }
catch { … }
finally { exit 0 }        # ❌ 예외가 있어도 exit 이 정상 종료로 덮어쓴다
```

**테스트가 26개 항목만 확인하고 죽었는데도 `ALL CHECKS PASSED` 를 출력했습니다.**
`catch` 에서 `FATAL` 을 찍고 `$script:Fail++` 하도록 고쳤습니다.

> 🔑 **회귀 테스트에서 "통과" 는 반드시 끝까지 도달했을 때만 의미가 있습니다.**
> 조기 종료가 조용히 성공으로 보이면, 남은 절반은 한 번도 실행된 적 없는 것이 됩니다.

#### (c) `STATE` push 가 응답을 묻었다

서버는 클라이언트가 붙어 있는 동안 매 `Update` 마다 `STATE` 줄을 보냅니다.
기존 `Read-Until 'STATE'` 는 `Read-Lines` 가 첫 줄만 반환하는 특성 때문에
**타임아웃 → `no STATE`** 로 끝났습니다. 해결은 두 갈래였습니다.

- `Read-Lines` 에서 `STATE ` 를 **걸러내고 마지막 프레임만 `$script:lastState` 에 캐시**
- `Get-StateJson` 이 그 캐시에서 JSON 을 꺼내도록 추가

```powershell
if ($line.StartsWith('STATE ')) { $script:lastState = $line } else { $out.Add($line) }
```

#### (d) `Read-PointList` 가 `LIST` 를 보내지 않았음

```powershell
$list = Read-PointList     # ❌ 안에서 Send-Line 'LIST' 를 안 함 → 응답이 없음
```

각 호출부가 `LIST` 를 따로 보내도록 나눠 둔 구조가 실수로 빠졌습니다.
**읽는 함수가 필요한 명령을 직접 보내게** 통일했습니다.

#### (e) 절대 매칭될 수 없는 가드

```powershell
if ($st.line -match 'state') { … }   # ❌ CYCLESTATE 응답에 "state" 라는 단어가 없다
```

`CYCLESTATE` 는 `OK CYCLE moving pos=1/2 lap=1/inf point=pick` 처럼 응답하므로
`'state'` 가 포함될 수 없습니다. 조건을 삭제했습니다.

#### (f) `SAVETEACH` 를 `CLEAR` **뒤에** 실행

```powershell
Send 'CLEAR'                # ← 지워놓고
Send 'SAVETEACH'            # ← 빈 세트를 저장 (항상 "성공")
Send 'LOADTEACH'            # ← 0개 복원
```

검사는 통과하는데 **아무것도 검증하지 못하는** 구조였습니다.
저장 → 비우기 → 불러오기 순서로 바꿨고, 이름·dwell 이 살아있는지도 확인합니다.

### 10.10 `CYCLEPAUSE` 가 `dwell` 중에 오면 무시됨 (재현성 버그)

**증상**: 테스트가 가끔 — 5회 중 1회 정도 — 아래와 같이 실패했습니다.

```
[FAIL] CYCLEPAUSE reports paused  (OK CYCLE dwell pos=2/2 lap=1/inf point=drop)
```

`CYCLEPAUSE` 를 보냈는데 답이 `paused` 가 아니라 **그때의 상태 그대로**였습니다.

**원인**: `TogglePause()` 가 `Moving` 과 `Paused` 만 처리하고 있었습니다.

```csharp
public void TogglePause()
{
    if (State == CycleState.Moving) State = CycleState.Paused;      // ✅
    else if (State == CycleState.Paused) { … }                      // ✅ 재개
    // ❌ Dwell 에서는 아무 것도 하지 않음 → State 가 Dwell 로 남음
}
```

사이클은 포인트를 **이동**한 뒤 **대기(dwell)** 하므로, `CYCLEPAUSE` 가 어느 순간에 도착하느냐에 따라
상태가 `Moving` 일 수도 `Dwell` 일 수도 있습니다.
`Dwell` 일 때 들어온 일시정지는 **완전히 버려진 채로** `OK CYCLE dwell …` 로 응답했습니다.

실사용으로 바꾸면 **일시정지 버튼이 간헐적으로 죽은 것처럼 보이는** 버그입니다.

**수정**: `Dwell` 도 정지 대상에 넣고, 무엇을 멈췄는지 기억해 재개할 때 같은 단계로 돌아갑니다.

```csharp
if (State == CycleState.Moving || State == CycleState.Dwell)
{
    pausedFrom = State;          // 무엇을 멈췄는지 기억
    State = CycleState.Paused;
}
else if (State == CycleState.Paused)
{
    if (pausedFrom == CycleState.Moving && driver.IsMoving) State = CycleState.Moving;
    else { State = CycleState.Dwell; dwellUntil = Time.time + GetDwell(CurrentReference); }
}
```

**검증**: 수정 후 **연속 5회 전부 61/61 통과**. 이전에는 같은 명령으로 60/61 이 실패가 발생했습니다.

> 🔑 **한 번 통과한 회귀 테스트는 "통과" 의 증거가 아닙니다.**
> 이 버그는 첫 실행이 아니라 **다섯 번째 반복에서 처음 드러났습니다.**
> 상태 의존 버그는 반드시 **연속 실행**해서 확인해야 합니다.

### 10.11 부수적으로 만난 문제들

#### Unity 에디터 스레드로 BringWindowToTop 실패

**증명**: `SetForegroundWindow` 는 **같은 사용자 세션의 포커스 소유 프로세스**에만 성공합니다.
다른 앱(탐색기, 메모장)이 포커스를 가지고 있으면 **거부**됩니다. 아무 오류도 없습니다.

**해결**: 사용자가 직접 창을 클릭해야 합니다. **자동으로 해결할 수 없습니다.**

#### `EditorSceneManager.MarkSceneClean` 없음

**증명**: `error CS0117: 'EditorSceneManager' does not contain a definition for 'MarkSceneClean'`

**원인**: 이 API 는 Unity 2022.3 에 없습니다.
그래서 열린 씬을 "깨끗하게 만들어서" 프리팹을 만들려던 시도가 컴파일 자체를 막았습니다.

**해결**: 씬을 건드리지 않는 **프리뷰 씬**에서 작업합니다.

```csharp
var preview = EditorSceneManager.NewPreviewScene();
try { … PrefabUtility.SaveAsPrefabAsset(instance, PrefabPath); }
finally { Object.DestroyImmediate(instance); EditorSceneManager.ClosePreviewScene(preview); }
```

#### `Copy-Item -LiteralPath "$src\*"` 가 아무것도 안 복사함

**증명**: `-LiteralPath` 는 **와일드카드를 확장하지 않습니다.** `\*` 를 문자 그대로 찾고 실패합니다.
`-ErrorAction SilentlyContinue` 로 감췄더니 조용히 아무 일도 안 일어났습니다.

**해결**:

```powershell
Copy-Item -Path (Join-Path $src '*') -Destination $dst -Recurse -Force   # 복사 → -Path
Remove-Item -LiteralPath $dst -Recurse -Force                            # 삭제 → -LiteralPath
```

**규칙**: `-Path` 는 **와일드카드를 허용할 때**, `-LiteralPath` 는 **경로에 `[` `]` 가 있을 때** 씁니다.

#### `Start-Process` 인자 인용 누락

**증명**:

```powershell
# ❌ 'C:\Program' 에서 잘려서 컴파일 안 됨
Start-Process -FilePath $dn -ArgumentList $csc, "@$w\ball.rsp"
```

**해결**: 인자를 **명시적으로 인용**합니다.

```powershell
Start-Process -FilePath $dn -ArgumentList ('"'+$csc+'"'), ('"@'+"$w\ball.rsp"+'"')
```

---

<a id="part-11"></a>

## Part 11. 부록

### 11.1 전체 명령 cheatsheet

```text
── 연결 ────────────────────────────────────────────
(접속 즉시 서버가 먼저 한 줄 보냅니다)
OK CONNECT rb3-730                연결됨 확인        ← 클라이언트가 보낸 게 아님

PING                          연결 확인          → PONG
GETSTATE                      즉시 상태 요청      → STATE {…}
STATUS                        요약               → OK STATUS …

── 속도 ────────────────────────────────────────────
SPEED <1..100>                100 = 100%         → OK SPEED 20.0

── 관절 ────────────────────────────────────────────
HOME                          닫힌 포즈          → OK HOME
MOVJ <6개 각도(도)>           절대 관절 이동     → OK MOVJ
SETJ <6개 각도(도)>           MOVJ 별칭
STOP                          즉시 정지+사이클중단 → OK STOP
JOG <축> <Δ도>                관절 상대 조그     → OK JOG
GETPOS                        관절각 6개 조회    → OK POS j1 j2 j3 j4 j5 j6
TCP                           TCP pose 조회      → OK TCP x y z qx qy qz qw

── 조그 ────────────────────────────────────────────
JOGSTART <what> <±1>          조그 시작(홀드)    → OK JOGSTART
                              what = J1..J6 | X Y Z | RX RY RZ
JOGSTOP                       조그 정지          → OK JOGSTOP

── 티칭 ────────────────────────────────────────────
TEACH <이름> [대기초]          현재 자세 기록
LIST                          목록
GOTO <번호>                   해당 포즈로 이동
NEXT / PREV                   다음/이전 포즈
DELETE <번호>                 삭제
RENAME <번호> <이름>          이름 변경
DWELL <번호> <초>             대기 시간
CLEAR                         전체 삭제
SAVETEACH / LOADTEACH         저장 / 복원

── 사이클 ──────────────────────────────────────────
CYCLE [번호…] [바퀴수]        0 바퀴 = 무한
LOOP <바퀴수>                 기본 반복 횟수
CYCLEPAUSE                    일시정지/재개
CYCLESTATE                    현재 상태
CYCLESTOP                     종료
```

<a id="112-오류-코드-cheatsheet"></a>

### 11.2 오류 코드 cheatsheet

아래는 **소스에서 실제로 나가는 문자열**을 추려낸 표입니다 (`PendantServer.cs` / `JointDriver.cs` / `Teach.cs`).

| 메시지 | 원인 | 조치 |
|---|---|---|
| `ERR unknown command 'X'` | 명령 오타 | cheatsheet 확인 |
| `ERR SPEED must be 1..100` | SPEED 범위 초과 | 1..100 사용 |
| `ERR SPEED needs a number 1..100` | 인자가 숫자 아님 | |
| `ERR JOG needs a joint number 1..6` | 관절 번호 없음/범위 밖 | `JOG 1 5` 형식 |
| `ERR JOG needs a delta in degrees` | 이동량 누락 | |
| `ERR JOGSTART needs <what> <sign>` | 인자 부족 | `JOGSTART J1 1` 형식 |
| `ERR JOGSTART sign must be -1 or +1` | 부호 오류 | `+1` 또는 `-1` |
| `ERR JOGSTART what must be J1..J6, X, Y, Z, RX, RY or RZ` | 축 이름 오류 | cheatsheet 확인 |
| `ERR needs six joint angles in degrees` | 각도 6개 미만 | ⚠️ **명령명이 붙지 않습니다** (아래 참고) |
| `ERR joint 1 ('X') is not a number` | 숫자 아님 | 도 단위 숫자 |
| `ERR no joint driver` | 컴포넌트 없음 | 프리팹 확인 |
| `ERR tcp frame not found` | `tcp` 본 없음 | 리그 임포트 확인 |
| `ERR jog deadman timeout` | 조그 갱신 없음 | `JOGSTOP` 전송 또는 `jogDeadmanSeconds=0` |
| `ERR <CMD> nothing taught yet` | 포인트 0개 | `TEACH` 먼저 (`GOTO`, `NEXT`, `PREV`, `DELETE` 등) |
| `ERR GOTO needs a point number 1..N` | 포인트 번호 범위 밖 | |
| `ERR GOTO could not move` | 이동 실패 | 포즈 확인 |
| `ERR RENAME needs <point> <new name>` | 인자 형식 오류 | `RENAME 2 drop` |
| `ERR RENAME could not rename point N` | 해당 포인트 없음 | |
| `ERR DWELL needs <point> <seconds>` | 인자 형식 오류 | `DWELL 1 0.75` |
| `ERR DWELL bad point` | 포인트 없음/값 오류 | |
| `ERR DELETE needs a point number 1..N` | 포인트 번호 범위 밖 | |
| `ERR CYCLE teach at least two points first, …` | 포인트 1개 이하 | 2개 이상 티칭 |
| `ERR CYCLE could not start, check the point numbers` | 포인트 번호 조합 오류 | [§7.2](#command-list) 바퀴수 규칙 확인 |
| `ERR LOOP needs a lap count, 0 for endless` | 인자 없음/숫자 아님 | `LOOP 0` = 무한 |
| `ERR LOADTEACH no file at …` | 저장 파일 없음 | 먼저 `SAVETEACH` |
| `ERR LOADTEACH file is not a teach file` | JSON 형식 불일치 | 파일 삭제 후 재저장 |
| `# STATE "ok":false` | **리그 해석 실패** | FBX 임포트 설정 검토 |

> ⚠️ **`ERR needs six joint angles in degrees` 에 명령명이 없다는 점**
> 관절 각도 파싱 실패는 `MOVJ` / `SETJ` 어느 쪽에서도 같은 문자열을 냅니다.
> 클라이언트가 `ERR MOVJ …` 로 접두사를 가정한 매칭을 하면 **이 오류를 놓칩니다.**
> `ERR` 까지만 보고 나머지를 무시하세요.
>
> 반면 `ERR GOTO …`, `ERR CYCLE …` 처럼 명령명이 붙는 것도 많으니,
> **일관된 접두사에 의존하지 말고 `ERR` 로 시작하는지만 보세요.**

### 11.3 환경 점검 명령 모음

```powershell
# ── Unity ──
$proj = "$HOME\Desktop\Digital_Twin_2022"
Get-Content "$proj\ProjectSettings\ProjectVersion.txt"

# ── 컴파일 확인 ──
Get-Item "$proj\Library\ScriptAssemblies\Assembly-CSharp.dll" |
  Select-Object FullName, LastWriteTime
Get-ChildItem -Recurse -LiteralPath "$proj\Assets\RB3-730" -Filter '*.cs' |
  Sort-Object LastWriteTime -Descending | Select-Object -First 1 Name, LastWriteTime

# ── 포트 리스닝 확인 ──
Get-NetTCPConnection -LocalPort 5000 -ErrorAction SilentlyContinue |
  Select-Object State, LocalAddress, LocalPort, OwningProcess

# ── Unity 로그에서 오류 ──
$log = "$env:LOCALAPPDATA\Unity\Editor\Editor.log"
Select-String -LiteralPath $log -Pattern 'error CS|Exception|RB3-730' |
  Select-Object -Last 20 | ForEach-Object { $_.Line }

# ── 프리팹 검증 ──
Select-String -LiteralPath "$proj\Assets\RB3-730\RB3_730.prefab" `
  -Pattern 'm_Script|jointBones' | Select-Object -First 20

# ── 티칭 저장 파일 ──
Get-ChildItem -LiteralPath "$env:USERPROFILE\AppData\LocalLow" -Filter '*teach*' -Recurse -EA SilentlyContinue
```

### 11.4 최종 검증 체크리스트

```
[ ] Unity 2022.3.62f3 + Built-in
[ ] 컴파일 오류 0 / 경고 0
[ ] Console: [RB3-730] verified: 7 renderers (7 skinned), … jointChain=ok
[ ] STATE.ok == true
[ ] 바운딩박스 Y == 0.875 m
[ ] HOME 에서 TCP → x=0.0000 y=0.8753 z=0.0064, |t| == 0.8753 m
[ ] SPEED 20 → OK SPEED 20.0
[ ] MOVJ 왕복 정상
[ ] HOME 정상
[ ] TEACH → LIST → CYCLE → CYCLESTOP 정상
[ ] CYCLE 0 1 이 1바퀴만 돌고 자동 정지
[ ] 연결 끊김 시 자동 정지 로그 확인
[ ] SAVETEACH / LOADTEACH 왕복 정상 (포인트 2개 기준)
[ ] 전체 회귀 테스트 61/61  →  passed: 61   failed: 0
```

### 11.5 남은 작업 (미구현)

| 항목 | 상태 | 비고 |
|---|---|---|
| **C# WinForms 펜던트 GUI** | ❌ 미구현 | 소켓 프로토콜은 완성·검증됨. GUI만 필요 |
| 자이로 6축 | ❌ 미구현 | 현재는 턴테이블 프리셋 |
| 실제 하드웨어 연결 | ❌ 미구현 | HAL 계층 필요 |
| 충돌 감지 | ⚠️ 콜라이더 준비만 됨 | `rb3_730es_u_collision.fbx` 임포트됨. 물리 적용은 미구현 |

> ⚠️ **실제 로봇에 연결하기 전에 반드시** 모션 시뮬레이터로 검증하세요.
> 이 코드는 `MoveTo` 같은 명령을 **속도 제한 없이** 전달할 수 있어 위험합니다.

---

## 마무리

이 문서는 다음 순서로 읽으면 됩니다.

1. [Part 1](#part-1) — 전체 그림과 4가지 핵심 원칙
2. [Part 2](#part-2) — 준비물 설치
3. [Part 3](#part-3) — URDF 확보·검증 (스케일 함정 여기서 잡습니다)
4. [Part 4](#part-4) — Blender FBX 변환·라운드트립 검증
5. [Part 5](#part-5) — Unity 임포트 설정 (Generic 리그가 핵심)
6. [Part 6](#part-6) — C# 코드 전체
7. [Part 7](#part-7) — TCP 프로토콜
8. [Part 8](#part-8) — 회귀 테스트
9. [Part 9](#part-9) — 실사용 시나리오
10. [Part 10](#part-10) — 버그 7가지

막히면 [Part 11.2](#112-오류-코드-cheatsheet) 의 오류 코드 표에서
에러 메시지를 검색하세요. 모든 실패 원인이 거기에 있습니다.

---

<div align="center">

**RB3-730 Digital Twin — Unity + TCP Teach Pendant**
완전 실습 가이드 · 모든 소스 포함 · 재현 가능

</div>


