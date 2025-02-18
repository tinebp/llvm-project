=============================
Offloading Design & Internals
=============================

.. contents::
   :local:

Introduction
============

This document describes the Clang driver and code generation steps for creating
offloading applications. Clang supports offloading to various architectures
using programming models like CUDA, HIP, and OpenMP. The purpose of this
document is to illustrate the steps necessary to create an offloading
application using Clang.

OpenMP Offloading
=================

.. note::
   This documentation describes Clang's behavior using the new offloading driver
   which. This currently must be enabled manually using ``-fopenmp-new-driver``.

Clang supports OpenMP target offloading to several different architectures such
as NVPTX, AMDGPU, X86_64, Arm, and PowerPC. Offloading code is generated by
Clang and then executed using the ``libomptarget`` runtime and the associated
plugin for the target architecture, e.g. ``libomptarget.rtl.cuda``. This section
describes the steps necessary to create a functioning device image that can be
loaded by the OpenMP runtime.  More information on the OpenMP runtimes can be
found at the `OpenMP documentation page <https://openmp.llvm.org>`__.

.. _Offloading Overview:

Offloading Overview
-------------------

The goal of offloading compilation is to create an executable device image that
can be run on the target device. OpenMP offloading creates executable images by
compiling the input file for both the host and the target device. The output
from the device phase then needs to be embedded into the host to create a fat
object. A special tool then needs to extract the device code from the fat
objects, run the device linking step, and embed the final image in a symbol the
host can use to register the library and access the symbols on the device.

Compilation Process
^^^^^^^^^^^^^^^^^^^

The compiler performs the following high-level actions to generate offloading
code:

* Compile the input file for the host to produce a bitcode file. Lower ``#pragma
  omp target`` declarations to :ref:`offloading entries <Generating Offloading
  Entries>` and create metadata to indicate which entries are on the device.
* Compile the input file for the target :ref:`device <Device Compilation>` using
  the :ref:`offloading entry <Generating Offloading Entries>` metadata created
  by the host.
* Link the OpenMP device runtime library and run the backend to create a device
  object file.
* Run the backend on the host bitcode file and create a :ref:`fat object file
  <Creating Fat Objects>` using the device object file.
* Pass the fat object file to the :ref:`linker wrapper tool <Device Linking>`
  and extract the device objects. Run the device linking action on the extracted
  objects.
* :ref:`Wrap <Device Binary Wrapping>` the :ref:`device images <Device linking>`
  and :ref:`offload entries <Generating Offloading Entries>` in a symbol that
  can be accessed by the host.
* Add the :ref:`wrapped binary <Device Binary Wrapping>` to the linker input and
  run the host linking action. Link with ``libomptarget`` to register and
  execute the images.

   .. _Generating Offloading Entries:

Generating Offloading Entries
-----------------------------

The first step in compilation is to generate offloading entries for the host.
This information is used to identify function kernels or global values that will
be provided by the device. Blocks contained in a ``#pragma omp target`` or
symbols inside a ``#pragma omp declare target`` directive will have offloading
entries generated. The following table shows the :ref:`offload entry structure
<table-tgt_offload_entry_structure>`.

  .. table:: __tgt_offload_entry Structure
    :name: table-tgt_offload_entry_structure

    +---------+------------+------------------------------------------------------------------------+
    |   Type  | Identifier | Description                                                            |
    +=========+============+========================================================================+
    |  void*  |    addr    | Address of global symbol within device image (function or global)      |
    +---------+------------+------------------------------------------------------------------------+
    |  char*  |    name    | Name of the symbol                                                     |
    +---------+------------+------------------------------------------------------------------------+
    |  size_t |    size    | Size of the entry info (0 if it is a function)                         |
    +---------+------------+------------------------------------------------------------------------+
    | int32_t |    flags   | Flags associated with the entry (see :ref:`table-offload_entry_flags`) |
    +---------+------------+------------------------------------------------------------------------+
    | int32_t |  reserved  | Reserved, to be used by the runtime library.                           |
    +---------+------------+------------------------------------------------------------------------+

