# RB3-730 표면 검사 티칭 데이터셋 — LLM 없이 진행하는 작업 가이드

> 이 문서는 LLM 없이(사람 + 일반 개발 도구만으로) `Digital_Twin_2022` Unity 프로젝트에
> 표면 검사 티칭 데이터셋 기능을 구현·검증하는 전 과정을 정리한 것입니다.
> 이미 구현이 완료된 상태의 재현/유지보수 안내서이며, 새 환경에서 처음부터 할 때의 진행 순서이기도 합니다.

---

## 0. 작업 개요

| 항목 | 내용 |
|---|---|
| 대상 프로젝트 | `C:\Users\Administrator\Desktop\Digital_Twin_2022` (Unity 2022.3.62f3) |
| 목적 | RB3-730 로봇 팔로 표면(터빈 블레이드/금형 패널/파이프)을 serpentine 래스터 주행하며 스캔점을 자동 생성 → 티칭 데이터셋으로 주입 |
| 산출 기능 | Inspector GUI(자동 배치 실행), 펜던트 `SCAN` 명령, Tools 메뉴, gizmo, 데이터셋 검증 로그 |
| 필수 검증 | (1) 오프라인 수학 하네스 ALL OK, (2) Unity 정식 컴파일 0 error / 0 warning, (3) 씬에서 수동 실행 QA |

### 대상 소스 파일 (총 9개, `.asmdef` 없음 → 전부 `Assembly-CSharp`에 귀속)

```
Assets/RB3-730/
├── Runtime/
│   ├── RB3_730_JogPanel.cs        # 수동 조작 패널 (기존)
│   ├── RB3_730_JointDriver.cs     # 관절 구동 (기존)
│   ├── RB3_730_Kinematics.cs      # FK/IK (기존)
│   ├── RB3_730_PendantServer.cs   # 펜던트 통신 — SCAN 명령 추가
│   ├── RB3_730_ScanPlan.cs        # [핵심 신규] 래스터 경로 계획
│   ├── RB3_730_Surface.cs         # 표면 메시/법선 — winding 수정
│   ├── RB3_730_SurfaceInspector.cs# [신규] Inspector + 주입 + gizmo
│   └── RB3_730_Teach.cs           # 티칭 포인트 — ReplaceAll 추가
├── Editor/
│   └── RB3_730_Setup.cs           # 씬 셋업 메뉴 — v11 + inspector 부착
├── Data/rb3_730es_u_joints.json   # 관절 정의
├── Models/…                       # FBX/머티리얼
└── RB3_730.prefab                 # 로봇 프리팹
```

설계 의도의 1차 레퍼런스: `C:\Users\Administrator\Desktop\3d.json`

---

## 1. Phase 0 — 기존 코드 파악 (반드시 먼저)

LLM 없이 진행할 때 가장 큰 차이는 "먼저 읽는 시간"입니다. 아래 순서로 파악합니다.

1. **README.md** (`Assets/RB3-730/README.md`) — 프리팹 구성·컴포넌트 관계.
2. **`JointDriver.cs`** — 6축 로봇의 관절 값이 어떻게 메시 변형에 반영되는지.
3. **`Kinematics.cs`** — FK(관절→TCP 포즈) / IK의 입력출력 시그니처. 스캔 계획은 이것만으로 검증됩니다.
4. **`Teach.cs`** — `TaughtPoint` 구조(`name`, `joints[6]`, `tcp[7]=xyz+quat`, `dwell`)와 데이터셋 로드/저장 방식.
5. **`Surface.cs`** — 표면 메시의 파라미터 좌표(u,v), 법선 계산, 드라이버 로컬↔월드 변환.
6. **`PendantServer.cs`** — 명령 디스패치 패턴(문자열 명령 → 핸들러).
7. **`Setup.cs`** — 씬 구성 메뉴와 프리팹 인스턴스화 방식. 컴포넌트 추가 패턴은 이 파일을 따릅니다.

> 팁: 각 파일 상단의 주석과 `3d.json`을 대조하면 설계 의도가 드러납니다.
> 코드베이스 규칙: **"Fix minimally, NEVER refactor while fixing"** — 기존 스타일을 그대로 따르고, 리팩터링은 하지 않습니다.

---

## 2. Phase 1 — 구현 순서 (의존 순서대로)

아래 순서를 지키면 어느 단계에서든 컴파일이 무너지지 않습니다. 각 단계는 IDE(Visual Studio / Rider / VS Code)에서 편집하고, 저장 후 **Unity 창으로 포커스를 옮겨 Console을 확인**하는 것으로 마무리합니다.

