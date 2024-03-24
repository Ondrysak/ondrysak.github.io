---
title: JerseyCTF 2024 - Password Manager
description: Writeup for Password Manager challenge
date: 2024-03-24
tags: ["ctf", "writeup", "rev"]
---

# Password Manager

Due to the strict password requirements at NICC (four characters), Mary Morse decided to write a password manager for herself. Unfortunately, all it does is tell you if you guessed the password correctly. Can you crack it?

## Analysis

We are provided with a statically linked 64-bit ELF executable named `pw`. Running it gives us some information:

```bash
$ ./pw
Usage is ./pw <FLAG>
$ ./pw IhaveNoIdea
That's not the password.
```

```
file pw
pw: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=916c182af19798695bc7552edbfc8e89eeb41fb5, for GNU/Linux 3.2.0, not stripped
❯ ./pw
Usage is ./pw <FLAG>
❯ ./pw IhaveNoIdea
That's not the password.
```
now lets inspect this in `ghidra`, after analyzing the binary we get to the `main` function easily. We spend some time trying to understand it adding a few comments and renaming the variables that seem important for this challenge.

```
                             **************************************************************
                             *                          FUNCTION                          *
                             **************************************************************
                             undefined8 __stdcall main(int param_1, char * * param_2,
             undefined8        RAX:8          <RETURN>                                XREF[1]:     00401d6a(W)  
             int               EDI:4          param_1
             char * *          RSI:8          param_2                                 XREF[1]:     00401dba(W)  
             undefined8        RDX:8          param_3
             undefined8        RCX:8          param_4
             undefined8        R8:8           param_5
             undefined8        R9:8           param_6
             undefined8        RAX:8          zero_me_out                             XREF[1]:     00401d6a(W)  
             undefined4        EAX:4          password_comparison_result
             undefined8        RSI:8          input_user                              XREF[1]:     00401dba(W)  
             undefined8        Stack[-0x10]:8 local_10                                XREF[2]:     00401d21(W), 
                                                                                                   00401de7(R)  
             undefined1        Stack[-0x15]:1 local_15                                XREF[1]:     00401d9c(W)  
             undefined1[19]    Stack[-0x28]   final_pass                              XREF[1]:     00401dab(*)  
             undefined2        Stack[-0x38]:2 third_constant                          XREF[1]:     00401d43(W)  
             undefined8        Stack[-0x40]:8 second_constant                         XREF[1]:     00401d3f(W)  
             undefined8        Stack[-0x48]:8 first_constant                          XREF[1]:     00401d3b(W)  
             undefined4        Stack[-0x4c]:4 iterator                                XREF[5]:     00401d71(W), 
                                                                                                   00401d7a(R), 
                                                                                                   00401d89(R), 
                                                                                                   00401d92(RW), 
                                                                                                   00401d96(R)  
             undefined4        Stack[-0x5c]:4 local_5c                                XREF[2]:     00401d11(W), 
                                                                                                   00401d49(R)  
             undefined8        Stack[-0x68]:8 local_68                                XREF[3]:     00401d14(W), 
                                                                                                   00401d4f(R), 
                                                                                                   00401da0(R)  
                             main                                            XREF[3]:     Entry Point(*), 
                                                                                          _start:00401c01(*), 004b1038(*)  
        00401d05 f3 0f 1e fa     ENDBR64
        00401d09 55              PUSH       RBP
        00401d0a 48 89 e5        MOV        RBP,RSP
        00401d0d 48 83 ec 60     SUB        RSP,0x60
        00401d11 89 7d ac        MOV        dword ptr [RBP + local_5c],param_1
        00401d14 48 89 75 a0     MOV        qword ptr [RBP + local_68],param_2
        00401d18 64 48 8b        MOV        RAX,qword ptr FS:[0x28]
                 04 25 28 
                 00 00 00
        00401d21 48 89 45 f8     MOV        qword ptr [RBP + local_10],RAX
        00401d25 31 c0           XOR        EAX,EAX
        00401d27 48 b8 4f        MOV        RAX,0x164d525e4351464f
                 46 51 43 
                 5e 52 4d 16
        00401d31 48 ba 57        MOV        param_3,0x655c65487a561657
                 16 56 7a 
                 48 65 5c 65
        00401d3b 48 89 45 c0     MOV        qword ptr [RBP + first_constant],RAX
        00401d3f 48 89 55 c8     MOV        qword ptr [RBP + second_constant],param_3
        00401d43 66 c7 45        MOV        word ptr [RBP + third_constant],0x581a
                 d0 1a 58
                             Check if two parameters were supplied
        00401d49 83 7d ac 02     CMP        dword ptr [RBP + local_5c],0x2
        00401d4d 74 22           JZ         LAB_00401d71
        00401d4f 48 8b 45 a0     MOV        RAX,qword ptr [RBP + local_68]
        00401d53 48 8b 00        MOV        RAX,qword ptr [RAX]
        00401d56 48 89 c6        MOV        param_2,RAX
        00401d59 48 8d 3d        LEA        param_1,[s_Usage_is_%s_<FLAG>_00495004]          = "Usage is %s <FLAG>\n"
                 a4 32 09 00
        00401d60 b8 00 00        MOV        EAX,0x0
                 00 00
        00401d65 e8 f6 ec        CALL       printf                                           int printf(char * __format, ...)
                 00 00
        00401d6a b8 01 00        MOV        zero_me_out,0x1
                 00 00
        00401d6f eb 76           JMP        LAB_00401de7
                             LAB_00401d71                                    XREF[1]:     00401d4d(j)  
        00401d71 c7 45 bc        MOV        dword ptr [RBP + iterator],0x0
                 00 00 00 00
        00401d78 eb 1c           JMP        LAB_00401d96
                             LAB_00401d7a                                    XREF[1]:     00401d9a(j)  
        00401d7a 8b 45 bc        MOV        zero_me_out,dword ptr [RBP + iterator]
        00401d7d 48 98           CDQE
        00401d7f 0f b6 44        MOVZX      zero_me_out,byte ptr [RBP + zero_me_out*0x1 + 
                 05 c0
        00401d84 83 f0 25        XOR        zero_me_out,0x25
        00401d87 89 c2           MOV        param_3,zero_me_out
        00401d89 8b 45 bc        MOV        zero_me_out,dword ptr [RBP + iterator]
        00401d8c 48 98           CDQE
        00401d8e 88 54 05 e0     MOV        byte ptr [RBP + zero_me_out*0x1 + -0x20],param_3
        00401d92 83 45 bc 01     ADD        dword ptr [RBP + iterator],0x1
                             LAB_00401d96                                    XREF[1]:     00401d78(j)  
        00401d96 83 7d bc 11     CMP        dword ptr [RBP + iterator],0x11
                             Notice that we iterate over 0x12 characters, making this iter
        00401d9a 7e de           JLE        LAB_00401d7a
        00401d9c c6 45 f3 00     MOV        byte ptr [RBP + local_15],0x0
        00401da0 48 8b 45 a0     MOV        zero_me_out,qword ptr [RBP + local_68]
        00401da4 48 83 c0 08     ADD        zero_me_out,0x8
        00401da8 48 8b 08        MOV        param_4,qword ptr [zero_me_out]
        00401dab 48 8d 45 e0     LEA        zero_me_out=>final_pass,[RBP + -0x20]
        00401daf ba 12 00        MOV        param_3,0x12
                 00 00
        00401db4 48 89 ce        MOV        param_2,param_4
        00401db7 48 89 c7        MOV        param_1,zero_me_out
        00401dba e8 11 f3        CALL       strncmp                                          int strncmp(char * __s1, char * 
                 ff ff
        00401dbf 85 c0           TEST       password_comparison_result,password_comparison
        00401dc1 75 13           JNZ        LAB_00401dd6
        00401dc3 48 8d 3d        LEA        param_1,[s_That's_the_password!_00495018]        = "That's the password!"
                 4e 32 09 00
        00401dca e8 51 69        CALL       puts                                             int puts(char * __s)
                 01 00
        00401dcf b8 00 00        MOV        password_comparison_result,0x0
                 00 00
        00401dd4 eb 11           JMP        LAB_00401de7
                             LAB_00401dd6                                    XREF[1]:     00401dc1(j)  
        00401dd6 48 8d 3d        LEA        param_1,[s_That's_not_the_password._0049502d]    = "That's not the password."
                 50 32 09 00
        00401ddd e8 3e 69        CALL       puts                                             int puts(char * __s)
                 01 00
        00401de2 b8 01 00        MOV        password_comparison_result,0x1
                 00 00
                             LAB_00401de7                                    XREF[2]:     00401d6f(j), 00401dd4(j)  
        00401de7 48 8b 4d f8     MOV        param_4,qword ptr [RBP + local_10]
        00401deb 64 48 33        XOR        param_4,qword ptr FS:[0x28]
                 0c 25 28 
                 00 00 00
        00401df4 74 05           JZ         LAB_00401dfb
        00401df6 e8 05 25        CALL       __stack_chk_fail                                 undefined __stack_chk_fail(undef
                 05 00
                             -- Flow Override: CALL_RETURN (CALL_TERMINATOR)
                             LAB_00401dfb                                    XREF[1]:     00401df4(j)  
        00401dfb c9              LEAVE
        00401dfc c3              RET

```

