# ***CHEAT***

*Here’s the crazy part of my L4D2 tool. It supports **polymorphic builds**, meaning each user can build the cheat with a unique signature. Polymorphic builds are annoying to pull off, but here’s how: shifting functions in memory, adding junk code (optional), encrypting strings, and renaming symbols at build time. Some might say this is overkill for a Valve cheat that isn’t widely shared — but honestly, it’s just cool as hell.*  

*Of course, this doesn’t fully protect against memory scans, PE header scans, or abnormal protection checks. That’s why we erase the PEH and use manual mapping.*  

***MORE INFO:*** [***MAIN INFO***](https://github.com/DestinyKubbyP/L4D2-Tool/tree/main), [***ADDON INFO***](https://github.com/DestinyKubbyP/L4D2-Tool/blob/main/main.md)


### ***Manual Mapping***

*Manual mapping loads a DLL without calling `LoadLibrary`, which keeps it hidden from listed modules. The process: allocate memory, copy DLL sections into their formatted locations, then use shellcode to execute. Afterward, we erase the PEH.*  


### ***Scattering***

*Scattering is one of the most fun techniques I've encountered while reversing. The idea is to take a DLL and split up its functions, placing them into code caves — unused memory regions within the target process (like a game). Instead of loading the DLL as a contiguous block, each function is manually placed somewhere else in the process’s memory space.*

*Once you've found suitable regions for your functions, you copy them there. However, this becomes tricky because of how x86/x64 assembly handles relative addressing — especially with `jmp`, `call`, and conditional branches.*  


### ***Why do we have to patch instructions?***

*Example:*  

```assembly
epic:
 mov eax, edi    ; 89 f8
 add eax, esi    ; 01 f0
 ret             ; c3
```

*This function originally lives at a certain address (say `0x1000`). If we move it to a code cave at `0x2000`, the instructions themselves are fine, but the `ret` will jump back incorrectly unless patched. Patching becomes tricky depending on the architecture. On x86 you can often insert the address directly; on x64 you usually need to load it into a register like `rax` and `jmp` there. Short jumps (`E8`) can only reach `±128` bytes, while near jumps can reach within `±2GB`. For longer distances, you need a register-based jump.*  


### ***Future Problem Theory***

**I haven’t run into this yet, but in x64 replacing and restoring registers dynamically can be very messy. *Example solution:***


***Original Function Without Stack Restore:***

```assembly
primary:
 mov rax,rdi 
 mov rax,[rax]
 mov rax,secondary
 call rax 
secondary :
 add rax,1000 ; ---adds to the rax address which is the secondary function start location. 
 ret
```


***Original Function With Stack Restore:***

```assembly
primary_patched:
 mov rax,rdi 
 mov rax,[rax]
 push rax ---this is where we fix it
 mov rax,secondary
 call rax 
secondary :
 pop rax ---when we pop it, it restores the rax value back to mov rax,[rax] before the push.
 add rax,1000
 ret
```

*By pushing `rax` before overwriting it, then popping it back later, we can safely restore the value and continue execution correctly. In short: `push` = save, `pop` = restore.*  

*Edit: this worked! I ended up doing this idea and it worked amazingly. What I did to get the scattering problem fixed: start at the address and find the instruction that needs patching. Once I find the address, I identify what type of opcode it’s using. For example, if I am on an `x86`-based architecture and I get a short jump, and the difference of the address exceeds `128 bytes`, then I am not able to properly patch the function so it directs code flow properly with the relocated functions of our DLL. What I do for calls is simple: get the start of the call opcode and add an offset of +1, then overwrite the relative address/near jump (replace the next 4 bytes with the relocated function start). For jumps and whatnot, I tend to push one of the registers like `edi`. Once I push `edi` on the stack, I change `edi`'s value to the relocated function start address. Once I've done that, I had to add an instruction that pops `edi` at the very start of the relocated function so it restores the original `edi` value.*