### 2.1 `Surface.cs` — 법선 방향修正 (기저 수정)
- 문제: 표면 법선 부호(`NormalSign`)와 삼각형 와인딩이 뒤집혀 있으면 래스터 접근 방향이 반대로 잡힘.
- 확인법: Scene 뷰에서 표면法線 gizmo가 바깥쪽을 향하는지, 프리팹 표면 위에 로봇 툴이 표면 **바깥**에서 접근하는지.
- 핵심 식: 면 법선 = `Cross(edge1, edge2)`의 와인딩 순서와 일치해야 함.

### 2.2 `ScanPlan.cs` — 핵심 알고리즘 (신규)
가장 어려운 부분. 아래 3개 함수로 구성합니다.

1. **`BuildRaster(...)`** — (u,v) 파라미터 공간에서 지그재그(serpentine) 라인 생성.
   - 입력: 시작/끝 v 경계, 라인 간격(standoff 기반 간격), 최대 법선 선회각.
   - 산출: 라인별 포즈 목록. 방향이 바뀔 때마다 도구 선회각이 제한 안에 들어가야 함.
2. **`SearchPlacement(...)`** — 후보 위치를 FK로 테스트하며 유효성을 필터링.
   ```
   SearchPlacement(..., vStart, vEnd,
       float maxNormalTurnDeg = 30f,   // 도구 선회 한계
       float maxJointJumpDeg = 45f)    // 인접 관절점프 한계
   ```
   - 후보마다 `Kinematics` FK로 관절값 계산 → 이전 점 대비 관절 점프·법선 선회角 검사 → 실패 시 그 지점에서 **컷**.
3. **`SolvePath(poses, maxJointJumpDeg)`** → `RB3_730_ScanResult`
   - `Ok`, `Count`, `WorstResidualM`(FK 잔차), `WorstOrientErrDeg`, `WorstJumpDeg`, `WorstStepM`, `MinMarginM`.
   - 마지막 점은 출발 위치로 복귀하는 **왕복(RT)** 처리.
4. **적응형 코너 세분 (`EmitRun` / `SplitPoint`)** — 선회가 한계를 넘는 구간에서만 점을 쪼개(서브디비전) 간격을 줄임. 코너에만 밀도가 올라가고 직선 구간은 불필요하게 점이 늘지 않게 합니다.

**이름 규칙 (검증 시 반드시 확인)**
| 포인트 | 이름 |
|---|---|
| 첫 점 (approach) | `SCAN-AP` |
| 래스터 내부 | `R{행}-{서열 2자리}` — v 경계마다 행 번호 리셋, 세분 삽입 점 포함 |
| 마지막 점 (return) | `SCAN-RT` |

**기본 파라미터**: `maxNormalTurnDeg = 30`, `maxJointJumpDeg = 45` — 3곳의 Inspector 호출부도 동일 기본값.

### 2.3 `Teach.cs` — `ReplaceAll` 추가
```csharp
public int ReplaceAll(IList<TaughtPoint> incoming)
```
- 기존 데이터셋을 통째로 교체하고 교체 개수를 반환. `SCAN-AP`/`SCAN-RT`/`R{row}-{ord}` 이름 체계 유지.

### 2.4 `SurfaceInspector.cs` — Inspector GUI + 주입 (신규)
- `OnInspectorGUI`: standoff, 라인 간격, samples/line, 파라미터 u/v 범위, 한계각 슬라이더 → **Run Scan** 버튼.
- 실행 흐름: 표면→법선 기준 좌표계 변환(`origin`, `toDriver`) → `BuildRaster` → `SolvePath` → `ReplaceTeachDataset` → 결과 로그.
- 로그 포맷 (검증 기준):
  ```
  [이름] dataset N points injected=… residual …E2 m orient …F3 deg jump …F1 deg step …F4 m margin …F3 m
  ```
- 좌표 변환 주의: **위치는 `transform.TransformPoint(local)`, 회전은 `transform.rotation * localQuat`**
  (Unity에는 `Transform.TransformRotation()`이라는 메서드가 **없음** — 실제 컴파일에서 잡힌 버그).
- Gizmo: 래스터 라인·표면 경계를 Scene 뷰에 표시.

### 2.5 `PendantServer.cs` — `SCAN` 명령
- 기존 명령 디스패치 패턴을 그대로 따라 `SCAN` 문자열 명령 → Inspector와 동일한 경로 계획 실행.
- `StartJog` 복구 누락에 주의(이 파일을 건드리는 김에 손상되기 쉬움).

