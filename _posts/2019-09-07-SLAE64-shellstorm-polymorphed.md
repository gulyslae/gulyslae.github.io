---
layout: post
title:  SLAE64 Assignment 6 - polymorphism
excerpt: create a polymorphic version of three shellcode taken from shellstorm
---

this time i choose three "uncommon" shellcode:  
* the first one changes hostname and shutdown (605)
* the second one runs a bindshell using nc (822)
* the third one is a shellcode created for SLAE64 (890)

i spent some time to try saving bytes, then i tried to hide syscall id to have a very cheap obfuscation.  
because i already used ror/rol for my SLAE32, i tried to not use it that much.

shellcodes original url are:
* [http://shell-storm.org/shellcode/files/shellcode-605.php](http://shell-storm.org/shellcode/files/shellcode-605.php)
* [http://shell-storm.org/shellcode/files/shellcode-822.php](http://shell-storm.org/shellcode/files/shellcode-822.php)
* [http://shell-storm.org/shellcode/files/shellcode-890.php](http://shell-storm.org/shellcode/files/shellcode-890.php)

i'm not discussing the morphism here, please look at .asm source code at [https://github.com/gulyslae/SLAE64](https://github.com/gulyslae/SLAE64)
* [https://github.com/gulyslae/SLAE64/605-my.asm](https://github.com/gulyslae/SLAE64/605-my.asm)
* [https://github.com/gulyslae/SLAE64/822-my.asm](https://github.com/gulyslae/SLAE64/822-my.asm)
* [https://github.com/gulyslae/SLAE64/890-my.asm](https://github.com/gulyslae/SLAE64/890-my.asm)

*SLAE64-1497*
*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/*
