---
layout: post
title:  SLAE64 extramile - hostname encoder/decoder
excerpt: encodes (and decodes) a payload using xor against sys_uname
---

i like this method to obfuscate payloads because they never run on
system that are not the real target.

the flow is very easy:
1. get hostname using syscall sys_uname
2. xor the shellcode using given value
3. print or execute

code is far from clean, for example i'm reusing the same function twice.  
i think i'll work on this code again in the future, but not now.


you can find the code on [my github](https://github.com/gulyslae/SLAE64/tree/master/xx-extramile)

and an asciinema rec of it working here:
[https://asciinema.org/a/bYP8go6GpwgvW4IXAaCDCaB11](https://asciinema.org/a/bYP8go6GpwgvW4IXAaCDCaB11)

*SLAE64-1497*
*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/*
