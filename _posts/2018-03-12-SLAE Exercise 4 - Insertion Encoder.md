---
layout: post
title: SLAE Exercise 4 - Insertion Encoder
subtitle: Linux x86 Insertion Encoder
tags: [SLAE]
---

**Overview**

The fourth task of the SLAE exam required a custom encoding shceme to be written. The requirements for this were as follows:

- PoC using execve-stack as the shellcode to encode and execute

**Requirements**

An insertion encoder was looked at for this exercise. This is done by inserting values into the original shellcode using a script which is then subsequently decoded when the shellcode is run by a decoder stub within the shellcode. The final shellcode will have a format similar to the following:


|Decoder stub   |  encoded shellcode|


For insertion encoder, a value will be inserted in-between the legitimate shellcode an example can be seen here whereby the value 0x10 is the inserted value:

Shellcode: 0x01, 0x02, 0x03, 0x04

Encoded shellcode: 0x01, 0x10, 0x02, 0x10, 0x03, 0x10, 0x04

To obtain the original shellcode, the role of the decoder stub for an insertion encoder is to move the legitimate shellcode values into the inserted values. An example of this can be seen below:

Encoded shellcode: 0x01, 0x10, 0x02, 0x10, 0x03, 0x10, 0x04

Iteration 1: 0x01, 0x02, 0x10, 0x03, 0x10, 0x04

Iteration 1: 0x01, 0x02, 0x03, 0x10, 0x04


*Knowing how  this works, what is needed to create this shellcode?*

- Shellcode that is to be encoded, for this the SLAE execve-stack shellcode as per exercise criteria. 
- A program to insert the required values into the original shellcode. 
- A decoder shellcode program to decode the altered shellcode and run it. 


**Shellcode**

The shellcode for the execve-stack program is as follows:

```
"\x31\xc0\x50\x68\x62\x61\x73\x68\x68\x62\x69\x6e\x2f\x68\x2f\x2f\x2f\x2f\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"

```

This is due to the requirements set out for this exercise within the exam. 

**Encoder**

For the program to insert the required values into the shellcode, the python script found [here](https://github.com/14Deep/SLAE/tree/master/Exercise%204) can be used to insert the required values into the required shellcode, including putting values at the end of the shellcode to let the decoder stub know when to stop decoding. 

The code is commented so should not need explaining, this can be seen below:

```
#!/usr/bin/python
# Python Insertion Encoder 

# input shellcode, for this exercise the execve-stack shellcode
shellcode = ("\x31\xc0\x50\x68\x62\x61\x73\x68\x68\x62\x69\x6e\x2f\x68\x2f\x2f\x2f\x2f\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80")

# the value that is wished to be inserted, can be changed to a required value
insertionvalue = 0x10

# the value that signals the end of the shellcode, make sure this is not the same as 'insertionvalue'
endvalue = '0xaa,0xaa'

# setting encoded to be populated in the upcoming loop
encoded = ""

print 'Encoded shellcode ... \n'

#loop through shellcode inserting the 'insertionvalue' inbetween
for x in bytearray(shellcode) :

	encoded += '0x' 			# add 0x to 'encoded'
	encoded += '%02x,' %x                   # add first shellcode byte in hex format (02x) and a comma
	encoded += '0x%02x,' % insertionvalue   # as above, but the chosen insertion value is used instead


encoded += endvalue				# add 'endvalue' to shellcode so the decoder knows when to stop

print encoded + '\n'				# print 'encoded'

print 'Total length of encoded shellcode is - %d' % len(bytearray(shellcode))

```


**Decoder**

The decoder utilises the jmp, call, pop technique to get the address of the encoded shellcode. From there, it iterates through the shellcode moving all of the legitimate values into the inserted valued. The shellcode is heavily commented and should clearly explain what is occurring at each step. The shellcode can be found [here](https://github.com/14Deep/SLAE/blob/master/Exercise%204/Encoder.py), and is also displayed below:

```
; Filename: InsertionDecoder.nasm
; Author:   Jake Badman
; Website:  14deep.github.io
; Purpose:  An insertion decoder to decode the shellcode for exercise 4 of SLAE 

global _start			

section .text
_start:

;Will be using jmp, call, pop to get the address of the encoded shellcode. 

				
xor eax, eax			     ; clearing registers
xor ebx, ebx
xor ecx, ecx

jmp short shellcode                  ; jumping to call_shellcode, which holds the encoded shellcode

decoder:

	pop esi                      ; pop address of Shellcode into esi
	lea edi, [esi + 1]           ; this is tracking the inserted values, edi will contain the value after esi, which is the inserted value
	inc al                       ; move 1 to eax, used as a counter for inserted values
	mov byte cl, [esi + 1]	     ; moving the first inserted value into cl, this will stay constant and be used to check when the shellcode has finished
decode:
	mov bl, byte [esi + eax]     ; bl will point to the inserted values
	xor bl, cl                   ; check if there is still shellcode to decode
	jnz short EncodedShellcode   ; if the zero flag is not set it means that the xor'd value wasn't the same as the inserted value meaning
				     ; the shellcode has finished. 


	mov bl, byte [esi + eax + 1] ; This moves the value after the inserted value into bl
	mov byte [edi], bl	     ; move bl, the legitimate value into the inserted value before it 
	inc edi			     ; increment edi so that it points to the next inserted value
	inc eax			     ; adding 2 to eax to point to the next inserted value
	inc eax	

	jmp short decode             ; loop until all shellcode has been decoded


shellcode:

	call decoder
	EncodedShellcode: db 0x31,0x10,0xc0,0x10,0x50,0x10,0x68,0x10,0x62,0x10,0x61,0x10,0x73,0x10,0x68,0x10,0x68,0x10,0x62,0x10,0x69,0x10,0x6e,0x10,0x2f,0x10,0x68,0x10,0x2f,0x10,0x2f,0x10,0x2f,0x10,0x2f,0x10,0x89,0x10,0xe3,0x10,0x50,0x10,0x89,0x10,0xe2,0x10,0x53,0x10,0x89,0x10,0xe1,0x10,0xb0,0x10,0x0b,0x10,0xcd,0x10,0x80,0x10,0xaa,0xaa

```


This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert Certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert

Student ID: 1092