which `ghidra` kindly helps us decompile to pseudo-C 

```

undefined8
main(int param_1,char **param_2,undefined8 param_3,undefined8 param_4,undefined8 param_5,
    undefined8 param_6)

{
  int password_comparison_result;
  undefined8 zero_me_out;
  ulong uVar1;
  undefined8 extraout_RDX;
  undefined8 extraout_RDX_00;
  undefined8 extraout_RDX_01;
  undefined8 uVar2;
  char *input_user;
  char *pcVar3;
  long in_FS_OFFSET;
  int iterator;
  undefined8 first_constant;
  undefined8 second_constant;
  undefined2 third_constant;
  byte final_pass [19];
  undefined local_15;
  ulong local_10;
  
  local_10 = *(ulong *)(in_FS_OFFSET + 0x28);
  first_constant = 0x164d525e4351464f;
  second_constant = 0x655c65487a561657;
  third_constant = 0x581a;
                    /* Check if two parameters were supplied
                        */
  if (param_1 == 2) {
                    /* Notice that we iterate over 0x12 characters, making this iteration go trough
                       all three constants */
    for (iterator = 0; iterator < 0x12; iterator = iterator + 1) {
      final_pass[iterator] = *(byte *)((long)&first_constant + (long)iterator) ^ 0x25;
    }
    local_15 = 0;
    input_user = param_2[1];
    password_comparison_result = strncmp((char *)final_pass,input_user,0x12);
    if (password_comparison_result == 0) {
      pcVar3 = "That\'s the password!";
      puts("That\'s the password!");
      zero_me_out = 0;
      uVar2 = extraout_RDX_00;
    }
    else {
      pcVar3 = "That\'s not the password.";
      puts("That\'s not the password.");
      zero_me_out = 1;
      uVar2 = extraout_RDX_01;
    }
  }
  else {
    input_user = *param_2;
    pcVar3 = "Usage is %s <FLAG>\n";
    printf("Usage is %s <FLAG>\n");
    zero_me_out = 1;
    uVar2 = extraout_RDX;
  }
  uVar1 = local_10 ^ *(ulong *)(in_FS_OFFSET + 0x28);
  if (uVar1 != 0) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail(pcVar3,input_user,uVar2,uVar1,param_5,param_6);
  }
  return zero_me_out;
}


```

