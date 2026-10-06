# 16-Bit RISC CPU Instruction Set Architecture — Design Outline

## 1. Overview

| Item | Specification |
|---|---|
| Architecture | RISC (Reduced Instruction Set Computer) |
| Instruction Width | 16-bit fixed-length |
| Register File | 16 × 16-bit General Purpose Registers (GPR) |
| Instruction Encoding | Prefix-based hierarchical expansion |
| Byte Order | Little-Endian |

## 2. Instruction Encoding Architecture

The ISA adopts a **hierarchical prefix encoding** scheme. The leading bits (prefix) of each instruction determine its format class, enabling multiple instruction formats within a fixed 16-bit width.

### 2.1 Format Classification

| Prefix | Prefix Length | Format Type | Addressing Mode | Available Slots |
|---|---|---|---|---|
| `00` / `01` / `10` | 2 bit | Register-Address Type | 12-bit address field | 3 × 4096 |
| `11` | 2 bit | Two-Address Type | 6-bit operand field | 63 instructions |
| `11` → `FF` | 8 bit | One-Address Type | 8-bit operand field | 15 instructions |
| `11` → `FF` → `FFF` | 12 bit | Zero-Address Type | 4-bit operand field | 16 instructions |

> **Note:** Prefixes `FF` and `FFF` are sub-encodings within the `11` two-address prefix space, forming a three-level decoding hierarchy: 2-bit → 8-bit → 12-bit.

### 2.2 Design Rationale

Under the 16-bit fixed-length constraint, a single flat opcode field cannot accommodate the full instruction set. Hierarchical prefix encoding is the necessary approach to support a rich instruction set while maintaining fixed-length encoding. Hardware decoding requires at most three stages of prefix discrimination, which is manageable in combinational logic.

## 3. Instruction Format Detail

### 3.1 Register-Address Type (Prefix `00` / `01` / `10`)

```
[15:14]  [13:12]  [11:0]
 Prefix  Sub-Op   12-bit Address Field
```

This format serves two purposes:

#### 3.1.1 Three-Register Arithmetic/Logic Operations

The 12-bit address field is divided into **three 4-bit register addresses**, supporting full GPR access:

```
[15:14]  [13:12]  [11:8]  [7:4]  [3:0]
 Prefix  Sub-Op    Rd      Rs1    Rs2
```

- `Rd`: Destination register (4-bit, full GPR access)
- `Rs1`, `Rs2`: Source registers (4-bit, full GPR access)

#### 3.1.2 Load/Store Operations

The 12-bit address field is partitioned as follows:

```
[15:14]  [13:12]  [11:10]  [9:8]  [7:0]
 Prefix  Sub-Op    Rd       Rb     Offset(imm8)
```

- `Rd` (2-bit): Destination register address — selects from **4 dedicated L/S registers**
- `Rb` (2-bit): Base register address — selects from **4 dedicated base registers**
- `Offset` (8-bit): Immediate offset

**Register Partitioning for Load/Store:**

| Role | Count | Selection | Purpose |
|---|---|---|---|
| Dedicated L/S Registers | 4 | 2-bit address | Data source/destination for load/store |
| Dedicated Base Registers | 4 | 2-bit address | Base address holding for memory access |

> **Design Trade-off:** In the 16-bit instruction width, after allocating 4 bits for the opcode and reserving 8 bits for the immediate offset (to support a meaningful address range), only 4 bits remain for register addressing. Allocating all 4 bits to either the destination or the base register would force the other to be implicit (single register), causing excessive context-saving overhead during conditional branches and memory access. The 4+4 partition is a balanced compromise.

### 3.2 Two-Address Type (Prefix `11`)

```
[15:14]  [13:8]  [7:0]
  `11`   Opcode   Operand Field
```

- `Opcode` (6-bit): Up to 63 distinct instructions (1 slot reserved for prefix extension to `FF`)
- `Operand Field` (8-bit): Register addresses and/or immediate data
- Register addresses in this format use **4-bit full GPR access** (unless otherwise specified)

#### 3.2.1 Upper-Immediate Load (LUI-type)

A subset of the `11` prefix space is reserved for **base register upper-byte load** instructions, analogous to RISC-V's `LUI`:

```
[15:14]  [13:8]  [7:6]   [5:0]
  `11`   LUI-Op   Rb     Imm8
```

- `Rb` (2-bit): Base register address — selects from the 4 dedicated base registers
- `Imm8` (8-bit): Immediate value loaded into the upper 8 bits of the base register

> **Purpose:** Enables construction of full 16-bit addresses via the base+offset addressing scheme used by the Load/Store format.

### 3.3 One-Address Type (Prefix `FF`)

```
[15:8]   [7:4]  [3:0]
  `FF`    Opcode  Operand
```

- Full prefix: 8-bit (`11111111`)
- `Opcode` (4-bit): Up to 15 distinct instructions
- `Operand` (4-bit): Single register address or operand specifier
- Register addresses use **4-bit full GPR access**

### 3.4 Zero-Address Type (Prefix `FFF`)

```
[15:4]    [3:0]
  `FFF`    Opcode
```

- Full prefix: 12-bit (`111111111111`)
- `Opcode` (4-bit): Up to 16 distinct instructions
- No operand field; operations are implicitly defined (e.g., `NOP`, `RET`, `HLT`, `INT`)

## 4. Register File Organization

| Register Group | Count | Address Width | Access Scope |
|---|---|---|---|
| General Purpose Registers (GPR) | 16 | 4-bit | Full access in arithmetic/logic/two-address/one-address formats |
| — Dedicated L/S Registers | 4 | 2-bit | Load/Store data path only |
| — Dedicated Base Registers | 4 | 2-bit | Base address for Load/Store; target of LUI |

> The 4 dedicated L/S registers and 4 dedicated base registers are **overlaid** onto the 16 GPRs (i.e., they are specific indices within R0–R15), not physically separate register files.

## 5. Addressing Modes Summary

| Mode | Description | Used By |
|---|---|---|
| Register Direct | Operand is a GPR | Arithmetic, logic, two-address, one-address |
| Base + Offset | Effective Address = `Base Register + imm8` | Load / Store |
| Immediate | Operand is an inline constant | LUI, arithmetic with immediate |
| Implicit | No explicit operand | Zero-address instructions |

## 6. Design Constraints and Trade-offs

1. **16-bit fixed-length** limits the opcode + operand space, necessitating hierarchical prefix encoding.
2. **Load/Store register partitioning (4+4)** is a compromise under the 4-bit register address constraint after allocating 8 bits to the immediate offset and 4 bits to the opcode.
3. **LUI-type instructions** are embedded within the `11` two-address prefix space rather than using a separate top-level prefix, preserving prefix encoding orthogonality.
4. **Little-Endian encoding** is adopted for byte-level memory access consistency.