The address of the global symbol will be set to the appropriate value by the
runtime once the device image is loaded. The flags are set to indicate the
handling required for the offloading entry. If the offloading entry is an entry
to a target region it can have one of the following
:ref:`entry flags <table-offload_entry_flags>`.

  .. table:: Target Region Entry Flags
    :name: table-offload_entry_flags

    +----------------------------------+-------+-----------------------------------------+
    |                Name              | Value | Description                             |
    +==================================+=======+=========================================+
    | OMPTargetRegionEntryTargetRegion | 0x00  | Mark the entry as generic target region |
    +----------------------------------+-------+-----------------------------------------+
    | OMPTargetRegionEntryCtor         | 0x02  | Mark the entry as a global constructor  |
    +----------------------------------+-------+-----------------------------------------+
    | OMPTargetRegionEntryDtor         | 0x04  | Mark the entry as a global destructor   |
    +----------------------------------+-------+-----------------------------------------+

If the offloading entry is a global variable, indicated by a non-zero size, it
will instead have one of the following :ref:`global
<table-offload_global_flags>` flags.

  .. table:: Target Region Global
    :name: table-offload_global_flags

    +-----------------------------+-------+---------------------------------------------------------------+
    |          Name               | Value | Description                                                   |
    +=============================+=======+===============================================================+
    | OMPTargetGlobalVarEntryTo   | 0x00  | Mark the entry as a 'to' attribute (w.r.t. the to clause)     |
    +-----------------------------+-------+---------------------------------------------------------------+
    | OMPTargetGlobalVarEntryLink | 0x01  | Mark the entry as a 'link' attribute (w.r.t. the link clause) |
    +-----------------------------+-------+---------------------------------------------------------------+

The target offload entries are used by the runtime to access the device kernels
and globals that will be provided by the final device image. Each offloading
entry is set to use the ``omp_offloading_entries`` section. When the final
application is created the linker will provide the
``__start_omp_offloading_entries`` and ``__stop_omp_offloading_entries`` symbols
which are used to create the :ref:`final image <Device Binary Wrapping>`.

This information is by the device compilation stage to determine which symbols
need to be exported from the device. We use the ``omp_offload.info`` metadata
node to pass this information device compilation stage.

Accessing Entries on the Device
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Accessing the entries in the device is done using the address field in the
:ref:`offload entry<table-tgt_offload_entry_structure>`. The runtime will set
the address to the pointer associated with the device image during runtime
initialization. This is used to call the corresponding kernel function when
entering a ``#pragma omp target`` region. For variables, the runtime maintains a
table mapping host pointers to device pointers. Global variables inside a
``#pragma omp target reclare`` directive are first initialized to the host's
address. Once the device address is initialized we insert it into the table to
map the host address to the device address.

Debugging Information
^^^^^^^^^^^^^^^^^^^^^

We generate structures to hold debugging information that is passed to
``libomptarget``. This allows the front-end to generate information the runtime
library uses for more informative error messages. This is done using the
standard :ref:`identifier structure <table-ident_t_structure>` used in
``libomp`` and ``libomptarget``. This is used to pass information and source
locations to the runtime.

  .. table:: ident_t Structure
    :name: table-ident_t_structure

    +---------+------------+-----------------------------------------------------------------------------+
    |   Type  | Identifier | Description                                                                 |
    +=========+============+=============================================================================+
    | int32_t |  reserved  | Reserved, to be used by the runtime library.                                |
    +---------+------------+-----------------------------------------------------------------------------+
    | int32_t |   flags    | Flags used to indicate some features, mostly unused.                        |
    +---------+------------+-----------------------------------------------------------------------------+
    | int32_t |  reserved  | Reserved, to be used by the runtime library.                                |
    +---------+------------+-----------------------------------------------------------------------------+
    | int32_t |  reserved  | Reserved, to be used by the runtime library.                                |
    +---------+------------+-----------------------------------------------------------------------------+
    |  char*  |  psource   | Program source information, stored as ";filename;function;line;column;;\\0" |
    +---------+------------+-----------------------------------------------------------------------------+

If debugging information is enabled, we will also create strings to indicate the
names and declarations of variables mapped in target regions. These have the
same format as the source location in the :ref:`identifier structure
<table-ident_t_structure>`, but the filename is replaced with the variable name.

.. _Device Compilation:

Offload Device Compilation
--------------------------

