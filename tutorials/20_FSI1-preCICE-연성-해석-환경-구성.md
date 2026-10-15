# 예제 20. preCICE 연성 해석 환경 구성

**참고**:

- 이 예제는 연성 라이브러리 [preCICE](https://precice.org)를 이용하여 OpenFOAM과 다른 해석 프로그램을 연성 해석하기 위한 환경을 구성하고, 환경이 동작하는지 검증한다. 해석 케이스는 다루지 않으며, 여기서 구성한 환경은 예제 21(유체-구조 연성 해석)과 예제 22(열-유체-구조 연성 해석)에서 사용한다.
- 사용하는 외부 소프트웨어의 라이선스는 다음과 같다: preCICE (LGPL 3.0), OpenFOAM-preCICE 어댑터 (GPL 3.0), CalculiX-preCICE 어댑터 (GPL 3.0), CalculiX (GPL 2.0).
- 이 예제는 OpenFOAM 튜토리얼(한국어판)에 새로 추가된 내용으로, [CC BY-NC-SA 3.0](https://creativecommons.org/licenses/by-nc-sa/3.0/) 라이선스 하에 배포된다.
- 이 자료는 ESI Group, ESI-OpenCFD 또는 OpenFOAM Foundation에 의해 승인된 자료가 아니다.

**호환성**:

- 이 예제의 명령어와 설정 파일은 [예제 0](00_Ubuntu-24.04-WSL-및-OpenFOAM-v2412-설치-OpenFOAM-솔버-목록.md)의 방법으로 설치한 OpenFOAM v2412 (WSL2, Ubuntu 24.04, /opt/OpenFOAM-v2412 소스 빌드) 환경을 기준으로 작성되었다.
- 사용한 버전은 preCICE v3.4.1, OpenFOAM-preCICE 어댑터 v1.4.0, CalculiX-preCICE 어댑터 v2.20.2 (CalculiX 2.20)이다.
- preCICE와 CalculiX 어댑터는 OpenFOAM에 포함되지 않은 외부 소프트웨어로, 공식 `.deb` 패키지로 시스템 디렉토리에 설치하므로 설치 과정에 관리자 권한(`sudo`)이 필요하다.
- 소요 시간은 패키지 설치 수 분(인터넷 속도에 따라 다름), 어댑터 빌드 약 1분, 설치 검증 수 분이다. 설치를 되돌리려면 `sudo apt remove libprecice3 calculix-precice3 && sudo apt autoremove`를 실행하고 `$FOAM_USER_LIBBIN/libpreciceAdapterFunctionObject.so` 파일을 지운다.

**선행 학습**:

- [예제 0. Ubuntu 24.04 (WSL) 및 OpenFOAM v2412 설치](00_Ubuntu-24.04-WSL-및-OpenFOAM-v2412-설치-OpenFOAM-솔버-목록.md) : OpenFOAM 소스 빌드와 환경 변수
- [예제 18. 사용자 정의 솔버 만들기](18_사용자-정의-솔버-만들기-전기수력학.md) : wmake와 사용자 라이브러리

---

## 목차

1. 배경
   - 1.1 분할 방식 연성과 preCICE
   - 1.2 어댑터
2. 예제 개요
3. 환경 구성
   - 3.1 구성 요소
   - 3.2 preCICE 및 CalculiX 어댑터 설치
   - 3.3 설치 확인과 라이브러리 충돌 문제
   - 3.4 preCICE 전용 환경 스크립트
   - 3.5 OpenFOAM-preCICE 어댑터 빌드
4. 설치 검증
   - 4.1 어댑터 불러오기 확인
   - 4.2 연성 시험 : solverdummy
   - 4.3 확인 범위와 다시 확인하는 방법
5. 실행 순서 요약
6. 문제 해결
7. 마무리
8. 참고 문헌

## 1. 배경

### 1.1 분할 방식 연성과 preCICE

유체와 구조물, 유체와 고체 사이의 열전달처럼 서로 다른 물리 현상이 경계면에서 영향을 주고받는 문제를 **연성 문제**(coupled problem)라 한다. 연성 문제를 푸는 방법은 크게 두 가지로 나뉜다.

- 모놀리식(monolithic) 방식 : 모든 물리 현상의 방정식을 하나의 프로그램에서 함께 푼다. 연성 문제마다 전용 코드가 필요하다.
- 분할(partitioned) 방식 : 각 물리 현상을 검증된 전문 해석 프로그램으로 따로 풀고, 경계면의 데이터만 주고받는다. 예를 들어 유체는 OpenFOAM으로, 구조물은 유한요소 해석 프로그램으로 푼다.

**preCICE**(Precise Code Interaction Coupling Environment)는 분할 방식 연성을 위한 오픈 소스 라이브러리이다[1]. 연성에 참여하는 해석 프로그램을 **participant**라 하며, preCICE는 participant 사이의 통신, 서로 다른 격자 사이의 데이터 보간, 시간 진행의 조율을 맡는다. 연성 해석을 할 때는 participant마다 별도의 프로그램(프로세스)을 동시에 실행한다. 이 책의 예제 21, 22에서는 두 프로그램을 같은 컴퓨터에서 실행하고 TCP 소켓으로 데이터를 주고받으며, 그 구성은 다음과 같다.

```text
 terminal 1                                 terminal 2
+-----------------------------------+      +-----------------------------------+
| Fluid participant                 |      | Solid participant                 |
|   OpenFOAM solver                 |      |   ccx_preCICE                     |
|   + OpenFOAM-preCICE adapter      |      |   (CalculiX + CalculiX-preCICE    |
|   + preCICE library               |      |    adapter) + preCICE library     |
+-----------------+-----------------+      +-----------------+-----------------+
                  +------ interface data (TCP sockets) ------+
```

두 프로그램이 모두 preCICE 라이브러리를 불러오므로, 3.3절에서 다루는 라이브러리 충돌 문제는 두 프로그램 모두에 해당한다. 연성의 이론(경계면 조건, 시간 창, 암시적 반복, 데이터 보간)과 preCICE 설정 파일은 예제 21에서 다루고, 이 예제에서는 연성 해석을 실행하는 데 필요한 프로그램을 설치하고 확인한다.

### 1.2 어댑터

각 해석 프로그램은 **어댑터**(adapter)를 통해 preCICE에 연결된다. 어댑터는 해석 프로그램의 경계면 데이터를 preCICE에 넘기고, preCICE에서 받은 데이터를 해석 프로그램의 경계 조건으로 넣어 주는 연결 코드이다.

| participant | 어댑터 | 형태 | 설치 방법 |
| --- | --- | --- | --- |
| OpenFOAM | OpenFOAM-preCICE 어댑터[2] | 함수 객체(function object) 라이브러리. 솔버를 수정하지 않고 controlDict에서 불러온다 | 소스 빌드 (3.5절) |
| CalculiX[3] | CalculiX-preCICE 어댑터 | 어댑터를 넣어 다시 컴파일한 CalculiX 실행 파일 `ccx_preCICE` | 공식 `.deb` 패키지 (3.2절) |

OpenFOAM-preCICE 어댑터는 OpenFOAM의 라이브러리와 함께 컴파일되므로, 사용하는 OpenFOAM에 맞게 직접 빌드한다. 반면 preCICE 라이브러리와 `ccx_preCICE`는 OpenFOAM과 관계없는 프로그램이므로 공식 `.deb` 패키지로 설치한다. OpenFOAM-preCICE 어댑터는 유체-구조 연성(`FSI`) 외에 열 연성(`CHT`) 등의 모듈도 제공하므로, 이 예제에서 구성한 환경은 예제 21의 유체-구조 연성 해석과 예제 22의 열-유체-구조 연성 해석에 모두 쓰인다. OpenFOAM v2412에 기본으로 포함되지 않은 이 도구들을 선택한 이유는 예제 21의 1.5절에서 설명한다.

## 2. 예제 개요

### 예제 구성

이 예제는 해석 케이스 없이 다음 순서로 진행한다.

1. preCICE와 CalculiX-preCICE 어댑터를 공식 `.deb` 패키지로 설치한다(3.2절).
2. 예제 0의 OpenFOAM 환경과 preCICE 사이의 라이브러리 충돌을 확인하고, 이를 피하는 환경 스크립트를 만든다(3.3, 3.4절).
3. OpenFOAM-preCICE 어댑터를 소스에서 빌드한다(3.5절).
4. OpenFOAM이 어댑터를 불러올 수 있는지, 두 프로그램이 preCICE로 연결되어 데이터를 주고받는지 확인한다(4장).

예제를 마치면 다음이 준비된다.

```text
~/precice/
├── debs/                      (.deb 패키지, 3.2절)
├── precice-of-env.sh          (환경 스크립트, 3.4절)
├── openfoam-adapter-v1.4.0/   (어댑터 소스, 3.5절)
└── solverdummies/             (연성 시험 프로그램, 4.2절)
$FOAM_USER_LIBBIN/libpreciceAdapterFunctionObject.so   (어댑터 라이브러리, 3.5절)
/usr/bin/ccx_preCICE, /usr/bin/precice-version, ...    (설치된 패키지, 3.2절)
```

### 목표

- preCICE, OpenFOAM-preCICE 어댑터, CalculiX-preCICE 어댑터로 구성된 연성 해석 환경을 직접 구축한다.
- 설치 과정에서 발생하는 라이브러리 충돌 문제의 원인을 이해하고 해결한다.
- 해석 케이스 없이 어댑터 불러오기와 두 프로그램의 연성을 시험하여, 환경이 동작하는지 확인한다.

### 완료 기준

다음 네 가지 확인을 모두 통과하면 이 예제를 마친 것이다.

1. preCICE와 CalculiX 실행(3.4절) : `precice-version`이 `3.4.1;`로 시작하는 줄을, `ccx_preCICE -v`가 `This is Version 2.20`을 출력한다.
2. 어댑터 빌드(3.5절) : `./Allwmake`가 마지막에 `=== OK: Building completed successfully! ===`를 출력한다.
3. 어댑터 불러오기(4.1절) : `foamHasLibrary`가 `Can load`로 시작하는 줄을 출력한다.
4. 두 프로그램의 연성(4.2절) : 터미널 2개에서 실행한 두 solverdummy가 모두 `Reached end at: final time-window: 2, final time: 2`를 출력한다.

## 3. 환경 구성

### 3.1 구성 요소

설치하는 구성 요소는 다음과 같다.

| 구성 요소 | 버전 | 설치 방법 | 설치 위치 |
| --- | --- | --- | --- |
| preCICE 라이브러리 | 3.4.1 | 공식 `.deb` 패키지 (`sudo apt`) | /usr/lib/x86_64-linux-gnu, /usr/bin |
| CalculiX-preCICE 어댑터 (`ccx_preCICE`) | 2.20.2 (CalculiX 2.20) | 공식 `.deb` 패키지 (`sudo apt`) | /usr/bin/ccx_preCICE |
| OpenFOAM-preCICE 어댑터 | 1.4.0 | 소스 빌드 | `$FOAM_USER_LIBBIN` |

설치 파일과 소스 코드는 홈 디렉토리 아래의 ~/precice/ 디렉토리에 모아 둔다.

### 3.2 preCICE 및 CalculiX 어댑터 설치

preCICE와 CalculiX 어댑터는 Ubuntu 24.04(코드명 noble)용 `.deb` 패키지가 공식 배포된다. 작업 디렉토리를 만들고 패키지와 체크섬 파일을 다운로드한다.

```bash
mkdir -p ~/precice/debs && cd ~/precice/debs
```

```bash
URL=https://github.com/precice/precice/releases/download/v3.4.1
wget $URL/libprecice3_3.4.1_noble.deb
wget $URL/libprecice3_3.4.1_noble.deb.sha256
```

```bash
URL=https://github.com/precice/calculix-adapter/releases/download/v2.20.2
wget $URL/calculix-precice3_2.20.2-1_amd64_noble.deb
wget $URL/calculix-precice3_2.20.2-1_amd64_noble.deb.sha256
```

다운로드한 파일이 손상되거나 변조되지 않았는지 체크섬으로 확인한다.

```bash
sha256sum -c *.sha256
```

```text
calculix-precice3_2.20.2-1_amd64_noble.deb: OK
libprecice3_3.4.1_noble.deb: OK
```

두 패키지를 설치한다. `apt`에 로컬 `.deb` 파일의 경로를 주면 필요한 의존 패키지까지 저장소에서 함께 설치한다. OpenFOAM-preCICE 어댑터의 빌드에 필요한 `pkgconf`도 함께 설치한다.

```bash
sudo apt update
```

```bash
sudo apt install -y --no-install-recommends \
    ./libprecice3_3.4.1_noble.deb ./calculix-precice3_2.20.2-1_amd64_noble.deb pkgconf
```

> **Note:** preCICE 패키지는 PETSc(`petsc-dev`)를 필수 의존 패키지로 요구하므로, PETSc와 그에 딸린 수치 라이브러리(HYPRE, MUMPS, ScaLAPACK, HDF5 등) 90여 개의 패키지가 함께 설치된다. 기존 패키지를 업그레이드하거나 제거하지는 않으며, OpenFOAM이 사용하는 OpenMPI(4.1.6)도 그대로 유지된다. 설치 마지막에 `N: Download is performed unsandboxed as root ...`라는 메시지가 출력될 수 있는데, 홈 디렉토리의 로컬 파일을 설치할 때 나오는 안내 메시지이므로 무시해도 된다.

### 3.3 설치 확인과 라이브러리 충돌 문제

설치가 끝나면 preCICE의 버전을 확인해 본다.

```bash
precice-version
```

[예제 0](00_Ubuntu-24.04-WSL-및-OpenFOAM-v2412-설치-OpenFOAM-솔버-목록.md)의 2.4절처럼 ~/.bashrc에서 OpenFOAM 환경을 자동으로 불러오도록 설정했다면, 버전 대신 아래의 오류가 출력된다. `ccx_preCICE -v`나 `precice-config-validate`를 실행해도 같은 오류가 발생한다.

```text
precice-version: symbol lookup error: /lib/x86_64-linux-gnu/libfftw3_mpi.so.3: undefined symbol: fftw_taint
```

설치가 잘못된 것이 아니다. 원인은 OpenFOAM 환경과 시스템 라이브러리 사이의 충돌이다. 원인을 이해하기 위해 먼저 몇 가지 용어를 정리한다.

> **Note:** 리눅스 프로그램은 실행될 때 필요한 기능의 상당 부분을 **공유 라이브러리**(이름이 `.so`로 끝나는 파일, Windows의 `.dll`에 해당)에서 불러온다. 프로그램은 기본적으로 /usr/lib 같은 시스템 디렉토리에서 공유 라이브러리를 찾지만, 환경 변수 `LD_LIBRARY_PATH`에 적힌 디렉토리가 있으면 그곳을 **먼저** 찾는다. **심볼**(symbol)은 라이브러리 안에 들어 있는 함수나 변수의 이름이다. 따라서 `undefined symbol: fftw_taint`는 "불러온 라이브러리에서 `fftw_taint`라는 함수를 찾지 못했다"는 뜻이다.

오류가 발생하는 과정은 다음과 같다.

1. preCICE는 PETSc에 연결되어 있고, PETSc는 시스템의 FFTW 라이브러리(`libfftw3_mpi.so.3`, `libfftw3.so.3`)를 사용한다.
2. 예제 0의 방법으로 OpenFOAM을 소스 빌드하면, ThirdParty 디렉토리에 FFTW 3.3.10이 따로 빌드되고, OpenFOAM 환경 스크립트(/opt/OpenFOAM-v2412/etc/bashrc)가 그 경로를 `LD_LIBRARY_PATH`에 추가한다.
3. `LD_LIBRARY_PATH`의 경로가 먼저 검색되므로, 프로그램이 실행될 때 시스템의 `libfftw3.so.3` 대신 ThirdParty의 `libfftw3.so.3`이 로드된다.
4. 시스템의 `libfftw3_mpi.so.3`은 `libfftw3.so.3`의 내부 함수(`fftw_taint`, `fftw_join_taint`)를 필요로 하는데, ThirdParty 빌드는 이 함수들을 외부로 공개하지 않는다. 이 라이브러리는 실행 시작 시점에 필요한 함수를 모두 찾도록 빌드되어 있으므로, 프로그램이 시작하자마자 종료된다.

`LD_LIBRARY_PATH`에 ThirdParty FFTW 경로가 들어 있는지 직접 확인한다.

```bash
echo $LD_LIBRARY_PATH | tr ':' '\n' | grep fftw
```

```text
/opt/ThirdParty-v2412/platforms/linux64Gcc/fftw-3.3.10/lib
```

> **Note:** 이 오류는 OpenFOAM 환경을 불러오지 않은 터미널에서는 발생하지 않는다. 즉 preCICE 자체는 정상적으로 설치되었다. 그러나 1.2절과 같이 어댑터가 OpenFOAM 솔버 안에서 preCICE를 불러오므로, OpenFOAM과 preCICE는 같은 프로세스 안에서 함께 실행된다. 따라서 OpenFOAM 환경을 유지한 채로 이 충돌을 해결해야 한다.

### 3.4 preCICE 전용 환경 스크립트

해결 방법은 OpenFOAM 환경을 불러온 뒤 `LD_LIBRARY_PATH`에서 **ThirdParty FFTW 경로 하나만** 제거하는 것이다. 그러면 시스템의 FFTW가 로드된다. 시스템 FFTW(Ubuntu 24.04 패키지)와 ThirdParty FFTW는 둘 다 버전 3.3.10으로, 함수 이름과 호출 방식(ABI, Application Binary Interface)이 같아 서로 바꿔 사용할 수 있다. 실제로 FFTW를 사용하는 OpenFOAM 유틸리티 `boxTurb`와 솔버 `dnsFoam`의 결과가 두 FFTW에서 완전히 같음을 확인하였다.

OpenFOAM 설치 파일을 고치는 대신, preCICE 해석을 할 때만 사용하는 환경 스크립트를 따로 만든다. 아래 명령 전체(`cat`으로 시작하는 줄부터 `EOF` 줄까지)를 복사하여 터미널에 붙여넣으면 ~/precice/precice-of-env.sh 파일이 만들어진다.

```bash
cat > ~/precice/precice-of-env.sh << 'EOF'
# ~/precice/precice-of-env.sh : load OpenFOAM v2412 for preCICE runs.
# The ThirdParty libfftw3.so.3 in LD_LIBRARY_PATH lacks fftw_taint/fftw_join_taint,
# which the system libfftw3_mpi (preCICE -> PETSc) needs. Remove only that directory;
# the system FFTW (same version 3.3.10) is then used instead.
source /opt/OpenFOAM-v2412/etc/bashrc
LD_LIBRARY_PATH=$(echo "$LD_LIBRARY_PATH" | tr ':' '\n' \
    | grep -v '/ThirdParty-v2412/platforms/[^/]*/fftw-' | paste -sd:)
export LD_LIBRARY_PATH
EOF
```

이후 preCICE를 사용하는 모든 터미널에서는 해석 전에 이 스크립트를 먼저 불러온다. 이 파일은 `./`나 `bash`로 실행하면 현재 터미널에 적용되지 않으므로 반드시 `source`로 불러온다.

```bash
source ~/precice/precice-of-env.sh
```

다시 설치를 확인한다.

```bash
precice-version
```

```text
3.4.1;no-info [git failed to run];...
```

```bash
ccx_preCICE -v
```

```text
This is Version 2.20
```

preCICE 3.4.1과 CalculiX 2.20이 정상적으로 실행되면 설치가 완료된 것이다. 첫 번째 출력의 `no-info [git failed to run]`은 패키지에 빌드 정보가 들어 있지 않다는 뜻으로, 오류가 아니다.

> **Note:** 이 스크립트는 OpenFOAM 환경을 다시 불러온 뒤 경로를 고치므로, OpenFOAM이 이미 로드된 터미널에서 여러 번 불러와도 문제가 없다. 반대로 스크립트를 불러온 뒤에 `source /opt/OpenFOAM-v2412/etc/bashrc`를 다시 실행하면 ThirdParty FFTW 경로가 되살아나므로 주의한다. 매번 입력하기 번거롭다면 ~/.bashrc에 `alias ofp='source ~/precice/precice-of-env.sh'`와 같은 alias를 추가해 두면 편리하다.

> **Note:** `ccx_preCICE -v`는 버전을 정상적으로 출력한 뒤에도 0이 아닌 종료 코드를 반환한다. 스크립트에서 환경이 올바른지 자동으로 검사하려면 종료 코드가 0인 `precice-version`을 사용한다(예제 21의 4.6절 (B)에서 만드는 runFSI.sh).

### 3.5 OpenFOAM-preCICE 어댑터 빌드

OpenFOAM-preCICE 어댑터는 소스 코드를 받아 현재 OpenFOAM 버전에 맞게 빌드한다. 빌드는 [예제 18](18_사용자-정의-솔버-만들기-전기수력학.md)의 2.3절에서 다룬 wmake로 수행되며, 반드시 3.4절의 환경 스크립트를 불러온 터미널에서 실행한다.

```bash
source ~/precice/precice-of-env.sh
```

```bash
cd ~/precice
```

```bash
git clone --depth 1 --branch v1.4.0 \
    https://github.com/precice/openfoam-adapter.git openfoam-adapter-v1.4.0
```

```bash
cd ~/precice/openfoam-adapter-v1.4.0 && ./Allwmake
```

1분 정도 지나면 아래와 같이 빌드 결과가 출력된다.

```text
Building the OpenFOAM-preCICE adapter...
...
Current OpenFOAM environment:
  WM_PROJECT = OpenFOAM
  WM_PROJECT_VERSION = v2412
...
The adapter will be built into .../platforms/linux64GccDPInt32Opt/lib
...
Everything looks fine in wmake.log.
Everything looks fine in ldd.log.
=== OK: Building completed successfully! ===
```

어댑터 라이브러리 libpreciceAdapterFunctionObject.so가 `$FOAM_USER_LIBBIN`에 생성되었는지 확인한다.

```bash
ls $FOAM_USER_LIBBIN/libpreciceAdapterFunctionObject.so
```

> **Note:** 환경 스크립트 없이 일반 OpenFOAM 환경에서 빌드해도 라이브러리 파일은 만들어지지만, 마지막에 `=== ERROR: Building completed with linking problems: there were undefined symbols. ===` 메시지가 출력된다. ldd.log를 열어 보면 `undefined symbol: fftw_taint`로, 3.3절과 같은 원인이다. 이 경우에도 실행할 때 환경 스크립트를 사용하면 정상 동작하지만, 처음부터 환경 스크립트를 불러온 뒤 빌드하는 것이 깔끔하다.

> **Note:** 어댑터는 사용자 라이브러리 디렉토리(`$FOAM_USER_LIBBIN`)에 설치되므로 관리자 권한이 필요 없고, OpenFOAM 설치(/opt/OpenFOAM-v2412)를 변경하지 않는다. OpenFOAM을 다시 빌드하거나 버전을 바꾸면 어댑터도 다시 빌드해야 한다. 설치가 기존 OpenFOAM에 영향을 주지 않았는지 확인하려면, 환경 스크립트를 불러오지 않은 **새 터미널**에서 [예제 0](00_Ubuntu-24.04-WSL-및-OpenFOAM-v2412-설치-OpenFOAM-솔버-목록.md)의 2.5절처럼 `simpleFoam -help`를 실행하고, 이전 예제의 케이스(예: 예제 1의 elbow)를 한 번 다시 실행해 본다. 결과는 이전과 같아야 한다.

## 4. 설치 검증

3장의 버전 확인은 각 프로그램이 실행되는지만 보여 준다. 연성 해석에서는 이에 더하여 OpenFOAM 솔버가 실행 중에 어댑터 라이브러리를 불러와야 하고, 두 프로그램이 preCICE로 연결되어 데이터를 주고받아야 한다. 이 장에서는 이 두 가지를 해석 케이스 없이 확인한다. 이 장의 명령은 모두 환경 스크립트를 불러온 터미널에서 실행한다.

### 4.1 어댑터 불러오기 확인

OpenFOAM 솔버는 controlDict의 `libs` 항목에 적힌 라이브러리를 실행을 시작할 때 불러온다. OpenFOAM 유틸리티 `foamHasLibrary`는 같은 방법으로 라이브러리를 불러 보고, 그 결과를 출력한다. `-verbose`를 주면 불러오기에 성공했을 때도 결과를 출력한다(주지 않으면 실패했을 때만 출력한다).

```bash
source ~/precice/precice-of-env.sh
```

```bash
foamHasLibrary -verbose libpreciceAdapterFunctionObject.so
```

```text
Can load "libpreciceAdapterFunctionObject.so"
```

환경 스크립트를 불러오지 않은 터미널(예제 0의 일반 OpenFOAM 환경)에서 같은 명령을 실행하면 다음과 같이 실패한다. 3.3절과 같은 원인이며, 연성 해석에서 유체 솔버가 어댑터를 불러오지 못하는 것도 이 상태이다. 새 터미널을 열어 직접 확인해 볼 수 있다.

```text
Could not load "libpreciceAdapterFunctionObject.so"
/lib/x86_64-linux-gnu/libfftw3_mpi.so.3: undefined symbol: fftw_taint
```

### 4.2 연성 시험 : solverdummy

preCICE 패키지에는 연성 기능만 시험하는 작은 C++ 프로그램 **solverdummy**의 소스 코드가 들어 있다(/usr/share/precice/examples/solverdummies/). solverdummy는 실제 해석을 하지 않고, 점 3개로 된 mesh에서 임의의 값을 주고받는다. 같은 프로그램을 participant 이름만 바꾸어(`SolverOne`, `SolverTwo`) 두 번 실행하면, preCICE가 두 프로그램을 연결하여 2개의 시간 창 동안 데이터를 교환한다. **시간 창**(time window)은 두 프로그램이 경계면 데이터를 주고받으며 함께 진행하는 시간 구간으로, 자세한 의미는 예제 21의 1.3절에서 다룬다. 연성 해석과 같이 두 프로그램을 동시에 실행하는 절차를 미리 연습하는 의미도 있다.

소스 디렉토리는 관리자만 쓸 수 있으므로 ~/precice/로 복사한 뒤, 예제 0에서 설치한 `cmake`로 빌드한다. 빌드도 환경 스크립트를 불러온 터미널에서 한다.

```bash
cp -r /usr/share/precice/examples/solverdummies ~/precice/
```

```bash
cd ~/precice/solverdummies/cpp && cmake . && make
```

```text
...
-- Build files have been written to: .../precice/solverdummies/cpp
[ 50%] Building CXX object CMakeFiles/solverdummy.dir/solverdummy.cpp.o
[100%] Linking CXX executable solverdummy
[100%] Built target solverdummy
```

Ubuntu 터미널을 2개 연다(Windows Terminal에서는 새 탭(+) 버튼 옆의 아래쪽 화살표 버튼을 눌러 Ubuntu-24.04를 선택하거나, 시작 메뉴에서 Ubuntu를 한 번 더 실행한다). 첫 번째 터미널에서 `SolverOne`을 실행한다(빌드한 터미널을 그대로 사용해도 된다).

```bash
source ~/precice/precice-of-env.sh
cd ~/precice/solverdummies/cpp
./solverdummy ../precice-config.xml SolverOne
```

출력이 `Setting up primary communication to coupling partner/s` 줄에서 멈추고 상대방을 기다린다. 두 번째 터미널에서 `SolverTwo`를 실행한다.

```bash
source ~/precice/precice-of-env.sh
cd ~/precice/solverdummies/cpp
./solverdummy ../precice-config.xml SolverTwo
```

1초 이내에 두 터미널 모두 다음과 같이 끝나고 프롬프트로 돌아온다(첫 번째 터미널의 마지막 부분).

```text
preCICE: it 2 (min: 2, max: 2), time-window 2 (max: 2), t 1, Dt 1, max-dt 1
DUMMY: Reading iteration checkpoint
preCICE: Time window completed
preCICE: Reached end at: final time-window: 2, final time: 2
DUMMY: Advancing in time
preCICE: Close communication channels
DUMMY: Closing C++ solver dummy...
```

`Reached end at: final time-window: 2, final time: 2`가 출력되면 두 프로그램이 preCICE로 연결되어 마지막 시간 창까지 데이터를 주고받은 것이다. 출력 중 `it 1`, `it 2`는 한 시간 창 안의 반복 번호이고, `Writing iteration checkpoint`와 `Reading iteration checkpoint`는 반복을 위해 시간 창 시작 시점의 상태를 저장하고 되돌리는 동작이다. 이 시험의 설정 파일(solverdummies/precice-config.xml)은 시간 창마다 정확히 2번 반복하도록 되어 있으며, 반복과 체크포인트의 의미는 예제 21의 1.3절에서 설명한다.

> **Note:** 출력의 앞부분에 `No convergence measures were defined for an implicit coupling scheme. It will always iterate the maximum amount iterations, which is 2.`라는 안내가 나온다. 이 시험의 설정 파일에 수렴 기준이 없어 항상 최대 반복 횟수(2회)만큼 반복한다는 뜻으로, 오류가 아니다.

> **Note:** 두 프로그램은 같은 디렉토리에서 실행해야 한다. preCICE는 연결 정보를 주고받는 디렉토리 아래에 precice-run/ 디렉토리를 만드는데, 이 시험의 설정 파일은 그 디렉토리(`exchange-directory`)를 지정하지 않았으므로 각 프로그램의 실행 디렉토리가 사용된다. 따라서 서로 다른 디렉토리에서 실행하면 두 프로그램이 상대방을 찾지 못하고 계속 기다린다. 예제 21에서는 이 디렉토리를 participant 디렉토리의 상위 디렉토리로 지정하여, 두 participant를 서로 다른 디렉토리에서 실행한다.

> **Note:** 두 터미널 **모두**에서 환경 스크립트를 불러와야 한다. 한쪽만 빠뜨리면 그쪽은 `./solverdummy: symbol lookup error: ... undefined symbol: fftw_taint`를 출력하고 바로 종료되며, 다른 한쪽은 계속 기다린다. 기다리는 프로그램은 `Ctrl+C`로 종료하고, 다시 실행하기 전에 ~/precice/solverdummies/cpp에서 `rm -rf precice-run`으로 남은 연결 정보를 지운다. 지우지 않으면 다음 실행에서 한쪽 프로그램이 `Unable to establish connection as a connection file already exists` 오류와 긴 역추적(backtrace)을 출력하며 비정상 종료하고, 다른 쪽은 계속 기다린다.

> **Note:** solverdummies/cpp/README.md에 적힌 `ctest -V`는 이 확인에 사용하지 않는다. 이 버전의 시험 스크립트(test.cmake)는 solverdummy에 인자를 잘못 넘겨 두 프로그램이 연성 없이 바로 종료되는데도, 결과와 관계없이 `Success!`와 `Passed`를 출력한다.

### 4.3 확인 범위와 다시 확인하는 방법

4.1, 4.2절의 시험으로 preCICE 라이브러리, 환경 스크립트, 소켓 통신, 어댑터 라이브러리의 불러오기가 확인되었다. 실제 연성 해석에 쓰이는 나머지 기능(OpenFOAM-preCICE 어댑터의 `FSI` 모듈과 이동 격자, `ccx_preCICE`의 연성, RBF 데이터 보간(solverdummy는 가장 가까운 점의 값을 옮기는 nearest-neighbor 보간만 사용한다), 암시적 반복의 가속)은 예제 21에서 preCICE 공식 튜토리얼 케이스를 실행하고, 그 결과를 preCICE가 공개한 참조 결과와 비교하여 최종 확인한다(예제 21의 5.3절). 병렬 실행은 예제 21의 4.8절에서 직렬 실행의 결과와 비교한다. 설치가 기존 OpenFOAM에 영향을 주지 않았는지는 3.5절의 마지막 Note와 같이 확인한다.

예제 21, 22를 나중에 다시 시작할 때는 3장을 반복할 필요 없이, 새 터미널에서 환경 스크립트를 불러온 뒤 예제 21의 3장과 같이 `precice-version`, `foamHasLibrary`(4.1절), `ccx_preCICE -v`의 출력만 다시 확인하면 된다. preCICE가 이미 설치된 다른 컴퓨터를 사용할 때는 다음 조건을 확인하고, 빠진 것만 3장의 해당 절을 따라 준비한 뒤 4.1, 4.2절의 확인을 한다.

- preCICE v3 (설정 파일의 문법이 v2와 다르다)
- 현재 사용하는 OpenFOAM으로 빌드한 OpenFOAM-preCICE 어댑터 (OpenFOAM을 다시 빌드했다면 3.5절의 빌드를 다시 한다)
- `ccx_preCICE` (CalculiX-preCICE 어댑터)
- 환경 스크립트 ~/precice/precice-of-env.sh. 예제 21, 22의 명령과 스크립트가 이 경로를 사용한다. OpenFOAM을 /opt/OpenFOAM-v2412가 아닌 곳에 설치했다면 스크립트의 `source` 줄의 경로를 바꾼다.

## 5. 실행 순서 요약

이 예제의 명령어를 한 번에 정리한 치트시트이다. 각 줄의 의미는 이전 절들에서 설명하였다.

```bash
# 1) install preCICE and the CalculiX adapter (only once, needs sudo)
mkdir -p ~/precice/debs && cd ~/precice/debs
URL=https://github.com/precice/precice/releases/download/v3.4.1
wget $URL/libprecice3_3.4.1_noble.deb $URL/libprecice3_3.4.1_noble.deb.sha256
URL=https://github.com/precice/calculix-adapter/releases/download/v2.20.2
wget $URL/calculix-precice3_2.20.2-1_amd64_noble.deb
wget $URL/calculix-precice3_2.20.2-1_amd64_noble.deb.sha256
sha256sum -c *.sha256
sudo apt update
sudo apt install -y --no-install-recommends \
    ./libprecice3_3.4.1_noble.deb ./calculix-precice3_2.20.2-1_amd64_noble.deb pkgconf

# 2) create the environment script (section 3.4) and load it in EVERY preCICE terminal
cat > ~/precice/precice-of-env.sh << 'EOF'
source /opt/OpenFOAM-v2412/etc/bashrc
LD_LIBRARY_PATH=$(echo "$LD_LIBRARY_PATH" | tr ':' '\n' \
    | grep -v '/ThirdParty-v2412/platforms/[^/]*/fftw-' | paste -sd:)
export LD_LIBRARY_PATH
EOF
source ~/precice/precice-of-env.sh
precice-version && ccx_preCICE -v

# 3) build the OpenFOAM-preCICE adapter (only once per OpenFOAM build)
cd ~/precice
git clone --depth 1 --branch v1.4.0 \
    https://github.com/precice/openfoam-adapter.git openfoam-adapter-v1.4.0
cd openfoam-adapter-v1.4.0 && ./Allwmake

# 4) check that the adapter can be loaded (section 4.1)
foamHasLibrary -verbose libpreciceAdapterFunctionObject.so

# 5) coupling test (section 4.2): build once, then run SolverOne in this terminal
cp -r /usr/share/precice/examples/solverdummies ~/precice/
cd ~/precice/solverdummies/cpp && cmake . && make
./solverdummy ../precice-config.xml SolverOne

# 5) ... and SolverTwo in a second terminal
source ~/precice/precice-of-env.sh
cd ~/precice/solverdummies/cpp
./solverdummy ../precice-config.xml SolverTwo
```

## 6. 문제 해결

| 증상 | 원인 | 해결 |
| --- | --- | --- |
| `symbol lookup error: ... libfftw3_mpi.so.3: undefined symbol: fftw_taint` | 일반 OpenFOAM 환경에서 preCICE를 사용하는 프로그램(`precice-version`, `ccx_preCICE`, `solverdummy` 등)을 실행함 (3.3절) | `source ~/precice/precice-of-env.sh` 후 다시 실행 |
| `foamHasLibrary`가 `Could not load "libpreciceAdapterFunctionObject.so"`를 출력 | 다음 줄이 `undefined symbol: fftw_taint`이면 위와 같은 원인. `cannot open shared object file`이면 어댑터를 빌드하지 않았거나 다른 OpenFOAM 환경에서 빌드함 | 환경 스크립트를 불러온 뒤 다시 확인. `ls $FOAM_USER_LIBBIN/libpreciceAdapterFunctionObject.so`로 어댑터 확인, 없으면 3.5절의 빌드 |
| 어댑터 빌드 끝에 `linking problems: there were undefined symbols` | 일반 OpenFOAM 환경에서 빌드함 (3.5절 Note) | 환경 스크립트를 불러온 뒤 `./Allwmake` 다시 실행 |
| 어댑터 빌드 중 preCICE를 찾지 못함 (`precice/precice.hpp: No such file or directory` 등) | preCICE 또는 `pkgconf` 미설치 | 3.2절의 `apt install` 확인 |
| `ccx_preCICE: command not found` | CalculiX 어댑터 패키지 미설치 | 3.2절의 `apt install` 확인 |
| solverdummy 빌드(`make`) 중 ``undefined reference to `fftw_join_taint'`` | 일반 OpenFOAM 환경에서 빌드함 | 환경 스크립트를 불러온 뒤 `make` 다시 실행 |
| `cmake .` 실행 시 `Could not find a package configuration file provided by "precice"` | preCICE 패키지 미설치 | 3.2절의 `apt install` 확인 |
| solverdummy가 `Setting up primary communication to coupling partner/s`에서 계속 멈춤 | 다른 터미널의 프로그램을 아직 실행하지 않았거나(이 경우는 정상), 그 프로그램이 오류로 종료됨, 또는 두 프로그램을 서로 다른 디렉토리에서 실행함, 또는 중단한 이전 실행의 precice-run/ 디렉토리가 남아 있음 | 다른 터미널의 출력 확인. 두 프로그램 모두 ~/precice/solverdummies/cpp에서 실행. 멈춘 프로그램을 `Ctrl+C`로 종료하고 `rm -rf precice-run` 후 다시 실행 |
| solverdummy가 `ERROR: Unable to establish connection as a connection file already exists at "./precice-run/..."`를 출력하고 `terminate called after throwing an instance of 'precice::Error'`와 역추적을 출력하며 비정상 종료 | 중단한 이전 실행의 연결 정보(precice-run/)가 남아 있음. 이때 다른 터미널의 프로그램은 계속 기다린다 | 두 프로그램을 모두 `Ctrl+C`로 종료하고 `rm -rf precice-run` 후 다시 실행 |
| 환경 스크립트를 불러올 때 `syntax error near unexpected token` 오류(그 앞에 `/opt/OpenFOAM-v2412/etc/bashrc`에 대한 `No such file or directory`) | Windows 편집기로 만든 파일의 줄 끝(CRLF) | `sed -i 's/\r$//' ~/precice/precice-of-env.sh` |

## 7. 마무리

이 예제에서 학습한 내용을 요약하면 다음과 같다.

- 분할 방식 연성과 preCICE, 해석 프로그램을 preCICE에 연결하는 어댑터의 역할
- 공식 `.deb` 패키지를 이용한 preCICE, CalculiX 어댑터 설치와 OpenFOAM-preCICE 어댑터 빌드
- OpenFOAM ThirdParty 라이브러리와 시스템 라이브러리 사이의 충돌 원인 분석과, 환경 스크립트를 이용한 해결
- `foamHasLibrary`와 solverdummy를 이용한 설치 검증

이 환경을 사용하는 다음 예제:

- [예제 21](21_유체-구조-연성-해석-preCICE-CalculiX.md) : OpenFOAM(유체)과 CalculiX(구조)의 양방향 유체-구조 연성 해석. preCICE 공식 튜토리얼 케이스를 실행하고 공식 참조 결과와 비교하여, 이 예제에서 구성한 환경을 최종 검증한다.
- [예제 22](22_열-유체-구조-연성-해석-열응력-열변형.md) : 예제 21의 케이스에 열 연성을 더하여, 열응력에 의한 열변형을 해석한다.

preCICE 튜토리얼 저장소에는 이 환경으로 실행할 수 있는 다른 케이스도 있다. 예를 들어 flow-over-heated-plate 케이스는 OpenFOAM 두 개(유체, 고체)를 열 연성한다.

## 8. 참고 문헌

[1] G. Chourdakis, K. Davis, B. Rodenberg, M. Schulte, F. Simonis, B. Uekermann, et al., "preCICE v2: A sustainable and user-friendly coupling library," *Open Research Europe*, vol. 2, p. 51, 2022. [https://doi.org/10.12688/openreseurope.14445.2](https://doi.org/10.12688/openreseurope.14445.2)

[2] G. Chourdakis, D. Schneider, and B. Uekermann, "OpenFOAM-preCICE: Coupling OpenFOAM with external solvers for multi-physics simulations," *OpenFOAM® Journal*, vol. 3, pp. 1–25, 2023. [https://doi.org/10.51560/ofj.v3.88](https://doi.org/10.51560/ofj.v3.88)

[3] G. Dhondt, *The Finite Element Method for Three-Dimensional Thermomechanical Applications*, Wiley, 2004.
