aarch32-jdk9 SIGILL on ARMv7 during Thread init
===============================================

[aarch32-jdk9](http://hg.openjdk.java.net/aarch32-port/jdk9/) (hotspot rev `d0514d28487e`) compiled natively on ARMv7 Fedora 23 crashes on startup.

Build flags:

    ./configure \
        --with-debug-level=slowdebug \
        --with-jvm-variants=core \
        --disable-warnings-as-errors 

Excerpts from the execution trace:

    create_initial_thread
    ...
    InterpreterRuntime::resolve_from_cache (thread=0xb490fc00, bytecode=Bytecodes::_invokespecial)
    ...
    InterpreterRuntime::resolve_from_cache (thread=0xb490fc00, bytecode=Bytecodes::_putfield)
    ...
    InterpreterRuntime::resolve_from_cache (thread=0xb490fc00, bytecode=Bytecodes::_putfield)
    ...
    ldrex   r0, [r9]
    ...
    strex   r9, lr, [r9]
    SIGILL

####Debugging

Run under GDB without breakpoints until SIGILL on the following instruction:

    0x72d7dae4: strex   r9, lr, [r9]

Print the templates table (to a gdb.log):

    TemplateInterpreter::print()
    > void

Find the `Stub` in `StubQueue` that contains the problematic instruction:

    TemplateInterpreter::code()->stub_containing((address) 0x72d7dae4)
    > 0x72d7da20

Find this address in the templates table:

    grep 0x72d7da20 gdb.log -A1
    > invokedynamic  186 invokedynamic  [0x72d7d880, 0x72d7da20]  416 bytes
    > new  187 new  [0x72d7da40, 0x72d7dca0]  608 bytes

Returned `Stub` pointer points to the end of the previous stub in queue, so the target stub is the `new` one. 

`static void _new();` is declared in [share/vm/interpreter/templateTable.hpp](http://hg.openjdk.java.net/aarch32-port/jdk9/hotspot/file/d0514d28487e/src/share/vm/interpreter/templateTable.hpp#l298) and defined in [cpu/aarch32/vm/templateTable_aarch32.cpp](http://hg.openjdk.java.net/aarch32-port/jdk9/hotspot/file/d0514d28487e/src/cpu/aarch32/vm/templateTable_aarch32.cpp#l3472).

Going through the implementation we can find [eden_allocate](http://hg.openjdk.java.net/aarch32-port/jdk9/hotspot/file/d0514d28487e/src/cpu/aarch32/vm/templateTable_aarch32.cpp#l3534) call and the [strex](http://hg.openjdk.java.net/aarch32-port/jdk9/hotspot/file/d0514d28487e/src/cpu/aarch32/vm/macroAssembler_aarch32.cpp#l2097) instruction inside it.

Looking into the details for this instruction:

    STREX<c> <Rd>, <Rt>, [<Rn>]
    if d == 15 || t == 15 || n == 15 then UNPREDICTABLE;
    if d == n || d == t then UNPREDICTABLE;

We can see that the first and third agruments must be different registers and the use of the same `r9` (`rscratch1`) register causes SIGILL.

This [change from jdk8u](http://hg.openjdk.java.net/aarch32-port/jdk8u/hotspot/rev/49c2278ec735#l1.43) solves the problem (corresponding [maillist thread](http://mail.openjdk.java.net/pipermail/aarch32-port-dev/2016-February/000083.html)).

####Details

 - hotspot error report: [hs_err_pid15670.log](https://github.com/akashche/aarch32-thread-init-sigill/blob/master/hs_err_pid15670.log)
 - shortened execution trace (980 lines, started just after [StubRoutines::call_stub](http://hg.openjdk.java.net/aarch32-port/jdk9/hotspot/file/d0514d28487e/src/share/vm/runtime/javaCalls.cpp#l406) call): [gdb_short.txt](https://github.com/akashche/aarch32-thread-init-sigill/blob/master/gdb_short.txt)
 - full execution trace (50 MB): [gdb.txt.xz](https://github.com/akashche/aarch32-thread-init-sigill/blob/master/gdb.txt.xz)
 - [.gdbinit](https://github.com/akashche/aarch32-thread-init-sigill/blob/master/.gdbinit) and [commands.txt](https://github.com/akashche/aarch32-thread-init-sigill/blob/master/commands.txt) used to record the trace

####Misc

Trace stripping grep:

    grep -E '^JavaCalls|^InterpreterRuntime|^Program|^=> 0x[0-9a-f]{8}:'

