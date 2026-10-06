# YouTube 다운로드 스크립트 정리 (download3.py)

pytubefix 기반 YouTube 다운로드 스크립트를 처음부터 다시 만들거나 고칠 수 있도록
작업 과정에서 발생한 문제와 해결 과정을 정리한 문서입니다.

> 참고: 작업이 끝난 시점에 `C:\Users\Administrator\Desktop\동영상` 폴더가
> 바탕화면에서 사라졌습니다(Desktop 폴더들이 `새 폴더`, `프로그램` 등으로
> 재편된 것으로 보입니다). 아래 5절에 완성된 전체 코드를 통째로 실어두었으니,
> 새 폴더를 만들고 그대로 저장하면 바로 재현됩니다.

---

## 1. 최초 문제: 아무것도 다운로드되지 않음

### 증상

```
$ python download3.py
Failed to get playlist links: 'list'
No links found. Exiting.
```

### 원인

스크립트는 항상 플레이리스트만 가정하고 있었습니다.

```python
playlist = Playlist(playlist_url, client='WEB')   # Playlist는 list= 파라미터가 필수
```

그런데 설정된 URL이 일반 영상 1개였습니다.

```
https://www.youtube.com/watch?v=9kM3mnxrMMM      # list= 없음 → KeyError 'list'
```

`list=`가 없으니 `Playlist()`가 `'list'` KeyError를 던지고, 예외 핸들러가 빈
리스트를 반환한 뒤 곧바로 종료했습니다. 즉 `download_video()`가 **한 번도
호출된 적이 없었습니다.** 네트워크나 봇 탐지 문제가 아니라 링크 파싱 단계에서
막힌 것입니다.

### 확인 방법

```powershell
python -c "from pytubefix import YouTube; v=YouTube('https://www.youtube.com/watch?v=9kM3mnxrMMM', client='WEB'); print(v.title)"
# How do microSD Cards Store 2 TB in such a TINY space?   ← 영상 자체는 접근 가능
```

영상 접근은 되는데 스크립트가 안 돌아가는 것 → 링크 파싱 로직 문제로 확정.

### 해결

`get_links()`를 추가해 URL 형태에 따라 분기하도록 했습니다.

```python
def get_links(url):
    """Accepts either a single video URL or a playlist URL and returns a link list."""
    if 'list=' in url:
        return get_playlist_links(url)
    return [url]
```

---

## 2. 단일 영상 / 플레이리스트 선택 기능

`input()`으로 어느 소스를 쓸지 고르게 하고, URL은 상수로 분리했습니다.

```python
SINGLE_VIDEO_URL = "https://www.youtube.com/watch?v=9kM3mnxrMMM"
PLAYLIST_URL = "https://www.youtube.com/playlist?list=PL6rx9p3tbsMt8YAmrZwHrabSAcyQR9ad2"

if __name__ == "__main__":
    # Select which source to download: 'single' or 'playlist'
    mode = input("Select source [single/playlist]: ").strip().lower()

    if mode == 'single':
        source_url = SINGLE_VIDEO_URL
    elif mode == 'playlist':
        source_url = PLAYLIST_URL
    else:
        print("Invalid selection. Exiting.")
        exit()

    print(f"Source: {source_url}")
    links = get_links(source_url)
    if not links:
        print("No links found. Exiting.")
    else:
        # Fewer concurrent workers reduce bot-detection risk
        with Pool(processes=2) as pool:
            pool.map(download_video, links)
```

동작:

```
$ python download3.py
Select source [single/playlist]: single
```

검증 결과:

| 입력 | 결과 |
|---|---|
| `single` | `['https://www.youtube.com/watch?v=9kM3mnxrMMM']` |
| `playlist` | 24개 링크 추출 성공 |

---

## 3. 두 번째 문제: `PoToken INVALID`

### 증상

```
SABR YouTube is forcing ads, wait 4.8 seconds to skip
https://www.youtube.com/watch?v=9kM3mnxrMMM download failed: SABR Maximum reload
attempts reached. Stream protection status: PoToken INVALID
```

### 원인: pytubefix 버전

pytubefix는 YouTube의 봇 탐지 우회를 위해 bundled `botGuard.js`를 Node.js로
돌려 **PoToken**(Proof-of-Origin Token)을 생성합니다. 그런데 설치된
**10.11.0에 번들된 `vm/botGuard.js`가 구버전**이어서, 발급된 토큰을 유튜브
서버가 거부했습니다.

확인 과정:

```powershell
python -c "import pytubefix; print(pytubefix.__version__)"
# 10.11.0
```

### 해결: 업그레이드

```powershell
python -m pip install --upgrade pytubefix
# Successfully installed pytubefix-11.2.0
```

11.2.0은 유효한 PoToken을 생성해서 문제가 사라졌습니다. 검증:

```
title: How do microSD Cards Store 2 TB in such a TINY space?
720p itag: 136
downloaded
size 36171998        ← 36MB 정상 수신
```

### 참고: client 옵션별 동작

`client=` 값을 바꿔 가며 확인한 결과:

| client | 결과 |
|---|---|
| `WEB` | OK (PoToken 필요) |
| `MWEB` | OK |
| `ANDROID` | HTTP 400 Bad Request |
| `IOS` | HTTP 400 Bad Request |
| `TV` | VideoUnavailable |
| `WEB_SPOR` / `WEB_EMBEDDED` / `TV_EMBEDDED` | KeyError (미지원) |

`client='WEB'`가 기본값이므로 그대로 쓰면 됩니다. PoToken 생성을 위해
시스템 Node.js(`C:\Program Files\nodejs\node.exe`)가 필요하며, 이 환경에는
이미 설치되어 있었습니다.

---

## 4. 세 번째 문제: ffmpeg 병합 실패 (exit status 1)

### 증상

```
  Merging audio...
... download failed: Command '[...ffmpeg.exe, -i, _v_How do microSD Cards Store
2 TB in such a TINY space?.mp4, ...]' returned non-zero exit status 1.
```

### 원인: 파일명에 `?`가 들어감

영상 제목이 "How do microSD Cards Store 2 TB in such a TINY **space?**" 인데,
`?`는 Windows 파일명 금지 문자입니다. 다음과 같이 어긋났습니다.

1. pytubefix가 `_v_...space?.mp4`로 저장 요청
2. **Windows가 `?`를 제거**해서 실제 파일은 `_v_...space.mp4`로 생성됨
3. ffmpeg 호출은 원래 이름(`?` 포함)으로 감 → `Invalid argument`, exit 1
4. `check=True`가 예외로 변환해 메시지에 `exit status 1`만 남고 **진짜 원인은
   숨겨짐**

직접 재현해서 확인:

```powershell
& "C:\Program Files\KMPlayer 64X\LAVFilters64\ffmpeg.exe" -i "_v_...space?.mp4" ...
# _v_How do microSD Cards Store 2 TB in such a TINY space?.mp4: Invalid argument
```

ffmpeg 자체는 멀쩡했습니다 (`aac` 인코더 지원 확인). KMPlayer 번들
ffmpeg이 `C:\Program Files\KMPlayer 64X\LAVFilters64\ffmpeg.exe`에 있어
자동 탐지됩니다.

### 해결 3가지

**1) 파일명 정규화 함수 추가**

```python
ILLEGAL_FILENAME_CHARS = r'<>:"/\|?*'

def sanitize_filename(name):
    cleaned = ''.join('_' if c in ILLEGAL_FILENAME_CHARS else c for c in name)
    return cleaned.rstrip(' .') or 'video'
```

4개 다운로드 분기(step 1~4)의 출력 파일명과 임시 파일명(`_v_`, `_a_`)에 모두
적용했습니다.

**2) ffmpeg 오류 메시지 노출**

`check=True` 대신 returncode를 직접 확인해서 stderr를 보여줍니다.

```python
def merge_audio_video(video_path, audio_path, output_path):
    print(f"  Merging audio...")
    cmd = [
        FFMPEG_PATH, "-i", video_path, "-i", audio_path,
        "-c:v", "copy", "-c:a", "aac",
        "-map", "0:v:0", "-map", "1:a:0",
        "-y", output_path
    ]
    result = subprocess.run(cmd, capture_output=True, text=True, errors='replace')
    if result.returncode != 0:
        raise RuntimeError(f"ffmpeg failed: {result.stderr.strip()[-500:]}")
    os.remove(video_path)
    os.remove(audio_path)
```

**3) 실패 시 임시 파일 정리**

```python
    except Exception as e:
        print(f"{link} download failed:", e)
        for leftover in ('_v_', '_a_'):
            for f in os.listdir('.'):
                if f.startswith(leftover):
                    try:
                        os.remove(f)
                    except OSError:
                        pass
```

### 결과

```
  Merging audio...
How do microSD Cards Store 2 TB in such a TINY space? (720p adaptive + ffmpeg)
```

```
Length   Name
------   ----
 42300013 How do microSD Cards Store 2 TB in such a TINY space_ [720p].mp4
```

42MB 생성, `downloaded_files.json`에도 기록 완료. 제목 끝의 `?`가 `_`로
바뀐 것이 sanitization 결과입니다.

---

## 5. 완성된 전체 코드 (download3.py)

새 폴더를 만들고 아래 내용을 그대로 `download3.py`로 저장하세요.

