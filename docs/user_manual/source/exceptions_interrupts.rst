.. _exceptions-interrupts:

Exceptions and Interrupts
=========================

|corev| implements trap handling for interrupts and exceptions according to the RISC-V Privileged Specification, version 1.11.
The ``irq_i[31:16]`` interrupts are a custom extension.

When entering an interrupt/exception handler, the core sets the ``mepc`` CSR to the current program counter and saves ``mstatus``.MIE to ``mstatus``.MPIE.
All exceptions cause the core to jump to the base address of the vector table in the ``mtvec`` CSR.
Interrupts are handled in either direct mode or vectored mode depending on the value of ``mtvec``.MODE. In direct mode the core
jumps to the base address of the vector table in the ``mtvec`` CSR. In vectored mode the core jumps to the base address
plus four times the interrupt ID. Upon executing an MRET instruction, the core jumps to the program counter previously saved in the
``mepc`` CSR and restores ``mstatus``.MPIE to ``mstatus``.MIE.

The base address of the vector table must be aligned to 256 bytes (i.e., its least significant byte must be 0x00) and can be programmed
by writing to the ``mtvec`` CSR. For more information, see the :ref:`cs-registers` documentation.

The core starts fetching at the address defined by ``boot_addr_i``. It is assumed that the boot address is supplied via a register
to avoid long paths to the instruction fetch unit.

Interrupt Interface
-------------------

:numref:`Interrupt interface signals` describes the interrupt interface.

.. table:: Interrupt interface signals
  :name: Interrupt interface signals

  +-------------------------+-----------+--------------------------------------------------+
  | Signal                  | Direction | Description                                      |
  +=========================+===========+==================================================+
  | ``irq_i[31:0]``         | input     | Active high, level sensistive interrupt inputs.  |
  |                         |           | Not all interrupt inputs can be used on          |
  |                         |           | |corev|. Specifically irq_i[15:12],              |
  |                         |           | irq_i[10:8], irq_i[6:4] and irq_i[2:0] shall be  |
  |                         |           | tied to 0 externally as they are reserved for    |
  |                         |           | future standard use (or for cores which are not  |
  |                         |           | Machine mode only) in the RISC-V Privileged      |
  |                         |           | specification. irq_i[11], irq_i[7], and irq_i[3] |
  |                         |           | correspond to the Machine External               |
  |                         |           | Interrupt (MEI), Machine Timer Interrupt (MTI),  |
  |                         |           | and Machine Software Interrupt (MSI)             |
  |                         |           | respectively. The irq_i[31:16] interrupts        |
  |                         |           | are a |corev| specific extension to the RISC-V   |
  |                         |           | Basic (a.k.a. CLINT) interrupt scheme.           |
  +-------------------------+-----------+--------------------------------------------------+

Interrupts
----------

The ``irq_i[31:0]`` interrupts are controlled via the ``mstatus``, ``mie`` and ``mip`` CSRs. |corev| uses the upper 16 bits of ``mie`` and ``mip`` for custom interrupts (``irq_i[31:16]``),
which reflects an intended custom extension in the RISC-V Basic (a.k.a. CLINT) interrupt architecture.
After reset, all interrupts, except for NMIs, are disabled.
To enable any of the ``irq_i[31:0]`` interrupts, both the global interrupt enable (``MIE``) bit in the ``mstatus`` CSR and the corresponding individual interrupt enable bit in the ``mie`` CSR need to be set. For more information, see the :ref:`cs-registers` documentation.

NMIs are never masked by the ``MIE`` bit. NMIs are masked however while in debug mode or while single stepping with ``STEPIE`` = 0 in the ``dcsr`` CSR.

If multiple interrupts are pending, they are handled in the fixed priority order defined by the RISC-V Privileged Specification, version 1.11 (see Machine Interrupt Registers, Section 3.1.9).
The highest priority is given to the interrupt with the highest ID, except for the Machine Timer Interrupt, which has the lowest priority. So from high to low priority the interrupts are
ordered as follows: 

* ``store bus fault NMI (129)``
* ``load bus fault NMI (128)``
* ``irq_i[31]``
* ``irq_i[30]``
* ...
* ``irq_i[16]``
* ``irq_i[11]``
* ``irq_i[3]``
* ``irq_i[7]``

The ``irq_i[31:0]`` interrupt lines are level-sensitive. The NMIs are triggered by load/store bus fault events.
To clear the ``irq_i[31:0]`` interrupts at the external source, |corev| relies on a software-based mechanism in which the interrupt handler signals completion of the handling routine to the interrupt source, e.g., through a memory-mapped register, which then deasserts the corresponding interrupt line.

In Debug Mode, all interrupts are ignored independent of ``mstatus``.MIE and the content of the ``mie`` CSR.

|corev| can trigger the following interrupts as reported in ``mcause``:

 +----------------+----------------+-------------------------------------------------+-----------------------------------------------------------------+
 | Interrupt      | Exception Code | Description                                     | Scenario(s)                                                     |
 +----------------+----------------+-------------------------------------------------+-----------------------------------------------------------------+
 |              1 |              3 | Machine Software Interrupt (MSI)                | ``irq_i[3]``                                                    |
 +----------------+----------------+-------------------------------------------------+-----------------------------------------------------------------+
 |              1 |              7 | Machine Timer Interrupt (MTI)                   | ``irq_i[7]``                                                    |
 +----------------+----------------+-------------------------------------------------+-----------------------------------------------------------------+
 |              1 |             11 | Machine External Interrupt (MEI)                | ``irq_i[11]``                                                   |
 +----------------+----------------+-------------------------------------------------+-----------------------------------------------------------------+
 |              1 |          31-16 | Machine Fast Interrupts                         | ``irq_i[31]``-``irq_i[16]``                                     |
 +----------------+----------------+-------------------------------------------------+-----------------------------------------------------------------+
 |              1 |            128 | Load bus fault NMI (imprecise)                  | ``data_err_i`` = 1 and ``data_rvalid_i`` = 1 for load           |
 +----------------+----------------+-------------------------------------------------+-----------------------------------------------------------------+
 |              1 |            129 | Store bus fault NMI (imprecise)                 | ``data_err_i`` = 1 and ``data_rvalid_i`` = 1 for store          |
 +----------------+----------------+-------------------------------------------------+-----------------------------------------------------------------+

