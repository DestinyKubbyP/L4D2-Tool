
# CHEAT

Yea, this is the pretty crazy part of my L4D2 tool. It has polymorphic build meaning. Each user has the option to build the cheat with there own signature. Polymorphic builds are pretty annoying to pull off, but let me explain. What we do to pull this off is shifting functions in memory, adding junk code(dont like doing this and choose to remove it, but is somthing to consider), encrpyt strings, and rename symbols on build time. Some may say this is overkill for a valve cheat thats not shared much, but its not only cool as fuck, its cool as fuck.

Obviously this does not avoid the problem of things like memory scans, PE headers, and abnormal protections scans. So we need to earse the PEH. The PEH defines basic data for the DLL or EXE. We also do basic manual mapping.

### Manually Maping

All manual mapping is loading your dll without calling some for of loadlibrary function. This hides us from the Listed modules. All we do to manual mape is find a big enough place of unused memory to import our dll. We can use VirtualAlloc, but I tend to avoid all WINAPI functions if possible. Once we get the address/place in memory where we can store our dll we copy the DLL data sections into the coressponding locations in which the dll was formated. I then use a shellcode tramp to execute the dll. NOW WE ERASE PEH.

### What does PEH(Portable Executable Header File) define?


```
1.File Type
2.Entry Point
3.Debug Data
4.Callbacks
5.Most importantly section layouts.
```

Earasing PEH is pretty simple. "But why earse PEH Peter?" and this is a amazing question. PEH are standard in every compiled DLL or EXE. Since they have predictable names,patterns, and locations it can make it very easy to detect a unreconginzed module and take action against us. For example of the section layouts : .text, .data, .rdata. First we will generate a string and rename the section layout names. This breaks any PE scans that use section names for there sig scans.

## Scattering

Scattering is one of the most fun techniques I've encountered while reversing. The idea is to take a DLL and split up its functions, placing them into code caves — unused memory regions within the target process (like a game). Instead of loading the DLL as a contiguous block, each function is manually placed somewhere else in the process’s memory space.

Once you've found suitable regions for your functions, you copy them there. However, this becomes hard to manage because of how x86/x64 assembly deals with relative addressing — especially with jmp, call, and conditional branches.

### Why do we have to patch instructions?
Let’s look at a basic example:
```assembly
epic:
    mov eax, edi    ; 89 f8
    add eax, esi    ; 01 f0
    ret             ; c3
```
This function holds its own place in memory when loaded. Say the instructions mov eax,edi is located at 0x1000. Say we find a code cave at 0x2000 which can store this function. We then pretty much copy and paste all the bytes of the assembly function at the location 0x2000. Ok so we shifted the function in memory, but the ret instruction which jumps back to instruction ahead of the function call so we have to patch this. Patching can become iffy depending on the architecture when it comes to relative jumping. With x86 you can simply input the address most times since its in range, but with x64 you usally have to store in a 8b reg like rax and then use the jmp instruction.Since we did scatter our functions in the games memory region we dont really have to worry about the limitations of short or near jumps. This is because a short jump(E8) can only reach 128 bytes from the current instruction and the near jump can jump to any location within +2 GB(which pretty much reaches all we need). If it comes down to it we have to store it in a register so we can peform a long jump(uncommon to occur). This sucks because we will have to account for stolen bytes and restore the rax reg. It hasnt happend to me since ive only coded the x86 version of the cheat and havent had the need to peform long jumps.

### Theory for future problem. 
I haven't ran into the issue Im theoryizing may occur in the upcoming future. Since in x64 dynamically replacing a register and restoring it to its old instruction with its previous instruction is not only stupid but extremly hard in cases cause you have traceback code flow with the specfic registers which is a huge hassle. What do I mean?


POSSIBLE SOLOUTION ↓

Orginal functions :
```
primary:
    mov rax,rdi 
    mov rax,[rax]
    call seconday --pretent this is the secondary function
secondary :
    add rax,1000;
    ret; -- jmps to instruction after "call rax" in the primary function.
```

Relocated functions :
```
primary_patched:
    mov rax,rdi 
    mov rax,[rax] ---the value needed to add too.
    push rax ---I want to take a snap shot of rax and then pop it on top of function secondary function to restore its value
    mov rax,secondary ---we have to asign a 8b reg to the function address.
    call rax --pretent this is the secondary function
secondary :
    pop rax; ---I pop/restore to the orginal adress value.
    add rax,1000; ---since I restored it it will give me the rax value before push of primary function.
    ret; -- jmps to instruction after "call secondary" in the primary function.
```

Whats different? Well the first thing I did differently was push the rax reg to the stack. This allowed me to save the value before we changed it to the secondary function location and called it. This is helpful since when we get to that location of the secondary function we can restore that register so we can do proper calculations on the value. Just think as push as save and pop as revert/restore/undo.
