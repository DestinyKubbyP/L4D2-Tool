# ***ADDON MANAGEMENT***
&nbsp;
*There are many ways to manage addons and implement toggling. I took a slightly different approach — a file-based shuffling method. I created my own folder in the base directory/debug output, and when mods are disabled, they’re placed there instead of in the game’s `addons` folder.*
&nbsp;
***EXAMPLE OF DISABLED MOD : C:\Users\DestinyTakes\source\repos\L4D2_Mod_Manager\L4D2_Mod_Manager\bin\Debug\net8.0-windows\Addons\3404846348.vpk***
&nbsp;
***EXAMPLE OF ENABLED MOD : C:\Program Files (x86)\Steam\steamapps\common\Left 4 Dead 2\left4dead2\addons\3404846348.vpk***
&nbsp;
*This lets me instantly see which mods are enabled without parsing JSON data. For example, if an addon is disabled (see the disabled mod example above), toggling it simply moves the VPK file back into the `addons` folder of Left 4 Dead 2.*  
&nbsp;
## ***WORKINGS OF THE ADDON MANAGER***
&nbsp;
*First, I define an addon class so we can have a structured table of information for each addon:*  
&nbsp;
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
&nbsp;
*This is the basic structure. It gives each addon some properties we can store in a `List<addon>`. The `steamtag` class might look a little out of place now, but it makes more sense later. Since we’re using a file-shuffling method, we need to scrape addons from two folders: the base directory and the L4D2 addons folder. To do that, we first create the necessary directories:*  
&nbsp;
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
&nbsp;
*Here, I create a `Settings` folder, an `Addons` folder, and an `options.json` file (for UI color preferences, saved mod packs, and user info — key-based since this tool is private).*
&nbsp;
*After creating the directories, I pull all addons into the base directory's addon folder. That way, I can read `options.json`, check what was previously enabled, and move the matching addons back into the L4D2 addons folder to re-enable them.*  
&nbsp;
*I then move the enabled addons into L4D2 addon folder. Now its time to update addon list and ui. This isn't too hard all we gotta do is scrape both addon folders and grab any addons. A lot of addon names are there ids. So I format the file name and extract the id. Once I extract all the ids of the I put them in a list and continue to network request.*
&nbsp;
#####HOW I GRAB ALL ADDONS
&nbsp;
```c#
    public static List<string> grab_addon_paths()
    {
        List<string> addons = new List<string>();

        foreach (string file in get_file_contents(resources.workshop_path))
        {
            if (file.Contains(".vpk"))
            {
                addons.Add(file);
            }
        }

        foreach (string file in get_file_contents(AppContext.BaseDirectory + "\\Addons"))
        {
            if (file.Contains(".vpk"))
            {
                addons.Add(file);
            }
        }

        return addons;
    }
```
&nbsp;
#####HOW I EXTRACT THE FILE NAMES
&nbsp;
```c#
    public static void pull_mods_to_path(string path,string new_folder_path) //puts all mods to a single folder for managment.
    {
        List<string> paths = grab_addon_paths();

        foreach (string mod_path in paths)
        {
            string fname = Path.GetFileName(mod_path);
            string new_des = Path.Combine(new_folder_path, fname);

            if (File.Exists(mod_path) && mod_path != new_des)
            {
                File.Move(mod_path, new_des);
            }
        }
    }
```

