# RING0GDB
RING0GDB is a gdbstub that gives you the ability to debug your ps4
kernel using gdb. Like PS4GDB it's integrated into Mira and starts in the 
background as soon as you load Mira.

In the next sections I will show you what steps are needed to get started.

## How to trigger RING0GDB
RING0GDB is automatically startet during the loading process of Mira. From a
technical point of view, it does not really get started, but installation
happens by overwriting two ISRs (Interrupt Service Routine). We overwrite the
routines for INT1 (debug) and INT3 (breakpoint).

This means that as soon as one of these two interrupts get fired, RING0GDB
takes control and allows us to connect to it with gdb just like we can do for
PS4GDB.

## Example debugging exec_self_imgact
exec_self_imgact is responsinble for loading/starting executable files on the
ps4. So our goal is to set a breakpoint into this function, and debug it as
soon as it gets called by the kernel.

Like mentioned before, we must trigger one of the two interrupts.
This can be easily achieved by running a kernel payload that looks like this:

```
int main(int argc, char *argv[]){
    asm("int $3");	
    return 0;
}
```

In this repository you can find a precompiled binary triggerRING0.bin which
contains this code.

At this point the ModuleLoader i've added to Mira comes very handy. Because
we can send this binary to port **9025** and it will start RING0GDB for us.

As soon as we send the payload to port **9025** klog should show us something
like this:

```
[handle_exceptionRing0] remcomOutBuffer allocated at 0xffffa6eb440dc000
received interrupt 03 - errorCode: 0x0
RAX: 0x0000000000000202		RBX:  0xffffa6eb12eac590
RCX: 0x000000000000040d		RDX:  0xffffffff841101ef
RSI: 0xffffff8065b8bab0		RDI:  0x0000000000000000
RBP: 0xffffff8065b8baa0		RSP:  0xffffff8065b8ba58
R8:  0xffffff8065b8bb80		R9:   0xffffffff85c40a78
R10: 0x0000000000000001		R11:  0xffffffff85ba0f98
R12: 0x0000000000000000		R13:  0xffffa6eb13540b68
R14: 0xffffff8065b8bab0		R15:  0xffffff8065dd4000
RIP: 0xffffff8065dd401a		FLAGS:0x0000000000000286
CS:  0x0000000000000020		SS:   0x0000000000000028
DS:  0xe8afafafafaf003b		ES:   0x000000000000003b
FS:  0x0080000000000013		GS:   0x0000000880b8001b
kernelbase: 0xffffffff8394c000
[handle_exceptionRing0] gdb_stub: Entering main loop...
[getpacketRing0] remcomInBufferRing0 allocated at 0xffffa6eb273b0000

<-[gdb_start_serverRing0] gdb_start_server
[gdb_start_serverRing0] sys_socket: 0x3
[gdb_start_serverRing0] sys_bind: 0x0
```

At this point RING0GDB is waiting for gdb to connect on port **9946**.

Like i did for PS4GDB, i've created a file (kernel.source) with the necessary commands
gdb should execute at launch, the file looks like this:

```
set architecture i386:x86-64
set disassembly-flavor intel
target remote 192.168.0.2:9946
```

Remember to replace the ip address with the ip address of your ps4.

Next we start gdb and load the source file, this is what the result looks like:

![RING0GDB1](https://i.postimg.cc/3RZ9GywB/ring0-01.png)

So gdb is connected and we can start debugging.
Our goal was to set a breakpoint at exec_self_imgact, so we need to find it's address.

In the previous klog, you see the kernel base address which in my case is **0xffffffff8394c000**,
and i know the slide of exec_self_imgact is 0x3CE730 (fw 6.72). So we have all we need to calculate the
address to where we need to set our breakpoint.

```
0xFFFFFFFF8394C000 + 0x3CE730 = 0xFFFFFFFF83D1A730
```
So we set our breakpoint

![RING0GDB2](https://i.postimg.cc/0NyYz93g/ring0-02.png)

At this point we can tell gdb to continue (continue command), the control is passed
back to gdb as soon as our breakpoint gets triggered.

To trigger the breakpoint it should be enough to start an application, in my 
case Playroom. As we can see, as soon as Playroom gets loaded our
breakpoint triggers and we can continue with our debugging session.

![RING0GDB2](https://i.postimg.cc/02CKDL79/ring0-03.png)

## Conclusion

Debugging kernel can be a pain because of kaslr and a lot of timing and lock problems
that can happen when you keep resources locked for too long. So do not expect too much
from this ;)

Have fun and don't be evil ;)
