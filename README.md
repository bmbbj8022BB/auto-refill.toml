# xml2toml.py

shadPS4용 XML 패치(`Bloodborne.xml`)를 bbhost용 TOML 패치로 바꿔 주는 스크립트입니다.

- 패치 여러 개가 든 XML에서 번호로 골라 변환합니다.
- XML에 없는 `expect`(패치 전 원래 바이트)와 `eboot` 해시는 eboot 파일을 읽어서 채웁니다.
- 파일은 컴퓨터 밖으로 나가지 않고, 90MB가 넘는 eboot도 필요한 부분만 읽습니다.

## 준비물

- Python 3 (`python --version`으로 확인)
- `Bloodborne.xml` (shadPS4 패치 목록, 1.09용)
- `eboot.elf` 또는 `eboot.bin` (**복호화된 ELF**여야 합니다. 암호화된 SELF는 거부합니다.)

세 파일(`xml2toml.py`, `Bloodborne.xml`, eboot)을 한 폴더에 둡니다.

## 가장 쉬운 사용법: 더블클릭

`xml2toml.py`를 더블클릭하거나 명령 프롬프트에서 인자 없이 실행합니다.

```
python xml2toml.py
```

1. 같은 폴더에서 XML과 eboot를 자동으로 찾습니다. eboot는 `eboot.elf`를 먼저 찾고, 없으면 `eboot.bin`을 씁니다.
2. 패치 목록이 번호, 기본 켜짐/꺼짐, 줄 수, 이름 순으로 나옵니다.
3. `번호>`에 입력합니다.
   - `45` : 한 개
   - `3,5` : 3번과 5번
   - `3-5` : 3번부터 5번까지
   - `all` : 전부
   - `q` : 종료
4. 선택한 패치가 3개 이하면 `option 키`를 물어봅니다. **보통은 비워 두세요.** 비우면 `option` 줄이 들어가지 않습니다. (아래 "option 줄" 참고)
5. 같은 폴더의 `toml_out` 폴더에 `45-patch.toml` 같은 파일이 저장되고, 3개 이하면 윈도우 메모장이 자동으로 열립니다.
6. Enter를 누르면 목록으로 돌아가 이어서 변환할 수 있습니다.

오류가 나면 창이 바로 닫히지 않고 Enter를 기다립니다.

## 명령줄 사용법

```
python xml2toml.py Bloodborne.xml --list
python xml2toml.py Bloodborne.xml eboot.elf --select 45 --option qol.restock --out restock.toml
python xml2toml.py Bloodborne.xml eboot.elf --find 자동보충 --out restock.toml
python xml2toml.py Bloodborne.xml eboot.elf --all --outdir toml_out
python xml2toml.py Bloodborne.xml eboot.elf --segments
```

| 옵션 | 설명 |
|---|---|
| `--list` | 패치 목록만 보기 (eboot 필요 없음) |
| `--select N` | 목록의 N번 패치를 변환 |
| `--find 글자` | 이름에 이 글자가 든 패치 하나를 변환 (딱 하나만 맞아야 함) |
| `--all` | 전부 변환해서 `--outdir` 폴더에 저장 |
| `--option 키` | bbhost 설정 키에 연결할 때만 사용. 안 주면 `option` 줄을 넣지 않음 |
| `--name 이름` | `[patch] name`. 안 주면 영문 이름, 한글뿐이면 `patch-번호` |
| `--description 글` | `[patch] description`. 안 주면 XML의 Name |
| `--out 파일` | 결과를 UTF-8 파일로 저장 |
| `--outdir 폴더` | `--all` 결과 폴더 (기본 `toml_out`) |
| `--base 값` | 주소에 더할 값을 직접 지정 (기본: 자동 보정) |
| `--segments` | eboot의 ELF 세그먼트 표만 출력 (진단용) |
| `--interactive` | 대화형 모드를 강제로 실행 |

PowerShell에서 `> 파일.toml`로 저장하면 UTF-16이 되어 한글 주석이 깨집니다. 꼭 `--out`을 쓰세요.

## 변환 규칙

기준이 된 bbhost TOML(`skip-intro`, `debug-camera`)과 같은 방식입니다.

1. XML의 `<Line>`을 주소순으로 모아, 주소가 이어지는 바이트를 하나의 구간으로 합칩니다.
2. 각 구간을 **64바이트씩** 잘라 site 하나로 만듭니다. 이름은 `code-00`, `code-01`, ...입니다.
3. `bytes`는 XML의 `Value`를 두 글자씩 끊어 소문자로 씁니다. `bytes16/32/64`는 리틀엔디언으로 바꿉니다. (`bytes`, `bytes8/16/32/64`만 지원하고, 다른 Type은 오류로 알려 줍니다.)
4. `expect`는 eboot에서 같은 주소의 바이트를 읽어 `bytes`와 같은 길이로 씁니다.
5. `eboot` 해시는 eboot 파일의 SHA-256입니다.

## 주소 보정

패치 주소는 파일 위치가 아니라 게임이 메모리에 올라간 주소입니다. 그래서 스크립트는 1.09 eboot에서 값이 알려진 4곳(`0x0202ee1c`는 `75 73`, 로고 3곳은 `4c 00 6f 00`)을 읽어 보고, 맞는 보정값을 자동으로 고릅니다.

