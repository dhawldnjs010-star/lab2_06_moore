# 실험 전 레포트: LAB2-06 Moore 상태 머신

작성자: 상혁 (2025440084) / 작성일: 2026-09-20 / 소스 커밋: `e4fb781` / workspace: `LAB1.code-workspace` (템플릿 v2.0.1) / OS: `Windows 11 Home 10.0.26200` / Python: `Python 3.14.7` / 시뮬레이터: Icarus Verilog `Icarus Verilog version 12.0 (devel) (s20150603-1539-g2693dd32b)`

> 이 레포트는 VS Code(Icarus) 시뮬레이션까지의 사전 검증이다. Vivado GUI와 실물 보드 결과는 실험 후 레포트([post](../post/post_report.md))에서 다룬다. 시각은 clk 상승 에지(5, 15, 25, … ns)의 1 ns 뒤, 즉 TB가 비교하는 시각으로 적었다.

## 목적과 예상 동작

상태 00 → 01 → 10 → 00을 순환하는 Moore 상태 머신을 설계한다. 출력이 현재 상태에만 의존하며 입력(advance)만 바뀌어도 출력이 즉시 바뀌지 않는다는 점을 확인한다.

### 포트 (`moore_cycle.v`의 `moore_cycle`)

| 포트 | 방향 | 비트 폭 | 설명 |
|---|---|---|---|
| clk | in | 1 | 상승 에지 기준 클록 |
| rst | in | 1 | 동기 active-high 리셋 |
| enable | in | 1 | 1이어야 전이 가능 |
| advance | in | 1 | 1이면 다음 상태로 전이 요청 |
| value | out (reg) | 2 | 현재 상태이자 출력 |

최상위(`lab2_moore.v`, `lab2_moore`): `enable = press`, `advance = switches[7]`(SW1), `led = {6'b000000, value}`.

### 동작 규칙과 경계 입력

- 규칙: rst이면 00. 아니면 enable && advance인 에지에서 00→01, 01→10, 10→00. 그 외에는 상태 유지. 출력 = 상태.
- 정상·경계: advance만 1로 바뀌어도(에지 전) 출력 유지, advance=0 또는 enable=0이면 유지, 10 다음은 00으로 순환.
- 시간 기준: TB 클록은 10 ns 주기(상승 에지 5, 15, 25, … ns)이고 입력은 에지 1 ns 뒤에 바꾼다. 레지스터는 다음 상승 에지에서 갱신된다. 실제 보드 클록(1 kHz)은 시뮬레이션의 10 ns와 별개다.

## 소스와 테스트벤치

- 설계 top: `lab2_moore` / 시뮬레이션 top: `tb_moore_cycle`
- 소스: [`src/moore_cycle.v`](../../src/moore_cycle.v), [`src/input_frontend.v`](../../src/input_frontend.v), [`src/lab2_moore.v`](../../src/lab2_moore.v)
- 테스트벤치: [`sim/tb_moore_cycle.sv`](../../sim/tb_moore_cycle.sv)
- 제약: [`constraints/lab2_moore.xdc`](../../constraints/lab2_moore.xdc)
- 설정: [`simulation.json`](../../simulation.json) (sources 3개, testbench `sim/tb_moore_cycle.sv`, simulation_top `tb_moore_cycle`)

| 파일 | 역할 |
|---|---|
| `src/moore_cycle.v` | 핵심 동작을 담은 코어 `moore_cycle`. TB가 이 모듈을 직접 검사한다. |
| `src/input_frontend.v` | 버튼·스위치를 클록에 맞추는 입력 회로. 리셋 2단 해제 동기화, 버튼·스위치 2단 동기화 플립플롭, STABLE_CYCLES=20(1 kHz에서 20 ms) 안정 확인 뒤 한 클록짜리 `press` 펄스를 만든다. |
| `src/lab2_moore.v` | 보드 top. 프런트엔드와 코어를 연결하고 LED로 출력한다. |
| `sim/tb_moore_cycle.sv` | 입력 자극, 기대값 계산, 자동 비교(`check`), PASS/FAIL 출력, VCD 생성, watchdog. |
| `constraints/lab2_moore.xdc` | 핀 번호·전압과 1 kHz 클록 정의. Icarus는 XDC를 읽지 않으므로 이 사전 시뮬레이션에는 사용되지 않는다. |
| `simulation.json` | VS Code 시뮬레이션 작업이 읽는 소스 목록·테스트벤치·시뮬레이션 top. |

### 테스트벤치 동작

