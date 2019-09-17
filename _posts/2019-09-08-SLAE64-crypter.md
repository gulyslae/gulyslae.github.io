---
layout: post
title:  SLAE64 Assignment 7 - custom cryptor
excerpt: custom cryptor
---

i'm not that strong in crypto, but i've read enough to be sure i won't
ever write my own encryption algorithm. i for sure can write an
obfuscator, but nothing more.

that's why i choose to use a user friendly language (python) and a
ready-to-use library.  
this time, because everybody needs an amaro after lot of work, i choose Fernet

i'll post python code to [https://github.com/gulyslae/SLAE64](https://github.com/gulyslae/SLAE64) as 07_cryptor.py

usage is pretty simple, just python2 e|d payload

because i know i'm lazy, i made it ready to accept shellcode in any format:
* 0x41,0x42,0x43
* 0x414243
* \x41\x42\x43
* or any combination of the three

for debugging purpose, i also print back decrypted payload when encrypting

*SLAE64-1497*
*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/*
