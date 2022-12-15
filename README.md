# Caravel_pcb

This git repo describes the whole process of PCB design for the Caravel_Board and verifying/testing the MPW_shuttle chips. <br>

# Verification

<b> Change the paths of `PDK_ROOT`, `PDK`, `GCC_PATH` and `GCC_PREFIX` according to the paths on the system you are working on. </b>

Steps to follow :

```
$ git clone https://github.com/DantuNandiniDevi/Caravel_pcb

$ cd Caravel_pcb/verification/verilog/dv/caravel/mgmt_soc/hkspi

$ export PDK_ROOT=/home/ubuntu/OpenLane/pdks

$ export PDK=sky130A

$ export GCC_PATH=/opt/riscv32/bin

$ export GCC_PREFIX=riscv32-unknown-elf

$ make 

$ gtkwave hkspi.vcd 

$ riscv32-unknown-elf-objdump --disassemble-all hkspi.elf > hkspi.disasm  <This to generate the disassembly file>
``` 

After opening gtkwave we can check multiple signals according to our requirements:

-> The memory address and data signals are at `hkspi_tb -> uut -> soc -> soc -> cpu`. Among the signals at the cpu module <b><I> mem_addr, mem_rdata and mem_wdata </b></I> can be accessed to evaluate the signals.

-> To check the Flash signal flow. We can see flash signals <b><I> flash_clk, flash_csb, flash_i0 and flash_i1  </b></I> at every level of the modules and also in the spiflash module.

