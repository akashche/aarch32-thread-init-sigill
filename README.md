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
    dsb ishst
    ...
    dsb ish
    ...
    ldrex   r0, [r9]
    ...
    strex   r9, lr, [r9]
    SIGILL

Details:

 - hotspot error report: TODO [hs_err_pid15670.log]()
 - shortened execution trace: TODO: [gdb_short.txt]().
 - full execution trace: TODO: [gdb.txt.xz]().
 - TODO: [.gdbinit]() and [commands.txt]() used to record the trace

