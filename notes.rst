Trusted RAM size: 256kb / 512kb

Shared RAM: (default) 4kb

Mnemonics:

TSP   - Test Secure-EL1 Payload
TZC   - Trust Zone Controller
SP    - Secure Payload
OEN   - Owning Entity Numbers (SMCCC call function)
SMCCC - SMC Calling Convention

Overview
========

* Trusted Firmware-A (TF-A) implements a subset of the Trusted Board
  Boot Requirements (TBBR) Platform Design Document (PDD) for Arm
  reference platforms.

* The TBB sequence starts when the platform is powered on and runs up
  to the stage where it hands-off control to firmware running in the
  normal world in DRAM. This is the cold boot path.

* TF-A also implements the Power State Coordination Interface (PSCI)
  PDD as a runtime service. PSCI is the interface from normal world
  software to firmware implementing power management use-cases (for
  example, secondary CPU boot, hotplug and idle). Normal world
  software can access TF-A runtime services via the Arm SMC (Secure
  Monitor Call) instruction. The SMC instruction must be used as
  mandated by the SMC Calling Convention (SMCCC).

* TF-A implements a framework for configuring and managing interrupts
  generated in either security state.

* TF-A also implements a library for setting up and managing the
  translation tables.

* All BL stages can be configured using DT-based configuration files
  (optional).

TF-A Stages
===========

* Boot Loader stage 1 (BL1) `AP Trusted ROM`

* Boot Loader stage 2 (BL2) `Trusted Boot Firmware`

* Boot Loader stage 3-1 (BL31) `EL3 Runtime Software`

* Boot Loader stage 3-2 (BL32) `Secure-EL1 Payload` (optional)

* Boot Loader stage 3-3 (BL33) `Non-trusted Firmware`

BL1
===

*Note:* This is (typically) placed in (trusted) `ROM`.

This stage begins execution from the platform's reset vector at
EL3. The reset address is platform dependent but it is usually located
in a Trusted ROM area. The BL1 data section is copied to trusted SRAM
at runtime.

1. Architectural initialization

* Exception vectors

* CPU initialization (CPU type dependent)

* Control register setup

2. Platform initialization

* Enable the Trusted Watchdog.

* Initialize the console.

* Configure the Interconnect to enable hardware coherency.

* Enable the MMU and map the memory it needs to access.

* Configure any required platform storage to load the next bootloader
  image (BL2).

* If the BL1 dynamic configuration file, TB_FW_CONFIG, is available,
  then load it to the platform defined address and make it available
  to BL2 via arg0.

* Configure the system timer and program the CNTFRQ_EL0 for use by
  NS-BL1U and NS-BL2U firmware update images.

3. Firmware Update detection and execution

(??? FWU ???)

4. BL2 image load and execution

* Print "Booting Trusted Firmware"

* bl1_plat_handle_pre_image_load()

* BL1 loads a BL2 raw binary image from platform storage
  (auth/decrypt?)

* bl1_plat_handle_post_image_load()

* BL1 passes control to the BL2 image at Secure *EL1*, starting from its
  load address.

BL2
===

*Running at Secure EL1* (normally)

1. Architectural initialization

* BL2 performs the minimal architectural initialization required for
  subsequent stages of TF-A and normal world software. EL1 and EL0 are
  given access to Floating Point and Advanced SIMD registers by
  setting the CPACR.FPEN bits.

2. Platform initialization

On Arm platforms, BL2 performs the following platform initialization:

* Initialize the console.

* Configure any required platform storage to allow loading further
  bootloader images.

* Enable the MMU and map the memory it needs to access.

* Perform platform security setup to allow access to controlled
  components.

* Reserve some memory for passing information to the next bootloader
  image EL3 Runtime Software and populate it.

* Define the extents of memory available for loading each subsequent
  bootloader image.

* If BL1 has passed TB_FW_CONFIG dynamic configuration file in arg0,
  then parse it.

3. Image loading in BL2

BL2 generic code loads the images based on the list of loadable images
provided by the platform. BL2 passes the list of executable images
provided by the platform to the next handover BL image.

4. SCP_BL2 (System Control Processor Firmware) image load

Optional.

