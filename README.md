# Bit-Accurate FP16 Multiplier-Accumulator

This project implements a bit-accurate IEEE-754 half-precision floating-point multiplier-accumulator (MAC) in SystemVerilog.

The design consists of a pipelined FP16 multiplier and a pipelined FP16 adder. It is verified using a 13-element dot product and compared against a Python NumPy FP16 golden model.

## Architecture

```text
FP16 inputs
    |
    v
3-stage multiplier
    |
    v
3-stage adder / accumulator
    |
    v
Accumulated FP16 result
```

The multiplier output is connected directly to the adder input. The adder feeds its result back as the running accumulator value.

## FP16 Format

Each input and output is a 16-bit IEEE-754 half-precision value:

```text
| Sign: 1 bit | Exponent: 5 bits | Fraction: 10 bits |
```

The exponent bias is 15. For normalized values, the hidden leading `1` is restored before arithmetic, creating an 11-bit mantissa.

## FP16 Multiplier Pipeline

The multiplier module is `fp16_mult53` and has a 3-cycle latency.

### Stage 1: Multiply

- Extracts the sign, exponent, and fraction fields.
- Restores the hidden leading `1` for normalized operands.
- Calculates the result sign using XOR.
- Calculates the raw exponent using the FP16 bias of 15.
- Multiplies two 11-bit mantissas to produce a 22-bit intermediate product.

```text
result_sign = sign_a ^ sign_b
result_exp  = exp_a + exp_b - 15
mantissa_product = mantissa_a * mantissa_b
```

### Stage 2: Normalize

The product is inspected and shifted to restore the normalized `1.xx` representation. The exponent is incremented when the product requires a right shift.

### Stage 3: Round and Pack

The multiplier extracts guard, round, and sticky bits and applies Round-to-Nearest-Even. If rounding overflows the mantissa, the exponent is incremented before packing the final 16-bit result.

## FP16 Adder Pipeline

The adder module is `fp16_add53` and has a 3-cycle latency. It receives the multiplier result and adds it to the running accumulator.

### Stage 1: Align and Swap

- Compares the exponents and mantissas of the two operands.
- Selects the operand with the larger magnitude.
- Right-shifts the smaller mantissa to align the binary points.
- Generates a sticky bit from discarded bits to preserve rounding information.

### Stage 2: Add or Subtract

The operand signs determine the arithmetic operation:

```text
same signs      -> mantissa addition
different signs -> mantissa subtraction
```

The aligned mantissas include extra precision bits for guard, round, and sticky information.

### Stage 3: Normalize and Round

The result is normalized before being packed into FP16 format.

- Overflow is handled by shifting the result right and increasing the exponent.
- A leading-zero counter handles cancellation after subtraction.
- The mantissa is shifted left when necessary and the exponent is reduced accordingly.
- Round-to-Nearest-Even is applied using guard, round, and sticky information.

The leading-zero correction is important when subtracting two values with similar magnitudes, because the result may contain several leading zeros.

## Modules

### `fp16_mult53`

Pipelined FP16 multiplier.

```text
Inputs:  clk53, rst53, a53, b53
Output:  prod53
Latency: 3 clock cycles
```

### `fp16_add53`

Pipelined FP16 adder and accumulator datapath.

```text
Inputs:  clk53, rst53, a53, b53
Output:  sum53
Latency: 3 clock cycles
```

### `tb_mac53`

SystemVerilog testbench that drives the 13-element dot product, waits for the pipeline to flush, and prints hardware and golden-model results.

## Verification Test Case

The design is tested with the following vectors:

```text
A = (0.1, 0.2, 0.25, -0.3, 0.4, 0.5, 0.55, 0.6,
     -0.75, 0.8, 0.875, 0.9, 0.7)

B = (0.25, -0.9, 0.125, 0.8, 0.875, -0.75, 0.3, 0.6,
     0.1, 0.2, -0.4, 0.55, -0.5)
```

The final expected accumulated value is:

```text
0x2420
```

Expected intermediate values:

```text
Index  Product  Accumulator
0       0x2666   0x2666
1       0xB1C2   0xB0F5
2       0x2800   0xAFEA
3       0xB3AE   0xB5D2
4       0x3599   0xA320
5       0xB600   0xB639
6       0x3147   0xB32B
7       0x35C3   0x305B
8       0xACCC   0x2BD4
9       0x311E   0x3313
10      0xB599   0xB01F
11      0x37EB   0x35DC
12      0xB59A   0x2420
```

## Pipeline Timing

The multiplier requires 3 clock cycles and the adder requires another 3 clock cycles. Therefore, the total input-to-accumulator latency is 6 clock cycles.

The testbench waits for six rising edges and adds a one-nanosecond delay before sampling the output:

```systemverilog
repeat (6) @(posedge clk53);
#1;
```

## Simulation

Compile the SystemVerilog source files with Icarus Verilog:

```bash
iverilog -g2012 -o mac_simulation.vvp fp16_mult.sv fp16_add.sv tb_mac.sv
```

Run the simulation:

```bash
vvp mac_simulation.vvp
```

The testbench generates a waveform file named `dump.vcd`. Open it with GTKWave:

```bash
gtkwave dump.vcd
```

The exact source filenames may differ depending on how the project files are organized. Include the multiplier, adder, and testbench files in the compile command.

## Python Golden Model

The reference model uses NumPy FP16 arithmetic:

```python
prod = np.float16(a_value) * np.float16(b_value)
accumulator = np.float16(accumulator + prod)
```

The hexadecimal representation of each FP16 value is used for bit-accurate comparison with the SystemVerilog outputs.

## Requirements

- SystemVerilog simulator with `always_ff`, `always_comb`, and unpacked array support
- Icarus Verilog, Verilator, Questa, or another compatible simulator
- Python 3 with NumPy for the golden model
- GTKWave for optional waveform inspection

## Notes

The implementation focuses on bit-accurate normalized FP16 arithmetic for the provided verification vectors.

Full production IEEE-754 support may additionally require explicit handling for NaN, infinity, subnormal values, overflow, and underflow.

<img width="871" height="742" alt="Screenshot 2026-09-15 233348" src="https://github.com/user-attachments/assets/4e7a435c-d12b-4948-863a-0340acce943e" />
