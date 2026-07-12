# Design and Implementation of a 16-bit ALU in Logisim

## i) Base 16-bit Adder/Subtractor (2's Complement)

### Full_Adder_1bit

Adding 2 bits presents four possible scenarios:

1. `0 + 0 = 0`
2. `1 + 0 = 1`
3. `0 + 1 = 1`
4. `1 + 1 = 10`

When two bits are added, a single bit position can only store a maximum value of 1. To preserve the positional value of the operation, we must account for a mathematical overflow. By storing and transmitting a Carry signal to the next most significant bit in the adder chain, the system successfully propagates this overflow throughout the entire calculation, preventing data loss.

To solve this, the `Full_Adder_1bit` performs the following Boolean operations:

- `Sum = A XOR B XOR Cin`
- `Cout = (A AND B) OR (Cin AND (A XOR B))`

This can be done with the hardware shown below:

- 3 pins of 1 bit
- 2 XOR gates
- 2 AND gates
- 1 OR gate

### AdderSubtractor_1bit

Performing both addition and subtraction in hardware requires binary math. Digital circuits subtract by adding the 2's complement of the second number to the first. The rule for 2's complement is: Invert the bits of the number you want to subtract (NOT), then add 1.

The `AdderSubtractor_1bit` uses an inversion layer before any arithmetic. It uses a control signal (`Op`) to decide what to do with the `B` input:

- When `Op = 0` (Addition): The B input passes through as introduced.
- When `Op = 1` (Subtraction): The B input is inverted (NOT).

To solve this, the `AdderSubtractor_1bit` performs the following Boolean operations:

- `B_internal = B XOR Op`
- `Sum = A XOR B_internal XOR Cin`
- `Cout = (A * B_internal) + (Cin * (A XOR B_internal))`

This can be done with the hardware shown below:

- 4 input pins of 1 bit (A, B, Cin, Op)
- 2 output pins of 1 bit (Sum, Cout)
- 1 XOR gate (The Inversion Layer)
- 1 Full_Adder_1bit subcircuit (The Math Engine)

### AdderSubtractor_16bit

To process 16-bit integers, we must cascade multiple 1-bit operators together into a unified architecture.

To solve this, the `AdderSubtractor_16bit` utilizes a "Ripple Carry" architecture. By connecting sixteen `AdderSubtractor_1bit` modules in series, the system allows the Carry-out (`Cout`) of one bit to physically ripple into the Carry-in (`Cin`) of the next, preserving positional value across all 16 bits. Besides, to complete the "+ 1" requirement of 2's complement subtraction, the `Op` control signal is routed directly into the `Cin` of the very first bit (Bit 0).

This can be read as:

- For each bit *i* (from 0 to 15): `Result[i] = A[i] ± B[i]`
- `Cin of Bit 0 = Op` (Inputs the required +1 when subtracting)
- `Cin of Bit i = Cout of Bit i-1 `(The Ripple Carry)

This can be done with the hardware shown below:

- 2 input pins of 16 bits (A, B)
- 1 input pin of 1 bit (Op)
- 1 output pin of 16 bits (Result)
- 3 Splitters (16-bit wide) to unpack and repack the data buses
- 16 AdderSubtractor_1bit subcircuits

## ii) Logic and Shift Units

### LogicUnit_16bit

Performing logic operations requires evaluating the binary states of two operands bit by bit. The logic unit provides fundamental bitwise operations essential for masking and evaluating data structures at the machine level. 

To solve this, the `LogicUnit_16bit` performs the following Boolean operations:

- `Out_AND = A AND B`
- `Out_OR = A OR B`
- `Out_XOR = A XOR B`
- `Out_NOT = NOT A`
- `Out_Logico = MUX(Out_AND, Out_OR, Out_XOR, Out_NOT, Sel_Logico)`

This can be done with the hardware shown below:

