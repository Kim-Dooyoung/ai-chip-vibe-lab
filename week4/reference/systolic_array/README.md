# Systolic Array (week4 reference) — `mac_prod` 위에 5 모듈로 조립

## 모듈 계층

```
systolic_top.v                ← 조립 (자체 로직 없음)
├── weight_loader.v           ← 1-cycle parallel broadcast
├── input_skew.v              ← row i 를 i cycle 지연 (계단형)
├── pe_array.v                ← R×C mac_prod 격자 + 배선만
│   └── mac_prod.v            ← week3/reference/prod_mac (외부 참조)
└── output_collect.v          ← bottom psum 캡처 + valid pipeline
```

**`mac_prod.v` 는 복사하지 않음** — Makefile 이 `week3/reference/prod_mac/` 의 파일을 그대로 참조 (single source of truth).

## 빌드 & 테스트 & 파형

```bash
source ../../../.venv/bin/activate

# 모듈별 단독 테스트 (모두 dump.vcd 생성됨)
make TARGET=pe_array          # 27 KB (16 mac_prod 포함)
make TARGET=input_skew        #  3 KB
make TARGET=weight_loader     #  1 KB
make TARGET=output_collect    #  1 KB
make TARGET=systolic_top      # 30 KB (full hierarchy)

# 모든 모듈 일괄
for T in pe_array input_skew weight_loader output_collect systolic_top; do
    make clean > /dev/null && make TARGET=$T 2>&1 | grep "TESTS="
done

# 파형 확인
surfer dump.vcd &
```

### 파형 분석 추천 순서

각 모듈 단독 VCD 부터 작은 것 → 큰 것 순으로:

1. **`make TARGET=output_collect && surfer dump.vcd`** — 가장 간단 (capture + valid pipeline)
2. **`make TARGET=weight_loader`** — start → load → done 펄스 시퀀스
3. **`make TARGET=input_skew`** — staircase 지연 패턴 (row 0~3 의 출력 시점 차이)
4. **`make TARGET=pe_array`** — 격자 배선 + 16 PE 의 내부 상태 (`g_row[i].g_col[j].pe`)
5. **`make TARGET=systolic_top`** — 5 모듈 전체 통합 동작 (`u_array`, `u_skew`, `u_wloader`, `u_collect`)

### Surfer 에서 보면 좋은 signal

**systolic_top VCD 의 경우:**

```
systolic_top.clk, .rst_n, .en
systolic_top.weight_start, .weight_done
systolic_top.act_in_flat, .act_valid_in
systolic_top.valid_out, .result_flat
systolic_top.u_skew.g_row[3].g_delay.pipe[2]     ← row 3 의 가장 깊은 stage
systolic_top.u_array.g_row[0].g_col[0].pe.weight_reg
systolic_top.u_array.g_row[3].g_col[3].pe.psum_out  ← 최종 출력 직전
```

→ wave 의 *대각선 활성화* 가 보임 (systolic 의 시각적 특징).

## 검증 결과 (M4 Max, cocotb 2.0.1, icarus 13.0)

| 모듈 | 테스트 수 | 통과 |
| --- | --- | --- |
| pe_array | 3 | 3/3 |
| input_skew | 3 | 3/3 |
| weight_loader | 2 | 2/2 |
| output_collect | 2 | 2/2 |
| systolic_top | 2 | 2/2 |
| **합계** | **12** | **12/12** |

## 재사용 패턴 (핵심)

### 1. Parameter 우선

모든 모듈이 `ROWS`, `COLS`, `DATA_WIDTH`, `ACC_WIDTH` 를 받음. iverilog 의 `-P` flag 로 인스턴스화 시 변경 가능.

### 2. Flat vector 인터페이스

`[ROWS*COLS*DW-1:0]` flat 으로 모든 array 신호 전달 — Verilog 2001 호환, 시뮬레이터/합성 도구 거의 모두 지원.

```verilog
// PE(i,j) 의 weight 슬라이스
weight_in_flat[(i*COLS + j + 1)*DATA_WIDTH-1 -: DATA_WIDTH]
```

### 3. 단일 책임

| 모듈 | 책임 |
| --- | --- |
| `pe_array` | mac_prod 인스턴스화 + 배선 *만* |
| `input_skew` | 계단형 지연 *만* |
| `weight_loader` | 1-cycle broadcast strobe *만* |
| `output_collect` | 캡처 + valid pipeline *만* |
| `systolic_top` | 조립 *만* (FSM 없음) |

→ 어떤 모듈만 교체해도 나머지 무영향.

### 4. 외부 dependency 차단

`mac_prod` 는 week3 위치 참조. Makefile 의 `MAC_PROD := $(abspath ../../../week3/.../mac_prod.v)` 한 줄만 수정하면 다른 MAC 으로 교체 가능 (같은 인터페이스 유지 조건).

### 5. VCD dump 가드

- 모든 모듈에 `ifndef NO_VCD_DUMP` guard
- 통합 빌드 시: `-DNO_VCD_DUMP` (sub-module dump 충돌 방지) + `-DDUMP_SYSTOLIC_TOP` (top 만 dump)
- → 합성 / 다중-instance 충돌 / 학습용 standalone 모두 동작

### 6. 공통 cocotb 헬퍼

`cocotb_helpers.py`:
- `tick(dut)` — cocotb 2.x NBA-safe 1 cycle 진행
- `reset_active_low(dut)` — rst_n 표준 리셋
- `pack_flat / unpack_flat / read_signed_vec` — flat ↔ Python list

→ 5개 testbench 가 모두 동일 헬퍼 사용.

## 데이터 흐름 시각화 (4×4 default)

```
testbench 가 매 cycle 1 행 입력
       │
       ▼  act_in_flat (parallel)
  ┌─────────────────┐
  │   input_skew    │  row k 를 k cycle 지연
  └────────┬────────┘
           ▼  staircase activations
  ┌─────────────────┐    load_weight 펄스
  │    pe_array     │ ◄────────────  weight_loader
  │  ┌─┬─┬─┬─┐      │
  │  ├─┼─┼─┼─┤      │
  │  ├─┼─┼─┼─┤      │  16 mac_prod
  │  └─┴─┴─┴─┘      │
  └────────┬────────┘
           ▼  psum_bot_flat
  ┌─────────────────┐
  │  output_collect │  1 cycle 캡처
  └────────┬────────┘
           ▼  result_flat, valid_out
```

## 결과 추출 패턴

`C[m][j]` 가 `history` 의 staircase 위치에 등장:

```python
C[m][j] == history[base + m + j][j]
# base = R (4×4 의 경우 4)
```

즉 같은 `j` 의 column 결과는 연속 cycle 에 나오고, 같은 `m` 의 row 결과는 j 만큼 cycle 차이.

## 한 줄 요약

> `mac_prod` 1개를 16번 인스턴스화 → `pe_array` 격자 → `input_skew` 가 시간 정렬 → `weight_loader` 가 가중치 broadcast → `output_collect` 가 캡처 → `systolic_top` 이 와이어만 잇는다. 12/12 PASS, surfer 로 wave 시각화 가능.