### 2.6 `Setup.cs` — 셋업 메뉴 v11
- 버전 마커 `v10 → v11` 갱신, 프리팹에 `SurfaceInspector` 컴포넌트 부착, Tools 메뉴 항목 추가.
- 패턴: 기존 `Setup` 코드가 반복하는 "프리팹 찾기 → `AddComponent` → 초기화" 흐름을 그대로 확장.

---

## 3. Phase 2 — 오프라인 검증 하네스 (LLM 없이 핵심)

Unity 씬 없이 알고리즘만 검증하기 위한 순수 C# 하네스를 만듭니다. **이 단계를 건너뛰면 씬에서야 에러를 발견해 시간을 크게 잃습니다.**

### 3.1 하네스 구성

작업 디렉터리 예: `%TEMP%\opencode\scan` (경로에 공백이 없어야 합니다)

| 파일 | 역할 |
|---|---|
| `UnityMath.cs` | `Vector3`, `Quaternion`, `Mathf` 등 Unity 타입의 **직접 구현(stub)** — 벡터/쿼터니언 연산, 내적/외적, 정규화, 각도 |
| `UnityGeom.cs` | 표면 노멀/파라미터 좌표 등 프로젝트 공용 기하 유틸 |
| `ScanTest.cs` | 래스터 계획 종합 테스트 (주 검증 도구) |
| `SurfTest.cs` | 표면 법선/와인딩 회귀 테스트 (별도 exe로) |

### 3.2 컴파일 명령 (.NET Framework csc — C#5)

```bat
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe /nologo /target:exe /optimize- ^
  /out:ScanTest.exe ^
  UnityMath.cs UnityGeom.cs ScanTest.cs ^
  "C:\Users\Administrator\Desktop\Digital_Twin_2022\Assets\RB3-730\Runtime\RB3_730_Surface.cs" ^
  "C:\Users\Administrator\Desktop\Digital_Twin_2022\Assets\RB3-730\Runtime\RB3_730_Kinematics.cs" ^
  "C:\Users\Administrator\Desktop\Digital_Twin_2022\Assets\RB3-730\Runtime\RB3_730_ScanPlan.cs"
```

실행: `ScanTest.exe` → 마지막 줄이 **`ALL OK`** 여야 함. 소요 시간 약 8~9초.

### 3.3 하네스 제약 조건 (지키지 않으면 컴파일 실패)