- 2 input pins of 16 bits (A, B)
- 1 input pin of 2 bits (Sel_Logico)
- 1 output pin of 16 bits (Out_Logico)
- Logic Gates (16-bit wide): AND, OR, XOR, NOT
- 1 Multiplexer (4-to-1)

### ShiftUnit_16bit

Bit shifting is mathematically equivalent to multiplying or dividing by powers of two, making it computationally faster than passing data through an arithmetic engine. This module displaces the bit array laterally based on control signals.

To solve this, the `ShiftUnit_16bit` performs the following operations:

- `Out_Shift = Shift(A, Sel_Shift)` (Logically shifting bits based on the selector code)

This can be done with the hardware shown below:

- Input pins for operand A and selection controls
- Splitters (to separate bits for routing)
- Multiplexers (to choose the shifted output)

## iii) Status Flags and Integrated ALU

### Flags

Status flags are a set of control bits that provide information about the result of the last operation executed by the ALU. These signals do not modify the operation's result; instead, they provide additional context used by the processor to make decisions, such as performing conditional jumps, detecting arithmetic errors, or determining if a result is negative.

The design implements four fundamental flags: Zero (Z), Carry (C), Overflow (V), and Sign (S). Each is calculated independently from the result obtained by the arithmetic unit, allowing all flags to be generated simultaneously without interfering with one another.

To solve this, the `Flags` unit performs the following operations:

- `Zero = 1 IF Result == 0x0000 ELSE 0`
- `Carry = CarryOut`
- `Overflow = CarryInMSB XOR CarryOut`
- `Sign = Result[15]`

This can be done with the hardware shown below:

- 1 input pin of 16 bits (Result)
- 2 input pins of 1 bit (CarryInMSB, CarryOut)
- 4 output pins of 1 bit (Zero, Carry, Overflow, Sign)
- 1 Comparator (16-bit) to verify if the result equals zero
- 1 Constant of 16 bits with a value of 0x0000
- 1 Splitter (16-bit) to extract the most significant bit
- 1 XOR gate (1-bit) to calculate the Overflow flag

### ALU_16bit_Principal

The Arithmetic Logic Unit (ALU) constitutes the functional core of the digital system, being responsible for executing arithmetic, logic, and shift operations on 16-bit data. To achieve a modular design and facilitate both debugging and hardware reuse, the ALU was built by integrating three independent functional units: an arithmetic unit, a logic unit, and a shift unit.

Each of these units internally performs its own operations and delivers a single 16-bit result. Subsequently, a main multiplexer selects which of these results will be sent to the ALU output according to the selected functional unit (`Sel_Unidad`). 

Once the corresponding operation is selected, the result is sent to the Flags module, which simultaneously calculates the Zero, Carry, Overflow, and Sign flags. The Zero and Sign flags are valid for any operation, while Carry and Overflow are only meaningful during arithmetic operations.

To solve this, the `ALU_16bit_Principal` performs the following operations:

- `Arith_Res = AdderSubtractor(A, B, Sel_Arit)`
- `Logic_Res = LogicUnit(A, B, Sel_Logico)`
- `Shift_Res = ShiftUnit(A, Sel_Shift)`
- `Result = MUX(Arith_Res, Logic_Res, Shift_Res, Sel_Unidad)`

This can be done with the hardware shown below:

- 2 input pins of 16 bits (A, B)
- 1 input pin of 2 bits (Sel_Unidad)
- 1 input pin of 1 bit (Sel_Arit)
- 1 input pin of 2 bits (Sel_Logico)
- 1 input pin of 1 bit (Sel_Shift)
- 1 AdderSubtractor_16bit subcircuit
- 1 LogicUnit_16bit subcircuit
- 1 ShiftUnit_16bit subcircuit
- 1 Main Multiplexer (16-bit) to select the corresponding unit's output
- 2 Multiplexers (1-bit) to control the Carry and CarryInMSB signals
- 1 Flags subcircuit
- 1 output pin of 16 bits (Result)
- 4 output pins of 1 bit (Zero, Carry, Overflow, Sign)

