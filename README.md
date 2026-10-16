# OpenFOAM 튜토리얼 (한국어판)

[![License: CC BY-NC-SA 3.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%203.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/3.0/)
[![OpenFOAM](https://img.shields.io/badge/OpenFOAM-v2412-blue)](https://www.openfoam.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20WSL2-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Tutorials](https://img.shields.io/badge/%EC%98%88%EC%A0%9C-22%EA%B0%9C-success)](#예제-목록)

OpenFOAM의 주요 기능을 학습하기 위한 한국어 예제집이다. OpenFOAM의 설치부터 격자 생성, 다상 유동, 난류, 병렬 처리, 화학 반응, 다물리 해석(전자기·전기화학), 사용자 정의 솔버 개발 등 OpenFOAM의 기본적인 사용법과 응용을 23개 예제를 통해 다룬다.

## 구성

이 예제집은 Technische Universität Wien, Institute of Chemical, Environmental & Bioscience Engineering의 **OpenFOAM Basic Training, 5th edition (2019)** ([www.cfd.at/tutorials](https://www.cfd.at/tutorials), 예제 1~14)을 한국어로 번역한 자료에, 9개 예제(예제 0, 15~22)를 추가하여 구성하였다.

각 예제는 아래의 구조로 구성된다.

1. **배경**: 다루는 주제에 대한 소개 및 관련 CFD 이론
2. **전처리**: 케이스 파일 구성 및 입력 설정 방법
3. **솔버 실행**: 솔버 실행 방법과 관련 명령어
4. **후처리**: ParaView 등을 이용한 결과 분석

단, 예제 0(설치 및 솔버 목록)과 예제 20(preCICE 연성 해석 환경 구성)은 해석 케이스를 다루지 않으므로 이 구조를 따르지 않는다.

## 호환성

- 원서: OpenFOAM 7, v1906
- 한국어판: OpenFOAM v2412 (WSL2, Ubuntu 24.04)

설치 절차는 [예제 0](./tutorials/00_Ubuntu-24.04-WSL-및-OpenFOAM-v2412-설치-OpenFOAM-솔버-목록.md)에 정리되어 있다.

## 예제 목록

| 예제 | 주제 | 솔버 | 대상 |
| :---: | :--- | :--- | :--- |
| [0](./tutorials/00_Ubuntu-24.04-WSL-및-OpenFOAM-v2412-설치-OpenFOAM-솔버-목록.md) | 설치 및 솔버 목록 | — | — |
| [1](./tutorials/01_기초-케이스-설정.md) | 기초 케이스 설정 | `icoFoam` | 2D, 엘보 곡관 |
| [2](./tutorials/02_내장-격자-생성-도구-blockMesh.md) | 격자 : `blockMesh` | `rhoPimpleFoam` / `sonicFoam` | 2D, 전향 계단 |
| [3](./tutorials/03_계산-영역-내-물리량-패치하기.md) | 계산 영역 내 물리량 패치하기 | `rhoPimpleFoam` / `sonicFoam` | 1D, 충격파 관 |
| [4](./tutorials/04_이산화-기법-1.md) | 이산화 기법 1 | `scalarTransportFoam` | 1D, 충격파 관 |
| [5](./tutorials/05_이산화-기법-2.md) | 이산화 기법 2 | `scalarTransportFoam` | 2D, 원 |
| [6](./tutorials/06_난류-정상-상태.md) | 난류 : 정상 상태 | `simpleFoam` | 2D, 후향 계단 |
| [7](./tutorials/07_난류-비정상-상태.md) | 난류 : 비정상 상태 | `pisoFoam` | 2D, 후향 계단 |
| [8](./tutorials/08_다상-유동-Volume-of-fluid-VOF.md) | 다상 유동 (VOF) | `interFoam` | 2D, 댐 붕괴 |
| [9](./tutorials/09_병렬-계산-기법.md) | 병렬 계산 기법 | `compressibleInterFoam` | 3D, Depth charge |
| [10](./tutorials/10_체류-시간-분포.md) | 체류 시간 분포 (RTD) | `simpleFoam` / `scalarTransportFoam` | 3D, T자형 배관 |
| [11](./tutorials/11_화학-반응.md) | 화학 반응 | `reactingFoam` | 2D, 혼합 엘보 |
| [12](./tutorials/12_격자-생성-도구-snappyHexMesh-단일-영역.md) | 격자 : `snappyHexMesh` (단일 영역) | `scalarTransportFoam` | 3D, 플랜지 |
| [13](./tutorials/13_격자-생성-도구-snappyHexMesh-다중-영역.md) | 격자 : `snappyHexMesh` (다중 영역) | `chtMultiRegionFoam` | 3D, 플랜지 |
| [14](./tutorials/14_샘플링.md) | 샘플링 | `sonicFoam` / `rhoPimpleFoam` | 3D, 충격관 |
| [15](./tutorials/15_직류-코로나-방전.md) | 직류 코로나 방전 | `electrostaticFoam` | 2D, 와이어-평판 방전 |
| [16](./tutorials/16_산화-환원-흐름-전지.md) | 산화 환원 흐름 전지 | `RfbFoam` (외부 솔버) | 3D, 흐름전지 half cell |
| [17](./tutorials/17_고체-산화물-연료-전지.md) | 고체 산화물 연료 전지 (SOFC) | `openFuelCell2` (외부 솔버) | 3D, SOFC unit cell |
| [18](./tutorials/18_사용자-정의-솔버-만들기-전기수력학.md) | 사용자 정의 솔버 만들기 : 전기수력학 | `electrostaticSimpleFoam` (사용자 정의) | 2D, 와이어-평판 전기집진기 |
| [19](./tutorials/19_다상-유동-복합-열전달.md) | 액체 제트 충돌 냉각 | `chtMultiRegionInterFoam` (외부 솔버) | 3D, 핀-휜 히트싱크 |
| [20](./tutorials/20_FSI1-preCICE-연성-해석-환경-구성.md) | FSI1: preCICE 연성 해석 환경 구성 | — | — |
| [21](./tutorials/21_FSI2-유체-구조-연성-해석-preCICE-CalculiX.md) | FSI2: 유체-구조 연성 해석 (FSI) | `pimpleFoam` + CalculiX (preCICE 연성) | 2D, 수직 탄성 플랩 |
| [22](./tutorials/22_FSI3-열-유체-구조-연성-해석-열응력-열변형.md) | FSI3: 열-유체-구조 연성 해석 | `buoyantBoussinesqPimpleFoam` + CalculiX (preCICE 연성) | 2D, 이중층(바이메탈) 플랩 |

## 학습 순서

OpenFOAM을 처음 학습하는 사용자는 먼저 예제 0~14를 순서대로 진행하는 것을 권장한다. 각 예제는 일반적인 CFD 해석 및 OpenFOAM 활용에 거의 필수적인 이론(유체역학, 수치해석 등)과 사용법(형상 정의, 격자 생성, 모델 설정, 후처리 기법 등)을 함께 다루며, 이전 예제에서 다룬 개념 및 설정 방법을 참조하여 반복 설명을 생략한다. 한 예제를 건너뛰면 다음 예제의 설명이 불충분할 수 있기 때문에, 임의로 생략하는 것은 권장하지 않는다.

그 이후의 예제는 특정 분야에서 사용되는 특정 모델의 사용법에 대한 설명이 주를 이룬다. 외부 솔버를 사용하거나 코드 수정, 해석 환경 구성 등 숙련도를 요하는 작업이 주를 이루므로, OpenFOAM 기본 사용법에 익숙해진 뒤에 학습하는 것을 권장한다.

## 라이선스

이 예제집은 원서와 동일하게 [Creative Commons Attribution-NonCommercial-ShareAlike 3.0 Unported (CC BY-NC-SA 3.0)](https://creativecommons.org/licenses/by-nc-sa/3.0/) 라이선스 하에 배포된다.

- **원저작자 표시 (BY)**: 원저작자(TU Wien)와 출처([www.cfd.at/tutorials](https://www.cfd.at/tutorials))를 명시할 것
- **비영리 (NC)**: 상업적 이용 불가
- **동일조건 변경허락 (SA)**: 2차 저작물은 동일/호환 라이선스로만 배포 가능

상세 내용은 [LICENSE.md](./LICENSE.md)를 참고한다.

## 면책 조항

이 자료는 ESI Group, ESI-OpenCFD 또는 OpenFOAM Foundation에 의해 승인된 자료가 아니다.

This offering is not approved or endorsed by ESI Group, ESI-OpenCFD or the OpenFOAM Foundation, the producer of the OpenFOAM® software and owner of the OpenFOAM® trademark.
