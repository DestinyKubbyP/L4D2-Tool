# ***ADDON MANAGEMENT***

*There are many ways to manage addons and implement toggling. I took a slightly different approach — a file-based shuffling method. I created my own folder in the base directory/debug output, and when mods are disabled, they’re placed there instead of in the game’s `addons` folder.*  

```EXAMPLE OF DISABLED MOD : C:\Users\DestinyTakes\source\repos\L4D2_Mod_Manager\L4D2_Mod_Manager\bin\Debug\net8.0-windows\Addons\3404846348.vpk```

```EXAMPLE OF ENABLED MOD : C:\Program Files (x86)\Steam\steamapps\common\Left 4 Dead 2\left4dead2\addons\3404846348.vpk```

*This lets me instantly see which mods are enabled without parsing JSON data. For example, if an addon is disabled (see the disabled mod example above), toggling it simply moves the VPK file back into the `addons` folder of Left 4 Dead 2.*  

  

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

  

```json
"enabled_mods" : {
    "asoidja.vpk" : {
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

  

# ***MOD PACK MANAGER***