## iv) IEEE 754 Half-Precision Floating Point Unit (FPU)

### IEEE754_Extractor

The IEEE 754 standard defines the format used by processors to represent floating-point numbers. To reduce circuit complexity without losing standard compatibility, the IEEE 754 Binary16 (Half Precision) variant was implemented, which uses a total of 16 bits distributed into three fields: one sign bit, five exponent bits, and ten mantissa bits.

The function of the extractor module is to separate a 16-bit data input into its three fundamental components so they can be processed by the different stages of the floating-point unit. Although the standard stores only the ten fractional bits, internal arithmetic operations consider a Hidden Bit with a logical value of one for all normalized numbers.

To solve this, the `IEEE754_Extractor` performs the following operations:

- `Sign = IEEE_In[15]`
- `Exponent = IEEE_In[14:10]`
- `Mantissa = IEEE_In[9:0]`
- `Real_Mantissa = 1.Mantissa` (Incorporating the Hidden Bit)

This can be done with the hardware shown below:

- 1 input pin of 16 bits (IEEE_In)
- 3 output pins (Sign of 1 bit, Exponent of 5 bits, Mantissa of 10 bits)
- 1 Splitter (16-bit) to separate the input data into individual signals
- 1 Splitter configured to rebuild the 5-bit exponent bus
- 1 Splitter configured to rebuild the 10-bit mantissa bus

### IEEE754_Comparator

Before adding or subtracting floating-point numbers, their radix points must be aligned. This module compares the exponents of both operands to identify the larger number and calculates the absolute distance between their magnitudes.

The circuit automatically reorganizes the inputs using multiplexers, ensuring the larger exponent is always connected to the first input of the subtractor and the smaller to the second. This guarantees the exponent difference is always positive. It also generates a `MayorEsA` control signal for subsequent stages.

To solve this, the `IEEE754_Comparator` performs the following operations:

- `Mayor_Es_A = Exp_A > Exp_B`
- `ExpMayor = MAX(Exp_A, Exp_B)`
- `ExpMenor = MIN(Exp_A, Exp_B)`
- `Diferencia = |Exp_A - Exp_B|`

This can be done with the hardware shown below:

- 2 input pins of 5 bits (ExpA, ExpB)
- 2 input pins of 10 bits (MantA, MantB)
- 1 Magnitude Comparator (5-bit)
- 2 Multiplexers (5-bit) to automatically select the larger and smaller exponent
- 1 OR gate to generate the MayorEsA signal
- 1 Subtractor (5-bit) to calculate the absolute difference
- 1 output pin of 1 bit (MayorEsA)
- 2 output pins of 5 bits (ExpMayor, ExpMenor)
- 1 output pin of 5 bits (Diferencia)

### IEEE754_Aligner

After determining which operand has the larger exponent, the next step is to equalize both exponents. The mantissa associated with the smaller exponent must be shifted to the right as many positions as indicated by the exponent difference.

This module first reconstructs the 11-bit real mantissas by incorporating the Hidden Bit. When the exponents are equal, the module performs an additional comparison between the mantissas to correctly identify the larger magnitude. The circuit automatically sorts the mantissas into `MantMayor` and `MantMenor`, shifts the smaller mantissa, and generates a `MayorFinal` signal.

To solve this, the `IEEE754_Aligner` performs the following operations:

- `Real_Mantissa = 1.Mantissa`
- `MantMayor = MAX_MANTISSA(MantA, MantB)`
- `MantMenor = MIN_MANTISSA(MantA, MantB)`
- `MantMenor_Aligned = MantMenor >> Diferencia`
- `MayorFinal = 1 IF Operando_A_Mayor ELSE 0`

This can be done with the hardware shown below:

- 1 input pin of 5 bits (ExpMayor)
- 1 input pin of 5 bits (Diferencia)
- 2 input pins of 1 bit (MayorEsA, ExpIguales)
- 2 input pins of 10 bits (MantA, MantB)
- 2 Splitters to reconstruct the 11-bit mantissas
- 1 Comparator (11-bit) to compare mantissas when exponents are equal
- 1 OR gate and 1 Multiplexer (1-bit) to generate MayorFinal
- 2 Multiplexers (11-bit) to sort the mantissas
- 1 Logical Shifter (11-bit) to shift the smaller magnitude mantissa
- 1 output pin of 5 bits (ExpOut)
- 2 output pins of 11 bits (MantMayor, MantMenor)
- 1 output pin of 1 bit (MayorFinal)

### IEEE754_MantissaALU

Once spatially aligned, the mantissas are passed into a dedicated arithmetic engine. This unit must decide whether to perform addition or subtraction by evaluating the original signs using an XOR gate.

When signs are equal, the mantissas are added directly. When the signs differ, it performs a magnitude subtraction. Because the mantissas were previously sorted, it is always guaranteed that `MantMayor >= MantMenor`, ensuring the result is never negative. A 12-bit temporary mantissa is produced to account for a possible carry.

To solve this, the `IEEE754_MantissaALU` performs the following operations:

- `Effective_Op = SignA XOR SignB`
- `MantTemp = MantMayor + MantMenor_Aligned` (If signs are identical)
- `MantTemp = MantMayor - MantMenor_Aligned` (If signs differ)
- `SignOut = SignA IF (SignA == SignB) ELSE Sign_of_MayorFinal`

This can be done with the hardware shown below:

- 2 input pins of 1 bit (SignA, SignB)
- 1 input pin of 1 bit (MayorFinal)
- 2 input pins of 11 bits (MantMayor, MantMenor)
- 1 input pin of 5 bits (ExpOut)
- 1 XOR gate to compare the signs
- 1 Adder (11-bit) and 1 Subtractor (11-bit)
- 2 Splitters to build the 12-bit temporary buses
- 1 Multiplexer (12-bit) to select the arithmetic operation
- 2 Multiplexers (1-bit) to determine the result's sign
- 1 output pin of 12 bits (MantTemp)
- 1 output pin of 1 bit (SignOut)
- 1 output pin of 5 bits (ExpTemp)

### IEEE754_Normalizer

The raw output of the mantissa ALU is rarely in the standard scientific notation format (1.xxxx). The normalizer forces the temporary mantissa to shift left or right until a leading '1' is detected.

The circuit detects the position of the first '1' within the five most significant bits and automatically calculates a shift between zero and four positions. It handles right-shifts if a carry out exists, and left-shifts if subtraction cancellations occurred. It also detects if the operation's result is exactly zero, adjusting the fields appropriately.

To solve this, the `IEEE754_Normalizer` performs the following operations:

- `MantNorm = MantTemp >> 1` (If carry out exists)
- `MantNorm = MantTemp << Adjustment` (If leading zeros exist)
- `ExpNorm = ExpTemp ± Adjustment` (Compensating for the mantissa shift)
- `Zero_Flag = 1 IF MantTemp == 0 ELSE 0`

This can be done with the hardware shown below:

- 1 input pin of 12 bits (MantTemp)
- 1 input pin of 5 bits (ExpTemp)
- 1 input pin of 1 bit (SignOut)
- 1 Splitter (12-bit) to extract individual bits
- 1 Comparator (12-bit) to detect a zero result
- Logic gates (NOT, AND, OR) to determine the shift distance
- 2 Logical Shifters (12-bit) for left and right displacement
- 1 Adder (5-bit) and 1 Subtractor (5-bit) for exponent correction
- 4 Multiplexers to select normalized outputs
- 1 output pin of 11 bits (MantNorm)
- 1 output pin of 5 bits (ExpNorm)
- 1 output pin of 1 bit (SignNorm)