```python
import os
import json
import subprocess
from pytubefix import Playlist, YouTube
from multiprocessing import Pool

downloaded_files_path = 'downloaded_files.json'

def load_downloaded_files():
    if os.path.exists(downloaded_files_path):
        with open(downloaded_files_path, 'r', encoding='utf-8') as file:
            return json.load(file)
    else:
        return []

def save_downloaded_files(downloaded_files):
    with open(downloaded_files_path, 'w', encoding='utf-8') as file:
        json.dump(downloaded_files, file, indent=4)

def get_links(url):
    """Accepts either a single video URL or a playlist URL and returns a link list."""
    if 'list=' in url:
        return get_playlist_links(url)
    return [url]

def get_playlist_links(playlist_url):
    try:
        # client='WEB' enables automatic PoToken generation via bundled nodejs
        playlist = Playlist(playlist_url, client='WEB')
        return list(playlist.video_urls)
    except Exception as e:
        print("Failed to get playlist links:", e)
        return []

FFMPEG_PATH = None

def _find_ffmpeg():
    common = [
        "ffmpeg",
        r"C:\Program Files\KMPlayer 64X\LAVFilters64\ffmpeg.exe",
        r"C:\Program Files\ffmpeg\bin\ffmpeg.exe",
        r"C:\ffmpeg\bin\ffmpeg.exe",
    ]
    for path in common:
        try:
            subprocess.run([path, "-version"], capture_output=True, check=True)
            return path
        except (FileNotFoundError, subprocess.CalledProcessError):
            continue
    return None

def has_ffmpeg():
    global FFMPEG_PATH
    if FFMPEG_PATH is None:
        FFMPEG_PATH = _find_ffmpeg()
    return FFMPEG_PATH is not None

ILLEGAL_FILENAME_CHARS = r'<>:"/\|?*'

def sanitize_filename(name):
    cleaned = ''.join('_' if c in ILLEGAL_FILENAME_CHARS else c for c in name)
    return cleaned.rstrip(' .') or 'video'

def merge_audio_video(video_path, audio_path, output_path):
    print(f"  Merging audio...")
    cmd = [
        FFMPEG_PATH, "-i", video_path, "-i", audio_path,
        "-c:v", "copy", "-c:a", "aac",
        "-map", "0:v:0", "-map", "1:a:0",
        "-y", output_path
    ]
    result = subprocess.run(cmd, capture_output=True, text=True, errors='replace')
    if result.returncode != 0:
        raise RuntimeError(f"ffmpeg failed: {result.stderr.strip()[-500:]}")
    os.remove(video_path)
    os.remove(audio_path)

def download_video(link):
    downloaded_files = load_downloaded_files()
    try:
        video = None
        for attempt in range(3):
            try:
                # client='WEB' auto-generates PoToken via nodejs to bypass
                # "This request was detected as a bot"
                video = YouTube(link, client='WEB')
                break
            except Exception as e:
                print(f"{link} attempt {attempt + 1} failed: {e}")
                if attempt == 2:
                    return
                import time
                time.sleep(5 * (attempt + 1))

        # 1) Try progressive 720p (audio+video combined, no ffmpeg needed)
        stream = video.streams.filter(
            file_extension="mp4", res="720p", progressive=True
        ).first()
        if stream:
            name, ext = os.path.splitext(stream.default_filename)
            outname = f"{sanitize_filename(name)} [720p]{ext}"
            if os.path.exists(outname):
                print(f"{outname} already downloaded, skipping.")
                return
            stream.download(filename=outname)
            print(f"{video.title} (720p progressive)")
            downloaded_files.append(outname)
            save_downloaded_files(downloaded_files)
            return

        # 2) Try best progressive (360p, no ffmpeg needed)
        stream = video.streams.filter(
            file_extension="mp4", progressive=True
        ).first()
        if stream:
            name, ext = os.path.splitext(stream.default_filename)
            outname = f"{sanitize_filename(name)} [360p]{ext}"
            if os.path.exists(outname):
                print(f"{outname} already downloaded, skipping.")
                return
            stream.download(filename=outname)
            print(f"{video.title} (360p progressive)")
            downloaded_files.append(outname)
            save_downloaded_files(downloaded_files)
            return

        # 3) Try adaptive 720p + audio merge (requires ffmpeg)
        if has_ffmpeg():
            vstream = video.streams.filter(
                file_extension="mp4", res="720p"
            ).first()
            astream = video.streams.get_audio_only()

            if vstream and astream:
                name, ext = os.path.splitext(vstream.default_filename)
                name = sanitize_filename(name)
                outname = f"{name} [720p]{ext}"
                vtemp = f"_v_{name}{ext}"
                atemp = f"_a_{name}{ext}"

                if os.path.exists(outname):
                    print(f"{outname} already downloaded, skipping.")
                    return

                vstream.download(filename=vtemp)
                astream.download(filename=atemp)
                merge_audio_video(vtemp, atemp, outname)
                print(f"{video.title} (720p adaptive + ffmpeg)")
                downloaded_files.append(outname)
                save_downloaded_files(downloaded_files)
                return

        # 4) Fallback: any adaptive + audio
        if has_ffmpeg():
            vstream = video.streams.filter(
                file_extension="mp4"
            ).first()
            astream = video.streams.get_audio_only()
            if vstream and astream:
                name, ext = os.path.splitext(vstream.default_filename)
                name = sanitize_filename(name)
                res = vstream.resolution or "unknown"
                outname = f"{name} [{res}]{ext}"
                vtemp = f"_v_{name}{ext}"
                atemp = f"_a_{name}{ext}"

                if os.path.exists(outname):
                    print(f"{outname} already downloaded, skipping.")
                    return

                vstream.download(filename=vtemp)
                astream.download(filename=atemp)
                merge_audio_video(vtemp, atemp, outname)
                print(f"{video.title} ({res} adaptive + ffmpeg)")
                downloaded_files.append(outname)
                save_downloaded_files(downloaded_files)
                return

        print(f"{link}: No suitable streams found")

    except Exception as e:
        print(f"{link} download failed:", e)
        for leftover in ('_v_', '_a_'):
            for f in os.listdir('.'):
                if f.startswith(leftover):
                    try:
                        os.remove(f)
                    except OSError:
                        pass

SINGLE_VIDEO_URL = "https://www.youtube.com/watch?v=9kM3mnxrMMM"
PLAYLIST_URL = "https://www.youtube.com/playlist?list=PL6rx9p3tbsMt8YAmrZwHrabSAcyQR9ad2"

if __name__ == "__main__":
    # Select which source to download: 'single' or 'playlist'
    mode = input("Select source [single/playlist]: ").strip().lower()

    if mode == 'single':
        source_url = SINGLE_VIDEO_URL
    elif mode == 'playlist':
        source_url = PLAYLIST_URL
    else:
        print("Invalid selection. Exiting.")
        exit()

    print(f"Source: {source_url}")
    links = get_links(source_url)
    if not links:
        print("No links found. Exiting.")
    else:
        # Fewer concurrent workers reduce bot-detection risk
        with Pool(processes=2) as pool:
            pool.map(download_video, links)
```

