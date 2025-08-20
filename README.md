
# ***L4D2-Tool***

*This is a tool made for the valve game LEFT 4 DEAD 2. It has pretty amazing features including Mod pack sharing, Addon management, vpk packer/unpacker, addon installer, addon subscription manager, responsive debug feedback, and chose able workshop paths. Hope you enjoy &lt;3*

*I'm going to be explaining how the features work and the apporaches I made which I found pretty inovative and unique. I used some pretty amazing packages like MoonSharp which made syntax handling and debuging for my script editor amazing. 
[Thanks xanathar for MoonSharp!](https://github.com/xanathar)*

#### ***NOTE : This is a private tool, but getting a key is free and all you need to do is contact DestinyTakes on discord at takenplace and see if your fit to be a tester. This will soon be released to the public after some pretty extensive testing of a lot of the features(MAINLY THE CHEAT FEATURES). This will take time do to the effort Im putting in to it to be completly undetectable.***


![Menu Example](https://github.com/DestinyKubbyP/L4D2-Tool/blob/main/Menu.PNG?raw=true)


# ***ADDON MANAGEMENT***

*There is plenty of ways to manage addons and implement togglbility. I tried to take a little different of a apporach. I used a file based suffling method. I create my own folder in my base directory/debug output and when mods are disabled they will be placed in the addons folder in my base directory.*


```EXAMPLE OF DISABLED MOD : C:\Users\DestinyTakes\source\repos\L4D2_Mod_Manager\L4D2_Mod_Manager\bin\Debug\net8.0-windows\Addons\3404846348.vpk```


```EXAMPLE OF ENABLED MOD : C:\Program Files (x86)\Steam\steamapps\common\Left 4 Dead 2\left4dead2\addons\3404846348.vpk```


*This allows me to easily see what mods are enabled without much struggle with of taking the time of parsing json data. So say we have a addon that is disabled(refer to disabled mod example) all that happens when we toggle this mod is the moving of the vpk file location to the addon folder of Left 4 Game 2*

*You may ask what the code may look like to peform this method. Its quite simple actually, but first we need to be sure of the selected item. This means we have to go way back to how we grab and update plugins.*

## ***WORKINGS OF ADDON MANAGER***

*First I define my addon class so we can a nice structured table of information for each addon.*
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
*This is basics of the code. It just simple gives a class some properties of the addon. This is just so we can easily store all data in a List(List<addon>). You will see the steamtag class and that may raise some flags(do to the lack of context and information so far). Now I simply grab all the paths of the addons. Since we are using that file suffling methods we have to scrape the contents of two different folders. The folder in our base directory and the addon folder in the game files of L4D2. This is pretty simple, but we need to setup directorys first. Its really easy all I do is use the base directory and create a couple folders underneath it as seen and pull all mods to base directory addon folder(will touch on this later).*
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
        } else
        {
            option_file = File.Open(addon_fig,FileMode.Open);
        }

        pull_mods_to_path(ResourceManager.resources.workshop_path, addon_path);
    }
```
*As seen in the code I make a couple files. A settings and addons folder and a options.json for storing ui element color prefrences, saved mod packs, user info(this is key based its "private")*

*Once the directorys are created it pulls all addons into base directory addon folder. This is so we can read our options.json see what was previously enabled and simply move the matching addons back into our Left 4 Dead 2 addon folder reenabling the mod. This is jumping a few steps ahead cause we havent even setup the config, but its pretty much a list like this.*

```json
    enabled_mods : {
        ["asoidja.vpk"] : {
            "type" : "script",
            "main_cat" : "Game Content",
            "name" : "Cool person"
        }
    }