## Coding solution script in python

Looking at the decompiled code, we notice that the password is generated by XORing three constants with `0x25`. We can directly compute this value and obtain the password.


```python
def compute_final_pass():
    # Constants
    first_constant = 0x164d525e4351464f
    second_constant = 0x655c65487a561657
    third_constant = 0x581a

    # Concatenate the first three constants as strings (assuming little-endian)
    constants_str = (
        first_constant.to_bytes(8, 'little') +
        second_constant.to_bytes(8, 'little') +
        third_constant.to_bytes(8, 'little')
    )

    # Compute final_pass by XORing with 0x25
    final_pass = bytearray()
    for byte in constants_str:
        final_pass.append(byte ^ 0x25)

    return final_pass

# Test the function
final_pass = compute_final_pass()
print("Final Password:", final_pass.decode())

```

Which leads us to the flag `jctf{wh3r3s_m@y@?}`

## Conclusion

In this write-up, we delved into the Password Manager challenge from JerseyCTF 2024. The challenge introduced a simplistic password manager that verified whether a given password was correct. Despite its apparent simplicity, the task provided an opportunity to apply reverse engineering techniques and overcome obstacles in a real-world scenario.

By analyzing the binary and reverse engineering its functionality, we gained insights into how the password verification process worked. Through careful examination and experimentation, we identified the mechanism behind the password generation and devised an effective strategy to obtain the correct password.