- 자극 순서: 리셋 → enable=1, advance=1(에지 전 1 ns 뒤 입력만 변경) → 4라운드: (전이 01, advance=0 유지, enable=0 유지, 전이 10, 전이 00) → 마지막에 한 번 더 전이 후 rst=1 리셋.
- 검사 횟수: 1(reset)+1(input alone)+4×5(S0 to S1, advance zero holds, disabled holds, S1 to S2, S2 to S0)+1(reset from nonzero) = 23
- 종료·watchdog: 마지막 검사 뒤 `finish` task가 `LAB2_PASS moore_cycle checks=N`을 출력하고 `$finish`한다. 별도로 100000 ns(100 µs) 뒤에 `watchdog timeout`으로 `$fatal` 처리한다. 예상 종료 시각은 226 ns이다.
- 이 TB는 코어 `moore_cycle`만 시험한다. 입력 동기화·디바운스와 실제 핀·타이밍이 통과했다는 뜻은 아니다.

### XDC 설명

`lab2_moore.xdc`은 포트 이름을 `lab2_moore.v`과 맞춰 핀을 지정한다. 모든 I/O는 `LVCMOS33`이고 `create_clock -name trainer_1khz -period 1000000.000 [get_ports clk]`로 주 클록을 1 kHz(주기 1,000,000 ns)로 정의하며 `set_false_path -from [get_ports {rst button sw[*]}]`로 비동기 입력을 타이밍 경로에서 제외한다.

| 포트 | 핀 | 보드 대응 |
|---|---|---|
| clk | B6 | 1 kHz 주 클록 |
| rst | K4 | 리셋 (active-high) |
| button | N8 | 스텝 버튼 |
| sw[7:0] | U4(sw[0]), V4, W1, W4, T1, U2, W3, Y1(sw[7]) | DIPSW8..DIPSW1 (DIPSW1..8 = sw[7]..sw[0]) |
| led[7:0] | N5(led[0]), M1, M3, M7, N7, M2, M4, L4(led[7]) | LED0..LED7 |

## VS Code 실행 과정

1. File → New Window → File → Open Workspace from File...로 `LAB1.code-workspace`를 연다. 확장(slang, VaporView, vscode-pdf)을 설치한다.
2. RTL·TB·XDC·`simulation.json`을 직접 입력하고 File → Save All.
3. Terminal → Run Task... → `01 Check tools`로 Git·Python·iverilog·vvp 버전을 확인한다.
4. `02 Simulate`를 실행해 `LAB2_PASS`와 종료 시각을 확인한다.
5. `03 Open waveform`으로 `build/sim/wave.vcd`를 VaporView로 연다. 신호: clk, rst, enable, advance, value[1:0] (value는 2진수).

정상 실행 로그(본인 로그로 교체하고 `evidence/pre/`에 복사):

```text
LAB2_PASS moore_cycle checks=23
sim/tb_moore_cycle.sv:19: $finish called at 226000 (1ps)
```

- 본인 실행 로그: [`../../evidence/pre/lab2_06_normal.log`](../../evidence/pre/lab2_06_normal.log)
- VCD: [`../../evidence/pre/lab2_06_wave_normal.vcd`](../../evidence/pre/lab2_06_wave_normal.vcd)
- 파형 캡처(VaporView): `evidence/pre/lab2_06_wave_zoom.png`(상승 에지 확대)
- 오류: 첫 실행에서 발생한 오류가 있으면 첫 오류 → 수정 → 재실행 로그 순서로 기록한다. (없으면 "없음", Python 실행 경로를 고쳤다면 그 내용 기입)

### 사전 파형 해석

| 시간 구간 | 입력 | 예상 | 실제 파형 | 해석 |
|---|---|---|---|---|
| 6 ns | rst=1 | value=00 | value=00 | 동기 리셋. |
| 7 ns | rst=0, enable=1, advance=1 (에지 전) | value=00 | value=00 | 입력만 바뀌었으므로 출력은 00 유지. 출력은 상승 에지에서만 바뀐다. |
| 16 ns | 에지 15 ns | value=01 | value=01 | enable·advance가 모두 1인 유효 에지: S0→S1. |
| 26 ns | advance=0 · 에지 25 ns | value=01 | value=01 | advance=0이므로 상태 유지. |
| 36 ns | advance=1, enable=0 · 에지 35 ns | value=01 | value=01 | enable=0이므로 유지. |
| 46 ns | enable=1 · 에지 45 ns | value=10 | value=10 | S1→S2. |
| 56 ns | 에지 55 ns | value=00 | value=00 | 경계: S2에서 다음 상태는 S0(순환). |
| 226 ns | rst=1 · 에지 225 ns | value=00 | value=00 | 01로 바뀐 상태에서 리셋 에지가 오면 00으로 복귀. |