### IEEE754_Packer

After mathematical normalization, the finalized floating-point components must be reassembled into the standard IEEE 754 half-precision word. 

The received mantissa includes the internal Hidden Bit, which is now discarded. The remaining ten fractional bits are located in the least significant part of the result, the normalized exponent occupies the five central bits, and the sign is placed in the most significant bit.

To solve this, the `IEEE754_Packer` performs the following bit operations:

- `IEEE_Out[15] = SignNorm`
- `IEEE_Out[14:10] = ExpNorm`
- `IEEE_Out[9:0] = MantNorm[9:0]` (Discarding the Hidden Bit)

This can be done with the hardware shown below:

- 1 input pin of 1 bit (SignNorm)
- 1 input pin of 5 bits (ExpNorm)
- 1 input pin of 11 bits (MantNorm)
- 1 Splitter (11-bit) to separate the Hidden Bit from the fraction
- 1 Splitter configured in combination mode to rebuild the 16-bit word
- 1 output pin of 16 bits (IEEE754_Out)

### IEEE754_Principal

This circuit constitutes the main floating-point unit of the ALU. Its function is to sequentially integrate all the developed modules to implement an arithmetic operation on numbers represented under the IEEE 754 Binary16 standard.

Unlike previous subcircuits, this unit does not perform operations directly on the data, but coordinates the flow of information between each stage of the floating-point coprocessor, effectively building a physical pipeline.

To solve this, the `IEEE754_Principal` establishes a sequential hardware pipeline:

- `Stage 1: Extractors -> Stage 2: Comparator & Aligner -> Stage 3: MantissaALU -> Stage 4: Normalizer -> Stage 5: Packer`

This can be done with the hardware shown below:

- 2 input pins of 16 bits (Operando A, Operando B)
- 2 IEEE754_Extractor modules
- 1 IEEE754_Comparator module
- 1 IEEE754_Aligner module
- 1 IEEE754_MantissaALU module
- 1 IEEE754_Normalizer module
- 1 IEEE754_Packer module
- 1 output pin of 16 bits (IEEE_Result)

## v) Top-Level Integration

### Circuito_Principal

This circuit corresponds to the top level of the project and represents the complete Arithmetic Logic Unit. Its objective is to integrate both the 16-bit integer ALU and the IEEE 754 Binary16 floating-point coprocessor, providing a single input and output interface for the system.

Both units simultaneously receive the same 16-bit input operands. A hardware switch (`ModoIEEE`) defines which module's answer commits to the main system bus. Because status flags only have meaning for integer operations, they are deactivated when the circuit operates in IEEE mode.

To solve this, the `Circuito_Principal` performs the following operational routing:

- `System_Out = MUX(ALU16bits_Result, IEEE754_Result, ModoIEEE)`

This can be done with the hardware shown below:

- 2 input pins of 16 bits (A, B)
- 1 input pin of 1 bit (Sel_Arit)
- 1 input pin of 2 bits (Sel_Logico)
- 1 input pin of 1 bit (Sel_Shift)
- 1 input pin of 2 bits (SelUnidad)
- 1 input pin of 1 bit (ModoIEEE)
- 1 ALU_16bit_Principal module
- 1 IEEE754_Principal module
- 1 Main Multiplexer (16-bit) to select the final result
- 4 Multiplexers (1-bit) to control the status flags based on the active mode
- 1 output pin of 16 bits (Resultado)
- 4 output pins of 1 bit corresponding to the status flags

## VI) IA generated flow chart

<img width="2816" height="1536" alt="flowchartalu" src="https://github.com/user-attachments/assets/c11b489f-dbe5-4ddc-9aa9-ec3dcc1b26dd" />


> **Figure 1.** *16 BIT ALU IMPLEMENTATION.* **Note.** Image generated by Gemini (Gemini, 2026) with the prompt "Draw a flow chart that neatly describes the behavior of the .cric file attached".
