# DE-Series-FIFO-Example
A run through on how to communicate to the FPGA fabric through the HPS's ethernet port. I'm doing this so future Badgers taking ECE 554 don't have to struggle with the low bandwidth of UART. This example was inspired by the kind people at Cornell who posted [this](https://people.ece.cornell.edu/land/courses/ece5760/DE1_SOC/HPS_peripherials/FPGA_addr_index.html)

# How to run this example for the DE1-SoC Board


# Roadmap to understanding your own FIFO implementation on any DE series board
* Download needed materials
   - [Your board's cd](http://download.terasic.com/)
   - [DE Series Linux image](https://www.intel.com/content/www/us/en/programmable/support/training/university/materials-software.html)
   
* In a new project directory read and complete up to and including section 3.3 of Intel's FPGA and linux [tutorial](ftp://ftp.intel.com/pub/fpgaup/pub/Intel_Material/18.0/Tutorials/Linux_On_DE_Series_Boards.pdf). Keep in mind quartus doesn't always play nice with directories that have spaces.
   - This tutorial teaches you how to communicate to your board's HPS system over serial as well as how to setup linux on the HPS system
   - The linux image suggested automatically configures your FPGA on boot. We won't be using this configuration but it is helpful for pure HPS applications.
   - For section 2.3, use the linux image for your board.
   - Even though you may not want to use VNC, you should configure connection over ethernet if your project requires it. Details are in section 2.7.
   - For section 3.3, to fully understand where the address 0xFF200000 comes from, look at the address map for Cyclone V [here](https://www.intel.com/content/www/us/en/programmable/hps/cyclone-v/hps.html).
   - Take a gander at section 3.4.3 if you plan to use native compiling and makefiles. You could try configuring startup scripts like .bashrc to ask the date and time over ethernet. Let me know if you get something like this working.
   - For more information about the SoC-Computer, take a look at [this](ftp.intel.com/Pub/fpgaup/pub/Intel_Material/18.0/Computer_Systems/DE1-SoC/DE1-SoC_Computer_ARM.pdf).
* In a new project directory, read and complete the My first HPS-FPGA tutorial found in `CD/UserManual/My_First_HPS-Fpga.pdf` or [here](http://terasic.yubacollegecompsci.com/resources/My_First_HPS-Fpga.pdf).
   - I didn't find it was neccesary to read my first HPS or my first FPGA, but any missing information should be found there.
   - Pay attention to qsys/platform designer usage. Some important points to notice:
     - Components are often abstractions of the ATX serial bus. Avalon memory mapped devices are like the name hints, memory mapped and thus must be given an address that is agreed upon.
     - Exported signals are what the FPGA fabric gets to communicate with. In this example, the conduit is exported. You'll see in larger examples that we often bridge a master clk and reset signals into qsys, so we do not have to wire every single reset or clock signal available.
     - I would highly recommend trying to create your own project using SystemBuilder (found in `CD/Tools/SystemBuilder`) to see how much of a pain it is to not use preconfigured projects like GHRD.
 
# Guide to creating your own FIFO implementation
* In previous FPGA-only work, FPGA projects started with SystemBuilder found in `CD/Tools/SystemBuilder/`. We could start the project there. However, we would have to connect the HPS system ourselves when we eventually instaitate it in Quartus. Much more tenacious people than I have already done that, so we will go from their work. Copy over `CD/Demonstrations/SOC_FPGA/dex_soc_GHRD` to your desired project directory. GHRD is a reference design that includes everything you need including an instantiated HPS on the top level. In this directory, there is a makefile that suggests it's useful. I would check this out if you have time especially since it might have scripts that help you setup [uboot](https://rocketboards.org/foswiki/Documentation/PreloaderUbootCustomization131).
## Qsys/Platform Designer
* From here, open the included quartus project file. Look at the top level then find your way over to qsys/platform designer which can be found in `tools->Qsys` or `tools->Platform Designer`.
* Upon opening, you will find that the HPS is already hooked up like in the previous examples. Along with the HPS being connected to onchip_memory (instead of SRAM), we have PIO like the last example for switches, buttons and LEDs. If you'd like, you could also connect the hex as an exercise. There also are JTAGs to an avalon bus master that I do not know how to use. Might be useful to figure out timing across the bus if needed. 
* Add two fifos of desired width and depth just like you did with the PIO. Make sure to turn backpressure off. I also do not use interrupts but instead use the status interface. Interrupts may be useful if you figure it out. Also, once you get AVALON_MM working the next step may be learning the avalon streaming interface.
![Where to find FIFO ip](https://github.com/nickbeckwith/DE-Series-FIFO-Example/blob/master/images/ip_fifo.jpg)
![FIFO ip configuration](https://github.com/nickbeckwith/DE-Series-FIFO-Example/blob/master/images/fifo_config.jpg)
* We now have two fifos that are renamed to h2f_fifo and f2h_fifo-- h being HPS and f being FPGA.
![Qsys initial](https://github.com/nickbeckwith/DE-Series-FIFO-Example/blob/master/images/qsys_start.jpg)
* Next we will export the data signals on the fpga side.
![Qsys export](https://github.com/nickbeckwith/DE-Series-FIFO-Example/blob/master/images/export.jpg)
* Finally, we will connect the clocks, resets and bridges to the HPS. We will be using the h2f lightweight AXI bus for status signals such as the CSR and the 'heavyweight' h2f and f2h AXI busses for data. Also, we connect the clocks and resets at this time instead of exporting them to prevent having to connect them in the top level.
![Qsys connected](https://github.com/nickbeckwith/DE-Series-FIFO-Example/blob/master/images/qsys_connect.jpg)
* To complete the qsys system, assign base addresses and interrupt numbers (if needed) which is found at `System->Assign Base Addresses`. These addresses are the memory offsets where the HPS can find the device. The actual address is found by adding the offset to the base bus address. For example, the HPS2FPGA AXI bus is found at 0xC0000000 so the f2hfifo out data is found at 0xC0010000. Generate the HDL and close out of qsys.
![Qsys final](https://github.com/nickbeckwith/DE-Series-FIFO-Example/blob/master/images/qsys_final.jpg)
## TODO insert how to change the fpga top level here. 
```Verilog
//=======================================================
//  create more readable signals
//=======================================================
wire h2f_full, h2f_empty, f2h_full, f2h_empty;
wire h2f_almost_empty, f2h_almost_full;
//assign h2f_full	= h2f_out_csr_readdata[7];
//assign h2f_empty	= ~(|h2f_out_csr_readdata);
//assign f2h_full	= f2h_in_csr_readdata[7];
//assign f2h_empty	= ~(|f2h_in_csr_readdata);
assign h2f_almost_empty = ~(|h2f_out_csr_readdata[7:4]);
assign f2h_almost_full  = f2h_in_csr_readdata[7] | (&f2h_in_csr_readdata[6:3]);
// reminder of other signals
// h2f_readdata, h2f_read
// f2h_in_writedata, f2h_in_write

// Confirmed: data read to q is 1 clock
// Confirmed: No write delay
// Notes: key 1 is pulsed, Key 2 is not pulsed. both act as read
// Actual FIFO operation
wire req_transaction;
reg transaction;
assign req_transaction = (KEY1_pulse | ~KEY[2]) & ~h2f_almost_empty & ~f2h_almost_full;

always @(posedge clk)
	transaction <= req_transaction;

assign f2h_in_writedata = h2f_readdata;
assign h2f_read = req_transaction;
assign f2h_in_write = transaction;
```
* This code gives us readable names for all the signals. Keep in mind we do not actually get a almost full and almost empty signal like we do for FIFO's instantiated with altera's megafunction wizard. Instead we create these signals by reading the fill line and using an arbitrary amount to determine the fill level. In the `h2f_almost_empty` case, we have it be set when the fill level is less than or equal to 15.
* `h2f_full` and similar signals that are in a seperate address in memory can only be accessed when the csr address is 1 and are unsable in the current code. Change the csr address to 1 if you would like to use these signals.
* We use almost empty and almost full to avoid queuing up an invalid write/read transaction.
* To deal with delays from the read operation, we add a single clock delay to the write operation to allow a write and a read operation to occur together.
* Key2 allows bursting of data between fifos while key1 exchanges two bytes per press. These are used mostly for testing.
## TODO generate header file
## Software side
The primary components of the software can be found in the example [main.c](src/main.c). In this section, I'll go through the file block by block. Our definitions are as follows.
``` C
// Pointers
// AXI bus address details
#define AXI_MASTER_BASE			(0xC0000000)			// fixed address for AXI bus in CV
#define AXI_MASTER_SPAN         	(0x00100000)			// Adjust to contain addresses you want
#define AXI_MASTER_MASK			(AXI_MASTER_SPAN - 1)		// If you pick span carefully, mask will
									// help you address the right mem block
// LW bus address details
#define LW_BUS_BASE 			(0xff200000)			// fixed address for LW bus in CV
#define LW_BUS_SPAN 			(0x00010000)			// Adjust to your liking. I don't need to 
#define LW_BUS_MASK 			(LW_BUS_SPAN -1) 		// reference the rest of my qsys stuff anyway
// pointers end
```
* AXI_MASTER_XXXX is information about the addressing of the AXI bus. The base is the base address as given by the Cyclone V specification [here](https://www.intel.com/content/www/us/en/programmable/hps/cyclone-v/hps.html). The Span is maximum addressable address and the mask is just for safety purposes to avoid a possible segfault.
* LW_BUS_XXXX is similar.
The macros used are to avoid function call procedures essentially making the functions inline.
``` C
// Macros
// Establish pointers
#define FIFO_WRITE				(*(FIFO_write_ptr))
#define FIFO_READ				(*(FIFO_read_ptr))
// Make status registers more readable
// FIFO base is fill-level and base plus 1 is status
// 0th bit is "full"
// 1st bit is "empty"	we're only using one of each for each fifo
#define WRITE_FIFO_FULL			((*(FIFO_write_status_ptr + 1)) & 1)
#define READ_FIFO_EMPTY			(((*(FIFO_read_status_ptr + 1)) & 2) >> 1)
#define WRITE_FILL_LEVEL		(*(FIFO_write_status_ptr))
#define READ_FILL_LEVEL			(*(FIFO_read_status_ptr))
// Function macros because why put actual functions on the stack?
// the nice people at Cornell say WAIT looks better than just {}
#define WAIT {}
// These block program execution until fifo is available for operation
#define FIFO_WRITE_BLOCK(data)	{while (WRITE_FIFO_FULL) WAIT;FIFO_WRITE=data;}
#define FIFO_READ_BLOCK(data)	{while (READ_FIFO_EMPTY) WAIT;data=FIFO_READ;}
// macros end
````
* An important thing to notice is that the csr register we found in qsys adheres to [this](https://www.intel.com/content/dam/www/programmable/us/en/pdfs/literature/hb/nios2/qts_qii55002.pdf) specification found at table 14-4. Therefore to get our control signals, we have to look at different memory mapped by the csr register. For example, the 33rd bit is the bit that is set when the fifo is full while the first 32 bits are the fifo's fill level.
* We also use blocking statements to write and read to the fifos. Writing and reading to fifos are just like any other memory mapped device, we write to a memory location and read from a memory location.
```C
// lw bus base and pointers
void *lw_bus_virtual_base;
volatile uint32_t *FIFO_write_status_ptr = NULL;
volatile uint32_t *FIFO_read_status_ptr = NULL;

// AXI_master bus base and pointers
void *axi_master_virtual_base;
volatile uint16_t *FIFO_write_ptr = NULL;
volatile uint16_t *FIFO_read_ptr = NULL;
```
* Be extremely careful about the size of your pointers. CSR registers are 32 bit wide and to make pointer arithmetic work well, you should use a pointer that refers to 32 bit data.
* Use the volatile data type to make sure the compiler knows that this data may change at any time: avoiding hurtful compiler optimizations.
```C
// === get FPGA address ====================================
int fd;
if ((fd = open( "/dev/mem", ( O_RDWR | O_SYNC))) == -1) {
   fprintf(stderr, "ERROR: can't find mem\n");
   exit(EXIT_FAILURE);
}
// =========================================================
```
* This is a stepping stone to read and write to our mem mapped devices.
```C
// === get lw virtual addresses ============================
	// lets map a lot of the lw bus mem map for now
	lw_bus_virtual_base = mmap(NULL, LW_BUS_SPAN, ( PROT_READ | PROT_WRITE ),
									MAP_SHARED, fd, LW_BUS_BASE);
	if (lw_bus_virtual_base == MAP_FAILED) {
		printf( "ERROR: mmap1() failed...\n" );
		close( fd );
		exit(EXIT_FAILURE);
	}
	// register locations
	FIFO_write_status_ptr = (uint32_t *)(lw_bus_virtual_base
								+ ((H2F_FIFO_IN_CSR_BASE & LW_BUS_MASK)));
	FIFO_read_status_ptr = (uint32_t *)(lw_bus_virtual_base
								+ ((F2H_FIFO_OUT_CSR_BASE & LW_BUS_MASK)));
// =========================================================
```
* We overzealously map memory addresses. Just be sure to refer to the right location in memory.
* This maps all LW-AXI memory locations such as the csr registers and LEDs.
* We do the same for the AXI memory bus but with the AXI address specifications.
* The H2F_FIFO_IN_CSR_BASE comes from the header file we generated earlier. You may have different names and should adjust accordingly. By using the generated header file, you can change qsys and assign new addresses but keep the same main.c.
```C
while(1) {
   FIFO_WRITE_BLOCK(fgetc(stdin));
}
```
* Reads from stdin and writes to fifo. Keep in mind that you can only write to your FIFO's size and no bigger unless you want your data masked.
```C
uint16_t buff;
while(1) {
   FIFO_READ_BLOCK(buff);
   fputs(buff, stdout);
}
```
* Writes to stdout. In your case, this is most likely to UART.