&nbsp;
### ***HOW DO WE GRAB MOD DATA?***
&nbsp;
----
*I use steams file detail api :"https://api.steampowered.com/ISteamRemoteStorage/GetPublishedFileDetails/v1/"*
&nbsp;
*With the list of addon ids we collect we will now query them with the file detail api. I will provide you the example in how queried it.*
&nbsp;
```c#
List<KeyValuePair<string, string>> post_data = new List<KeyValuePair<string, string>>();
post_data.Add(new KeyValuePair<string, string>("itemcount", ids.Count.ToString()));

foreach(string id in ids)
{
	post_data.Add(new KeyValuePair<string,string>($"publishedfileids[{index}]",id)); //what this does is pretty simple it appends a bunch of ids to a array of ids so it can query a ton of mods in one go.
	index++;
}
```
&nbsp;
#####JSON RESPONSE EXAMPLE :
&nbsp;
```json
{
  "response": {
    "result": 1,
    "resultcount": 2,
    "publishedfiledetails": [
      {
        "publishedfileid": "1234567890",
        "result": 1,
        "creator": "76561198000000001",
        "creator_app_id": 550,
        "consumer_app_id": 550,
        "filename": "maps/custom_map1.vpk",
        "file_size": 10485760,
        "file_url": "https://steamusercontent-a.akamaihd.net/ugc/111111111111111111/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA/",
        "hcontent_file": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
        "hcontent_preview": "BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB",
        "title": "Custom Map 1",
        "description": "A custom Left 4 Dead 2 map with special infected tweaks.",
        "time_created": 1620000000,
        "time_updated": 1625000000,
        "visibility": 0,
        "banned": 0,
        "ban_reason": "",
        "subscriptions": 15234,
        "favorited": 2034,
        "lifetime_subscriptions": 20000,
        "lifetime_favorited": 2500,
        "views": 40321,
        "tags": [
          { "tag": "Map" },
          { "tag": "Survival" }
        ]
      },
      {
        "publishedfileid": "9876543210",
        "result": 1,
        "creator": "76561198000000002",
        "creator_app_id": 550,
        "consumer_app_id": 550,
        "filename": "addons/realistic_weapons.vpk",
        "file_size": 5242880,
        "file_url": "https://steamusercontent-a.akamaihd.net/ugc/222222222222222222/CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC/",
        "hcontent_file": "CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC",
        "hcontent_preview": "DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD",
        "title": "Realistic Weapons Pack",
        "description": "Overhauls weapon models and sounds to be more realistic.",
        "time_created": 1610000000,
        "time_updated": 1628000000,
        "visibility": 0,
        "banned": 0,
        "ban_reason": "",
        "subscriptions": 45872,
        "favorited": 6321,
        "lifetime_subscriptions": 60000,
        "lifetime_favorited": 7000,
        "views": 100321,
        "tags": [
          { "tag": "Weapons" },
          { "tag": "HD" }
        ]
      }
    ]
  }
}
```
&nbsp;
*As you can see in this example there are two addons and they have there own props. We can use the structure below to deseralize this json data into a usable c# table.*
&nbsp;
#####THE ADDON STRUCTURE :
&nbsp;
```c#    
	public class Root
    {
        public WorkshopResponse response { get; set; }
    }

    public class steamtag
    {
        public string tag { get; set; }
    }

    public class WorkshopResponse
    {
        public int result { get; set; }
        public int resultcount { get; set; }
        public List<PublishedFileDetail> publishedfiledetails { get; set; }
    }

    public class PublishedFileDetail
    {
        public string publishedfileid { get; set; }
        public int result { get; set; }
        public string creator { get; set; }
        public int creator_app_id { get; set; }
        public int consumer_app_id { get; set; }
        public string filename { get; set; }

        [JsonConverter(typeof(ResourceManager.ParseStringToLongConverter))]
        public long file_size { get; set; }

        public string file_url { get; set; }
        public string title { get; set; }
        public string description { get; set; }
        public List<steamtag> tags { get; set; }
}
```
&nbsp;
*Now that we have the addon list with the needed data we can now update our ui. If you've taken a peek at the ui already you can tell it has a main-catagoreys and sub-catagoreys, but how do we determine these? All that is provided to us tag wise, is the steam tag array. First we have to review the main catagoreys. I pass the addon into two functions called `grab_main_cat` and `grab_sub_cat`. I will now show you what the functions and list look like.*
&nbsp;
```c#
    public static Dictionary<string, ListBox> survivorLists;
    public static Dictionary<string, ListBox> weaponLists;
    public static Dictionary<string, ListBox> gameContentLists;
    public static Dictionary<string, ListBox> gameModeLists;
    public static Dictionary<string, ListBox> itemLists;
    public static Dictionary<string, ListBox> infectedLists;
    public static void init_list()
    {
        weaponLists = new Dictionary<string, ListBox>() {
            { "Shotgun", ui.ShotgunList },
            { "Sniper", ui.SniperList },
            { "Melee", ui.MeleeList },
            { "Pistol", ui.PistolList },
            { "Rifle", ui.RifleList },
            { "SMG", ui.SMGList },
            { "M60", ui.M60List },
            { "Throwable", ui.ThrowableList },
            { "Grenade Launcher", ui.GrenadeLauncherList }
        };

        survivorLists = new Dictionary<string, ListBox>()
        {
            { "Nick", ui.NickList },
            { "Bill", ui.BillList },
            { "Francis", ui.FrancisList },
            { "Louis", ui.LouisList },
            { "Zoey", ui.ZoeyList },
            { "Coach", ui.CoachList },
            { "Ellis", ui.ElisList },
            { "Rochelle", ui.RochelleList }
        };

        gameContentLists = new Dictionary<string, ListBox>()
        {
            { "Campaigns",ui.CampaignsList},
            { "Weapons", ui.WeaponList},
            { "Items", ui.ItemList},
            { "Sounds", ui.SoundList},
            { "Scripts",ui.ScriptList},
            { "UI",ui.UIList},
            { "Miscellaneous",ui.MiscellaneousList},
            { "Models",ui.ModelList},
            { "Textures",ui.TextureList}
        };

        gameModeLists = new Dictionary<string, ListBox>()
        {
            { "Single Player",ui.SingePlayerList},
            { "Co-op", ui.Co_opList},
            { "Versus", ui.VersusList},
            { "Scavenge", ui.ScavengeList},
            { "Survival",ui.SurvivalList},
            { "Realism",ui.RealsimList},
            { "Realism Versus",ui.RealismVersusList},
            { "Mutations",ui.MutationList}
        };

        itemLists = new Dictionary<string, ListBox>()
        {
            { "Adrenaline",ui.AdrenalineList},
            { "Defibrillator", ui.DefibrillatorList},
            { "Pills", ui.PillsList},
            { "Medkit", ui.MedkitList},
            { "Other",ui.OtherList},
        };

        infectedLists = new Dictionary<string, ListBox>()
        {
            { "Common Infected",ui.CommonInfectedList},
            { "Special Infected", ui.SpecialInfectedList},
            { "Boomer", ui.BoomerList},
            { "Charger", ui.ChargerList},
            { "Hunter",ui.HunterList},
            { "Jockey",ui.JockeyList},
            { "Smoker",ui.SmokerList},
            { "Spitter",ui.SpitterList},
            { "Tank",ui.TankList},
            { "Witch",ui.WitchList}
        };
    };

    public static string[] survivor_tags = {
        "Bill",
        "Francis",
        "Louis",
        "Zoey",
        "Coach",
        "Ellis",
        "Nick",
        "Rochelle"
    };

    public static string[] infected_tags =
    {
        "Common Infected",
        "Special Infected",
        "Boomer",
        "Charger",
        "Hunter",
        "Jockey",
        "Smoker",
        "Spitter",
        "Tank",
        "Witch"
    };

    public static string[] game_content_tags =
    {
        "Campaigns",
        "Weapons",
        "Items",
        "Sounds",
        "Scripts",
        "UI",
        "Miscellaneous",
        "Models",
        "Textures",
    };

    public static string[] game_modes =
    {
        "Single Player",
        "Co-op",
        "Versus",
        "Scavenge",
        "Survival",
        "Realism",
        "Realism Versus",
        "Mutations"
    };

    public static string[] weapon_tags =
    {
        "Grenade Launcher",
        "M60",
        "Melee",
        "Pistol",
        "Rifle",
        "Shotgun",
        "SMG",
        "Sniper",
        "Throwable"
    };

    public static string[] item_tags =
    {
        "Adrenaline",
        "Defibrillator",
        "Medkit",
        "Pills",
        "Other"
    };
```
&nbsp;
*This is a simple function that assignes a designated key to a certain ui list element. So we can easily acsess list elements without to much work. You can also see the array of strings that simply just contain the sub_cats(I will explain later why what I did was bad)*
&nbsp;
#####HOW I GRAB MAIN TAG
&nbsp;
```c#
    public static string grab_main_cat(List<steamtag> steamtags)
    {
        string final_cat = "";
        bool found_sur = false;
        bool found_inf = false;
        bool found_gc = false;
        bool found_gm = false;
        bool found_wps = false;
        bool found_items = false;

        foreach(steamtag tag in steamtags)
        {
            if(survivor_tags.Contains(tag.tag))
            {
                found_sur = true;
            }

            if (infected_tags.Contains(tag.tag))
            {
                found_inf = true;
            }

            if (game_content_tags.Contains(tag.tag))
            {
                found_gc = true;
            }

            if (game_modes.Contains(tag.tag))
            {
                found_gm = true;
            }

            if (weapon_tags.Contains(tag.tag))
            {
                found_wps = true;
            }

            if (item_tags.Contains(tag.tag))
            {
                found_items = true;
            }
        }

        if(found_wps)
        {
            final_cat = "Weapons";
        } else if(found_items)
        {
            final_cat = "Items";
        } else if(found_sur)
        {
            final_cat = "Survivor";
        } else if(found_inf)
        {
            final_cat = "Infected";
        } else if(found_gm)
        {
            final_cat = "Game Mode";
        } else
        {
            final_cat = "Game Content";
        }

        return final_cat;
    }
```
&nbsp;
#####HOW I GRAB SUB TAG
&nbsp;
```c#
    public static string grab_sub_cat(string cat,List<steamtag> tags)
    {
        foreach(steamtag tag in tags)
        {
            if (cat == "Weapons" && weapon_tags.Contains<string>(tag.tag)) 
            {
                return tag.tag;
            }

            if (cat == "Infected" && infected_tags.Contains<string>(tag.tag))
            {
                return tag.tag;
            }

            if (cat == "Game Mode" && game_modes.Contains<string>(tag.tag))
            {
                return tag.tag;
            }

            if (cat == "Game Content" && game_content_tags.Contains<string>(tag.tag))
            {
                return tag.tag;
            }

            if (cat == "Survivor" && survivor_tags.Contains<string>(tag.tag))
            {
                return tag.tag;
            }

            if (cat == "Items" && item_tags.Contains<string>(tag.tag))
            {
                return tag.tag;
            }
        }


        return "";
    }
```
&nbsp;
*When grabbing the main cat we check if the addons tags are found in any of the list of strings that are under the main cats. If the addon has the tags of the main cats, sub cats. Then we know that the main categorey matches the one with the same tags underneath. This can be iffy depending on how the user handles there tags. I tried to handle it in a way where certain cats take prority over others. So it organizes it in a more sufistcated way. Once it finds main/sub cat of the addon we simply instance a new addon object and add it to our current addon list. We then call our update_ui function which is pretty simple.*