Some systems have a separate System Control Processor (SCP) for power,
clock, reset and system control. BL2 loads the optional SCP_BL2 image
from platform storage into a platform-specific region of secure
memory. The subsequent handling of SCP_BL2 is platform specific. For
example, on the Juno Arm development platform port the image is
transferred into SCP's internal memory using the Boot Over MHU (BOM)
protocol after being loaded in the trusted SRAM memory. The SCP
executes SCP_BL2 and signals to the Application Processor (AP) for BL2
execution to continue.

5. EL3 Runtime Software image load

BL2 loads the EL3 Runtime Software (BL31) image from platform storage
into a platform- specific address in trusted SRAM. If there is not
enough memory to load the image or image is missing it leads to an
assertion failure.

6. AArch64 BL32 (Secure-EL1 Payload) image load

Optional. TSP.

BL2 loads the optional BL32 image from platform storage into a
platform- specific region of secure memory.

This information is passed to BL31 (if used).

7. BL33 (Non-trusted Firmware) image load

BL2 loads the BL33 image (e.g. UEFI or other test or boot software)
from platform storage into non-secure memory as defined by the
platform.

BL2 relies on EL3 Runtime Software to pass control to BL33 once secure
state initialization is complete. Hence, BL2 populates a
platform-specific area of memory with the entrypoint and Saved Program
Status Register (SPSR) of the normal world software image. The
entrypoint is the load address of the BL33 image. The SPSR is
determined as specified in Section 5.13 of the Power State
Coordination Interface PDD. This information is passed to the EL3
Runtime Software.

8. AArch64 BL31 (EL3 Runtime Software) execution

BL2 execution continues as follows:

* BL2 passes control back to BL1 by raising an SMC, providing BL1 with
  the BL31 entrypoint. The exception is handled by the SMC exception
  handler installed by BL1.

* BL1 turns off the MMU and flushes the caches. It clears the
  SCTLR_EL3.M/I/C bits, flushes the data cache to the point of
  coherency and invalidates the TLBs.

* BL1 passes control to BL31 at the specified entrypoint at EL3.

AArch64 BL31
============

1. Architectural initialization

* Currently, BL31 performs a similar architectural initialization to
  BL1 as far as system register settings are concerned. Since BL1 code
  resides in ROM, architectural initialization in BL31 allows override
  of any previous initialization done by BL1.

* BL31 initializes the per-CPU data framework, which provides a cache
  of frequently accessed per-CPU data optimized for fast, concurrent
  manipulation on different CPUs. This buffer includes pointers to
  per-CPU contexts, crash buffer, CPU reset and power down operations,
  PSCI data, platform data and so on.

* It then replaces the exception vectors populated by BL1 with its
  own. BL31 exception vectors implement more elaborate support for
  handling SMCs since this is the only mechanism to access the runtime
  services implemented by BL31 (PSCI for example). BL31 checks each
  SMC for validity as specified by the SMC Calling Convention before
  passing control to the required SMC handler routine.

* BL31 programs the CNTFRQ_EL0 register with the clock frequency of
  the system counter, which is provided by the platform.

2. Platform initialization

BL31 performs detailed platform initialization, which enables normal
world software to function correctly.

* Initialize the console.

* Configure the Interconnect to enable hardware coherency.

* Enable the MMU and map the memory it needs to access.

* Initialize the generic interrupt controller.

* Initialize the power controller device.

* Detect the system topology.

3. Runtime services initialization

* BL31 is responsible for initializing the runtime services. One of
  them is PSCI.

4. AArch64 BL32 (Secure-EL1 Payload) image initialization

* If a BL32 image is present then there must be a matching Secure-EL1
  Payload Dispatcher (SPD) service.

* When the BL32 has completed initialization at Secure-EL1, it returns
  to BL31 by issuing an `SMC`.

* On return from the handler the framework will exit to EL2 and run
  BL33.

* By nature, BL32 is optional. (`Plugin` extension to SMCCC)

5. BL33 (Non-trusted Firmware) execution

*General-purpose boot loader - U-Boot, UEFI, etc.*

* EL3 Runtime Software initializes the EL2 or EL1 processor context
  for normal-world cold boot, ensuring that no secure state
  information finds its way into the non-secure execution state.

* EL3 Runtime Software uses the entrypoint information provided by BL2
  to jump to the Non-trusted firmware image (BL33) at the highest
  available Exception Level (EL2 if available, otherwise EL1).
