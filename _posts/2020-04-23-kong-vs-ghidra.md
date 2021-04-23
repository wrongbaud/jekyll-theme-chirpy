---
layout: post
title:  "Godzilla Vs. Kong Vs ... Ghidra? - Ghidra Scripting, PCode Emulation, and Password Cracking"
image: ''
date:   2021-04-23 00:06:31
tags: [ghidra,gba,pcode,reversing,firmware]
description: 'Godzilla Vs. Kong Vs ... Ghidra? - Ghidra Scripting, PCode Emulation and Password Cracking'
categories: [ghidra]
---

# Godzilla Vs. Kong Vs ... Ghidra? - Ghidra Scripting, PCode Emulation, and Password Cracking on a GBA ROM

## Overview

This month Godzilla Vs. Kong is being released. As a long-time Toho/kaiju fan, I thought it would only be appropriate to size up these two monsters against Ghidra! Over the next few months, we will be reverse engineering various King Kong and Godzilla games to see how well they hold up against an additional opponent, Ghidra!

To start, we're going to be reverse-engineering the Game Boy Advance (GBA) game Kong: King of Atlantis. This game utilizes a password system to get access to the various levels. We will be reverse engineering that functionality and hopefully putting together some scripts to aid in our analysis! Can King Kong hold his own against Ghidra's monstrous API? Let's find out.

