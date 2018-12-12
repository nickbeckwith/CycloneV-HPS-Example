# DE-SoC-FIFO-Example
A run through on how to communicate to the FPGA fabric through the HPS's ethernet port. I'm doing this so future Badgers taking ECE 554 don't have to struggle with the low bandwidth of UART. This example was inspired by the kind people at Cornell who posted [this](https://people.ece.cornell.edu/land/courses/ece5760/DE1_SOC/HPS_peripherials/FPGA_addr_index.html)

## How to run this example for the DE1-SoC Board


## Roadmap to understanding your own FIFO implementation on any DE series board
* Download needed materials
   - [Your board's cd](http://download.terasic.com/)
   - [DE Series Linux image](https://www.intel.com/content/www/us/en/programmable/support/training/university/materials-software.html)
   
* Read and complete up to and including section 3.3 of Intel's FPGA and linux [tutorial](ftp://ftp.intel.com/pub/fpgaup/pub/Intel_Material/18.0/Tutorials/Linux_On_DE_Series_Boards.pdf).
   - This tutorial teaches you how to communicate to your board's HPS system over serial as well as how to setup linux on the HPS system
   - The linux image suggested automatically configures your FPGA on boot. We won't be using this configuration but it is helpful for pure HPS applications.
   - For section 2.3, use the linux image for your board.
   - Even though you may not want to use VNC, you should configure connection over ethernet if your project requires it. Details are in section 2.7.
   - For section 3.3, to fully understand where the address 0xFF200000 comes from, look at the address map for Cyclone V [here](https://www.intel.com/content/www/us/en/programmable/hps/cyclone-v/hps.html).
   - Take a gander at section 3.4.3 if you plan to use native compiling and makefiles. You could try configuring startup scripts like .bashrc to ask the date and time over ethernet. Let me know if you get something like this working.
   - For more information about the SoC-Computer, take a look at [this](ftp://ftp.intel.com/Pub/fpgaup/pub/Intel_Material/18.0/Computer_Systems/DE1-SoC/DE1-SoC_Computer_ARM.pdf).
* Read and complete the My first HPS-FPGA tutorial found in `CD/UserManual/My_First_HPS-Fpga.pdf` or [here](http://terasic.yubacollegecompsci.com/resources/My_First_HPS-Fpga.pdf).
   - I didn't find it was neccesary to read my first HPS or my first FPGA, but any missing information should be found there.
   - Pay attention to qsys/platform designer usage. Some important points to notice:
     - Components are often abstractions of the ATX serial bus. Avalon memory mapped devices are like the name hints, memory mapped and thus must be given an address that is agreed upon.
     - Exported signals are what the FPGA fabric gets to communicate with. In this example, the conduit is exported. You'll see in larger examples that we often bridge a master clk and reset signals into qsys, so we do not have to wire every single reset or clock signal available.
     - I would highly recommend trying to create your own project using SystemBuilder (found in `CD/Tools/SystemBuilder`) to see how much of a pain it is to not use preconfigured projects like GHRD.
 
## Guide to creating your own FIFO implementation
* In previous FPGA-only work, FPGA projects started with SystemBuilder found in `CD/Tools/SystemBuilder/`. We could start the project there. However, we would have to connect the HPS system ourselves when we eventually instaitate it in Quartus. Much more tenacious people than I have already done that, so we will go from their work. Copy over `CD/Demonstrations/SOC_FPGA/dex_soc_GHRD` to your desired project directory. GHRD is a reference design that includes everything you need including an instantiated HPS on the top level. In this directory, there is a makefile that suggests it's useful. I would check this out if you have time especially since it might have scripts that help you setup [uboot](https://rocketboards.org/foswiki/Documentation/PreloaderUbootCustomization131).

* From here, open the included quartus project file. Before looking at the top level, find your way over to qsys. 