표의 값은 상승 에지 직후 안정된 값이다. `LAB2_PASS`만 적지 않고 각 행에서 입력, 이전 상태, 다음 상태를 비교한다. 파형 캡처에서 위 시각을 확대해 본인 화면으로 확인한다.

## 코드 수정·실패·복구 실험

- 변경: S1의 다음 상태를 S0으로 바꾼다(`2'b01: value <= 2'b10;` → `value <= 2'b00;`).
- 변경한 파일과 위치: `src/moore_cycle.v` 11행
- 테스트벤치 기대값은 바꾸지 않는다.

```diff
-    2'b01: value <= 2'b10;
+    2'b01: value <= 2'b00;
```

실행 전 계산: 46 ns의 에지(45 ns)에서 이전 상태는 01, enable=1, advance=1이므로 정상 회로는 10으로 간다. 변경 회로는 00으로 가서 TB의 `value===2` 검사가 46 ns에 실패한다. 16 ns의 S0 to S1은 변경과 무관하게 통과한다.

| 단계 | 소스 커밋 또는 해시 | 실행 폴더·로그 링크 | 입력·기대값·실제값 | 해석 |
|---|---|---|---|---|
| 정상 코드 | `e4fb781` | [normal.log](../../evidence/pre/lab2_06_normal.log) | 46 ns 기대 value=10, 실제 value=10. `LAB2_PASS moore_cycle checks=23` | 모든 검사 통과, 226 ns 종료. |
| 지정한 RTL 변경 | 미커밋 수정본(`e4fb781` 기준, 로컬 실행) | [mod.log](../../evidence/pre/lab2_06_mod.log) | 46 ns 기대 value=10, 실제 value=00. `LAB2_FAIL S1 to S2 time=46000`, `FATAL: sim/tb_moore_cycle.sv:12: check failed` | `S1 to S2` 검사가 변경을 발견했다(로그의 time은 ps 단위, 46000 ps = 46 ns). |
| 원래 코드로 복구 | `e4fb781` | [recover.log](../../evidence/pre/lab2_06_recover.log) | 복구 후 전체 검사 재실행. `LAB2_PASS moore_cycle checks=23`, `$finish called at 226000 (1ps)` | PASS와 종료 시각이 정상 실행과 같고 새 VCD를 확인한다. |

- 첫 실패 이후에는 `$fatal`로 시뮬레이션이 끝나므로 뒤의 검사는 실행되지 않는다. 변경 전후 파형은 각각 별도 폴더에 보관한다.
- 문법 오류를 경험했다면 오류 위치로 이동한 화면, 원인, 수정 내용과 재실행 로그도 이 절에 연결한다.

## 보드 실험 계획

- 부품·프로젝트: Vivado 2026.1, RTL Project `lab2_moore`, 부품 `xc7s75fgga484-1`(정확히 -1). Design Sources: `moore_cycle.v`, `input_frontend.v`, `lab2_moore.v`(Copy sources 해제). Simulation Sources: `tb_moore_cycle.sv`(Set as Top: `tb_moore_cycle`). Constraints: `lab2_moore.xdc`. Project Summary의 Top module name은 `lab2_moore`이다.
- 장비 클록·제약: Combo II-DLD S75 주 클록 B6을 1 kHz로 맞추고 XDC의 `trainer_1khz`(1,000,000 ns)와 일치시킨다.
- 입력: DIPSW1=`sw[7]`=advance, N8=press(=enable), K4=리셋. 주 클록은 1 kHz.
- 출력: LED[1:0]=value, LED[7:2]=0.

| 조작 | 예상 LED / 동작 |
|---|---|
| K4 초기화 | LED[1:0]=00 |
| SW1=1로만 변경 (N8 안 누름) | 00 유지 |
| N8 한 번 | 01 |
| SW1=0에서 N8 한 번 | 01 유지 |
| SW1=1로 바꾸고 N8 두 번 | 10 → 00 |

SW1은 동기화 회로를 거치므로 실제 변경 시점과 약간의 지연이 있을 수 있다.

촬영할 장면: 보드 전체(배선·입력·출력이 함께 보이는 사진)와 위 표의 조작별 LED 상태 사진·영상. 예상되는 차이: 버튼·스위치는 동기화·안정 확인 지연이 있고, 빠른 신호는 눈으로 구분되지 않을 수 있다.

이 단계에서는 Vivado GUI와 실물 보드의 결과를 수행한 것처럼 기록하지 않는다. 합성·구현·bit 생성과 실제 장치 기록은 실험 후 레포트에서 다룬다.