```c#
    public static void update_ui() 
    {
        ui = ResourceManager.resources.main;
        init_list();
        clear_ui();

        foreach (Addons.addon addon in Addons.current_addons)
        {
            if (addon.main_cat == "Survivor" && survivorLists.ContainsKey(addon.sub_cat))
            {
                survivorLists[addon.sub_cat].Items.Add(addon.name);
            } else if(addon.main_cat == "Weapons" && weaponLists.ContainsKey(addon.sub_cat))
            {
                weaponLists[addon.sub_cat].Items.Add(addon.name);
            } else if(addon.main_cat == "Items" && itemLists.ContainsKey(addon.sub_cat))
            {
                itemLists[addon.sub_cat].Items.Add(addon.name);
            } else if(addon.main_cat == "Infected" && infectedLists.ContainsKey(addon.sub_cat)) 
            {
                infectedLists[addon.sub_cat].Items.Add(addon.name);
            } else if(addon.main_cat == "Game Mode" && gameModeLists.ContainsKey(addon.sub_cat))
            {
                gameModeLists[addon.sub_cat].Items.Add(addon.name);
            } else if (addon.main_cat == "Game Content" && gameContentLists.ContainsKey(addon.sub_cat))
            {
                gameContentLists[addon.sub_cat].Items.Add(addon.name);
            }
        }
    }
```

