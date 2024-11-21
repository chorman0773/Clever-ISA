# Assembly Syntax Recommendations

The following document indicates recommended syntax for assembly

## Part A - Basic instructions, Syntax

### General Syntax

The following abnf describes the general syntax of the assembly. Lexical nonterminals are specified in all capital letters. Syntactic non-terminals are specified in all lowercase letters. 
In Syntactic Non-terminals, each element (terminal or non-terminal) may be separated by any amount of whitespace other than newline or carriage return:
```abnf
LABEL-START := %x41-5A / %x61-7A / "_" / "." / "$" / "@"

LABEL-PART := <LABEL-START> / %x30-39

LABEL := <LABEL-START> [*<LABEL-PART>]

comment := ("#" / ";" / "//") [*<NOT-NEWLINE>]

NEWLINE := [<comment>] [%xD] %xA

MNEMONIC := (%x41-5A / %x61-7A / "_") [*(%x41-5A / %x61-7A / %x30-39 / "_")]

VARIANT := ([*%x30-39] <MNEMONIC>) / (["-"] *%x30-39)

OPCODE-NAME := <MNEMONIC> ["." <VARIANT>]

size-specifier := "byte" / "short" / "half" / "single" / "double" / "quad"

REGISTER-NAME := (%x41-5A / %x61-7A) [*(%x41-5A / %x61-7A / %x30-39)] # Note: See Appendix C for a full list

HEX-DIGIT := %x30-39 / %x41-46 / %x61-66

DEC-LITERAL := *(%x30-39) ["d"]

HEX-LITERAL := ("0x" *<HEX-DIGIT>) / (%x30-39 [*<HEX-DIGIT>] "h")

OCT-LITERAL := ("0o" *(%x30-37)) / (*(%x30-37) "o")

BIN-LITERAL := ("0b" *(%x30-31)) / (*(%x30-31) "b")

int-literal := ["-"] (<DEC-LITERAL> / <HEX-LITERAL> / <OCT-LITERAL> / <BIN-LITERAL>)

immediate-value := <int-literal> / <label> [("+"/"-") <int-literal>]

immediate-operand := (<immediate-value> ["+" "ip"]) / "rel" <immediate-value>

memory-address-base := <REGISTER-NAME> / <immediate-operand> / "abs" <immediate-value>
memory-address-offset := ([<DEC-LITERAL> "*"] <register-name>) / (<register-name> ["*" <DEC-LITERAL>])
memory-address := <memory-address-base> 
    / (<memory-address-offset> / <memory-address-base>) "+" <register-name> 
    / <register-name> "+" (<memory-address-base>/<memory-address-offset>)

memory-reference := "[" [<size-specifier>] <memory-address>"]"

base-operand := <register-name> / <immediate-operand> / <memory-reference>

operand := <size-specifier> [<base-operand>] / <base-operand>

instruction := [*(<label>":")] <opcode-name> [<operand> [*("," <operand>)]]

directive-name := "." (%x41-5A / %x61-7A / "_" / ".") [*(%x41-5A / %x61-7A / %x30-39 / "_" / ".")]

directive-line := <directive-name> [*<NOT-NEWLINE>]

program-line := [<instruction> / <label> ":" / <directive-line>] <newline>

program := *<program-line>

```

`NOT-NEWLINE` means any valid character other than `%x0A` or `%x0D`. 

The `LABEL-PART` and `LABEL-START` lexical non-terminals may be modified when using Unicode XID support defined in Part D.

### Labels

A label may preceed any instruction, followed by a `:`. 
A label matches the following regex `[A-Za-z_.$@][A-Za-z0-9_.$@]*`.

Labels starting with a decimal number are reserved - their support is documented in Part D - Recommended Syntax Extensions for Inline Assembly in Systems Languages. Additionally, support for Unicode XID is documented there.


### Mnemonics

Each instruction is assigned a mnemonic based on the name in the specification. The full list of instructions in the version is listed in Appendix A.

### Instruction Variants

Instructions that vary in the `h` field will typically include this variation as a suffix separated  by a `.`. 

The style depends on the exact field. For a single bit, this is typically the name of the bit. 

The following suffixes are defined as follows:
* Registers - not specified as a suffix
    * For GPR Specializations, the register in the appropriate operand position with no size specifier
    * For other instructions, the register name placed in the first operand position.
* Size Control: The explicit name of the size control. There is no default. 
    * If present on an instruction with 0 operands that is not a prefix, written in the operand position instead of as a suffix.
* `f`, `l` , `w` (for `mul`/`imul`): The letter being present sets the bit and absent clears it.
* `w` (in shift instructions): The letter `w` indicates the bit being set, and the letter `s` indicates the bit being clear.
* Condition Code: Condition Code Suffix immediately following the mnemonic (no `.` in the suffix)
* `u`: The letter `u` indicates the bit being set, and the letter `s` indicates the bit being cleared.
* `i`: The letter `i` being present indicates the bit being set, and the letter `i` being absent indicates the bit being clear.
* `ss` (in `movfx` and `movxf`) is written as the `1<<ss`. 
* `w` (in conditional branches): The signed 4-bit integer value of the branch weight

All supported variants are listed in Appendix B.

### Operand Syntax

#### Size Specifier

Immediately to any operand, or before the address of a memory reference, a size specifier may appear. The specifier is one of the following:
* `byte` (size 1)
* `short` (12 bits)
* `half` (size 2)
* `single` (size 4)
* `double` (size 8)
* `quad` (size 16)

`quad` may only appear on a vector register or an immediate/memory operand.
`short` may only appear on an immediate operand.

Omitting a size specifier, except on a register operand, yields an unspecified but valid width. 
When applied to an immediate operand or a memory address, the width is sufficient to encode the value of the operand.

When omitted on a vector register pair, `quad` size is inferred.  
When omitted on a general purpose register for an instruction that supports GPR specialization for that operand:
* If the operand is the only operand, `double` is inferred,
* If there is another operand, the same operand size is inferred.

In all other cases, the width is unspecified but valid for the instruction and register type.

#### Register Operands

A Register operand is specified by the name of the register specified in the specification. Any alias name is accepted.
Full List of valid register names (excluding regno names) is specified in Appendix C.
In addition, a register can be specified as `r{num}` where `{num}` is the register number. 
This syntax cannot be used to refer to a vector pair register.

A vector register operand can be written as `v{num}` where `{num}` is the vector pair number. The vector pair `v{num}` refers to the pair of `v{num}l` and `v{num}h`.

#### Immediate Operands

An immediate operand may be written as one of the following forms:
* An integer literal, written in decimal, hexadecimal, octal, or binary