.. note::

   Load bus fault and store bus fault are handled as imprecise non-maskable interrupts
   (as opposed to precise exceptions).

Exceptions
----------

|corev| can trigger the following exceptions as reported in ``mcause``:

 +----------------+----------------+---------------------------------------+---------------------------------------------------------------------------+
 | Interrupt      | Exception Code | Description                           | Scenario(s)                                                               |
 +----------------+----------------+---------------------------------------+---------------------------------------------------------------------------+
 |              0 |              1 | Instruction access fault              | Execution attempt from I/O region.                                        |
 +----------------+----------------+---------------------------------------+---------------------------------------------------------------------------+
 |              0 |              2 | Illegal instruction                   |                                                                           |
 +----------------+----------------+---------------------------------------+---------------------------------------------------------------------------+
 |              0 |              3 | Breakpoint                            | Instruction address breakpoint.                                           |
 |                |                |                                       | Load/store/AMO address breakpoint.                                        |
 |                |                |                                       | Environment break.                                                        |
 +----------------+----------------+---------------------------------------+---------------------------------------------------------------------------+
 |              0 |              5 | Load access fault                     | Non-naturally aligned load access attempt to an I/O region.               |
 |                |                |                                       | Load-Reserved attempt to region without atomic support.                   |
 +----------------+----------------+---------------------------------------+---------------------------------------------------------------------------+
 |              0 |              7 | Store/AMO access fault                | Non-naturally aligned store access attempt to an I/O region.              |
 |                |                |                                       | Store-Conditional or Atomic Memory Operation (AMO) attempt                |
 |                |                |                                       | to region without atomic support.                                         |
 +----------------+----------------+---------------------------------------+---------------------------------------------------------------------------+
 |              0 |             11 | Environment call from M-Mode (ECALL)  |                                                                           |
 +----------------+----------------+---------------------------------------+---------------------------------------------------------------------------+
 |              0 |             48 | Instruction bus fault                 | ``instr_err_i`` = 1 and ``instr_rvalid_i`` = 1 for instruction fetch      |
 +----------------+----------------+---------------------------------------+---------------------------------------------------------------------------+

If an instruction raises multiple exceptions, the priority, from high to low, is as follows: 

* ``instruction address breakpoint (3)``
* ``instruction access fault (1)``
* ``instruction bus fault (48)``
* ``illegal instruction (2)``
* ``environment call from M-Mode (11)``
* ``environment break (3)``
* ``load/store/AMO address breakpoint (3)``
* ``store/AMO access fault (7)``
* ``load access fault (5)``

Exceptions in general cannot be disabled and are always active. 
All exceptions are precise.
Whether the PMA will actually cause exceptions depends on its configuration.
|corev|  raises an illegal instruction exception for any instruction in the RISC-V privileged and unprivileged specifications that is explicitly defined as being
illegal according to the ISA implemented by the core, as well as for any instruction that is left undefined in these specifications unless the instruction encoding
is configured as a custom |corev| instruction for specific parameter settings as defined in (see :ref:`custom-isa-extensions`).
An instruction bus error leads to a precise instruction interface bus fault if an attempt is made to execute the instruction that has an associated bus error.
Similarly an instruction fetch with a failing PMA check only leads to an instruction access exception if an actual execution attempt is made for it.

Nested Interrupt/Exception Handling
-----------------------------------

|corev| does support nested interrupt/exception handling in software.
The hardware automatically disables interrupts upon entering an interrupt/exception handler.
Otherwise, interrupts/exceptions during the critical part of the handler, i.e. before software has saved the ``mepc`` and ``mstatus`` CSRs, would cause those CSRs to be overwritten.
If desired, software can explicitly enable interrupts by setting ``mstatus``.MIE to 1 from within the handler.
However, software should only do this after saving ``mepc`` and ``mstatus``.
There is no limit on the maximum number of nested interrupts.
Note that, after enabling interrupts by setting ``mstatus``.MIE to 1, the current handler will be interrupted also by lower priority interrupts.
To allow higher priority interrupts only, the handler must configure ``mie`` accordingly.

The following pseudo-code snippet visualizes how to perform nested interrupt handling in software.

.. code-block:: c
   :linenos:

   isr_handle_nested_interrupts(id) {
     // Save mpec and mstatus to stack
     mepc_bak = mepc;
     mstatus_bak = mstatus;

     // Save mie to stack (optional)
     mie_bak = mie;

     // Keep lower-priority interrupts disabled (optional)
     mie = mie & ~((1 << (id + 1)) - 1);

     // Re-enable interrupts
     mstatus.MIE = 1;

     // Handle interrupt
     // This code block can be interrupted by other interrupts.
     // ...

     // Restore mstatus (this disables interrupts) and mepc
     mstatus = mstatus_bak;
     mepc = mepc_bak;

     // Restore mie (optional)
     mie = mie_bak;
   }

Nesting of interrupts/exceptions in hardware is not supported.