*Now the easy part we simply loop through the addons and if there main cat is (Survior obviously I do manual checks so you get the gist of what Im saying) then its sub cat to grab the ui element and add the addon and boom.

*Update Idea : Instead of storing all the sub cats in a array of strings and manualy checking each array. We make a class named categorey and it has the fields of the ~~main cat,~~(not needed), sub cat, addon name and list element or we can pass the addon object.*
&nbsp;
*I can then use a array of categorey classes for each addon.  We will then loop through that array grab and the sub_cat. The sub_cat is important due to how I handle my grouping. My ListBoxes are first the sub cat and are followed up by `List`. For example lets say Main is our form. The list element would be grabbed like this  : `Main.Controls[sub_cat + "List"]` now that we have our ui element we simply add it to ui list element.*

#####JSON EXAMPLE
```json
"49123.vpk" : [
	["ui_list"] = list,
	["addon_id"] = id;
	["addon_name"] = name;
]
```
*This is how I now want to store them. So instead of looping through all the main cat list values with the assigned list for key validation or if its contained.*

#####CURRENT
```c#
        foreach (Addons.addon addon in Addons.current_addons)
        {
            if (addon.main_cat == "Survivor" && survivorLists.ContainsKey(addon.sub_cat))
            {
                survivorLists[addon.sub_cat].Items.Add(addon.name);
            } else if(addon.main_cat == "Weapons" && weaponLists.ContainsKey(addon.sub_cat))
            {
                weaponLists[addon.sub_cat].Items.Add(addon.name);
            } else if(addon.main_cat == "Items" && itemLists.ContainsKey(addon.sub_cat))
            {
                itemLists[addon.sub_cat].Items.Add(addon.name);
            } else if(addon.main_cat == "Infected" && infectedLists.ContainsKey(addon.sub_cat)) 
            {
                infectedLists[addon.sub_cat].Items.Add(addon.name);
            } else if(addon.main_cat == "Game Mode" && gameModeLists.ContainsKey(addon.sub_cat))
            {
                gameModeLists[addon.sub_cat].Items.Add(addon.name);
            } else if (addon.main_cat == "Game Content" && gameContentLists.ContainsKey(addon.sub_cat))
            {
                gameContentLists[addon.sub_cat].Items.Add(addon.name);
            }
        }
```
#####THEORY
```c#
        foreach (Addons.addon addon in Addons.current_addons)
        {
			addon.ui_list.Items.Add(addon.addon_name);
        }
```


