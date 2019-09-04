---
layout: post
title:  SLAE32 Assignment 6 - polymorphism
excerpt: create a polymorphic version of three shellcode taken from shellstorm
---

to challenge myself, i choose the three shellcode almost randomly.  
i say almost because i found that lot of shellcode have similar structure: push hardcoded string on the stack and have a syscall.  
it would be hard to use three different morphism algorithm, on very similar code, and to not waste too much bytes - leave alone to save some...

i spent some time to try saving bytes from just one of them, the one
that reads /etc/passwd (861), with a saving of 7 bytes (~6%).

i felt i've did enough and just tried to do some more interesting morphism for the other two shellcode, like using ror and rol to completely change pushed string.

shellcodes original url are:
* [links](http://shell-storm.org/shellcode/files/shellcode-632.php)
* [links](http://shell-storm.org/shellcode/files/shellcode-841.php)
* [links](http://shell-storm.org/shellcode/files/shellcode-861.php)

i'm not discussing the morphism here, please look at .asm source code at [links](https://github.com/gulyslae/SLAE32)

*SLAE-1037*
*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/*
