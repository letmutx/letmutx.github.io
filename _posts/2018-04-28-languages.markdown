---
layout: post
title:  "Types of languages"
date:   2018-04-28 11:57:48 +0530
categories: programming languages
---

Someone asked me what the difference between compiled, interpreted and bytecode based languages is and I couldn't give a satisfactory answer verbally and so this is an attempt to give a more complete answer.

People usually write programs in a high level languages: simple for people to read, understand and modify. Because a computer only understands machine code(numbers encoded in binary) these programs need to be converted into binary. A _compiler_ can do this for us. You write a program in a high level language in text, use the compiler to convert it into the machine code which the computer can understand and execute.

There are different computer architectures (x86 - the most popular desktop processor, ARM - most popular mobile processor) and the machine code for different architectures is different. So, if you have a computer program and you want to run it independent of the underlying architecture of the computer you have to compile your program for every architecture(one for ARM, x86 etc..) which is possible but isn't necessary in all the cases.

You can work around this problem by distributing the text you wrote(not the compiled version) and converting the program into platform dependent code each time the program is run. A special program called _interpreter_ will convert your program into the platform-dependent machine code. This needs the interpreter to be installed on the target computer where you want the program to be run. This is the case with Python: you give the program text, you expect the Python interpreter to be installed on the target computer and run your program using the Python interpreter.

There are two problems with this - 1. You're giving away your program in a way everyone can read and understand(which is not desired in some cases). 2. Every time your program runs, it needs to be parsed, validated and then run. (2) is text processing: a very costly operation and takes a lot of time.

Both these problems can be solved to a great extent with p-code based languages. You compile the program into some representation(often called bytecode) which is platform independent and also fast to parse and validate. And then have a program which can run the bytecode wherever you want. This still needs a compilation step: into the bytecode. This is the case with Java: you write a program in text, compile it into the bytecode, distribute the bytecode, expect the Java interpreter to be installed on the target computer and run your bytecode on the target computer.
