# 📦 추가 자료 — `mac_v2.v` (강화된 MAC 구조)

> Week 3-2 의 `mac.v` 는 *교육용 최소 MAC*. 이 폴더는 *production 으로 가려면 무엇을 더해야 하는지* 를 보여주는 *advanced 예시*.
>
> **필수 학습 자료 아님** — Week 3-2 진행에 영향 없음. *RTL 깊이 들어가고 싶은 학생* 의 선택 학습.

## 🎯 학습 목적

`mac.v` (기본) ↔ `mac_v2.v` (강화) 를 *나란히* 보며:
- *왜 production RTL 이 그렇게 복잡한지* 직관적으로 이해
- *각 추가 신호 / 로직이 *어떤 문제* 를 해결하는지* 파악
- 본인 NPU 설계 (Week 3-1) 시 *어느 수준까지 구현할지* 판단 근거

## 📋 파일 구성

| 파일 | 역할 |
| --- | --- |
| `mac_v2.v` | 강화된 MAC (75줄) — 8가지 개선 반영 |
| `test_mac_v2.py` | cocotb testbench (6 테스트) |
| `Makefile` | cocotb 표준 빌드 |
| `README.md` | 이 문서 |

## 🚀 실행

```bash
source ../../../.venv/bin/activate
cd week3/reference/advanced_mac
make            # 6 테스트 실행 (TESTS=6 PASS=6 기대)
```

## ⚖️ `mac.v` vs `mac_v2.v` — 비교 표

| 항목 | `mac.v` (기본) | `mac_v2.v` (강화) | 해결하는 문제 |
| --- | --- | --- | --- |
| **데이터 폭** | 8-bit, 32-bit hardcoded | `DATA_WIDTH`, `ACC_WIDTH` 파라미터 | INT4/INT16 재사용 |
| **Reset** | `rst` (active-high) | `rst_n` (active-low) | 산업 표준 따름 |
| **Enable** | 없음 | `en` 신호 | stall / power saving |
| **Clear** | reset 뿐 | `clear_acc` 신호 | 새 dot product 시작 (전체 reset 없이) |
| **Valid 신호** | 없음 (항상 누적) | `in_valid` / `acc_valid` | 외부 모듈과 handshake |
| **Pipeline** | 1-stage (곱+덧셈 한 cycle) | 2-stage (곱 → 누적) | 고주파 동작 (>300 MHz) |
| **Overflow** | silent wrap | saturation + `overflow` flag | 정확성 / 디버깅 |
| **`$dumpfile`** | 모듈 안에 포함 | 없음 (testbench 가 dump) | 합성 가능 RTL |

## 🔬 8가지 개선의 *왜* — 각각의 의미

### 1️⃣ Parameterization

```verilog
module mac_v2 #(
    parameter DATA_WIDTH = 8,
    parameter ACC_WIDTH  = 32
) ( ... );
```

**왜?** Week 4-4 에서 *응용에 따라 dtype 바꾸기* 가능 (INT4 → INT8 → INT16). hardcoded 면 매번 새 모듈.

### 2️⃣ Active-low Reset (`rst_n`)

**왜?** 산업 표준. 칩 외부에서 *전원 ON 직후* reset 라인이 *0 으로 떨어졌다가 1 로 올라옴*. 외부 noise 에서 안전.

```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) ...
```

### 3️⃣ Enable 신호 (`en`)

```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) ...
    else if (en) ...   // ← en=0 이면 아무 일도 안 함
end
```

**왜?**
- *Stall* — 외부 메모리에서 입력 못 가져왔을 때 *현재 상태 유지*
- *Power saving* — 사용 안 할 때 clock gating 가능
- `mac.v` 는 *매 cycle 무조건 누적/reset* → garbage 누적 위험

### 4️⃣ Clear Accumulator (`clear_acc`)

```verilog
prod_clear <= clear_acc & in_valid;
// ...
acc_base = prod_clear ? 0 : acc_ext;
acc_next = acc_base + prod_ext;
```

**왜?** *새 dot product 시작* 시 acc 를 *0 으로 다시 시작* 해야 함. `mac.v` 는 *full reset* 만 가능 → *clock 한 사이클 손실* + 모든 상태 잃음.

`clear_acc=1` 이면 *이번 cycle 의 곱셈 결과가 새 acc 의 시작* 이 됨. 깔끔.

### 5️⃣ Valid 신호 Pipeline

```verilog
input wire in_valid;
output reg acc_valid;
// in_valid → prod_valid (stage 1) → acc_valid (stage 2)
```

**왜?** Systolic array / NPU 컨트롤러 는 *언제 결과가 유효한지* 알아야 함. `mac.v` 는 *acc 가 항상 변함* → 외부에서 *언제 읽을지* 모름.

AXI-Stream 같은 *valid/ready handshake* 의 절반 (valid 만).

### 6️⃣ 2-stage Pipeline

```verilog
// Stage 1: 곱
prod_reg <= in_data * weight;

// Stage 2: 누적
acc <= acc + prod_reg;
```

**왜?**
- `mac.v` 한 cycle 안에 *곱 (5ns) + 덧셈 (3ns)* = 8ns → **125 MHz 한계**
- 분리하면 각 단계 ~5ns → **200 MHz 가능**
- 진짜 NPU 는 더 잘게 쪼개서 *1 GHz+* 동작
- **대가**: latency 1 cycle 증가 + pipeline warmup 필요

### 7️⃣ Saturation + Overflow Flag

```verilog
wire signed [33:0] acc_next_raw = acc_ext + prod_ext;
wire overflow_detected = (acc_next_raw[32] != acc_next_raw[31]);  // sign bit mismatch

if (overflow_detected) begin
    acc      <= acc_next_raw[32] ? ACC_MIN : ACC_MAX;
    overflow <= 1;
end
```

