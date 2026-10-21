Quick Start
===========

fRAC includes an Ethernet and TCP stack and a few accelerators, all implemented on the FPGA.
A TCP client can send application requests over TCP and receive responses with low latency.
Accelerators sit in slots, which can be swapped at runtime through partial reconfiguration.


Requirements
------------

Hardware
~~~~~~~~

1. Alveo U280.
3. USB cable for JTAG.

Software
~~~~~~~~

1. Ubuntu. The paper's measurements ran on 20.04, and the evaluation testbeds
   run 24.04.
2. Vivado 2022.2, whose hardware manager programs the FPGA over JTAG. Building
   bitstreams also needs Vitis HLS 2022.2 and a license for the UltraScale+
   Integrated 100G Ethernet Subsystem (CMAC).
5. CMake >= 3.5, for building bitstreams.

.. warning::

   Programming the U280 over JTAG takes it off PCIe if it was up there, for
   example running the XRT shell from its flash. Some servers treat that
   surprise link-down as a fatal error and reset. Take the card off the bus
   and disable its link first (``scripts/flashfpga.sh --help`` shows the
   commands), or use a host on which the card is not enumerated.

Generating bitstream
--------------------
.. code-block:: sh

   $ cd ~
   $ git clone https://github.com/accl-kaust/frac.git
   $ cd frac
   $ make ip CMAKE=/usr/bin/cmake
   $ ./bin/spinhdl --parallel 8 weave spinhdl.yaml \
                    --units spin.yaml \
                    --static static.yaml


Program the FPGA
----------------

Program the Alveo U280 FPGA with the full image. We have a shell script for
this. The full image starts with top_k, the first accelerator listed for each
cell in ``spinhdl.yaml``, in every slot.

.. code-block:: sh

   $ cd ~/frac
   $ ./scripts/programfpga.sh example/frac/jtag/frac.bit

.. code-block:: output

   programming ~/frac/example/frac/jtag/frac.bit
   PROGRAM_OK: xcu280_u55c_0

``programfpga.sh`` uses the Vivado in ``/tools/Xilinx/Vivado/2022.2``. Set
``VIVADO_ROOT`` if yours is elsewhere, and ``HW_TARGET`` if the host has more
than one JTAG cable; ``--help`` lists the options.

.. note::

   Wait a few seconds after programming the FPGA before sending it any packets
   or requests.

Configure the Host Network
--------------------------

The checked-in fRAC design uses FPGA address ``172.24.1.52`` and TCP port
``2888``. Configure the connected host interface with a different address in
the same subnet. The example uses ``172.24.1.2/24``. The MTU must be at least
8232, because fRAC's TCP stack sends 8192-byte segments; we use 9000.

Check the TCP Connection
------------------------

Just try ping-ing FPGA.

.. code-block:: sh

   $ ping 172.24.1.52

If ping succeeds, your machine can reach the FPGA and possibly fRAC.

You can also send some test packets to fRAC accelerators and see if they are live.

.. code-block:: sh

   $ cd ~/frac
   $ go run ./scripts/testfuncs.go -slots 0,1 -counter

You will get a response something like this. Both slots hold top_k, which
answers with a single line that ``testfuncs`` does not recognise; the 300 ms is
``testfuncs`` waiting for further lines that do not come. Both are expected.

.. code-block:: output

  workload 0x0000 (slot 0):
  unrecognised; closest is or_slot (ff 00 00 .., 8-bit line) (1/16 words)   64B in 301.58273ms
  00000000  b9 aa aa aa 00 00 00 00  b7 aa aa aa b6 aa aa aa  |................|
  00000010  b5 aa aa aa b4 aa aa aa  b3 aa aa aa b2 aa aa aa  |................|
  00000020  b1 aa aa aa b0 aa aa aa  af aa aa aa ae aa aa aa  |................|
  00000030  ad aa aa aa ac aa aa aa  ab aa aa aa aa aa aa aa  |................|

  workload 0x0001 (slot 1):
  unrecognised; closest is or_slot (ff 00 00 .., 8-bit line) (1/16 words)   64B in 301.57862ms
  00000000  b9 aa aa aa 00 00 00 00  b7 aa aa aa b6 aa aa aa  |................|
  00000010  b5 aa aa aa b4 aa aa aa  b3 aa aa aa b2 aa aa aa  |................|
  00000020  b1 aa aa aa b0 aa aa aa  af aa aa aa ae aa aa aa  |................|
  00000030  ad aa aa aa ac aa aa aa  ab aa aa aa aa aa aa aa  |................|

Reconfigure with an Accelerator
-------------------------------

Try reconfiguring with any accelerator

.. code-block:: sh

   $ cd ~/frac
   $ go run ./scripts/reconfslots.go -slot 1 -chunk-size 256 \
            -hbm-addr 0x10004000 -query-status example/frac/icap/c01_f09.bin

Then try sending packets and it should change the response.

Finally, program with echo for the next step

.. code-block:: sh

   $ go run ./scripts/reconfslots.go -slot 1 -chunk-size 256 \
            -hbm-addr 0x10004000 -query-status example/frac/icap/c01_f09.bin


.. code-block:: sh

   $ go run ./scripts/testfuncs.go -slots 0,1 -counter

.. code-block:: output

  workload 0x0000 (slot 0):
  unrecognised; closest is or_slot (ff 00 00 .., 8-bit line) (1/16 words)   64B in 302.053461ms
  00000000  b9 aa aa aa 00 00 00 00  b7 aa aa aa b6 aa aa aa  |................|
  00000010  b5 aa aa aa b4 aa aa aa  b3 aa aa aa b2 aa aa aa  |................|
  00000020  b1 aa aa aa b0 aa aa aa  af aa aa aa ae aa aa aa  |................|
  00000030  ad aa aa aa ac aa aa aa  ab aa aa aa aa aa aa aa  |................|

  workload 0x0001 (slot 1):
  echo_slot (data returned unchanged)  16/16 words   128B in 1.17679ms
  00000000  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
  00000010  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
  00000020  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
  00000030  ff ff ff ff ff ff ff ff  80 00 00 00 fd ff 01 00  |................|
  00000040  aa aa aa aa ab aa aa aa  ac aa aa aa ad aa aa aa  |................|
  00000050  ae aa aa aa af aa aa aa  b0 aa aa aa b1 aa aa aa  |................|
  00000060  b2 aa aa aa b3 aa aa aa  b4 aa aa aa b5 aa aa aa  |................|
  00000070  b6 aa aa aa b7 aa aa aa  b8 aa aa aa b9 aa aa aa  |................|