The input file is compiled for each active device toolchain. The device
compilation stage is performed differently from the host stage. Namely, we do
not generate any offloading entries. This is set by passing the
``-fopenmp-is-device`` flag to the front-end. We use the host bitcode to
determine which symbols to export from the device. The bitcode file is passed in
from the previous stage using the ``-fopenmp-host-ir-file-path`` flag.
Compilation is otherwise performed as it would be for any other target triple.

When compiling for the OpenMP device, we set the visibility of all device
symbols to be ``protected`` by default. This improves performance and prevents a
class of errors where a symbol in the target device could preempt a host
library.

The OpenMP runtime library is linked in during compilation to provide the
implementations for standard OpenMP functionality. For GPU targets this is done
by linking in a special bitcode library during compilation, (e.g.
``libomptarget-nvptx64-sm_70.bc``) using the ``-mlink-builtin-bitcode`` flag.
Other device libraries, such as CUDA's libdevice, are also linked this way. If
the target is a standard architecture with an existing ``libomp``
implementation, that will be linked instead. Finally, device tools are used to
create a relocatable device object file that can be embedded in the host.

.. _Creating Fat Objects:

Creating Fat Objects
--------------------

A fat binary is a binary file that contains information intended for another
device. We create a fat object by embedding the output of the device compilation
stage into the host as a named section. The output from the device compilation
is passed to the host backend using the ``-fembed-offload-object`` flag. This
inserts the object as a global in the host's IR. The section name contains the
target triple and architecture that the data corresponds to for later use.
Typically we will also add an extra string to the section name to prevent it
from being merged with other sections if the user performs relocatable linking
on the object.

.. code-block:: llvm

  @llvm.embedded.object = private constant [1 x i8] c"\00", section ".llvm.offloading.nvptx64.sm_70."

The device code will then be placed in the corresponding section one the backend
is run on the host, creating a fat object. Using fat objects allows us to treat
offloading objects as standard host objects. The final object file should
contain the following :ref:`offloading sections <table-offloading_sections>`. We
will use this information when :ref:`Device Linking`.

  .. table:: Offloading Sections
    :name: table-offloading_sections

    +----------------------------------+--------------------------------------------------------------------+
    |             Section              | Description                                                        |
    +==================================+====================================================================+
    | omp_offloading_entries           | Offloading entry information (see :ref:`table-tgt_offload_entry`)  |
    +----------------------------------+--------------------------------------------------------------------+
    | .llvm.offloading.<triple>.<arch> | Embedded device object file for the target device and architecture |
    +----------------------------------+--------------------------------------------------------------------+

.. _Device Linking:

Linking Target Device Code
--------------------------

Objects containing :ref:`table-offloading_sections` require special handling to
create an executable device image. This is done using a Clang tool, see
:doc:`ClangLinkerWrapper` for more information. This tool works as a wrapper
over the host linking job. It scans the input object files for the offloading
sections and runs the appropriate device linking action. The linked device image
is then :ref:`wrapped <Device Binary Wrapping>` to create the symbols used to load the
device image and link it with the host.

The linker wrapper tool supports linking bitcode files through link time
optimization (LTO). This is used whenever the object files embedded in the host
contain LLVM bitcode. Bitcode will be embedded for architectures that do not
support a relocatable object format, such as AMDGPU or SPIR-V, or if the user
passed in ``-foffload-lto``.

.. _Device Binary Wrapping:

Device Binary Wrapping
----------------------

Various structures and functions are used to create the information necessary to
offload code on the device. We use the :ref:`linked device executable <Device
Linking>` with the corresponding offloading entries to create the symbols
necessary to load and execute the device image.

Structure Types
^^^^^^^^^^^^^^^