# PATHING AND LIST UTILS

&nbsp;
#####COLLECT ALL LIST
```c#
    public static List<ListBox> collect_all_list()
    {
        List<ListBox> lists = new List<ListBox>();

        foreach(ListBox list in infectedLists.Values)
        {
            lists.Add(list);
        }

        foreach (ListBox list in itemLists.Values)
        {
            lists.Add(list);
        }

        foreach (ListBox list in gameModeLists.Values)
        {
            lists.Add(list);
        }

        foreach (ListBox list in gameContentLists.Values)
        {
            lists.Add(list);
        }

        foreach (ListBox list in survivorLists.Values)
        {
            lists.Add(list);
        }

        foreach (ListBox list in weaponLists.Values)
        {
            lists.Add(list);
        }

        return lists;
    }
```
#####HOW I CLEAR LISTS
```c#
    public static void clear_ui()
    {
        List<ListBox> lists = collect_all_list();

        foreach(ListBox box in lists)
        {
            box.Items.Clear();
        }
    }
```


#UNFINSHED STUFF HERE ->

#####THE MOD STRUCTURE :
&nbsp;
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
		public string id;
    }
}

deseralize_js structed = JsonSeralizer.Deserialize<deseralize_js>();
```
&nbsp;
#####FILE WATCHING :
&nbsp;
```c#
    public static void start_file_watch(string path)
    {
        fileSystem = new FileSystemWatcher(path);
        fileSystem.EnableRaisingEvents = true;
        fileSystem.Filter = "*.*";
        fileSystem.Created += file_created;
        fileSystem.Deleted += file_destroyed;
    }
```