---

## 6. 환경 요약

| 항목 | 값 |
|---|---|
| Python | 3.12.7 (Anaconda, `C:\ProgramData\anaconda3`) |
| pytubefix | 11.2.0 (10.11.0에서 업그레이드 필수) |
| Node.js | `C:\Program Files\nodejs\node.exe` (PoToken 생성용) |
| ffmpeg | `C:\Program Files\KMPlayer 64X\LAVFilters64\ffmpeg.exe` |

---

## 7. 트러블슈팅 요약

| 증상 | 원인 | 해결 |
|---|---|---|
| `Failed to get playlist links: 'list'` / `No links found.` | 단일 영상 URL을 `Playlist()`로 파싱 | `get_links()` 추가, `list=` 여부로 분기 |
| `PoToken INVALID` / `SABR Maximum reload attempts reached` | pytubefix 10.11.0의 구버전 `botGuard.js` | `pip install --upgrade pytubefix` → 11.2.0 |
| ffmpeg `returned non-zero exit status 1` (원인 메시지 없음) | 제목의 `?`가 Windows에서 제거돼 파일명 불일치 | `sanitize_filename()` 적용 + stderr 출력 + 임시 파일 정리 |
| `AttributeError: 'Stream' object has no attribute 'progressive'` | pytubefix 11.x 속성명 변경 | `is_progressive` 사용 (스크립트의 `filter(progressive=True)`는 정상 동작) |
| `AttributeError: 'Stream' object has no attribute 'file_extension'` | pytubefix 11.x 속성명 변경 | `filter(file_extension=...)`는 내부적으로 처리되어 정상 동작 |

주의: pytubefix 11.x에서는 이 영상처럼 progressive 스트림이 하나도 없는 경우가
많습니다. 이때는 step 1·2가 항상 실패하고 step 3(ffmpeg 병합) 또는 step 4로
넘어가는 것이 정상 동작입니다.

---

## 8. 사용법

```powershell
cd <폴더 경로>
python download3.py
```

```
Select source [single/playlist]: single      # 또는 playlist
```

- `single` — `SINGLE_VIDEO_URL` 1개 다운로드
- `playlist` — `PLAYLIST_URL` 안의 24개 다운로드 (worker 2개 병렬)
- URL을 바꾸려면 상위 스크립트의 `SINGLE_VIDEO_URL` / `PLAYLIST_URL` 수정
- `downloaded_files.json`에 다운로드 이력이 쌓이며, 이름이 같은 파일은 건너뜁니다