# ***L4D2-Tool***

*This is a tool made for the Valve game **Left 4 Dead 2**. It comes with some powerful features, including: mod pack sharing, addon management, VPK packer/unpacker, addon installer, addon subscription manager, responsive debug feedback, and customizable workshop paths. Hope you enjoy <3*  

*In this README I’ll explain how the features work and the approaches I took — some of which I think are pretty innovative and unique. I used some amazing packages, like **MoonSharp**, which made syntax handling and debugging for my script editor awesome.  
[Thanks xanathar for MoonSharp!](https://github.com/xanathar)*  

#### ***NOTE: This is a private tool, but getting a key is free — all you need to do is contact `DestinyTakes` on Discord (`takenplace`) to see if you’re a good fit for testing. This will be released publicly after extensive testing of many features (mainly the cheat functions). This takes time because I’m working hard to make it completely undetectable.***  

![Menu Example](https://github.com/DestinyKubbyP/L4D2-Tool/blob/main/Menu.PNG?raw=true)

---
# ***ADDON MANAGEMENT***

*There are many ways to manage addons and implement toggling. I took a slightly different approach — a file-based shuffling method. I created my own folder in the base directory/debug output, and when mods are disabled, they’re placed there instead of in the game’s `addons` folder.*  

```EXAMPLE OF DISABLED MOD : C:\Users\DestinyTakes\source\repos\L4D2_Mod_Manager\L4D2_Mod_Manager\bin\Debug\net8.0-windows\Addons\3404846348.vpk```

```EXAMPLE OF ENABLED MOD : C:\Program Files (x86)\Steam\steamapps\common\Left 4 Dead 2\left4dead2\addons\3404846348.vpk```

*This lets me instantly see which mods are enabled without parsing JSON data. For example, if an addon is disabled (see the disabled mod example above), toggling it simply moves the VPK file back into the `addons` folder of Left 4 Dead 2.*  

---

## ***WORKINGS OF THE ADDON MANAGER***

*First, I define an addon class so we can have a structured table of information for each addon:*  

```c#
public class steamtag
{
    public string tag { get; set; }
}

public class addon
{
    public string name;
    public string type;
    public List<steamtag> tags;
    public string main_cat;
    public string sub_cat;

    public addon(string name, string type, List<steamtag> tags, string main_cat,string sub_cat)
    {
        this.name = name;
        this.main_cat = main_cat;
        this.type = type;
        this.tags = tags;
        this.sub_cat = sub_cat;
    }
}
```

*This is the basic structure. It gives each addon some properties we can store in a `List<addon>`. The `steamtag` class might look a little out of place now, but it makes more sense later. Since we’re using a file-shuffling method, we need to scrape addons from two folders: the base directory and the L4D2 addons folder. To do that, we first create the necessary directories:*  

```c#
public static void create_directorys(string path)
{
    string setting_path = path + "\\Settings";
    string addon_path = path + "\\Addons";
    string addon_fig = setting_path + "\\options.json";

    if (!File.Exists(setting_path))
    {
        Directory.CreateDirectory(setting_path);
    }

    if (!File.Exists(addon_path))
    {
        Directory.CreateDirectory(addon_path); 
    }

    if (!File.Exists(addon_fig))
    {
        option_file = File.Create(addon_fig);
    } 
    else
    {
        option_file = File.Open(addon_fig,FileMode.Open);
    }

    pull_mods_to_path(ResourceManager.resources.workshop_path, addon_path);
}
```

*Here, I create a `Settings` folder, an `Addons` folder, and an `options.json` file (for UI color preferences, saved mod packs, and user info — key-based since this tool is private).*  

*After creating the directories, I pull all addons into the base directory’s addon folder. That way, I can read `options.json`, check what was previously enabled, and move the matching addons back into the L4D2 addons folder to re-enable them.*  

---

```json
["enabled_mods"] = {
    ["asoidja.vpk"] = {
        "type" : "script",
        "main_cat" : "Game Content",
        "name" : "Cool person"
    }
}
```

*Then we deserialize the JSON into a live object:*  

```c#
public class deseralize_js
{
    public class result
    {
        public Dictionary<string, Dictionary<string, mod_struct>> enabled_mods;
    }

    public class mod_struct
    {
        public string type;
        public string name;
        public string main_cat;
    }
}

deseralize_js structed = JsonSeralizer.Deserialize<deseralize_js>();
```

---

# ***MOD PACK MANAGER***

*(section TBD — you can expand this as needed)*  

---

# ***CHEAT***

*Here’s the crazy part of my L4D2 tool. It supports **polymorphic builds**, meaning each user can build the cheat with a unique signature. Polymorphic builds are annoying to pull off, but here’s how: shifting functions in memory, adding junk code (optional), encrypting strings, and renaming symbols at build time. Some might say this is overkill for a Valve cheat that isn’t widely shared — but honestly, it’s just cool as hell.*  

*Of course, this doesn’t fully protect against memory scans, PE header scans, or abnormal protection checks. That’s why we erase the PEH and use manual mapping.*  

---

### ***Manual Mapping***

*Manual mapping loads a DLL without calling `LoadLibrary`, which keeps it hidden from listed modules. The process: allocate memory, copy DLL sections into their formatted locations, then use shellcode to execute. Afterward, we erase the PEH.*  

---

### ***Scattering***

*Scattering is one of the most fun techniques I've encountered while reversing. The idea is to take a DLL and split up its functions, placing them into code caves — unused memory regions within the target process (like a game). Instead of loading the DLL as a contiguous block, each function is manually placed somewhere else in the process’s memory space.*

*Once you've found suitable regions for your functions, you copy them there. However, this becomes tricky because of how x86/x64 assembly handles relative addressing — especially with `jmp`, `call`, and conditional branches.*  

---

### ***Why do we have to patch instructions?***

*Example:*  

```assembly
epic:
 mov eax, edi    ; 89 f8
 add eax, esi    ; 01 f0
 ret             ; c3
```

*This function originally lives at a certain address (say `0x1000`). If we move it to a code cave at `0x2000`, the instructions themselves are fine, but the `ret` will jump back incorrectly unless patched. Patching becomes tricky depending on the architecture. On x86 you can often insert the address directly; on x64 you usually need to load it into a register like `rax` and `jmp` there. Short jumps (`E8`) can only reach `±128` bytes, while near jumps can reach within `±2GB`. For longer distances, you need a register-based jump.*  

---

### ***Future Problem Theory***

**I haven’t run into this yet, but in x64 replacing and restoring registers dynamically can be very messy. *Example solution:***

**Orginal Function Without Stack Restore :**

```assembly
primary:
 mov rax,rdi 
 mov rax,[rax]
 mov rax,secondary
 call rax 
secondary :
 add rax,1000; ---adds to the rax address which is the secondary function start location. 
 ret
```
**Orginal Function With Stack Restore:**

```assembly
primary_patched:
 mov rax,rdi 
 mov rax,[rax]
 push rax ---this is where we fix it
 mov rax,secondary
 call rax 
secondary :
 pop rax ---when we pop it it restores the rax value back to mov rax,[rax] before the push.
 add rax,1000
 ret
```

*By pushing `rax` before overwriting it, then popping it back later, we can safely restore the value and continue execution correctly. In short: `push` = save, `pop` = restore. Edit: this worked! I ended up doing this idea and it worked amazingly. What I did to get the scattering problem fixed. Start of the address and find the instuction that needs patching. Once I find the address, I identify what type of op code its using. For example if I am on `x86` based arch and I get a short jump and the difference of the address of exceeds `128 bytes` then I am not able to properly patch the function so it directs code flow properly with the relocated functions of our DLL. What I do for calls is simple get the start of the call op code and add a offset of +1 and we most overwrite the realtive address/near jmp. For jmps and what not I just tend push one of the registers like edi. Once I push edi on the stack I change edi's value to the relocated function start address. Once I've done that I had to add a instruction that pops edi at the very start of the relocated function. So it restores the original edi value.*