```

*Once this is all setup and we then desteralize the json into a live object.*

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

*Once we use*

# ***MOD PACK MANAGER***

# ***CHEAT***

*Yea, this is the pretty crazy part of my L4D2 tool. It has polymorphic build meaning. Each user has the option to build the cheat with there own signature. Polymorphic builds are pretty annoying to pull off, but let me explain. What we do to pull this off is shifting functions in memory, adding junk code(dont like doing this and choose to remove it, but is somthing to consider), encrpyt strings, and rename symbols on build time. Some may say this is overkill for a valve cheat thats not shared much, but its not only cool as fuck, its cool as fuck.*

*Obviously this does not avoid the problem of things like memory scans, PE headers, and abnormal protections scans. So we need to earse the PEH. The PEH defines basic data for the DLL or EXE. We also do basic manual mapping.*

### ***Manually Maping***

*All manual mapping is loading your dll without calling some for of loadlibrary function. This hides us from the Listed modules. All we do to manual mape is find a big enough place of unused memory to import our dll. We can use VirtualAlloc, but I tend to avoid all WINAPI functions if possible. Once we get the address/place in memory where we can store our dll we copy the DLL data sections into the coressponding locations in which the dll was formated. I then use a shellcode tramp to execute the dll. NOW WE ERASE PEH.*

### ***What does PEH(Portable Executable Header File) define?***


```
    1.File Type
    2.Entry Point
    3.Debug Data
    4.Callbacks
    5.Most importantly section layouts.
```

*Earasing PEH is pretty simple. "But why earse PEH Peter?" and this is a amazing question. PEH are standard in every compiled DLL or EXE. Since they have predictable names,patterns, and locations it can make it very easy to detect a unreconginzed module and take action against us. For example of the section layouts : .text, .data, .rdata. First we will generate a string and rename the section layout names. This breaks any PE scans that use section names for there sig scans.*

## ***Scattering***

*Scattering is one of the most fun techniques I've encountered while reversing. The idea is to take a DLL and split up its functions, placing them into code caves — unused memory regions within the target process (like a game). Instead of loading the DLL as a contiguous block, each function is manually placed somewhere else in the process’s memory space.*

*Once you've found suitable regions for your functions, you copy them there. However, this becomes hard to manage because of how x86/x64 assembly deals with relative addressing — especially with jmp, call, and conditional branches.*

### ***Why do we have to patch instructions?***
*Let’s look at a basic example:*
```assembly
    epic:
        mov eax, edi    ; 89 f8
        add eax, esi    ; 01 f0
        ret             ; c3
```
*This function holds its own place in memory when loaded. Say the instructions mov eax,edi is located at 0x1000. Say we find a code cave at 0x2000 which can store this function. We then pretty much copy and paste all the bytes of the assembly function at the location 0x2000. Ok so we shifted the function in memory, but the ret instruction which jumps back to instruction ahead of the function call so we have to patch this. Patching can become iffy depending on the architecture when it comes to relative jumping. With x86 you can simply input the address most times since its in range, but with x64 you usally have to store in a 8b reg like rax and then use the jmp instruction.Since we did scatter our functions in the games memory region we dont really have to worry about the limitations of short or near jumps. This is because a short jump(E8) can only reach 128 bytes from the current instruction and the near jump can jump to any location within +2 GB(which pretty much reaches all we need). If it comes down to it we have to store it in a register so we can peform a long jump(uncommon to occur). This sucks because we will have to account for stolen bytes and restore the rax reg. It hasnt happend to me since ive only coded the x86 version of the cheat and havent had the need to peform long jumps.*

### ***Theory for future problem.*** 
*I haven't ran into the issue Im theoryizing may occur in the upcoming future. Since in x64 dynamically replacing a register and restoring it to its old instruction with its previous instruction is not only stupid but extremly hard in cases cause you have traceback code flow with the specfic registers which is a huge hassle. What do I mean? POSSIBLE SOLOUTION ↓*

*Orginal functions :*

```assembly
    primary:
        mov rax,rdi 
        mov rax,[rax]
        call seconday --pretent this is the secondary function
    secondary :
        add rax,1000;
        ret; -- jmps to instruction after "call rax" in the primary function.
```

*Relocated functions :*

```assembly
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

*Whats different? Well the first thing I did differently was push the rax reg to the stack. This allowed me to save the value before we changed it to the secondary function location and called it. This is helpful since when we get to that location of the secondary function we can restore that register so we can do proper calculations on the value. Just think as push as save and pop as revert/restore/undo.*
