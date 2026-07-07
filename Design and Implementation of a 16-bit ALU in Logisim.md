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

## ii) Unidad de Desplazamiento de 16 bits (Shifter)

### Unidad_Desplazamiento_16bit

Desplazar bits hacia la izquierda o hacia la derecha dentro de un número binario es el equivalente en hardware de multiplicar o dividir por potencias de 2. En un desplazamiento lógico de una posición, cada bit se mueve a un índice adyacente. El bit que cae por el borde del registro se descarta permanentemente, y la nueva posición vacía se rellena con una constante cero.

En lugar de utilizar compuertas lógicas activas complejas, un desplazamiento fijo de 1 bit se puede implementar en su totalidad mediante el enrutamiento físico de cables. Al desempaquetar el bus de entrada de 16 bits en cables individuales utilizando divisores (*splitters*), el sistema puede reasignar físicamente las líneas de datos para lograr el desplazamiento de forma instantánea, sin retardo de propagación. Luego, se utiliza un multiplexor para que actúe como árbitro, seleccionando qué ruta cableada (Izquierda o Derecha) se envía a la salida final en función de una señal de control.

Para resolver esto, la `Unidad_Desplazamiento_16bit` realiza las siguientes operaciones de enrutamiento:

- **Cuando `Sel_Shift = 0` (Desplazamiento Lógico a la Izquierda - SHL):** Cada bit de entrada se mueve una posición hacia arriba (`Result[i+1] = A[i]`). El bit menos significativo se rellena con un cero lógico (`Result[0] = 0`), y el bit original `A[15]` se descarta.
- **Cuando `Sel_Shift = 1` (Desplazamiento Lógico a la Derecha - SHR):** Cada bit de entrada se mueve una posición hacia abajo (`Result[i-1] = A[i]`). El bit más significativo se rellena con un cero lógico (`Result[15] = 0`), y el bit original `A[0]` se descarta.

Esto se implementa con el hardware que se detalla a continuación:

- 1 pin de entrada de 16 bits (`A`)
- 1 pin de entrada de 1 bit (`Sel_Shift`)
- 1 pin de salida de 16 bits (`Out_Shift`)
- 3 Divisores (*Splitters* de 16 bits de ancho) para desempaquetar el bus principal y volver a empaquetar los dos buses desplazados.
- 2 Constantes de 1 bit (Valor `0x0`) para rellenar las posiciones de bits vacías.
- 1 Multiplexor (Ancho de datos de 16 bits, 1 bit de selección, Habilitación desactivada) para elegir la ruta de la operación.
## iii) Unidad Lógica de 16 bits (Operaciones Booleanas)

### Unidad_Logica_16bit

Mientras que la unidad aritmética procesa los bits como valores numéricos, la unidad lógica trata cada bit de forma independiente como un interruptor booleano (Verdadero/Falso). Esta unidad es fundamental para realizar operaciones de enmascaramiento de bits (*masking*), comprobación de condiciones y manipulación de datos a bajo nivel en la arquitectura del procesador.

El diseño implementa las cuatro compuertas lógicas fundamentales operando en paralelo sobre buses de 16 bits. Dado que Logisim soporta el procesamiento de múltiples bits de forma nativa en sus compuertas, no es necesario separar los cables individualmente, lo que reduce la latencia teórica del circuito. Un multiplexor actúa como selector de ruta para determinar qué resultado lógico se envía al bus principal.

Para resolver esto, la `Unidad_Logica_16bit` realiza las siguientes operaciones bit a bit simultáneamente:

- `Resultado_AND = A AND B` (Utilizado para apagar bits específicos).
- `Resultado_OR = A OR B` (Utilizado para encender bits específicos).
- `Resultado_XOR = A XOR B` (Utilizado para comparar igualdad o invertir condicionalmente).
- `Resultado_NOT = NOT A` (Utilizado para la inversión total de los estados de la entrada A).

El control de la unidad se define mediante la señal de 2 bits `Sel_Logico`:
- `00`: Salida = AND
- `01`: Salida = OR
- `10`: Salida = XOR
- `11`: Salida = NOT

Esto se implementa con el hardware que se detalla a continuación:

- 2 pines de entrada de 16 bits (`A`, `B`)
- 1 pin de entrada de 2 bits (`Sel_Logico`)
- 1 pin de salida de 16 bits (`Out_Logico`)
- 1 compuerta AND (16 bits)
- 1 compuerta OR (16 bits)
- 1 compuerta XOR (16 bits)
- 1 compuerta NOT (16 bits)
- 1 Multiplexor (Ancho de datos de 16 bits, 2 bits de selección)
