# VerySimpleCPU
VerySimpleCPU implementation in Logic Circuit

Modified Very Simple CPU is a teaching architecture to explain CPU implementation.
I don't remember the book, if you know the book or are the author (in the same book there is a description of the Relative Simple CPU) tell me the title
and your name, and i will gladly add you in the credits. Thank you.

## Architecture
The architecture is a 6 bit one with only 4 instruction. The main register is 8 bit wide.

The visible register set is made by two registers:  
 * PC (the instruction pointer), 8 bit   
 * AC (the accumulator), 8 bit  
 

Full Register Set (Visible + Hidden)  

PC - 8 bit  -- Visible, Program Counter  
AC - 8 bit  -- ACcumulator register  
DR - 8 bit  -- Data Register, contains the read data  
AR - 8 bit  -- Address Register, output to the memory  
IR - 2 bit  -- Instruction Register, contains the 2 bit opcode  

## Encoding  

The encoding is composed of 2 bit opcode and 6 bit address.

## Instructions  

**Legend**  

    * **@** : address  
    * **<-** : assign  
    * **,** : parallel (e.g. IR <- X, AR <- Y means that both IR and AR get updated in the same clock cycle)  
    * **;** : await next clock  
    * **+** : binary add  
    * **&** : binary and  

### ADD @ : opcode(00) address( 6 bit )  

    1. IR <- MEM[PC], AR <- MEM[PC];  
    2. DR <- MEM[AR];   
    3. AC <- AC + DR;  

### AND @ : opcode(01) address( 6 bit )  

    1. IR <- MEM[PC], AR <- MEM[PC];  
    2. DR <- MEM[AR];   
    3. AC <- AC & DR;  

### JMP @ : opcode(10) address( 6 bit )  

    1. IR <- MEM[PC], AR <- MEM[PC];  
    2. PC <- AR;  

### STA @ : opcode(11) address( 6 bit )  

    1. IR <- MEM[PC], AR <- MEM[PC];  
    2. MEM[AR] <- AC;  

# Implementation

## Internal Bus  

**BUS_0** := connect( AR(read), DR(read), IR(read), MEM_D_IN(write) );                The BUS_0 has only one writer (MEM_D_IN)  
**BUS_1** := connect( PC(read), PC(write), AR(write), MEM_ADR(read) );                The BUS_1 has two writer (AR & PC), and two reader (PC & MEM_ADR)  


## Signals

    AR_RB0          AR <- BUS_0  
    DR_RB0          DR <- BUS_0  
    IR_RB0          IR <- BUS_0  
    PC_RB1          PC <- BUS_1  
    PC_WB1          PC -> BUS_1  
    AR_WB1          AR -> BUS_1  
    ALU_MUX         ALU <- select( AC + DR, AC & DR )  
    AC_LD           AC <- ALU  
    W_MEM           MEM[ MEM_ADR ] <- AC  
    PC_INC          PC <- PC + 1  


## Direct links

    AC -> ALU_PORT_0
    DR -> ALU_PORT_1
    AC -> MEM_D_OUT
    IR[0] -> ALU_SELECT


## Microcontrol States

    INIT : PC_WB1;  
    FETCH: AR_RB0, IR_RB0; 
    ADD0 : ~PC_WB1, AR_WB1, PC_INC;  
    ADD1 : DR_RB0;  
    ADD2 : AC_LD;  
    AND0 : ~PC_WB1, AR_WB1, PC_INC;  
    AND1 : DR_RB0;  
    AND2 : AC_LD;  
    JMP0 : ~PC_WB1, AR_WB1, PC_INC;  
    JMP1 : PC_RB1;  
    STA0 : ~PC_WB1, AR_WB1, PC_INC;  
    STA1 : W_MEM;  
    END  : PC_WB1;  


## States Reduction

ADD0 = AND0 = JMP0 = STA0 -> PMEM  
ADD1 = AND1 -> ALU1  
ADD2 = AND2 -> ALU2  

FETCH : AR_RB0, IR_RB0;  
PMEM: ~PC_WB1, AR_WB1, PC_INC;  
{  
    ALU1: DR_RB0;  
    ALU2: AC_LD;  
    JMP1: PC_RB1;  
    STA1: W_MEM;  
}  
END: PC_WB1, ~AR_WB1;  

## States Encoding  

INIT = 000  
FETCH = 001  
PMEM = 010  
JMP1 = 011  
STA1 = 100  
ALU1 = 101  
ALU2 = 110  
END = 111  

## Opcodes to Microcontrol States  

    00 -> 0, 1, 2, 5, 6, 7, 1  
    01 -> 0, 1, 2, 5, 6, 7, 1  
    10 -> 0, 1, 2, 3, 7, 1  
    11 -> 0, 1, 2, 4, 7, 1  

## Microcontrol Signals  

INC_MPC = 1  
LD_MPC = (PMEM * IJMP') + STA1 + JMP1 + END = ((PMEM * IJMP')' * (STA1') * (END'))'  

Will skip to the targets: STA1, ALU1, END, FETCH  

PMEM (010) to STA1 (100)  
PMEM (010) to ALU1 (101)  
STA1 (100) to END (111)  
JMP1 (011) to END (111)  
END (111) to FETCH (001)  

STA1(100) and ALU1(101) differ by a single bit, so: IO[0] = (PMEM * ISTA)'  
END(111)  
FETCH(001)  

I0 : (PMEM * ISTA)' = O2' * O1 * O0' * (IR0 * IR1)'  
I1 : PMEM' * END' = (O2' * O1 * O0')'  
I2 : END' = (O2 * O1 * O0)'  

    FIELD : 9_SGN_CNTRL  
    8_SGN_CNTRL / AR_RB0, DR_RB0, IR_RB0, PC_RB1, PC_WB1, AR_WB1, AC_LD, W_MEM, PC_INC  
    INIT :          0,      0,      0,      0,      1,      0,      0,      0,      0  
    FETCH:          1,      0,      1,      0,      1,      0,      0,      0,      0  
    PMEM :          0,      0,      0,      0,      0,      1,      0,      0,      1  
    JMP1 :          0,      0,      0,      1,      0,      0,      0,      0,      0  
    STA1 :          0,      0,      0,      0,      0,      0,      0,      1,      0  
    ALU1 :          0,      1,      0,      0,      0,      0,      0,      0,      0  
    ALU2 :          0,      0,      0,      0,      0,      0,      1,      0,      0  
    END  :          0,      0,      0,      0,      1,      0,      0,      0,      0  


Initial Program in RAM  

    @14 = 31 hex(1F)  
    @15 = 71 hex(47)  
    @16 = 0  hex(0)  

    @0 = ADD @14 hex(38)  
    @1 = AND @15 hex(3D)  
    @2 = STA @17 hex(47)  
    @3 = AND @16 hex(41)  
    @4 = JMP @0  hex(02)  
