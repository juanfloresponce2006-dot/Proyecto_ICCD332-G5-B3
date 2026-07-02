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
- 2 Splitters (16-bit wide) to unpack and repack the data buses
- 16 AdderSubtractor_1bit

Traduce tal cual al español 