1. **C#5 문법만 사용**: `$"보간문자열"`, `nameof()`, 표현식 본문 멤버(`=>`), `out var` 전부 **금지**.
   (이 하네스는 구형 `csc.exe`로 컴파일하기 때문 — Unity 쪽은 C#9라 문법이 다름.)
2. **`Main` 중복 금지**: `SurfTest.cs`, `Probe.cs`는 각각 독립 exe로 컴파일합니다.
   `ScanTest.exe` 빌드에 섞으면 `CS0017` (Main 여러 개)로 실패합니다.
3. **검증 대상 소스만 포함**: 하네스가 필요로 하는 Unity 스텁과 대상 파일만 넣고, 프로젝트 나머지 파일은 넣지 않습니다.

### 3.4 통과 기준 (회귀 스펙 — 숫자로 확인)

| 모델 | 포인트 수 | tool turn ≤ | joint jump ≤ | margin ≥ | FK 잔차 |
|---|---|---|---|---|---|
| TurbineBlade | ≥130 (실제 162) | 30° (29.84°) | 45° (34.95°) | 0 m (0.1388 m) | <1e-5 (약 3.4e-7 m) |
| MouldedPanel | 130 | 30° (8.88°) | 45° | 0 m (0.2052 m) | 동일 |
| PipeBody | 130 | 30° (28.52°) | 45° (37.24°) | 0 m (0.0868 m) | 동일 |

- `SurfTest.exe`: 표면 법선/와인딩 회귀도 **ALL OK**.
- **하네스 통과 ≠ Unity 컴파일 통과**입니다. 다음 Phase가 필요합니다.

---

## 4. Phase 3 — Unity 정식 컴파일 검증

### 방법 A: 사람이라면 이것만 하면 됨 (권장)

1. Unity 창으로 포커스 이동 → `Ctrl+R` (Assets → Refresh).
2. **Console** 창에서 0 error / 0 warning 확인.
3. 이 방법이 Unity가 실제로 하는 컴파일 그 자체이므로 가장 확실합니다.

### 방법 B: 자동화/헤드리스 검증 (Unity 창을 못 쓸 때)

Unity가 쓰는 **같은 컴파일러 + 같은 rsp**를 직접 실행합니다.

1. **rsp 위치 확인**
   ```
   Library\Bee\artifacts\1900b0aE.dag\Assembly-CSharp.rsp         # 런타임
   Library\Bee\artifacts\1900b0aE.dag\Assembly-CSharp-Editor.rsp  # 에디터
   ```
   (rsp는 ~241개 `-r:` 참조, `/langversion:9.0`, `UNITY_*` define, 소스 생성기 analyzer 목록을 담고 있음)

2. ** rsp를 복사해 소스 목록을 현재 파일로 갱신** — 새롭게 만든 `.cs`는 rsp에 자동 반영되지 않습니다.
   런타임 rsp의 기존 소스 줄(`.cs`로 끝나는 줄)을 지우고 현재 8개 파일을 추가:
   ```
   "Assets/RB3-730/Runtime/RB3_730_JogPanel.cs"
   … (총 8개: JointDriver, Kinematics, PendantServer, ScanPlan, Surface, SurfaceInspector, Teach 포함)
   ```

3. **출력 경로를 임시 폴더로 리다이렉트** — Unity 실행 중이므로 Unity의 아티팩트를 건드리면 안 됩니다.
   `-out:` / `-refout:` / `-pdb:` 를 임시 경로(`%TEMP%\unitycheck\`)로 교체.
   ⚠️ rsp의 `-r:` 경로는 **상대경로**이므로 실행 시 작업 디렉터리가 프로젝트 루트여야 합니다.

4. **실행** (msbuild의 csc가 아니라 Unity 자체 Roslyn 사용):
   ```powershell
   cd C:\Users\Administrator\Desktop\Digital_Twin_2022
   & "C:\Program Files\Unity\Hub\Editor\2022.3.62f3\Editor\Data\NetCoreRuntime\dotnet.exe" `
     exec "C:\Program Files\Unity\Hub\Editor\2022.3.62f3\Editor\Data\DotNetSdkRoslyn\csc.dll" `
     /nostdlib /noconfig "@C:\…\unitycheck\rt.rsp"
   ```
   - **EXIT=0 + 출력 없음 = 0 error / 0 warning.**
5. 에디터 rsp도 동일하게, 단 `Assembly-CSharp.ref.dll` 참조를 **방금 만든 신규 ref.dll**로 교체 후 실행.

---

## 5. 함정 목록 (이 순서로 시간을 낭비하지 말 것)

LLM으로 작업할 때 실제로 발이 묶였던 지점들입니다. 사람도 동일 함정에 걸립니다.

| # | 함정 | 증상 | 해결 |
|---|---|---|---|
| 1 | **msbuild/VS Roslyn 크래시** | `csc.exe`가 EXIT=-2146232797 (0x80131623)로 **조용히 종료**. 아무 메시지 없음. Unity 타입을 참조하는 코드를 컴파일할 때마다 발생. | 원인은 코드가 아님(Unity 설치 환경 문제). MSBuild 경로의 csc로 Unity 프로젝트를 검증하려 하지 말 것. **Unity의 `DotNetSdkRoslyn\csc.dll`만 사용** (Phase 3 방법 B). |
| 2 | **Editor.log 스테이 에러** | Console/로그에 CS0103 등이 떠 있는데 현재 코드에는 해당 라인이 아님. | 로그의 에러 라인 번호를 **현재 파일과 반드시 대조**. 이 프로젝트의 24건은 편집 중간 상태의 잔재였음. 파일 수정 시각(mtime)과 로그 시각을 비교해 최신 여부 확인. |
| 3 | **rsp 소스 목록 누락** | 새 `.cs`를 만들어도 rsp에는 없음 → Unity가 모르는 파일. | rsp 생성 시각 확인 후 소스 목록 직접 갱신 (Phase 3 방법 B-2). |
| 4 | **존재하지 않는 Unity API** | `transform.TransformRotation(...)` 같은 없는 메서드 사용 → CS1061. | IDE를 반드시 Unity 프로젝트로 열어 IntelliSense가 Unity 어셈블리를 보게 하거나, Phase 3 컴파일로 확인. (`TransformPoint`의 회전 대응물은 `transform.rotation * local`.) |
| 5 | **하네스 vs Unity 문법 괴리** | 하네스 ALL OK인데 Unity에서 에러, 또는 그 반대. | 두 검증 경로(Phase 2 + Phase 3)를 **둘 다** 통과해야 완료. |
| 6 | **Main 중복** | `CS0017`. | `SurfTest`/`Probe`는 독립 exe로. |
| 7 | **Unity 자동 리프레시 안 됨** | 파일을 수정해도 Console 갱신 없음. | `kAutoRefresh` 레지스트리 미설정 환경. Unity 창 클릭 → `Ctrl+R` 강제 Refresh. |
| 8 | **Editor.log 잠금** | 실행 중인 Unity가 로그 파일을 잠금 → 일반 파일 읽기 실패. | PowerShell에서 `FileStream(..., FileShare.ReadWrite)`로 읽기. |
| 9 | **중간 상태 편집의 잔재** | 테스트 변수·중간 메서드가 코드에 남음 (예: 미사용 `camberPos`). | 컴파일 warning(예: CS0219)을 방치하지 말고 즉시 제거 — 0 warning이 완료 기준. |

---

## 6. Phase 4 — 씬 반영 및 수동 QA

컴파일 통과 후 실제 동작 확인:

1. **Setup 메뉴 실행** (`Tools` → RB3-730 셋업) — 프리팹 갱신, `SurfaceInspector` 컴포넌트 부착 확인 (v11).
2. **Inspector에서 Run Scan** — 콘솔 로그에서 확인:
   - `injected=` 실제 포인트 수 (>0)
   - `residual` ≈ 0 (FK 잔차)
   - `jump` ≤ 45°, `margin` > 0
3. **Scene 뷰 gizmo** — 래스터 라인이 표면 바깥에 평행하게 깔리는지, 코너에서만 밀도가 높아지는지.
4. **펜던트 `SCAN`** — 서버 경유 실행이 같은 결과를 내는지.
5. **로봇 동작** — 티칭 데이터셋 로드 후 로봇이 스캔 경로를 실제로 따라가는지(가장 느리고 확실한 검증).

---

## 7. 최종 완료 체크리스트

- [ ] 하네스 `ScanTest.exe` → `ALL OK` (3개 모델, 기준 숫자 충족)
- [ ] 하네스 `SurfTest.exe` → `ALL OK`
- [ ] Unity `Ctrl+R` 후 Console **0 error / 0 warning** (또는 Phase 3 방법 B: EXIT=0, 출력 없음)
- [ ] Setup 메뉴 v11 실행 성공, 프리팹에 inspector 컴포넌트 존재
- [ ] Inspector Run Scan 로그: injected > 0, jump ≤ 45°, margin > 0, residual ≈ 0
- [ ] 펜던트 `SCAN` 동작 확인
- [ ] PointName 규칙 준수: `SCAN-AP` / `R{row}-{ord}` / `SCAN-RT`
- [ ] 중간 상태 잔재 없음 (미사용 변수·중복 코드 0)

---

## 부록 A. 작업 시간 배분 참고 (LLM 미사용 시)

| 단계 | 예상 비중 | 비고 |
|---|---|---|
| Phase 0 코드 파악 | ~20% | 생략하면 이후 전부 재작업 위험 |
| Phase 1 구현 | ~35% | ScanPlan 알고리즘이 대부분 |
| Phase 2 하네스 구축 | ~15% | 한 번 만들면 이후 회귀에 계속 사용 |
| Phase 3 컴파일 검증 | ~10% | 방법 A(Unity Refresh)면 짧음 |
| Phase 4 QA | ~20% | 씬 재현·수치 확인 |

## 부록 B. 핵심 명령 모음

```bat
:: 1) 하네스 컴파일 (C#5, 작업 디렉터리 = 하네스 폴더)
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe /nologo /target:exe /optimize- ^
  /out:ScanTest.exe UnityMath.cs UnityGeom.cs ScanTest.cs ^
  "<프로젝트>\Assets\RB3-730\Runtime\RB3_730_Surface.cs" ^
  "<프로젝트>\Assets\RB3-730\Runtime\RB3_730_Kinematics.cs" ^
  "<프로젝트>\Assets\RB3-730\Runtime\RB3_730_ScanPlan.cs"
ScanTest.exe

:: 2) Unity 정식 컴파일 (작업 디렉터리 = 프로젝트 루트, rsp는 소스 갱신+출력 리다이렉트 후)
cd /d "<프로젝트>"
"C:\Program Files\Unity\Hub\Editor\2022.3.62f3\Editor\Data\NetCoreRuntime\dotnet.exe" ^
  exec "C:\Program Files\Unity\Hub\Editor\2022.3.62f3\Editor\Data\DotNetSdkRoslyn\csc.dll" ^
  /nostdlib /noconfig "@<임시>\rt.rsp"
```

---
*작성 기준: 2026-10-06 작업 세션 결과 (Unity 2022.3.62f3 / Windows)*