# Firmware - Asmita/Aman/Ayyappa
## Basic Firmware flow (C code, Header files and conversion to Hex file)
* We write a C code which we want to run on the Caravel chip and include the header file - [defs_mpw-two-mfix.h](https://github.com/DantuNandiniDevi/Caravel_pcb/blob/main/verification/verilog/dv/caravel/mgmt_soc/hkspi/defs_mpw-two-mfix.h) in it. It includes the definitions of all the mprj pins, logic analyzer pins, Flash SPI control register, Counter timer configurations, Bit fields for Counter-timer configuration, SPI Master register Configuration, Bit fields for SPI master configuration, Individual bit fields for the GPIO pad control. All these definition values are as per the [Efabless Caravel “harness” SoC documentation](https://caravel-harness.readthedocs.io/en/latest/). We have run a blink.c code and a simple multiplication.c code.
![](images/initial_flow.png) 
* As shown in the flowchart above, this C code is converted to a hex file through the RISCV32IM GCC GNU Toolchain in the [Makefile](https://github.com/DantuNandiniDevi/Caravel_pcb/blob/main/verification/verilog/dv/caravel/mgmt_soc/hkspi/Makefile).

## Python script flow (Script used to send Hex file to Caravel)
* This hex file is sent to the Caravel flash through the Python Script explained in the flowchart below:
![](images/pythonscript_flow.png) 

Basically, the firmware is used to send the C code, in hex format, with all the default pin and register definitions to the Caravel chip so that it can be run on it.

## Waveform to show start of the Main part of the C code in the .vcd generated according to the disassembled code:
![](images/firmware.png) 
Here we can see that the execution of main starts at around 23.3 us.


# Host to FT232 & FT232 to Host - Nandini/Oishi/Nisha
<p align="center">
<img src="images/ft232_blockdiagram.jpeg">
</p>

## Working of FT232

* The FT232H defaults to asynchronous serial UART interface mode of operation (used in RS232 config). UART interface support for 7 or 8 data bits, 1 or 2 stop bits and Odd/ Even/ Mark/ Space/ No Parity.
* It is compatible with Hi-Speed 480 Mbits/sec and Full-Speed 12Mbits/sec. Data transfer rates is upto 12 MBaud at TTL levels.


## Housekeeping SPI
Housekeeping SPI is a SPI responder that cen be accessed from a remote host through 4-pin serial interface. The SPI implementation is mode 0, with new data on SDI captured on the SCK rising edge, and output data presented on the falling edge of SCK. The SPI pins are shared with user area GPIO.
### Timing diagram of Housekeeping SPI
<p align="center">
<img src="images/housekeepingSPI.jpeg">
</p>
* The command sequence has one command word (8 bits), followed by one address word (8 bits), followed by one or more data words (8 bits).
* The SPI implementation enables data transmission when CSB pin is low. The new data is captured on SDI at the rising edge of SCK and output data is presented on the falling edge of SCK which is received on the SDO line.
* After CSB is set low, the SPI is always in the "command" state, awaiting for a new command.

### Housekeeping SPI modes
There are 2 modes of operations : streaming and n-byte mode
* In **streaming mode** operation, the data is sent or received continuously, one byte at a time, with the internal address incrementing for each byte. Streaming mode operation continues until CSB is raised to end the transfer.
* In **n-byte mode** operation, the number of bytes to be read and/or written is encoded in the command word, and may have a value from 1 to 7 (note that a value of zero implies streaming mode). After n bytes have been read and/or written, the SPI returns to waiting for the next command. No toggling of CSB is required to end the command or to initiate the following command.

There is another mode called [pass though mode](https://github.com/DantuNandiniDevi/Caravel_pcb#pass-thorugh-mode) which is explained in the flash working part of the document.

## Data Flow in the circuit

<p align="center">
<img src="images/FT232_flow.jpeg">
</p>

-> The Bitstream generated from the firmware is passed onto the board using the USB module.

-> The USB module connects to a FT232 module which converts the incoming bitstream into an SPI inferface signals.

-> The SPI interface signals from FT232 are connected to the SPI interface of the caravel.

-> There is a different flash interface to connect flash to caravel. The signals from the FT232 are sent to flash using the housekeeping flash interface through caravel.

Bootcode/Firmware signals at the FT232 input  interface.

![image](https://user-images.githubusercontent.com/62461290/202191303-8644b8c8-6f92-4bfa-a48d-23df9846ae1d.png)

Bootcode/Firmware signals at the caravel interface.

![image](https://user-images.githubusercontent.com/62461290/202191516-01cf639b-6e5f-4fa4-9865-2ebd183ce522.png)

Bootcode/Firmware signals at the External Flash interface.

![image](https://user-images.githubusercontent.com/62461290/202191006-de838f8b-c530-4a78-b42d-e223dad0e640.png)

In the below picture we can observe the signals at every negative clock edge the `flash_io1` data signal is valid. It matches with the `spi_out` register and the CPU read it (as seen in `mem_rdata`) in the consecutive clock cycle and executes accordingly.

![image](https://user-images.githubusercontent.com/62461290/202193770-75c253c2-82b9-44f1-b29c-a3a036019a26.png)



# Flash Working - Saketh/Yathin/Raj
## Pass Thorugh Mode
* There are 2 pass through modes. The first pass through modes used flash in the management project. The second pass through mode corresponds to a secondary flash that can be defined in the user project. 
* In the first pass through mode, the CPU is immediately set into reset. Then sets the flas_csb to low which initiates the data transfer to QSPI.
* Now all the SPI signals(SDI,SCK) are directly applied to QSPI Flash pins flash io0 and flash clk respectively. QSPI flash io1 is directly connected to SDO of HKSPI.
* Once the data transfer is complete, the CSB pin of both QSPi and HKSPI goes high. Now the CPU is brought out of reset and starts executing instructions at the program start address.  
* Using the pass through mode, we can transfer data to spi flash without any external access.
* Currently management SOC has only 4 pin spi mode. User project may elect quad SPI using 6 pin mode.When the 6 pin mode is used, gpio pins 36,37 gets converted into flash io2 and flas io3 pins respectively.


## Waveforms at flash pins of caravel
* Initially, an intrustruction with opcode 0xAB is sent through flash io0 pin to release from deep power down.
* Once it gets released, we give another instruction with opcode 0x03 to read 3 byte address coming from flash io1 pin.
* We next observe HPSPI_CSB becoming low, which indicates HKSPI is active. Then at every posedge of HKSPI, we get valid data.

![](images/flashpins.png) 
 
## Waveforms at pins of spi flash
![](images/spiflashpins.png)

## Flash Specifications
We are using a QUAD I/O SPI NOR Flash wiht 32mb memory. <br />
Given below is the instructions set of the flash we used.
<br />
![](images/instructions.png)


# Bootcode Understanding - Mayank/Yashwant/Prabal

## Reset Flow:

![](images/Reset_flow_0.png)

![](images/Reset_flow.png)

- The reset is triggered exteranlly from outside the caravel chip. This leads to the reset of the
padframe chip_io throught "resetb" which is triggered at exactly 1us.

- Eventually this lead to triggering of reset of various components in management core such as PLL , caravel_ clocking, housekeeping etc. which is also denoted by "resetb" at 1.001us. 

- From caraval_clocking the reset of management_soc is triggered at 1.005us which is denoted by "resetn". 


## Bootcode flow:

The Booting flow during reset and start are almost the same, as during the POR circuit generate the same reset signal.

### STEP-1

- Initializing all the register file from x1 to x31 with value 0, except x2 (Stack pointer) and x0 (Hardwired zero register) which has been initialized by the reset itself (it is also initiated during start when POR circuit generate the same reset signal).

- The address loaded in the Stack Pointer is 0x00000400.

![](images/PicoRV_Register.png)

### STEP-2

- Copying the initialization value of .data section which involves moving the data from _sidata (start address for the initialization values of the .data section defined in linker script) to between  _sdata and _edata. (start & end address for the .data section defined in linker script respectively).

- Initializing the value present in .bss section (block start symbol) to zero. The .bss section address starts from _sbss to _ebss . 

NOTE :
- The .data section and .bss section are present in RAM (as described in linker file) with the origin address of 0x01000000.  

![](images/STEP-2.png)

- The program code to run after reset and other data are present in the Flash with origin address 0x10000000. 
- The Bootcode is executed using Execute-In-Place (XIP) method.
- Execute-In-Place (XIP) is a method of executing code directly from the serial Flash memory without copying the code to the RAM. The serial Flash memory is seen as    another memory in the MCU's memory address map.
- The _sidata of .data section is 0x10000210. which stated as _etext is the diassembly file. 
- THe _sdata and _edata for this implementation is 0x0, which shows that the .data section was empty. 


### STEP-3

![](images/STEP_3.png)

-  Setting the SPI to manual mode for sending optional WREN command and again  setting it back to MEMIO mode. (optional)

- Setting the CS (Chip Select) bit HIGH and IO0 as output through SPI control register at address 0x28000000 and enabling manual control of SPI.

- Chip Select (CS) is turned HIGH, by setting the value of SPI control Register to 0x120.

- Set the register at address 0x28000003 to 0x0.

- Sending optional WREN command through SPI. These command enable access to files, IO , networking etc

- Once the command have been send the SPI is returned to the previous mode (MEMIO).

- This is done by setting the register at 0x28000003 to 0x80.

NOTE : 

For Step-3 to take place, a2 register should not be initialised with zero. This disables the WREN command transission. 


## Implementation: 

- The reset signal to the padframe "resetb" was activated as 1us. 
 
- This triggered the reset flow till the picorv32 "resetn" which was activated at 1.005us.
 
- The starting address of flash (0x10000000) where the bootcode and rest of the programme are present got loaded in the CPU at 1.009us. 
 
- The instructon present in the (0x10000000) got executed at 1.343us. 
 
- The instruction present in address (0x100000B8) marks the end of bootcode excecution.
 
- The entire bootcode was executed till 6.747us.
 
- The period for which the bootcode ran is approximately 5.738us.

- After that the .main section began to execute which included the firmware and the blink program.

### Start of Bootcode:

![](images/Bootcode_start.png)

### End of Bootcode:

![](images/Bootcode_end.png)

Note: 
In this implementation the WREN command were not transmitted. 