Several different structures are used to store offloading information. The
:ref:`device image structure <table-device_image_structure>` stores a single
linked device image and its associated offloading entries. The offloading
entries are stored using the ``__start_omp_offloading_entries`` and
``__stop_omp_offloading_entries`` symbols generated by the linker using the
:ref:`table-tgt_offload_entry`.

  .. table:: __tgt_device_image Structure
    :name: table-device_image_structure

    +----------------------+--------------+----------------------------------------+
    |         Type         |  Identifier  | Description                            |
    +======================+==============+========================================+
    |         void*        |  ImageStart  | Pointer to the target code start       |
    +----------------------+--------------+----------------------------------------+
    |         void*        |   ImageEnd   | Pointer to the target code end         |
    +----------------------+--------------+----------------------------------------+
    | __tgt_offload_entry* | EntriesBegin | Begin of table with all target entries |
    +----------------------+--------------+----------------------------------------+
    | __tgt_offload_entry* |  EntriesEnd  | End of table (non inclusive)           |
    +----------------------+--------------+----------------------------------------+

The target :ref:`target binary descriptor <table-target_binary_descriptor>` is
used to store all binary images and offloading entries in an array.

  .. table:: __tgt_bin_desc Structure
    :name: table-target_binary_descriptor

    +----------------------+------------------+------------------------------------------+
    |         Type         |    Identifier    | Description                              |
    +======================+==================+==========================================+
    |        int32_t       |  NumDeviceImages | Number of device types supported         |
    +----------------------+------------------+------------------------------------------+
    |  __tgt_device_image* |   DeviceImages   | Array of device images (1 per dev. type) |
    +----------------------+------------------+------------------------------------------+
    | __tgt_offload_entry* | HostEntriesBegin | Begin of table with all host entries     |
    +----------------------+------------------+------------------------------------------+
    | __tgt_offload_entry* |  HostEntriesEnd  | End of table (non inclusive)             |
    +----------------------+------------------+------------------------------------------+

Global Variables
----------------

:ref:`table-global_variables` lists various global variables, along with their
type and their explicit ELF sections, which are used to store device images and
related symbols.

  .. table:: Global Variables
    :name: table-global_variables

    +--------------------------------+---------------------+-------------------------+---------------------------------------------------------+
    |            Variable            |         Type        |       ELF Section       |                    Description                          |
    +================================+=====================+=========================+=========================================================+
    | __start_omp_offloading_entries | __tgt_offload_entry | .omp_offloading_entries | Begin symbol for the offload entries table.             |
    +--------------------------------+---------------------+-------------------------+---------------------------------------------------------+
    | __stop_omp_offloading_entries  | __tgt_offload_entry | .omp_offloading_entries | End symbol for the offload entries table.               |
    +--------------------------------+---------------------+-------------------------+---------------------------------------------------------+
    | __dummy.omp_offloading.entry   | __tgt_offload_entry | .omp_offloading_entries | Dummy zero-sized object in the offload entries          |
    |                                |                     |                         | section to force linker to define begin/end             |
    |                                |                     |                         | symbols defined above.                                  |
    +--------------------------------+---------------------+-------------------------+---------------------------------------------------------+
    | .omp_offloading.device_image   |  __tgt_device_image | .omp_offloading_entries | ELF device code object of the first image.              |
    +--------------------------------+---------------------+-------------------------+---------------------------------------------------------+
    | .omp_offloading.device_image.N |  __tgt_device_image | .omp_offloading_entries | ELF device code object of the (N+1)th image.            |
    +--------------------------------+---------------------+-------------------------+---------------------------------------------------------+
    | .omp_offloading.device_images  |  __tgt_device_image | .omp_offloading_entries | Array of images.                                        |
    +--------------------------------+---------------------+-------------------------+---------------------------------------------------------+
    | .omp_offloading.descriptor     | __tgt_bin_desc      | .omp_offloading_entries | Binary descriptor object (see :ref:`binary_descriptor`) |
    +--------------------------------+---------------------+-------------------------+---------------------------------------------------------+

.. _binary_descriptor:

Binary Descriptor for Device Images
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This object is passed to the offloading runtime at program startup and it
describes all device images available in the executable or shared library. It
is defined as follows:

.. code-block:: c

  __attribute__((visibility("hidden")))
  extern __tgt_offload_entry *__start_omp_offloading_entries;
  __attribute__((visibility("hidden")))
  extern __tgt_offload_entry *__stop_omp_offloading_entries;
  static const char Image0[] = { <Bufs.front() contents> };
  ...
  static const char ImageN[] = { <Bufs.back() contents> };
  static const __tgt_device_image Images[] = {
    {
      Image0,                            /*ImageStart*/
      Image0 + sizeof(Image0),           /*ImageEnd*/
      __start_omp_offloading_entries,    /*EntriesBegin*/
      __stop_omp_offloading_entries      /*EntriesEnd*/
    },
    ...
    {
      ImageN,                            /*ImageStart*/
      ImageN + sizeof(ImageN),           /*ImageEnd*/
      __start_omp_offloading_entries,    /*EntriesBegin*/
      __stop_omp_offloading_entries      /*EntriesEnd*/
    }
  };
  static const __tgt_bin_desc BinDesc = {
    sizeof(Images) / sizeof(Images[0]),  /*NumDeviceImages*/
    Images,                              /*DeviceImages*/
    __start_omp_offloading_entries,      /*HostEntriesBegin*/
    __stop_omp_offloading_entries        /*HostEntriesEnd*/
  };

Global Constructor and Destructor
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The global constructor (``.omp_offloading.descriptor_reg()``) registers the
device images with the runtime by calling the ``__tgt_register_lib()`` runtime
function. The constructor is explicitly defined in ``.text.startup`` section and
is run once when the program starts. Similarly, the global destructor
(``.omp_offloading.descriptor_unreg()``) calls ``__tgt_unregister_lib()`` for
the destructor and is also defined in ``.text.startup`` section and run when the
program exits.

Offloading Example
------------------

This section contains a simple example of generating offloading code using
OpenMP offloading. We will use a simple ``ZAXPY`` BLAS routine.

.. code-block:: c++

    #include <complex>

    using complex = std::complex<double>;

    void zaxpy(complex *X, complex *Y, complex D, std::size_t N) {
    #pragma omp target teams distribute parallel for
      for (std::size_t i = 0; i < N; ++i)
        Y[i] = D * X[i] + Y[i];
    }

    int main() {
      const std::size_t N = 1024;
      complex X[N], Y[N], D;
    #pragma omp target data map(to:X[0 : N]) map(tofrom:Y[0 : N])
      zaxpy(X, Y, D, N);
    }

This code is compiled using the following Clang flags.

.. code-block:: console

    $ clang++ -fopenmp -fopenmp-targets=nvptx64 -O3 zaxpy.cpp -c

The output section in the object file can be seen using the ``readelf`` utility

.. code-block:: text

  $ llvm-readelf -WS zaxpy.o
  [Nr] Name                                       Type
  ...
  [34] omp_offloading_entries                     PROGBITS
  [35] .llvm.offloading.nvptx64-nvidia-cuda.sm_70 PROGBITS

Compiling this file again will invoke the ``clang-linker-wrapper`` utility to
extract and link the device code stored at the section named
``.llvm.offloading.nvptx64-nvidia-cuda.sm_70`` and then use entries stored in
the section named ``omp_offloading_entries`` to create the symbols necessary for
``libomptarget`` to register the device image and call the entry function.

.. code-block:: console

    $ clang++ -fopenmp -fopenmp-targets=nvptx64 zaxpy.o -o zaxpy
    $ ./zaxpy

We can see the steps created by clang to generate the offloading code using the
``-ccc-print-phases`` option in Clang. This matches the description in
:ref:`Offloading Overview`.

.. code-block:: console

    $ clang++ -fopenmp -fopenmp-targets=nvptx64 -ccc-print-phases zaxpy.cpp
    # "x86_64-unknown-linux-gnu" - "clang", inputs: ["zaxpy.cpp"], output: "/tmp/zaxpy-host.bc"
    # "nvptx64-nvidia-cuda" - "clang", inputs: ["zaxpy.cpp", "/tmp/zaxpy-e6a41b.bc"], output: "/tmp/zaxpy-07f434.s"
    # "nvptx64-nvidia-cuda" - "NVPTX::Assembler", inputs: ["/tmp/zaxpy-07f434.s"], output: "/tmp/zaxpy-0af7b7.o"
    # "x86_64-unknown-linux-gnu" - "clang", inputs: ["/tmp/zaxpy-e6a41b.bc", "/tmp/zaxpy-0af7b7.o"], output: "/tmp/zaxpy-416cad.o"
    # "x86_64-unknown-linux-gnu" - "Offload::Linker", inputs: ["/tmp/zaxpy-416cad.o"], output: "a.out"