![Password Screen](https://wrongbaud.github.io/assets/img/kong/kvg.jpg)

### Reccomended Reading
* If you're not familiar with ARM/Thumb operating modes, I would highly recommend that you review [this post](https://www.embedded.com/introduction-to-arm-thumb/). 
* If you would like to follow along, you will need to get your [Ghidra development environment set up correctly](https://wrongbaud.github.io/posts/ghidra-debugger/).

## Goals

In this post, I want to demonstrate and explain the following:

1. Using pre-existing Ghidra scripts to aid in your analysis
2. Writing your own scripts to automate analysis tasks
3. Utilize and emulate Ghidra's intermediate language PCode to aid in your analysis
4. How to crack some GBA passwords!

Kong: King of Atlantis is a side-scrolling action game released for the Game Boy Advance in 2005. It allowed you to play as various characters from the cartoon and utilized a password system for accessing levels. Passwords were displayed at the end of each level and could be entered from the password screen: 

![Password Screen](https://wrongbaud.github.io/assets/img/kong/good_job_pw_screen.png)


## Initial ROM Analysis

After our [last post](https://wrongbaud.github.io/posts/ghidra-debugger/) we now have a loader for GBA ROMs and the ability to debug them using an emulator/gdb. 

First, we will load the Kong: King of Atlantis ROM and select the GBA loader. After running autoanalysis, we can see that Ghidra has defined a large number of functions for us: 

![Functions Automatically Found](https://wrongbaud.github.io/assets/img/kong/function_count.png)

If we take a cursory look through all of these functions, we can see that most of them have a similar function prologue:

![Function Prologue Two](https://wrongbaud.github.io/assets/img/kong/func_pro_2.png)

From this, we can infer a couple of things about this codebase:

* A large amount of this code seems to be operating in [ARM's Thumb mode](https://www.cs.umd.edu/~meesh/cmsc411/website/proj01/arm/thumb.html)
* Most of these functions seem to begin with the same function prologue

While Ghidra did an excellent job at identifying many functions, we can still see some locations where autoanalysis stopped and could not define code/functions. 

![Undefined Function](https://wrongbaud.github.io/assets/img/kong/example_undefined_function.png)

If we attempt to disassemble this data as THUMB code, we see the following appear:

![Proper Disassembly](https://wrongbaud.github.io/assets/img/kong/proper_disassembly.png)

Some segments of the ROM remain unanalyzed and could contain code that we are interested in. Given that there remains a decent amount of undefined code, we have two options here moving forward to complete our analysis:

1. We can search for byte sequences / undefined code regions and create them manually
2. We can search for byte/instruction sequences that match the ```push``` instruction and attempt to automate function discovery
    * **Note**: Just because we are seeing many functions utilizing Thumb mode and beginning with a ```push``` instruction does not mean that _all_ of the functions will be like this. We are simply trying to automate what we have already observed, so these methods may still leave some functions undefined

Option one will certainly work but will take some time and will not assist us when we inevitably run into this again when reversing other GBA ROMs. If performed correctly, option two will result in a repeatable analysis tool and allow us to dive into Ghidras API as well, so let's go with option two. 

Before writing our own tool to do this, we should check and see if someone has already done something similar. Let's take a look at Ghidra's Script manager.


## Ghidra Scripting and Script Manager

Ghidra's API supports both Java and Python. You can find examples of each in Ghidra's script manager.  When attempting to automate or create a new Ghidra script, it is always good to review the existing scripts. If we cannot find what we need, we can likely find something that illustrates portions of what we want to do.  We can do this by clicking on the ```Window``` -> ```Script Manager``` doing so will bring up the following window:

![Script Window](https://wrongbaud.github.io/assets/img/kong/script_window.png)

If we look at the tree in the left of the scripts window, we can see that the scripts are categorized. Let's start by taking a look at the Functions option and see what already been developed for us:

![Script Window Functions](https://wrongbaud.github.io/assets/img/kong/script_window_functions.png)

**RE Tip:** Always check the scripts included in Ghidra when you're attempting to write a new analysis feature. There is nothing wrong with standing on the shoulders of other folks when doing RE (or any kind of work).

Sure enough, there is a script called ```MakeFunctionsScript.java``` the description is as follows:

```
Script to ask the user for a byte sequence that is a common function start  make functions at those locations  if code has only one block it asks the user where the data block is and splits the program into  code and data blocks  
```

This is very close to what we need, so let's give it a try using the bytes: ```0x30 0xB5``` this is the machine code for the instruction: ```push { r4, r5, lr } ```. 

If we double click on this script, the following window appears: 

![Script Arguments](https://wrongbaud.github.io/assets/img/kong/script_arguments.png)

After providing our byte sequence and running the script, we see the following lines of output in the console window:

```
MakeFunctionsScript.java> Function already exists at address: 082c44a0
MakeFunctionsScript.java> Function already exists at address: 082c4918
MakeFunctionsScript.java> Function already exists at address: 082c53d4
MakeFunctionsScript.java> Made function at address: 082c556c
MakeFunctionsScript.java> Function already exists at address: 082c604c
MakeFunctionsScript.java> Made function at address: 082c6138
```

Excellent! Just by searching the script repository, we were able to find another way to streamline our analysis. Let's take a look at some of the newly defined functions and see what they look like:

![Bad Function 1](https://wrongbaud.github.io/assets/img/kong/bad_function_data_section.png)

If we try to disassemble this manually in ARM mode, no assembly is generated. If we disassemble it in Thumb mode, we see the following:

![Bad Disassembly](https://wrongbaud.github.io/assets/img/kong/bad_disassembly.png)

While this sequence of bytes disassembles, this does not look like valid code. To those of you who are new to RE, this may seem a little ambiguous. How do we know that this probably isn't valid code? Some things to look for when determining whether disassembly is valid or not include:

* Is a clear entry point present? (Look for a function prologue such as a ```push``` instruction, etc.)
* Is a return instruction or method present?
* Are any instructions repeated with identical arguments (e.g.: ```MOV r0, r0```)
* Is this code segment surrounded (or close to) valid code?
* Are there any references to this code segment anywhere? (XREFs, jump tables, etc.)

We don't see any of these characteristics in the snippet posted above! So this is probably not valid code, but what is it? If we scroll up in the database, we come across strings of increasing characters like this:

![Bad Disassembly](https://wrongbaud.github.io/assets/img/kong/characters.png)

What might this data be? Remember that we're analyzing an entire Game Boy Advance Rom, so this data could be used to draw sprites, play audio, etc. In addition to code, there are tons of resources packed into this ROM, like sprites and audio files! Let's see if we can look at this ROM and determine where some of this data is located to tell our function analysis plugin to ignore these regions. If we look at the functions that have been defined thus far, we see a gap in the addresses:

![Function Gap](https://wrongbaud.github.io/assets/img/kong/function_gap.png)

If we look at what is contained within this gap, some patterns can be found. Take this byte sequence, for example:

![Bad Bytes](https://wrongbaud.github.io/assets/img/kong/bad_bytes.png)

If we try to disassemble these bytes, we get the following:

![Bad Bytes](https://wrongbaud.github.io/assets/img/kong/bad_bytes_disassembled.png)

Let's revisit the superficial criteria that we came up with earlier to determine if a byte sequence contained valid code or not:

* ~~~Is a clear entry point present? (Look for a function prologue such as a ```push``` instruction, etc.)~~~
* ~~~Is a return instruction or method present?~~~
* ~~~Is this code segment surrounded (or close to) valid code?~~~
* ~~~Are there any references to this code segment anywhere? (XREFs, jump tables, etc.)~~~

It's not looking great for this segment. If we continue to review this range manually, we find other elements of interesting data such as:

![Bad Bytes](https://wrongbaud.github.io/assets/img/kong/bad_bytes_3.png)

At this point, you're probably wondering what all of this extra data might be. Remember that this is an entire GBA ROM image. These images typically didn't contain any kind of filesystem for resource management or things of that nature, so the game's resources needed to be embedded into the ROM! This range could represent map data, tile data, audio effects, or any other in-game asset. For now, we are going to have our analysis script ignore the region between ```0x3BDC``` and ```0x2B42b0```. 

While I don't want to do a line by line code review, I do want to highlight some of the more helpful API calls that the provided ```MakeFunctionsScript.java``` script takes advantage of:

1. ```currentProgram.getMemory()``` - Returns the memory space of the program being analyzed
2. ```askBytes()``` - This function returns a sequence of bytes provided by the user
3. ```currentProgram.getMemory().getBlocks()``` - Returns an array of ```MemoryBlock``` objects that represent the memory map of the target binary
4. ```memory.findBytes(start, end, functionBytes, null, true, monitor)``` - Searches the memory space for the provided byte sequence
5. ```getFunctionContaining(found)``` - Returns the function that contains the address ```found```
6. ```disassemble(found)``` - Attempts to disassemble code at the provided address ```found```

We will need to understand how these work to modify the behavior to fit our use case, so with that - let's try to write our own version of this script that will work on this GBA ROM. 

### Writing Custom Scripts

At this point, we can manually run this script with new byte sequences to try to identify additional functions, but what if we could just edit this existing script to iterate over multiple byte sequences and disassemble where the byte sequence is found?

Using the script manager, we can edit this pre-existing script by right-clicking it and selecting: ```Edit with basic editor``` doing this opens up a simple editor:

![Basic Editor](https://wrongbaud.github.io/assets/img/kong/basic_editor.png)

As you might have noticed, some of the code that gets disassembled by this script is _not_ in thumb mode. Look at the following byte sequence, for example:

![Undefined Function Example](https://wrongbaud.github.io/assets/img/kong/example_undefined_function.png)

If we disassemble this sequence of bytes using the ```disassemble``` API call, we see the following assembly as a result:

![Improper Disassembly](https://wrongbaud.github.io/assets/img/kong/improper_disassembly.png)

While these are technically properly disassembled ARM instructions, they're not being disassembled in the correct mode. If we manually disassemble this byte sequence in Thumb mode, we see the following:

![Proper Disassembly](https://wrongbaud.github.io/assets/img/kong/proper_disassembly.png)

Now, this looks much better! We even have a branch instruction that jumps to a previously defined function. This is a sign that we selected the proper mode. 

Now that we have an idea of how this script works and how we need to change it to suit our use case, we need to modify the function finder script to:

1. Locate our specified sequence of bytes that represent a ```push``` instruction
2. Attempt to generate Thumb mode code at said address

Earlier, we tried to use the ```disassemble()``` API call to create code, but that generated ARM instructions. After looking through the included scripts with Ghidra, I found an example of generating Thumb code. You can find it below:

```
Address found = parseAddress("0x8000b58");
// We set the true flag here to disassembly in Thumb mode
ArmDisassembleCommand cmd = new ArmDisassembleCommand(found,null,true);
// Apply the command and as a result, perform the disassembly
cmd.applyTo(currentProgram);
```
Now that we can create Thumb mode code, we can modify the pre-existing ```MakeFunctionsScript.java``` program to look for ```push``` instructions and disassembly the bytes in thumb mode where these byte sequences are found. But remember, we'd like to see all instances of this push instruction (or at least as many as we can) without having to continually run this script. 

The 16 bit ```push``` instruction is encoded as follows:

| Bit | Value |
| --- | ----- | 
| 15 | 1 | 
| 14 | 0 | 
| 13 | 1 | 
| 12 | 1 | 
| 11 | 0 | 
| 10 | 1 | 
| 9 | 0 | 
| 8 | X (Used for LR) | 
| 7 | R7 | 
| 6 | R6 | 
| 5 | R5 | 
| 4 | R4 | 
| 3 | R3 | 
| 2 | R2 | 
| 1 | R1 | 
| 0 | R0 | 

This means that we want to look for all byte sequences that share the upper 6 bits of the table above, allowing for all possible register configurations to be decoded if they are found. Is this overkill? -- Yes,, but it makes an excellent example of how to try to automate some of the manual aspects of reverse engineering a ROM file like this. We will modify the existing Ghidra plugin to iterate over these byte sequences and create Thumb mode code when these values are found. 

We want to write this so that we can use it on other GBA ROMs in the future, so the first thing that I would like this script to do is ask the user for a region of memory to perform analysis on, we can do that with the following function:

```java
// Give the user an option to choose a start address and end address for the script
public Address[] getBlockInfo() throws Exception {
    int regionCount = askInt("Get Num of Regions","How many different memory regions would you like to analyze?");
    Address [] blocks = new Address[regionCount*2];
    for (int x = 0;x < regionCount; x+=2) {
        Address startAddress = askAddress("Get Start Address","Please enter the starting address of the region you wish to analyze");
        Address endAddress = askAddress("Get End Address","Please enter the end address for the region you wish to analyze");
        blocks[x] = startAddress;
        blocks[x+1] = endAddress;
    }
    return blocks;
}
```
There are only a few API calls being used in the above loop:

1. ```askInt()``` - Similar to the ```askByte()``` function we reviewed earlier
2. ```askAddress()``` - See above description

We use this subroutine to define the regions that we want to perform the analysis. The analysis loop can be seen below:

```java
// Using the provided byte sequence and address range, iterate over the addresses and look for instructions!
public int getFunctions(Address start,Address end,byte[] inst_sequence) {
    int funcCount = 0;
    boolean keepSearching = true;
    // Let the user know which byte sequence we are looking for.
    print("Searching for byte sequence: " );
    for(byte b: inst_sequence) {
        print(String.format("%02X", b));
    }
    println("");
    // Get the memory space for the current program that we are analyzing.
    Memory memory = currentProgram.getMemory();
    Address currentAddr = start;
    while(keepSearching && (!monitor.isCancelled())&& (start.compareTo(end) <= 0)) {
        // Search the memory region that we provided for our byte sequence
        Address found = memory.findBytes(start, end, inst_sequence, null, true, monitor);
        if(found != null){
            if(getFunctionContaining(found) == null) {
                //Create our command to disassemble code in thumb mode
                ArmDisassembleCommand cmd = new ArmDisassembleCommand(found,null,true);;
                cmd.applyTo(currentProgram);
                if(cmd.getDisassembledAddressSet() != null){
                    // Code was properly disassembled, create a function!
                    Function func = createFunction(found, null);
                    if (func != null) {
                        println("Made function at address: " + found.toString());
                        // Add the length of our function here so that we don't have to iterate through all of the created code.
                        start = found.add(func.getBody().getNumAddresses());
                        funcCount++;
                        break;
                    }
                }
            }
            start = found.add(2);
        // Nothing was found with memory.findBytes, time to bail!
        }else {
            keepSearching = false;
        }
    }		
    return funcCount;
}
```

So, what exactly is going on in the above loop? We provide a start and end address for the analysis and a byte sequence to search for. From there, we take advantage of the ```findBytes``` function, which will return addresses containing the provided byte sequence. 

```java
Address found = memory.findBytes(start, end, inst_sequence, null, true, monitor);
```
If a proper byte sequence is found, we check to see if it is already in a function. If not, we attempt to disassemble THUMB mode code at that address. 

```java
if(found != null){
    if(getFunctionContaining(found) == null) {
        //Create our command to disassemble code in thumb mode
        ArmDisassembleCommand cmd = new ArmDisassembleCommand(found,null,true);;
        cmd.applyTo(currentProgram);
```

If the code is disassembled correctly, we create a function and add the length of that function to our current address.

```java
if(cmd.getDisassembledAddressSet() != null){
    // Code was properly disassembled, create a function!
    Function func = createFunction(found, null);
    if (func != null) {
        println("Made function at address: " + found.toString());
        // Add the length of our function here so that we don't have to iterate through all of the created code.
        start = found.add(func.getBody().getNumAddresses());
        funcCount++;
        break;
    }
```

Finally we will tie both of these together in the ```run()``` function as follows:

```java
public void run() throws Exception {
    println("GBA Function Generation");
    int foundCount = 0;
    byte [] inst_bytes = new byte[] {0x00,(byte)0xB5};
    Address[] addrBlocks = getBlockInfo();
    for(int inst_byte = 0; inst_byte< 0xFF;inst_byte++) {
        for (int x = 0; x< addrBlocks.length; x += 2) {
            inst_bytes[0] = (byte)inst_byte;
            foundCount += getFunctions(addrBlocks[x],addrBlocks[x+1],inst_bytes);
        }
    }
    println("Made "+foundCount+ " functions");
}
```

After running this script and providing the previously mentioned regions, we generate 141 additional functions! Not bad for a few lines of Java that we borrowed from a pre-existing script. 

## Hunting for Kong's Passwords

Now that we have a fair amount of functions analyzed let's see if we can track down the code that is handling the processing of the passwords. Let's start by exploring what we know of the password system:

1. Passwords are made up of 7 characters
2. Passwords are only letters, and 17 possible values can be used
3. Passwords are immediately parsed after 7 characters are entered

We can start by looking for instructions that contain the immediate value 7. We can also see if there are any usable string references in the ROM. Let's start by looking for instructions containing the value 7. This can be done by navigating to ```Search -> For Scalars``` window from the top navigation bar. From this window, we will input the value 7 and click search; this causes the following window to appear:

![Scalar Search](https://wrongbaud.github.io/assets/img/kong/7_search.png)

There are 372 results, but only a handful of them utilize ```cmp``` instructions. The hypothesis here is that as you enter the password, a counter is incrementing with each character (notice that there is no "enter" button at the password screen and automatically parses the password after seven characters). After searching through the results containing a ```cmp```  with the value 7, we come across the function ```FUN_082ca72e``` that contains the following:

```c
if (in_stack_0000000c < 0x1e) {
    draw_string(10,1,"Wrong PassWord");
}
else {
    draw_string(10,1,"              ");
}
```

Alright, it looks like we are probably on the right track here; let's see what other functions this subroutine calls. We can do this in Ghidra by looking at the call tree. With the function selected in the decompiler window, click on the ```Window``` button in the navigation bar, and then click ```Function Call Trees```, the following window will appear:

![Function Call Tree](https://wrongbaud.github.io/assets/img/kong/function_call_tree.png)

This window contains two tables:

1. Incoming Calls - These are the functions that call this function
2. Outgoing Calls - These are the function that this function calls

Before we dive too deep into the functions that this function calls, let's see if we can better understand what is going on. Remember that we navigated to this function because we saw a comparison against the immediate value 7. That comparison can be seen code segment extracted from the function below (be warned, it's a pretty big one!)

```c

void FUN_082ca72e(void)

{
  char cVar1;
  undefined input_code;
  byte bVar2;
  int iVar3;
  undefined1 *pwd_dest;
  undefined *puVar4;
  undefined *puVar5;
  uint uVar6;
  int unaff_r8;
  byte *unaff_r10;
  uint in_stack_0000000c;
  
code_r0x082ca72e:
  FUN_082da814();
  input_related(&DAT_0300097c,&DAT_0300097e);
  FUN_082ca100();
  if (DAT_03004644 != '\x06') {
    FUN_082caf8c();
  }
  if ((((DAT_0300097e & 0x10) == 0) || (4 < DAT_03004650)) || (unaff_r8 != 0)) {
    if ((((DAT_0300097e & 0x20) != 0) && (DAT_03004650 != 0)) && (unaff_r8 == 0)) {
      DAT_03004650 = DAT_03004650 - 1;
    }
  }
  else {
    DAT_03004650 = DAT_03004650 + 1;
  }
  if ((((DAT_0300097e & 0x80) == 0) || (1 < DAT_03004651)) || (unaff_r8 != 0)) {
    if (((DAT_0300097e & 0x40) == 0) || (DAT_03004651 == 0)) goto LAB_082ca812;
    if (unaff_r8 == 0) {
      DAT_03004651 = DAT_03004651 - 1;
      goto LAB_082ca812;
    }
LAB_082ca82c:
    if (in_stack_0000000c < 0x1e) {
      draw_string(10,1,"Wrong PassWord");
    }
    else {
      draw_string(10,1,"              ");
    }
  }
  else {
    DAT_03004651 = DAT_03004651 + 1;
LAB_082ca812:
    if (unaff_r8 != 0) goto LAB_082ca82c;
    draw_string(10,1,"Enter PassWord");
  }
  if (unaff_r8 != 0) {
    if (300 < unaff_r8 + 1U) {
      PWD_LEN_CNT = 0;
      PWD_CHAR_1 = 0;
      PWD_CHAR_2 = 0;
      PWD_CHAR_3 = 0;
      PWD_CHAR_4 = 0;
      PWD_CHAR_5 = 0;
      PWD_CHAR_6 = 0;
      PWD_CHAR_7 = 0;
      PWD_CHAR_8 = 0;
    }
    goto LAB_082cac06;
  }
  if (((DAT_0300097e & 1) == 0) || (7 < PWD_LEN_CNT)) {
    if (((DAT_0300097e & 2) == 0) || (DAT_02006cfc + 1 == DAT_02006d00 + 1)) goto LAB_082cac06;
    FUN_082ca39c();
    FUN_082d5b04(1,0xc0,0xf);
    while (iVar3 = FUN_082d5b44(), iVar3 != 0) {
      FUN_082da814();
    }
    FUN_082ca514();
    FUN_082d5b04(0,0xc0,0xf);
    while (iVar3 = FUN_082d5b44(), iVar3 != 0) {
      FUN_082da814();
    }
    FUN_082cbd4c();
    goto code_r0x082ca72e;
  }
  if ((DAT_03004651 != 2) || (DAT_03004650 != 5)) {
    if (DAT_03004651 == 0) {
      if (DAT_03004650 == 0) {
        pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
        input_code = 10;
        goto LAB_082caabe;
      }
      if (DAT_03004650 != 1) {
        if (DAT_03004650 == 2) {
          pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
          input_code = 0xb;
        }
        else {
          if (DAT_03004650 == 3) {
            pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
            input_code = 2;
          }
          else {
            if (DAT_03004650 == 4) {
              pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
              input_code = 0xc;
            }
            else {
              if (DAT_03004650 != 5) goto LAB_082caac0;
              pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
              input_code = 3;
            }
          }
        }
        goto LAB_082caabe;
      }
      (&PASSCODE_DEST)[PWD_LEN_CNT] = 1;
    }
    else {
      if (DAT_03004651 == 1) {
        if (DAT_03004650 == 0) {
          pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
          input_code = 4;
        }
        else {
          if (DAT_03004650 == 1) {
            pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
            input_code = 0xd;
          }
          else {
            if (DAT_03004650 == 2) {
              pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
              input_code = 0xe;
            }
            else {
              if (DAT_03004650 == 3) {
                pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                input_code = 5;
              }
              else {
                if (DAT_03004650 == 4) {
                  pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                  input_code = 0xf;
                }
                else {
                  if (DAT_03004650 != 5) goto LAB_082caac0;
                  pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                  input_code = 6;
                }
              }
            }
          }
        }
      }
      else {
        if (DAT_03004651 != 2) goto LAB_082caac0;
        if (DAT_03004650 == 0) {
          pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
          input_code = 7;
        }
        else {
          if (DAT_03004650 == 1) {
            pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
            input_code = 0x10;
          }
          else {
            if (DAT_03004650 == 2) {
              pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
              input_code = 0x11;
            }
            else {
              if (DAT_03004650 == 3) {
                pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                input_code = 8;
              }
              else {
                if (DAT_03004650 != 4) goto LAB_082caac0;
                pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                input_code = 0x12;
              }
            }
          }
        }
      }
LAB_082caabe:
      *pwd_dest = input_code;
    }
LAB_082caac0:
    (&PWD_CHAR_1)[PWD_LEN_CNT] = 1;
    if (PWD_LEN_CNT < 7) {
      PWD_LEN_CNT = PWD_LEN_CNT + 1;
    }
    play_gax_speech_effect(0x50);
    if ((PWD_LEN_CNT != 7) ||
       (bVar2 = check_password_1(PASSCODE_DEST,(uint)PWD_CHAR_1,(uint)PWD_CHAR_2,(uint)PWD_CHAR_3,
                                 (uint)PWD_CHAR_4,(uint)PWD_CHAR_5,(uint)PWD_CHAR_6), bVar2 != 1))
    goto LAB_082cac06;
    FUN_08000680(&DAT_02006d08,0);
    password_check(PASSCODE_DEST,PWD_CHAR_1,PWD_CHAR_2,PWD_CHAR_3);
    DAT_03004680 = 5;
    calls_pw_print();
    unaff_r8 = 0;
    in_stack_0000000c = 0;
    DAT_03004650 = 0;
    DAT_03004651 = 0;
    DAT_03004652 = 0;
    *unaff_r10 = 0;
    PWD_LEN_CNT = 0;
    PWD_CHAR_1 = 0;
    PWD_CHAR_2 = 0;
    PWD_CHAR_3 = 0;
    PWD_CHAR_4 = 0;
    PWD_CHAR_5 = 0;
    PWD_CHAR_6 = 0;
    PWD_CHAR_7 = 0;
    PWD_CHAR_8 = 0;
    PWD_PARSING_RESULT = 0;
    *(undefined *)(DAT_02006cfc + 0x1b0) = 0;
    goto code_r0x082ca72e;
   
   // SEE APPENDIX FOR ENTIRE SUBROUTINE

```

One of the first things that we notice when looking at this function is a large series of if statements assign a value (labeled ```input_code```) to a memory location pointed to by the variable that I have named ```PASSCODE_DEST```. If we examine the values for the various ```input_code``` assignments, we see that the highest one is ```0x12```. Why is this number significant? Remember that ```0x11``` possible characters can be entered from the password screen! 

**But wait!** We see a ```0x12``` get assigned, but only ```0x11``` possible values available? Correct, the input codes are indexed at ```1``` and not ```0```, so this still adds up!

These values likely correspond with the character entered in the password screen, but how are they determined? We can see from the if statement that two variables determine the ```input_code``` which is later stored at the ```PASSCODE_DEST``` location: ```DAT_03004651``` and ```DAT_03004650```. Let's fire up mGBA and examine these memory values as we enter a password. We will also inspect the ```PASSCODE_DEST``` memory location and ```PWD_LEN_CNT``` and see how they are used. 

If "A" is selected in the password screen, both locations are ```0```, see the memory view below:


![Password Select 1](https://wrongbaud.github.io/assets/img/kong/passwd_select_1.png)

If "B" is selected, we see the following:

![Password Select 2](https://wrongbaud.github.io/assets/img/kong/passwd_select_2.png)

If H is selected, we see:

![Password Select 3](https://wrongbaud.github.io/assets/img/kong/passwd_select_3.png)

See the pattern yet? The values at these two memory locations represent the X and Y coordinates for the current character selection. These X and Y coordinates are then translated and used to assign the ```input_code``` value that eventually gets stored at ```PASSCODE_DEST```.

For example, let's examine the following code segment:

```c
if (DAT_03004651 == 0) {
    if (DAT_03004650 == 0) {
    pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
    input_code = 10;
    goto LAB_082caabe;
    }
```

This means that if we enter an "A" , the actual value stored in memory at ```PASSCODE_DEST``` is 10. We can verify this by entering 5 A's and examining memory at the address pointed to by ```PASSCODE_DEST```, which before I labeled it was ```DAT_030027e0```.

![Password Entered](https://wrongbaud.github.io/assets/img/kong/all_as.png)

Excellent! Here we can see that 5 0xA's were stored at this address. So with this information, we can generate a table of what character values map to which input codes. The table can be seen below:


| Character | Code | 
| --------- | ---- |
| A | 0xA | 
| B | 0x1 | 
| C | 0xB |
| D | 0x2 | 
| E | 0xC |
| F | 0x3 |
| G | 0x4 |
| H | 0xD |
| I | 0xE |
| J | 0x5 |
| K | 0xf |
| L | 0x6 |
| M | 0x7 |
| N | 0x10 |
| O | 0x11 |
| P | 0x8 |
| Q | 0x12 |

At this point in our analysis, we have identified the following:

1. Which function seems to be handling the password input verification
2. Memory locations for the codes representing the entered password

So how are these memory locations used? What happens after a password is entered? Let's look at the original reason that we visited this subroutine. To begin with, the comparison against 7:

```c
if ((PWD_LEN_CNT != 7) ||
    (bVar2 = check_password_1(PASSCODE_DEST,(uint)PWD_CHAR_1,(uint)PWD_CHAR_2,(uint)PWD_CHAR_3,
                                (uint)PWD_CHAR_4,(uint)PWD_CHAR_5,(uint)PWD_CHAR_6), bVar2 != 1))
goto LAB_082cac06;
```

We can see that if ```PWD_LEN_COUNT``` is not equal to seven, or the function that I have labeled ```check_password_1``` returns something not equal to 1 we branch to ```LAB_082cac06```, this location ultimately returns us to the top of the password processing loop. 

So we want to make sure that when we enter our 7 character password, the ```check_password_1``` function returns a 1. After that, there are several functions called which perform further processing on the user-provided password. 

Before we dig into ```check_password_1``` we should get a solid understanding of what is being passed to it. While the decompilation, in this case, looks very reasonable, it never hurts to verify against the assembly. If we look at the assembly code leading up to the function call for ```check_password_1``` we see the following:

```
                             LAB_082caae4                                    XREF[1]:     082caae0(j)  
        082caae4 22 4d           ldr        r5,[->PASSCODE_DEST]                             = 030027e0
        082caae6 28 78           ldrb       r0,[r5,#0x0]=>PASSCODE_DEST                      This is R0...
        082caae8 69 78           ldrb       r1,[r5,#0x1]=>PWD_CHAR_1                         This is R1
        082caaea aa 78           ldrb       r2,[r5,#0x2]=>PWD_CHAR_2                         This is R2
        082caaec eb 78           ldrb       r3,[r5,#0x3]=>PWD_CHAR_3                         This is R3
        082caaee 2c 79           ldrb       r4,[r5,#0x4]=>PWD_CHAR_4
        082caaf0 00 94           str        r4,[sp,#0x0]=>param_5
        082caaf2 6c 79           ldrb       r4,[r5,#0x5]=>PWD_CHAR_5
        082caaf4 01 94           str        r4,[sp,#param_6]
        082caaf6 ac 79           ldrb       r4,[r5,#0x6]=>PWD_CHAR_6
        082caaf8 02 94           str        r4,[sp,#param_7]
        082caafa 01 f0 39 ff     bl         check_password_1                                 byte check_password_1(byte param

```

We can see from this assembly snippet that the first four characters are being passed through ```r0```-```r3```. The other three characters are passed in on the stack. We can set a breakpoint using [our gdb setup from the previous post](https://wrongbaud.github.io/posts/ghidra-debugger/) and dump the register/stack content to verify this. Here is the register/stack content with 7 ```G```'s are entered:

```
(gdb) info reg
r0             0x4                 4
r1             0x4                 4
r2             0x4                 4
r3             0x4                 4
r4             0x4                 4
r5             0x30027e0           50341856
r6             0x30027e8           50341864
r7             0x3004654           50349652
r8             0x0                 0
r9             0x7                 7
r10            0x3004653           50349651
r11            0x0                 0
r12            0x264               612
sp             0x3007bc0           0x3007bc0
lr             0x82caaff           137145087
pc             0x82cc970           0x82cc970
cpsr           0x6000003f          1610612799
(gdb) x/10w $sp
0x3007bc0:      4       4       4       0
0x3007bd0:      0       0       0       -1
0x3007be0:      0       0
(gdb) 
```

Excellent, we can see that as expected r0-r3 contain 0x4 which is the input_code for the character G. This value is also seen on the stack three times. Now let's examine the ```check_password_1``` function and see what it contains:

```c
byte check_password_1(byte param_1,uint param_2,uint param_3,uint param_4,uint param_5,uint param_6,
                     uint param_7)

{
  uint uVar1;
  uint uVar2;
  uint uVar3;
  byte bVar4;
  uint passwd_byte_2;
  uint password_counter;
  uint passwd_byte_3;
  uint passwd_byte_4;
  uint passwd_byte_5;
  uint passwd_byte_6;
  uint passwd_byte_7;
  uint uVar5;
  uint uVar6;
  uint uVar7;
  uint uVar8;
  uint uVar9;
  uint uVar10;
  char passwd_nonce [8];
  undefined return_value;
  uint local_30;
  uint local_2c;
  uint local_28;
  char nonce_val;
  
  passwd_byte_2 = param_2 & 0xff;
  passwd_byte_3 = param_3 & 0xff;
  passwd_byte_4 = param_4 & 0xff;
  passwd_byte_5 = param_5 & 0xff;
  passwd_byte_6 = param_6 & 0xff;
  passwd_byte_7 = param_7 & 0xff;
  bVar4 = 1;
  local_30 = 0;
  local_2c = 0;
                    /* First thing that we see here is that all of the values have to be less than
                       8?
                        */
  local_28 = 0;
  if (((((8 < param_1) || (8 < passwd_byte_2)) || (8 < passwd_byte_3)) ||
      ((8 < passwd_byte_4 || (8 < passwd_byte_5)))) || ((8 < passwd_byte_6 || (8 < passwd_byte_7))))
  {
    return 0;
  }
                    /* Starts with B
                       If it starts with B, then the return value (1) is stored in local_3a */
  return_value = 1;
  if (param_1 == 1) {
    passwd_nonce[1] = 0;
                    /* Return value (1) stored here */
    passwd_nonce[2] = 1;
    passwd_nonce[3] = 2;
    passwd_nonce[4] = 3;
    passwd_nonce[5] = 4;
    passwd_nonce[6] = 5;
  }
  else {
                    /* Starts with D */
    if (param_1 == 2) {
      passwd_nonce[1] = 5;
      passwd_nonce[2] = 0;
      passwd_nonce[3] = 1;
      passwd_nonce[4] = param_1;
      passwd_nonce[5] = 3;
      passwd_nonce[6] = 4;
    }
    else {
                    /* Starts with F */
      if (param_1 == 3) {
        passwd_nonce[1] = 4;
        passwd_nonce[2] = 5;
        passwd_nonce[3] = 0;
        passwd_nonce[4] = 1;
        passwd_nonce[5] = 2;
        passwd_nonce[6] = param_1;
      }
      else {
                    /* Starts with G */
        if (param_1 == 4) {
          passwd_nonce[1] = 3;
          passwd_nonce[2] = param_1;
          passwd_nonce[3] = 5;
          passwd_nonce[4] = 0;
          passwd_nonce[5] = 1;
          passwd_nonce[6] = 2;
        }
        else {
                    /* Starts with J */
          if (param_1 == 5) {
            passwd_nonce[1] = 2;
            passwd_nonce[2] = 3;
            passwd_nonce[3] = 4;
            passwd_nonce[4] = param_1;
            passwd_nonce[5] = 0;
          }
          else {
                    /* Starts with L */
            if (param_1 == 6) {
              passwd_nonce[1] = 1;
              passwd_nonce[2] = 2;
              passwd_nonce[3] = 3;
              passwd_nonce[4] = 4;
            }
            else {
                    /* Starts with M */
              if (param_1 == 7) {
                passwd_nonce[1] = 3;
                passwd_nonce[2] = 1;
                passwd_nonce[3] = 2;
                passwd_nonce[4] = 4;
              }
              else {
                if (param_1 != 8) goto LAB_082ccad8;
                passwd_nonce[4] = 4;
                passwd_nonce[2] = 1;
                passwd_nonce[3] = 2;
                passwd_nonce[1] = 3;
              }
            }
            passwd_nonce[5] = 5;
            return_value = 0;
          }
          passwd_nonce[6] = return_value;
        }
      }
    }
  }
LAB_082ccad8:
  password_counter = 1;
  uVar5 = 0;
  uVar7 = 0;
  uVar9 = 0;
  do {
    nonce_val = passwd_nonce[password_counter];
    uVar6 = uVar5;
    uVar8 = uVar7;
    uVar10 = uVar9;
    uVar1 = local_30;
    uVar2 = local_2c;
    uVar3 = local_28;
    if (nonce_val == '\0') {
      uVar10 = passwd_byte_2;
      if (((((password_counter != 1) && (uVar10 = passwd_byte_3, password_counter != 2)) &&
           (uVar10 = passwd_byte_4, password_counter != 3)) &&
          ((uVar10 = passwd_byte_5, password_counter != 4 &&
           (uVar10 = passwd_byte_6, password_counter != 5)))) &&
         (uVar10 = passwd_byte_7, password_counter != 6)) {
        uVar10 = uVar9;
      }
    }
    else {
      if (nonce_val == '\x01') {
        uVar1 = passwd_byte_2;
        if (((password_counter != 1) && (uVar1 = passwd_byte_3, password_counter != 2)) &&
           (((uVar1 = passwd_byte_4, password_counter != 3 &&
             ((uVar1 = passwd_byte_5, password_counter != 4 &&
              (uVar1 = passwd_byte_6, password_counter != 5)))) &&
            (uVar1 = local_30, password_counter == 6)))) {
          uVar1 = passwd_byte_7;
        }
      }
      else {
        if (nonce_val == '\x02') {
          uVar2 = passwd_byte_2;
          if (((password_counter != 1) && (uVar2 = passwd_byte_3, password_counter != 2)) &&
             ((uVar2 = passwd_byte_4, password_counter != 3 &&
              (((uVar2 = passwd_byte_5, password_counter != 4 &&
                (uVar2 = passwd_byte_6, password_counter != 5)) &&
               (uVar2 = local_2c, password_counter == 6)))))) {
            uVar2 = passwd_byte_7;
          }
        }
        else {
          if (nonce_val == '\x03') {
            uVar8 = passwd_byte_2;
            if ((((password_counter != 1) && (uVar8 = passwd_byte_3, password_counter != 2)) &&
                (uVar8 = passwd_byte_4, password_counter != 3)) &&
               (((uVar8 = passwd_byte_5, password_counter != 4 &&
                 (uVar8 = passwd_byte_6, password_counter != 5)) &&
                (uVar8 = uVar7, password_counter == 6)))) {
              uVar8 = passwd_byte_7;
            }
          }
          else {
            if (nonce_val == '\x04') {
              uVar3 = passwd_byte_2;
              if ((((password_counter != 1) && (uVar3 = passwd_byte_3, password_counter != 2)) &&
                  ((uVar3 = passwd_byte_4, password_counter != 3 &&
                   ((uVar3 = passwd_byte_5, password_counter != 4 &&
                    (uVar3 = passwd_byte_6, password_counter != 5)))))) &&
                 (uVar3 = local_28, password_counter == 6)) {
                uVar3 = passwd_byte_7;
              }
            }
            else {
              if ((((nonce_val == '\x05') && (uVar6 = passwd_byte_2, password_counter != 1)) &&
                  ((uVar6 = passwd_byte_3, password_counter != 2 &&
                   (((uVar6 = passwd_byte_4, password_counter != 3 &&
                     (uVar6 = passwd_byte_5, password_counter != 4)) &&
                    (uVar6 = passwd_byte_6, password_counter != 5)))))) &&
                 (uVar6 = uVar5, password_counter == 6)) {
                uVar6 = passwd_byte_7;
              }
            }
          }
        }
      }
    }
    local_28 = uVar3;
    local_2c = uVar2;
    local_30 = uVar1;
    password_counter = password_counter + 1 & 0xff;
    uVar5 = uVar6;
    uVar7 = uVar8;
    uVar9 = uVar10;
  } while (password_counter < 7);
  if (((((uVar10 != 1) || (uVar8 != 8)) && ((uVar10 != 2 || (uVar8 != 7)))) &&
      ((uVar10 != 3 || (uVar8 != 6)))) &&
     ((((uVar10 != 4 || (uVar8 != 5)) &&
       ((((uVar10 != 5 || (uVar8 != 4)) && ((uVar10 != 6 || (uVar8 != 3)))) &&
        ((uVar10 != 7 || (uVar8 != 2)))))) && ((uVar10 != 8 || (uVar8 != 1)))))) {
    bVar4 = 0;
  }
  if (local_30 + 4 != local_28) {
    bVar4 = 0;
  }
  if (8 - local_2c != uVar6) {
    bVar4 = 0;
  }
  return bVar4;
}
```

Here we have another lengthy subroutine, but there is an interesting check at the beginning:

```
  if (((((8 < param_1) || (8 < passwd_byte_2)) || (8 < passwd_byte_3)) ||
      ((8 < passwd_byte_4 || (8 < passwd_byte_5)))) || ((8 < passwd_byte_6 || (8 < passwd_byte_7))))
  {
    return 0;
  }
```

It would appear that _all_ valid passwords have to contain coded values less than 8! This means that the following characters are the _only_ good characters for a password:

| Character | Code | 
| --------- | ---- |
| B | 0x1 | 
| D | 0x2 | 
| F | 0x3 |
| G | 0x4 |
| J | 0x5 |
| L | 0x6 |
| M | 0x7 |

Next, we can see that based on the first character in the password, an array that I have labeled ```passwd_nonce``` is populated with values ```0-6```, this array is then iterated over in a sizeable do-while loop. 

Let's regroup here and review how our inputs are processed:

1. Letters are mapped to ```input_code``` values based on the X/Y coordinates of the letter that was selected
2. This list of 7 ```input_code``` values is passed to ```check_password_1```
3. ```check_password_1``` populates another array I've labeled ```passwd_nonce``` based on the first letter of the provided password.
4. Lots of calculations are performed comparing both the ```passwd_nonce``` array and the original values passed into ```check_passwd_1```

The next logical step here would be to analyze these while loops and figure out exactly how all these values are related ... 

But wait a minute ... we're supposed to be channeling our favorite kaiju with this post. Would they spend time worrying about details, or would they apply brute force methods? 

![Ghidra Wailing](https://64.media.tumblr.com/75b1ec67a8d075a3cd891f2e8c473a01/6088640a3acbc2ce-b8/s640x960/293917fca6f3a4d1a4a39ef29e8e3a590e417861.gifv)

Remember, we know that only 7 possible values can be provided to this function, and those values themselves must be less than 8. This leaves us with 7^7 possible combinations. Let's ask ourselves a question that is almost always relevant when reverse engineering: What Would Kong/Godzilla/Ghidra do?

**Brute Force!**

Using Ghidras PCode emulation capabilities, we can emulate this function 7^7 times and record which password combinations allow this function to return 1. But before we dive into that, let's talk about PCode.

## PCode  

PCode is a language defined within Ghidra that is used to model processor behavior. This language describes what instructions _do_ and how they affect the CPU and its memory space. In the same way that a hardware design language models how a CPU operates logically, PCode is used to define how instructions work via a [predetermined set of operators](https://ghidra.re/courses/languages/html/pcodedescription.html). [The documentation](https://ghidra.re/courses/languages/html/pcoderef.html) refers to it as a register transfer language, and you can also think of it as an Intermediate Language (IL). The PCode for a given instruction defines precisely what the instruction _does_. For example, if we examine the PCode for the ARM instruction ```MOV r8,r1``` we see the following:

```
    r8 = COPY r1 // This is the PCode
```

```COPY``` is one of the many available PCode operations used to define instruction behavior, and this specific PCode example copies the value of r1 into r8, which is precisely what the ```MOV r8,r1``` instruction does!

[Sleigh](https://ghidra.re/courses/languages/html/sleigh.html) is used to define a processor and translate machine code instructions into a sequence of PCode operations. These PCode operations attempt to model the processor's behavior and serve as an intermediate language that can be analyzed across each CPU architecture that Ghidra supports. This is why the decompiler works across multiple architectures because it is the PCode that is being analyzed instead of the assembly language itself.

Sleigh defines the actual decoding of the instruction, which is used to generate the disassembly that we can see in the listing view. For each decoded instruction, there is a PCode representation that defines how that specific instruction behaves. For example, if we look at the sleigh file for the 6502 processor, we see the following:

```
:AND OP1     is (cc=1 & aaa=1) ... & OP1
{ 
	A = A & OP1; 
	resultFlags(A);
}
```

The first line: ```:AND OP1     is (cc=1 & aaa=1) ... & OP1``` is used to decode the actual instruction and formats the listing display. The code contained in the brackets is the actual PCode, which in this case defines the ```AND``` instruction behavior for the 6502 processor. This instruction performs a logical AND operation on register A and stores the result in A. 

When the analysis is performed, the SLEIGH file is used to decode the instruction, then the PCode is generated. This PCode is then analyzed by the decompiler, which then generates our decompiler output. If you want to examine the PCode in the memory listing, click the following icon at the top of the listing display:

![Icon](https://wrongbaud.github.io/assets/img/kong/icon_listing.png)

After that, enable the PCode display by right-clicking the PCode bar and clicking enable:

![PCode](https://wrongbaud.github.io/assets/img/kong/pcode_enable.png)

**Note:** I would not recommend leaving this because it makes the disassembly challenging to read.

![PCode](https://wrongbaud.github.io/assets/img/kong/pcode_in_listing.png)

So why do we care about any of this? What does this have to do with our password cracking goals? Ghidra can be used to emulate these PCode operations! This will allow us to emulate a specific segment of PCode with specific register and memory values.  

Using Ghidra's PCode emulation, we can emulate a subroutine, set breakpoints, and even examine register/memory contents. We will emulate the ```check_password_1``` function with all possible combinations of password values and see which values cause the function to return the value ```1```. This will be used to generate a list of all possible passwords for this game.

## Emulating PCode

We will use Ghidra's PCode emulation to emulate our function of interest with every possible password combination. To do this, we will need to do the following:

1. Set up memory values and register values to reflect the password being tested
2. Set a breakpoint on the return instruction and check the return value

We know from our breakpoint that we set earlier that the first four characters of the password are passed in ```r0``` through ```r3``` and the other three are passed in on the stack. We also know the state of all of the registers and can set them to the same values we dumped out using GDB. 

Lucky for us, Ghidra's ```EmulatorHelper``` class has everything we need to properly set up memory. Let's take a look at the function that I wrote to prepare the memory space before emulation:

```java
	private void SetupGBAMemory(byte [] passwdVals) {
		// Set our stack pointer value to the same value we examined in GDB
		emuHelper.writeRegister(emuHelper.getStackPointerRegister(), 0x3007bc0);
		try {
			// Write the first four characters of our password that we are testing
			emuHelper.writeRegister("r0",passwdVals[0]);
			emuHelper.writeRegister("r1",passwdVals[1]);
			emuHelper.writeRegister("r2",passwdVals[2]);
			emuHelper.writeRegister("r3",passwdVals[3]);
			// Copy values in from the dump obtained in GDB
			emuHelper.writeRegister("r4",4);
			emuHelper.writeRegister("r5",0x30027e0);
			emuHelper.writeRegister("r6",0x30027e8);
			emuHelper.writeRegister("r7",0x3004654);
			emuHelper.writeRegister("r8",0x0);
			emuHelper.writeRegister("r9",0x7);
			emuHelper.writeRegister("r10",0x3004653);
			emuHelper.writeRegister("r11",0x0);
			emuHelper.writeRegister("r12",0x264);
			emuHelper.writeRegister("sp", 0x3007bc0);
			emuHelper.writeRegister("lr", 0x82caaff);
			emuHelper.writeRegister("cpsr", 0x6000003f);
			emuHelper.writeRegister("CY", 0);
			// Write the last three characters onto the stack
			emuHelper.writeStackValue(0, 4, passwdVals[4]);
			emuHelper.writeStackValue(4, 4, passwdVals[5]);
			emuHelper.writeStackValue(8, 4, passwdVals[6]);
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
```

The two functions that we use here are:
* ```writeRegister```: Used to write to a register value for a given emulation context 
* ```writeStackValue```: Used to write to a value on the stack

After calling this function, our emuHelper object now has all of the proper memory values populated in order to emulate the ```check_password_1``` function. Now we need to actually emulate it, to do this we will use the following subroutine:

```java
public void passwd_crack(byte[] passwdVals) throws Exception{
    returnAddress = getAddress(0x82cccba);
    mainFunctionEntry = getSymbolAddress("check_password_1");
    
    // Obtain entry instruction in order to establish initial processor context
    Instruction entryInstr = getInstructionAt(mainFunctionEntry);
    // Instantiate our emulator helper
    emuHelper = new EmulatorHelper(currentProgram);
    char[] passwdChars = {'B','D','F','G','J','L','M'};
    SetupGBAMemory(passwdVals);
    emuHelper.writeRegister(emuHelper.getPCRegister(), mainFunctionEntry.getOffset());
    
    try {
        emuHelper.setBreakpoint(returnAddress);
        // Execution loop until return from function or error occurs
        while (!monitor.isCancelled()) {
            emuHelper.run(mainFunctionEntry, entryInstr, monitor);
            Address executionAddress = emuHelper.getExecutionAddress();
            if (monitor.isCancelled()) {
                println("Emulation cancelled");
                return;
            }
            if (executionAddress.equals(returnAddress)) {
                byte retVal = emuHelper.readRegister("r0").byteValue();
                if(retVal == 1) {
                    String password = "";
                    for(int x =0;x<7;x++) {
                        password += passwdChars[passwdVals[x]-1];
                    }
                    println("Valid password found with password Vals: " + Arrays.toString(passwdVals) + "Password: "+password);
                    bw.write(password);
                    bw.newLine();
                }
                return;
            }
        }
    }
    finally {
        emuHelper.dispose();
    }
}
	

```

Let's walk through what this subroutine is doing. First, we set the appropriate return address to use as our stop point for the emulator. This will be the return instruction at the end of the ```check_password_1``` function. 

```
returnAddress = getAddress(0x82cccba);
```

Next, we define the entry point for our emulation. This will be the address of the ```check_password_1``` function and the first instruction in this function. 

```
mainFunctionEntry = getSymbolAddress("check_password_1");
// Obtain entry instruction in order to establish initial processor context
Instruction entryInstr = getInstructionAt(mainFunctionEntry);
```

After this, we create our [```EmulatorHelper```](https://ghidra.re/ghidra_docs/api/ghidra/app/emulator/EmulatorHelper.html) object and call the ```SetupGBAMemory``` function with the byte array that was provided to this function. Then we set the program counter register to the function entry point:

```
emuHelper = new EmulatorHelper(currentProgram);
SetupGBAMemory(passwdVals);
emuHelper.writeRegister(emuHelper.getPCRegister(), mainFunctionEntry.getOffset());
```

The following ```try```/```catch``` block contains the code that will start the emulation, set a breakpoint at our return address, and monitor the emulator for this breakpoint to be executed. If the breakpoint is triggered and the return instruction is being executed, we read out the ```r0``` value and check if it is ```1```, which would indicate a valid password. If this is the case, then we write the password out to a file for further examination! The important functions to understand here are:

* ```emuHelper.setBreakpoint(returnAddress)```: Sets a breakpoint on our return address
* ```emuHelper.run(mainFunctionEntry, entryInstr, monitor);```: Launches the emulation, this blocks until execution stops (i.e our breakpoint is hit!)
* ```emuHelper.readRegister("r0")```: Allows for the reading of register values during PCode emulation
* ```emuHelper.getExecutionAddress()```: Returns the current execution address of the emulator

With this subroutine, we can provide a password to test, set up memory appropriately and then emulate the PCode for our function of interest. We can then check the return value to see if a valid password was provided, if so we record that to a file. So now all we have to do is call this function 7^7 times with each possible password combination. To generate these combinations we'll use the following subroutine:

```java
private void permute(byte[] a, int k) {
    int n = a.length;
    if (k < 1 || k > n)
        throw new IllegalArgumentException("Illegal number of positions.");
    int[] indexes = new int[n];
    int total = (int) Math.pow(n, k);
    byte[] passTest = {1,1,1,1,1,1,1};
    while (total-- > 0) {
        for (int i = 0; i < n - (n - k); i++)
            passTest[i] = a[indexes[i]];
        try {
            // Emulate our PCode!
            passwd_crack(passTest);
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        for (int i = 0; i < n; i++) {
            if (indexes[i] >= n - 1) {
                indexes[i] = 0;
            } else {
                indexes[i]++;
                break;
            }
        }
    }
}
```

Finally, we can wrap all of this up in our main run function:

```java
protected void run() throws Exception {
    println("Kong Emulation Script Starting...");
    fw = new FileWriter("/home/wrongbaud/kong-passwords.txt", true);
    bw = new BufferedWriter(fw);
    byte[] chars = {1,2,3,4,5,6,7};
    permute(chars, 7);
    bw.close();
}
```

Let's run it! We can select our ```KongBruteForce.java``` script from the Scripts window and let it wreak havoc on this binary. 

```
KongBruteForce.java> Kong Emulation Script Starting...
KongBruteForce.java> 2021/04/22 23:32:45
KongBruteForce.java> Kong Emulation Script Ending...found 882 passwords
KongBruteForce.java> 2021/04/23 01:34:46
KongBruteForce.java> Finished!
```

Success! It looks like we have generated [882 valid passcodes](https://github.com/wrongbaud/ghidra-utils/blob/main/GBA/kong-passwords.txt) for this game! They appear to be level select cheats, with variations on the playable characters and the number of provided lives. 
 I have not taken the time to analyze every one of them, so some may have repeated functionality. Here are some examples that spawn a Kong level with a varying amount of lives:

Result of passcode LBMDJBM
![7 Lives](https://wrongbaud.github.io/assets/img/kong/7_lives.png)

Result of passcode GJJBGBM
![4 Lives](https://wrongbaud.github.io/assets/img/kong/4_lives.png)

Result of Passcode GLJBFBM
![7 Lives](https://wrongbaud.github.io/assets/img/kong/3_lives.png)

I have attached the [codes](https://github.com/wrongbaud/ghidra-utils/blob/main/GBA/kong-passwords.txt) at the end of this post. There are a _lot_ that I haven't tested, so have fun. We have 882 passcodes for the GBA ROM Kong: King of Atlantis!

## Conclusion

This was a much longer post than I anticipated. Thank you for taking the time to read it. I hope you enjoyed my ramblings about GBA ROM reversing and hopefully learned a little something about extending Ghidra along the way. 

With this exercise, we learned:

* How to write Ghidra scripts in Java
* Steps to determine whether code is valid or not
* How to write PCode emulation harness in Java and perform PCode emulation

Using these techniques, we were able to identify the function containing the password parsing and verification. Once these were identified, we developed a PCode emulation harness to brute force all 823543 possible password combinations. I think that it's safe to say that Ghidra beat Kong this time. [Here are all 882 available passwords for the ROM](https://github.com/wrongbaud/ghidra-utils/blob/main/GBA/kong-passwords.txt)]. All of the scripts that were developed for this work [can be found in my ghidra-utils repository](https://github.com/wrongbaud/ghidra-utils).

All of the scripts used here can be found in my [ghidra-utils](https://github.com/wrongbaud/ghidra-utils) repository. I plan on updating this with scripts that I write throughout these posts and with links to other helpful Ghidra resources. All pull requests are welcome!

If you would like to learn more about Ghidra, check out [this free course](https://hackaday.io/course/172292-introduction-to-reverse-engineering-with-ghidra) that I  put together with HackadayU. If you want to learn more about reverse engineering embedded systems and extend Ghidra to analyze the code that runs on them [my courses through VoidStar Security](https://www.voidstarsec.com/hardware-training) may be of interest to you. Lastly, if you have any questions about this post or want to reach out, please feel free to do so on [Twitter](https://twitter.com/wrongbaud) or at contact@voidstarsec.com

## Appendix

### Password Parsing Function

```c

void FUN_082ca72e(void)

{
  char cVar1;
  undefined input_code;
  byte bVar2;
  int iVar3;
  undefined1 *pwd_dest;
  undefined *puVar4;
  undefined *puVar5;
  uint uVar6;
  int unaff_r8;
  byte *unaff_r10;
  uint in_stack_0000000c;
  
code_r0x082ca72e:
  FUN_082da814();
  input_related(&DAT_0300097c,&DAT_0300097e);
  FUN_082ca100();
  if (DAT_03004644 != '\x06') {
    FUN_082caf8c();
  }
  if ((((DAT_0300097e & 0x10) == 0) || (4 < DAT_03004650)) || (unaff_r8 != 0)) {
    if ((((DAT_0300097e & 0x20) != 0) && (DAT_03004650 != 0)) && (unaff_r8 == 0)) {
      DAT_03004650 = DAT_03004650 - 1;
    }
  }
  else {
    DAT_03004650 = DAT_03004650 + 1;
  }
  if ((((DAT_0300097e & 0x80) == 0) || (1 < DAT_03004651)) || (unaff_r8 != 0)) {
    if (((DAT_0300097e & 0x40) == 0) || (DAT_03004651 == 0)) goto LAB_082ca812;
    if (unaff_r8 == 0) {
      DAT_03004651 = DAT_03004651 - 1;
      goto LAB_082ca812;
    }
LAB_082ca82c:
    if (in_stack_0000000c < 0x1e) {
      draw_string(10,1,"Wrong PassWord");
    }
    else {
      draw_string(10,1,"              ");
    }
  }
  else {
    DAT_03004651 = DAT_03004651 + 1;
LAB_082ca812:
    if (unaff_r8 != 0) goto LAB_082ca82c;
    draw_string(10,1,"Enter PassWord");
  }
  if (unaff_r8 != 0) {
    if (300 < unaff_r8 + 1U) {
      PWD_LEN_CNT = 0;
      PWD_CHAR_1 = 0;
      PWD_CHAR_2 = 0;
      PWD_CHAR_3 = 0;
      PWD_CHAR_4 = 0;
      PWD_CHAR_5 = 0;
      PWD_CHAR_6 = 0;
      PWD_CHAR_7 = 0;
      PWD_CHAR_8 = 0;
    }
    goto LAB_082cac06;
  }
  if (((DAT_0300097e & 1) == 0) || (7 < PWD_LEN_CNT)) {
    if (((DAT_0300097e & 2) == 0) || (DAT_02006cfc + 1 == DAT_02006d00 + 1)) goto LAB_082cac06;
    FUN_082ca39c();
    FUN_082d5b04(1,0xc0,0xf);
    while (iVar3 = FUN_082d5b44(), iVar3 != 0) {
      FUN_082da814();
    }
    FUN_082ca514();
    FUN_082d5b04(0,0xc0,0xf);
    while (iVar3 = FUN_082d5b44(), iVar3 != 0) {
      FUN_082da814();
    }
    FUN_082cbd4c();
    goto code_r0x082ca72e;
  }
  if ((DAT_03004651 != 2) || (DAT_03004650 != 5)) {
    if (DAT_03004651 == 0) {
      if (DAT_03004650 == 0) {
        pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
        input_code = 10;
        goto LAB_082caabe;
      }
      if (DAT_03004650 != 1) {
        if (DAT_03004650 == 2) {
          pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
          input_code = 0xb;
        }
        else {
          if (DAT_03004650 == 3) {
            pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
            input_code = 2;
          }
          else {
            if (DAT_03004650 == 4) {
              pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
              input_code = 0xc;
            }
            else {
              if (DAT_03004650 != 5) goto LAB_082caac0;
              pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
              input_code = 3;
            }
          }
        }
        goto LAB_082caabe;
      }
      (&PASSCODE_DEST)[PWD_LEN_CNT] = 1;
    }
    else {
      if (DAT_03004651 == 1) {
        if (DAT_03004650 == 0) {
          pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
          input_code = 4;
        }
        else {
          if (DAT_03004650 == 1) {
            pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
            input_code = 0xd;
          }
          else {
            if (DAT_03004650 == 2) {
              pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
              input_code = 0xe;
            }
            else {
              if (DAT_03004650 == 3) {
                pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                input_code = 5;
              }
              else {
                if (DAT_03004650 == 4) {
                  pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                  input_code = 0xf;
                }
                else {
                  if (DAT_03004650 != 5) goto LAB_082caac0;
                  pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                  input_code = 6;
                }
              }
            }
          }
        }
      }
      else {
        if (DAT_03004651 != 2) goto LAB_082caac0;
        if (DAT_03004650 == 0) {
          pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
          input_code = 7;
        }
        else {
          if (DAT_03004650 == 1) {
            pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
            input_code = 0x10;
          }
          else {
            if (DAT_03004650 == 2) {
              pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
              input_code = 0x11;
            }
            else {
              if (DAT_03004650 == 3) {
                pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                input_code = 8;
              }
              else {
                if (DAT_03004650 != 4) goto LAB_082caac0;
                pwd_dest = &PASSCODE_DEST + PWD_LEN_CNT;
                input_code = 0x12;
              }
            }
          }
        }
      }
LAB_082caabe:
      *pwd_dest = input_code;
    }
LAB_082caac0:
    (&PWD_CHAR_1)[PWD_LEN_CNT] = 1;
    if (PWD_LEN_CNT < 7) {
      PWD_LEN_CNT = PWD_LEN_CNT + 1;
    }
    play_gax_speech_effect(0x50);
    if ((PWD_LEN_CNT != 7) ||
       (bVar2 = check_password_1(PASSCODE_DEST,(uint)PWD_CHAR_1,(uint)PWD_CHAR_2,(uint)PWD_CHAR_3,
                                 (uint)PWD_CHAR_4,(uint)PWD_CHAR_5,(uint)PWD_CHAR_6), bVar2 != 1))
    goto LAB_082cac06;
    FUN_08000680(&DAT_02006d08,0);
    password_check(PASSCODE_DEST,PWD_CHAR_1,PWD_CHAR_2,PWD_CHAR_3);
    DAT_03004680 = 5;
    calls_pw_print();
    unaff_r8 = 0;
    in_stack_0000000c = 0;
    DAT_03004650 = 0;
    DAT_03004651 = 0;
    DAT_03004652 = 0;
    *unaff_r10 = 0;
    PWD_LEN_CNT = 0;
    PWD_CHAR_1 = 0;
    PWD_CHAR_2 = 0;
    PWD_CHAR_3 = 0;
    PWD_CHAR_4 = 0;
    PWD_CHAR_5 = 0;
    PWD_CHAR_6 = 0;
    PWD_CHAR_7 = 0;
    PWD_CHAR_8 = 0;
    PWD_PARSING_RESULT = 0;
    *(undefined *)(DAT_02006cfc + 0x1b0) = 0;
    goto code_r0x082ca72e;
  }
  if (PWD_LEN_CNT != 0) {
    PWD_LEN_CNT = PWD_LEN_CNT - 1;
  }
  (&PWD_CHAR_1)[PWD_LEN_CNT] = 0;
LAB_082cac06:
  if (DAT_03004651 != 0) {
    if (DAT_03004651 != 1) {
      if (DAT_03004651 != 2) goto LAB_082cad54;
      if (DAT_03004650 == 0) {
        DAT_03004652 = 0x21;
LAB_082cad1a:
        *unaff_r10 = 0x40;
      }
      else {
        if (DAT_03004650 == 1) {
          DAT_03004652 = 0x40;
          *unaff_r10 = 0x40;
          goto LAB_082cad54;
        }
        if (DAT_03004650 == 2) {
          DAT_03004652 = 0x60;
        }
        else {
          if (DAT_03004650 == 3) {
            DAT_03004652 = 0x80;
            goto LAB_082cad1a;
          }
          if (DAT_03004650 == 4) {
            DAT_03004652 = 0xa0;
            bVar2 = 0x40;
            goto LAB_082cad38;
          }
          if (DAT_03004650 != 5) goto LAB_082cad54;
          DAT_03004652 = 0xc0;
        }
        *unaff_r10 = 0x40;
      }
      goto LAB_082cad54;
    }
    if (DAT_03004650 == 0) {
      DAT_03004652 = 0x21;
      bVar2 = 0x20;
LAB_082cad38:
      *unaff_r10 = bVar2;
      goto LAB_082cad54;
    }
    if (DAT_03004650 == 1) {
      DAT_03004652 = 0x40;
    }
    else {
      if (DAT_03004650 == 2) {
        DAT_03004652 = 0x60;
LAB_082cacba:
        *unaff_r10 = 0x20;
        goto LAB_082cad54;
      }
      if (DAT_03004650 == 3) {
        DAT_03004652 = 0x80;
      }
      else {
        if (DAT_03004650 == 4) {
          DAT_03004652 = 0xa0;
          goto LAB_082cacba;
        }
        if (DAT_03004650 != 5) goto LAB_082cad54;
        DAT_03004652 = 0xc0;
      }
    }
    *unaff_r10 = 0x20;
    goto LAB_082cad54;
  }
  if (DAT_03004650 == 0) {
    DAT_03004652 = 0x21;
    *unaff_r10 = 0;
    goto LAB_082cad54;
  }
  if (DAT_03004650 == 1) {
    DAT_03004652 = 0x40;
  }
  else {
    if (DAT_03004650 == 2) {
      DAT_03004652 = 0x60;
LAB_082cac4e:
      *unaff_r10 = 0;
      goto LAB_082cad54;
    }
    if (DAT_03004650 == 3) {
      DAT_03004652 = 0x80;
    }
    else {
      if (DAT_03004650 == 4) {
        DAT_03004652 = 0xa0;
        goto LAB_082cac4e;
      }
      if (DAT_03004650 != 5) goto LAB_082cad54;
      DAT_03004652 = 0xc0;
    }
  }
  *unaff_r10 = 0;
LAB_082cad54:
  uVar6 = 0;
  do {
    uVar6 = uVar6 + 1 & 0xff;
  } while (uVar6 != 8);
  do {
    cVar1 = *(char *)(uVar6 + 0x30027d8);
    if (cVar1 == '\x01') {
      puVar5 = &DAT_02006d6c + uVar6 * 100;
      puVar4 = &DAT_081a1d30;
LAB_082caec2:
      FUN_080003fc(puVar5,puVar4);
      FUN_08000680(puVar5,0);
    }
    else {
      if (cVar1 == '\x02') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a2310;
        goto LAB_082caec2;
      }
      if (cVar1 == '\x03') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a28f0;
        goto LAB_082caec2;
      }
      if (cVar1 == '\x04') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a29b0;
        goto LAB_082caec2;
      }
      if (cVar1 == '\x05') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a34b0;
        goto LAB_082caec2;
      }
      if (cVar1 == '\x06') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a3630;
        goto LAB_082caec2;
      }
      if (cVar1 == '\a') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a3b50;
        goto LAB_082caec2;
      }
      if (cVar1 == '\b') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a4650;
        goto LAB_082caec2;
      }
      if (cVar1 == '\n') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a1c70;
        goto LAB_082caec2;
      }
      if (cVar1 == '\v') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a2250;
        goto LAB_082caec2;
      }
      cVar1 = *(char *)(uVar6 + 0x30027d8);
      if (cVar1 == '\f') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a2830;
        goto LAB_082caec2;
      }
      if (cVar1 == '\r') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a2ed0;
        goto LAB_082caec2;
      }
      if (cVar1 == '\x0e') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a2f90;
        goto LAB_082caec2;
      }
      if (cVar1 == '\x0f') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a3570;
        goto LAB_082caec2;
      }
      if (cVar1 == '\x10') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a3c10;
        goto LAB_082caec2;
      }
      if (cVar1 == '\x11') {
        puVar5 = &DAT_02006d6c + uVar6 * 100;
        puVar4 = &DAT_081a4130;
        goto LAB_082caec2;
      }
      if (cVar1 == '\x12') {
        FUN_080003fc(&DAT_02006d6c + uVar6 * 100,&DAT_081a4710);
        FUN_08000680(&DAT_02006d6c + uVar6 * 100,0);
      }
    }
    if ((&PASSCODE_DEST)[uVar6] == '\x01') {
      store_1_at_r0(&DAT_02006d6c + uVar6 * 100);
    }
    else {
      FUN_0800060c(&DAT_02006d6c + uVar6 * 100);
    }
    FUN_08000408(&DAT_02006d6c + uVar6 * 100);
    uVar6 = uVar6 + 1 & 0xff;
    if (uVar6 == 0x10) {
      FUN_08000684(&DAT_02006d08,DAT_03004652,*unaff_r10 + 0x3a,0x80);
      FUN_08000704(&DAT_02006d08);
      FUN_08000408(&DAT_02006d08);
      FUN_08000684(&DAT_020073ac,(uint)PWD_LEN_CNT * 0x10 + 0x50,0x20,0);
      FUN_08000704(&DAT_020073ac);
      store_1_at_r0(&DAT_020073ac);
      FUN_08000408(&DAT_020073ac);
      FUN_08002e90();
                    /* WARNING: Subroutine does not return */
      FUN_082ca72e();
    }
  } while( true );
}
```