**왜?**
- INT32 누적도 *큰 batch + 큰 weight* 면 overflow 가능
- `mac.v` 는 *silent wrap* — 2147483647 + 1 → -2147483648 (값이 *반전*)
- 진짜 칩: saturation (max 에서 멈춤) + flag 출력 → 디버깅 가능
- Overflow detection: 33-bit signed 결과의 *bit 32 (extra sign)* 과 *bit 31 (would-be sign)* 가 다르면 overflow

### 8️⃣ `$dumpfile` 제거

```verilog
// mac.v 에 있던 부분:
initial begin
    $dumpfile("dump.vcd");
    $dumpvars(0, mac);
end
```

**왜 빼는가?**
- `$dumpfile` 은 *시뮬레이션 전용* — 합성기는 무시하지만 *코드 품질 저하*
- *진짜 합성 가능 RTL* 에는 들어가면 안 됨
- cocotb 의 `WAVES=1` 환경변수 또는 testbench 가 dump 처리

> 💡 **mac.v 의 `$dumpfile`** 은 *교육 편의를 위한 의도된 단순화*. cocotb 의 `make WAVES=1` 사용법을 학생이 모를 때 *자동으로 dump.vcd 가 생기게* 함.

## 🧪 cocotb 테스트 6종 (`test_mac_v2.py`)

| 테스트 | 검증 내용 |
| --- | --- |
| `test_basic_accumulation` | 3개 곱셈 누적 — 기능 정확성 |
| `test_enable_holds` | en=0 일 때 acc 유지 |
| `test_clear_acc` | clear_acc 로 새 시퀀스 시작 |
| `test_no_silent_wrap` | 100×127² = 1.6M 누적 시 overflow=0 |
| `test_valid_pipeline_propagation` | in_valid → acc_valid 2-cycle 안에 도착 |
| `test_random_vs_numpy` | 100 INT8 pair, NumPy 와 정확히 일치 |

### 검증 결과 (M4 Max)

```
TESTS=6 PASS=6 FAIL=0
test_basic_accumulation           PASS
test_enable_holds                 PASS
test_clear_acc                    PASS
test_no_silent_wrap               PASS
test_valid_pipeline_propagation   PASS
test_random_vs_numpy              PASS  (RTL=2413, NumPy=2413)
```

## 🎓 학생이 가져갈 학습 포인트

### 1. *production RTL = 기본 RTL + 8가지 디테일*

각 디테일이 *교과서에 없는* 실무 지식. *어느 회사 RTL coding guide* 를 봐도 비슷한 항목들이 강조됨.

### 2. *복잡성에는 이유가 있다*

`mac.v` 의 단순함이 *나쁜* 게 아니라 *교육 의도*. 진짜 칩 설계자는 *언제 어떤 디테일이 필요한지* 판단력을 갖춰야.

### 3. *Verify 가 작성보다 어렵다*

`mac.v` 의 cocotb 4 테스트 → `mac_v2.v` 의 6 테스트. 기능이 늘면 *테스트 부담* 도 비선형 증가. *Verification engineer* 가 별도 직군인 이유.

### 4. *Trade-off 명시*

| 추가 기능 | 비용 |
| --- | --- |
| Pipeline | 면적 ↑, latency ↑, 복잡도 ↑ |
| Saturation | 면적 ↑ (비교기), 약간의 지연 |
| Valid 신호 | 인터페이스 복잡도 ↑ |
| Parameterization | 합성 시 elaboration 비용 |

**공짜 점심 없음** — Week 4-4 의 *specialization 비용* 메시지와 연결.

## 💼 강사 활용 — 회차 후 *심화 학습* 자료로

| 시점 | 활용 |
| --- | --- |
| Week 3-2 회차 끝 | *"더 가고 싶은 학생은 advanced_mac 보세요"* 한 줄 안내 |
| Week 3-5 (회고) | *"내 NPU 의 MAC 을 production 수준으로 만들려면 무엇이 필요?"* 토의 |
| Week 4-2 (cost 모델) | *"왜 pipeline / saturation 이 면적을 늘리는지"* 의 구체적 예 |
| Week 4-5 (발표) | RTL 부분을 *advanced* 까지 한 학생 — 발표에서 가산점 |
| 코스 종료 후 | *"FPGA / ASIC 으로 가려면 +6가지"* 답변의 구체적 사례 |

## 🔬 실험 — 직접 비교해보기 (학생용 도전)

같은 1024-element dot product 를 두 MAC 에서 실행해서 비교:

```bash
# 1. 기본 mac.v 의 cycle 측정 (이미 Week 3-4 reference 에서)
cd ../04_cycle_compare && make
# → RTL: 1025 cycles

# 2. mac_v2.v 도 비슷한 패턴으로 (advanced testbench 작성 필요)
# 예상: 2-stage pipeline 이므로 1024 + 2 = 1026 cycles
```

> 💡 *Pipeline 이 깊을수록 latency 가 늘어남* 의 직접 측정.

## 📐 한 줄 요약

> **`mac.v` = 교육용 최소 MAC. `mac_v2.v` = production 으로 가는 *한 걸음*.**
>
> *진짜 NPU MAC* 은 여기서 또 +몇 가지 (DSP 추론, multi-cycle multiplier, AXI 인터페이스, scan chain DFT 등) 가 더해짐. *진짜 칩 설계자가 되려는 학생* 의 *다음 학습 entry point*.

claude --resume d3561eb5-a6bf-436b-9cf1-2b253db62bdd