실행하면 이런 줄이 나옵니다.

- `주소 보정값: 0x... (알려진 위치 4/4곳 확인)` : 정상입니다.
- `ELF 헤더로는 주소가 안 맞아서, 파일에서 로고 문자열을 찾아 보정했습니다` : 2차 방법으로 보정했다는 뜻입니다. 아래에 `코드 위치 확인: 맞음`이 나와야 합니다.
- `주소 자동 보정 실패` : 1.09가 아닌 eboot이거나, 이미 패치된 파일이거나, 주소 계산이 다른 경우입니다. 아래에 나오는 세그먼트 표를 확인하세요.

맨 위의 `xml2toml v5 ...` 줄로 어떤 버전의 스크립트를 실행 중인지 알 수 있습니다.

## 결과 TOML에서 직접 고칠 곳 (모두 선택 사항)

- **`option`** : 기본으로는 들어가 있지 않습니다. 설정 파일 옵션에 연결하고 싶을 때만 직접 추가하세요. (아래 "option 줄" 참고)
- **`name`과 site 이름** : 번호 이름 대신 `skip-intro`, `logo-fromsoft`처럼 바꿀 수 있습니다. 한 패치 안에서 겹치면 안 되고, 영문, 숫자, `-`, `_`만 쓰세요.
- **맨 위 설명 주석** : XML의 Note를 옮기기만 합니다. 자세한 설명은 직접 쓰세요.
- 파일 맨 위의 "Off by default."는 XML의 `isEnabled`와 상관없이 항상 들어갑니다.

`address`, `expect`, `bytes`는 바꾸지 마세요.

## option 줄

TOML의 `option = "..."`은 **그 패치를 bbhost 설정 파일의 옵션에 연결**합니다. (예: `startup.skip_intro`는 `bbhost.toml`의 `[startup]` 아래 `skip_intro = true`로 켭니다.)

- `option`이 **있는** 패치: 패치 목록에서 `Follows 옵션이름, ...`이라고 나오고, 체크박스를 직접 누를 수 없습니다. 설정 파일의 값을 따라갑니다.
- `option`이 **없는** 패치: 체크박스로 직접 켜고 끕니다. (예: 원래 들어 있던 `deflate-level`, `aerial-vectors-480`)

그래서 스크립트는 기본으로 `option` 줄을 넣지 않습니다. `option = "Auto-Refill"`처럼 임의의 이름을 넣으면 체크박스가 비활성이 됩니다. 이미 넣었다면 그 줄을 지우세요.

(이 동작은 bbhost 화면에 나온 문구를 보고 정리한 것이고, bbhost 코드를 확인한 것은 아닙니다.)

## 확인된 것과 확인되지 않은 것

확인됨:
- 스킵 인트로(1번)와 디버그카메라(34번)를 실제 eboot에서 변환한 결과가 기준 TOML과 일치합니다. 디버그카메라는 15개 site의 주소, `expect`, `bytes`가 전부 같았습니다.
- 해시는 `941f887a562aae054fac35af8cc8f27cf075f3d4cc2e029fb5ae2a663aaa5ae7`로 같게 나왔습니다.

확인되지 않음:
- 나머지 패치의 `expect` 값. 같은 계산이니 맞을 가능성이 높지만, 기준 TOML이 있는 패치만 대조해 봤습니다. 데이터 영역(`0x04d9...` 이후)은 스킵 인트로 3곳만 확인됐습니다.
- bbhost가 site 이름을 어떻게 쓰는지. 기준 파일에도 이름이 제각각이라 라벨일 가능성이 높지만, bbhost 코드를 보지 못했습니다.
- 윈도우에서 메모장이 자동으로 열리는지. 리눅스에서만 시험했습니다.

## 문제 해결

| 증상 | 확인할 것 |
|---|---|
| 더블클릭하면 다른 프로그램으로 열림 | `.py` 파일을 우클릭해 연결 프로그램을 Python으로 지정 |
| 까만 창이 바로 닫힘 | 명령 프롬프트에서 `python xml2toml.py`로 실행해 메시지 확인 |
| `이 파일은 복호화되지 않은 SELF입니다` | 복호화된 ELF eboot을 사용 |
| `알려진 1.09 해시와 다릅니다` | 다른 버전/파일일 수 있음. `eboot.elf`와 `eboot.bin`을 바꿔서 시도 |
| `원래 값이 바꿀 값과 같습니다` | 이미 패치된 eboot일 수 있음 |
| bbhost 목록에서 체크박스가 비활성이고 `Follows ...`라고 나옴 | TOML의 `option = ...` 줄을 삭제 |
| 결과가 예전과 똑같음 | `toml_out`의 예전 파일을 보고 있거나 옛 스크립트를 실행 중. 버전 줄 확인 |
| 한글이 깨짐 | `--out`으로 저장 (PowerShell의 `>`는 UTF-16) |

## 한계

- 1.09 eboot 기준으로 만들어져 있습니다. 주소 자동 보정의 기준값이 1.09용입니다.
- 암호화된 SELF는 읽지 못합니다.
- XML Type은 `bytes`, `bytes8/16/32/64`만 지원합니다. (현재 `Bloodborne.xml`에는 `bytes`, `bytes16`, `bytes32`만 있습니다.)
