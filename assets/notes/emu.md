# Emulation

Types of state/information - "architected" state - programmer can touch or view it

general-purpose registers (GPRs), memory, control register, condition codes for
branching or conditional control flow, etc.

non-architected state: (we _typically_ dont have to worry about these!)
- values on the address pins of the CPU
- CPU caches


# Emulator needs to concern itself with...
## architected state
- registers (this includes stack ptr)
- PC (program counter/instruction pointer)
- memory (program code, data, stack, etc.)
- architected IRQ (interrupt) state
- condition codes (CC)
- control registers (program status word PSW), x86 - cr0, cr1, cr2

- model-specific registers on x86 chips are an example of a gray area: non-architected, but we'd still have to emulate them


## emulator example

### Our ISA constraints
- 16-bit addresses
- word addressable (word size is 16 bits)
- 8 GPRs
- fixed-width instructions
- little endian ("low byte low address")



```
PHYSICAL MEMORY MAP

|-----------------------| 0xffff
  AVAILABLE RAM
  ...
  ...
  ...
  ..
|------------------------|
  CARTRIDGE ROM
...

|------------------------| <--- PC reset value (b000)
|    GPU (a000)
|------------------------|
|  AVAILBLE RAM          |
|------------------------|
| KEYBOARD               |  <----MMIO regions
|------------------------|
|   NETWORK CARD         |
|------------------------| 0x0000
```

====================================

```C
#define NUM_GPRS 8
#define RAM_SZ 65K
#define RAM_RESET_VALUE 0xffff
#define PC_RESET_VAL 0xb000

// emulated machine state
struct cpu_state {
    uint16_t pc; // program counter -- its an address
    uint16_t regs[NUM_GPRS];

    uint16_t ram[RAM_SZ];
    struct psw {
        uint8_t current_ring : 1;
        uint8_t flags : 3;
        uint8_t irq_raised : 1;
        uint8_t reserved : 11;
    } 
}

// populating values based on "reset state"
struct cpu_state * init() {

    struct cpu_state * cpu = malloc(sizeof(struct cpu_state));
    
    // set reset values
    cpu->psw.current_ring = 0;
    cpu->psw.flags = 0;
    cpu->psw.irq_raised = 0;
    cpu->psw.reserved = 0;

    for (int i = 0; i < RAM_SZ; i++) {
        cpu->ram[i] = RAM_RESET_VALUE;
    }

    for (int i = 0; i < NUM_GPRS ;i++) {
        cpu->regs[i] = 0;
    }

    cpu->pc = PC_RESET_VALUE;

    return cpu;
}

inline uint16_t read_mem(struct cpu_state * cpu, uint16_t addr) {

    if (addr_in_device_range(addr)) {
        device_read(cpu, addr);
    return cpu->ram[addr];
}

inline void write_mem(struct cpu_state * cpu, uint16_t addr, uint16_t value) {
    cpu->ram[addr] = value;
}

#define OP_ADD 0x00
#define OP_NOT 0x01
#define OP_AND 0x02

// opcode dst reg, src1, src2
// ADD R0, R1, R2

enum instr_type {
    ADD,
    NOT,
    AND,
}

struct instr {
    enum instr_type instr;
    uint8_t sr1;
    uint8_t sr2;
    uint8_t dst;
}

//  instruction encoding for ADD instruction
// |opcode (4 bits) | dst (3) | sr1 (3) | sr2 (3) | 3 bits unused|
struct instr * decode (uint16_t raw) {

    struct instr * instr = malloc(sizeof(struct instr));

    uint8_t opcode = raw >> 12;

    switch (opcode) {
        case OP_ADD:
            instr->instr = ADD;
            instr->dst = (raw << 4) >> 13;
            instr->sr1 = (raw << 7) >> 10;
            instr->sr2 = (raw << 10) >> 7;
            break;
        case OP_NOT:
            break;
        case OP_AND:
            break;
        default:
            break;
    }


}

void (*)exec_routines[NUM_INSTRS] = {
    exec_add,
    exec_sub,
    exec_not,
    exec_and,
    ...
    }


void exec_add (struct cpu_state * cpu) {
    cpu->regs[instr->dst] = cpu->regs[instr->sr1] + cpu->regs[instr->sr2];
    cpu->pc += 1;
}


// decode-and-dispatch style emulation
void emulate(struct cpu_state * cpu) {

    // instruction cycle goes on forever
    while (1) {
        // instruction fetch
        uint16_t raw_instr = read_mem(pc)

        // decode
        struct instr * instr = decode(raw_instr);

        // execute
        switch (instr->instr) {
            case ADD:
                // add 2 src registers and store it in dst register
                exec_routines[instr](cpu);
                break;
            case SUB:
            ..
            .
            default:
                break;
        }

    }

}

int main () {
    struct cpu_state * cpu = init();
    emulate(cpu);
}
```

