using Terraria.ModLoader;
using Terraria.ID;
using Terraria;
using System.IO;
using System.Reflection;
using System;
using Terraria.ModLoader.Config;
using Terraria.ModLoader.Core;
using System.ComponentModel;
using System.Collections.Generic;
using System.Linq;
using System.Diagnostics;
using Mono.Cecil;
using MonoMod.Cil;
using Newtonsoft.Json;
using System.Net;
using BetterMusic.Conf;
using Terraria.ModLoader.IO;
using static Terraria.ModLoader.ModContent;

// TODO Replacements for overworldday and altoverworldday not working, each vanilla theme switches to the other.

//TODO More rain options (blizzard theme plays even if its just snowing)
//TODO Mod biome music replacement
//TODO MINOR inBossRange bool is true if there is any boss close to the player (unreliable check if the actual npc in question is close)
//TODO MINOR If a music is set to play at some % hp (phase 2 of bossfight) and the boss has reached phase 2 & regained hp, and if the music has an additional requirement such as underground, and if the player leaves underground for a short time and reenters, the 2nd phase music will not play anymore.
//TODO Better checking if npc health has reached 0%
//TODO Condition only replacement: replace music if conditions are met, regardless of the current musicID
//TODO Allow for multiple different music replacements for the same enemy, and add better auto priorites. What should be possible: set p1 theme for golem, set p2 theme for golem once his body is damaged. The priority of p2 theme should be higher than p1 so that it plays
//TODO Add "custom requirement" functionality. Should allow to make a requirement for any bool in Terraria.Main and Terraria.Main.LocalPlayer
//TODO Boss volume config even for non replaced music & song volume normalization & other music trickery such as seeking, interrupting, and only playing for a specific amount of time


namespace BetterMusic.ModLoad
{
    internal static class ModLoader
	{
		public static readonly Assembly TerrariaAssembly;
		public static readonly Type ModCompile;
		public static readonly Type Interface;

		public class MessageOptions
		{
			public string _message;
			public int _gotoMenu;
			public Terraria.UI.UIState _state = null;
			public string _altButtonText = "";
			public Action _altButtonAction = null;

			public MessageOptions(string msg, int gotoMenu = 888, Terraria.UI.UIState state = null, string altButtonText = "", Action altButtonAction = null)
            {
				_message = msg;
				_gotoMenu = gotoMenu;
				_state = state;
				_altButtonText = altButtonText;
				_altButtonAction = altButtonAction;

				if (_state == null)
					_state = ModLoad.ModLoader.getModLoaderState("modsMenu");
			}
			public MessageOptions(MessageOptions msg)
            {
				_message = msg._message;
				_gotoMenu = msg._gotoMenu;
				_state = msg._state;
				_altButtonText = msg._altButtonText;
				_altButtonAction = msg._altButtonAction;
			}
		}

		static ModLoader()
        {
			TerrariaAssembly = System.Reflection.Assembly.Load("Terraria");
			ModCompile = TerrariaAssembly.GetType("Terraria.ModLoader.Core.ModCompile");
			Interface = TerrariaAssembly.GetType("Terraria.ModLoader.UI.Interface");
		}

		public static void extractMod(Mod mod)
		{
			IDisposable modHandle = null;
			if (Directory.Exists(PATH.EXTRACT_DIR))
            {
				Utils.ForceEmptyInsanoMode(PATH.EXTRACT_DIR);
				//Directory.Delete(PATH.EXTRACT_DIR, true);
			}
			Directory.CreateDirectory(PATH.EXTRACT_DIR);

			TmodFile modFile = (TmodFile)mod.GetType()
				.GetProperty("File", BindingFlags.Instance | BindingFlags.NonPublic)
				.GetValue(mod, null);
			modHandle = modFile.Open();
			Type contentConverters = TerrariaAssembly.GetType("Terraria.ModLoader.Core.ContentConverters");
			MethodInfo reverse = contentConverters.GetMethod("Reverse", BindingFlags.NonPublic | BindingFlags.Static);
			foreach (var entry in modFile)
			{
				var name = entry.Name;
				Action<Stream, Stream> converter = null;
				object[] parameters = new object[] { name, converter };
				reverse.Invoke(contentConverters, parameters);
				converter = (Action<Stream, Stream>)parameters[1];

				string[] filter = new string[]
				{ "setup.bat", "BetterMusic.XNA.dll", "Info", "xXDONOTFORGETXx.txt", "IMPORTANT.txt", "BetterMusic.log", "icon_old.png" };

				if (filter.Contains(name))
					continue;
				else if (name == "BetterMusic.txt")
					name = "BetterMusic.cs";

				var path = Path.Combine(PATH.EXTRACT_DIR, name);
				Directory.CreateDirectory(Path.GetDirectoryName(path));

				using (var dst = File.OpenWrite(path))
				using (var src = modFile.GetStream(entry))
				{
					if (converter != null)
						converter(src, dst);
					else
						src.CopyTo(dst);
				}
			};

			if (File.Exists(Path.Combine(PATH.EXTRACT_DIR, "Info")))
				File.Delete(Path.Combine(PATH.EXTRACT_DIR, "Info"));

			File.Copy(Path.Combine(PATH.EXTRACT_DIR, "BetterMusic.cs"), Path.Combine(PATH.EXTRACT_DIR, "BetterMusic.txt"), true);
			File.Copy(Path.Combine(PATH.EXTRACT_DIR, "bwuild.txt"), Path.Combine(PATH.EXTRACT_DIR, "build.txt"), true);

			modHandle?.Dispose();

			Utils.log("[EXTRACTED]");
		}

		public static void buildMod(string path)
		{
			Utils.log("[BUILDING]");
			Type consoleBuildStatus = (Type)ModCompile
				.GetMember("ConsoleBuildStatus", BindingFlags.NonPublic | BindingFlags.Instance)[0];
			var statusObj = Activator.CreateInstance(consoleBuildStatus);
			var modCompObj = Activator.CreateInstance(ModCompile, new object[] { statusObj });

			MethodInfo build = modCompObj.GetType().GetMethods(BindingFlags.NonPublic | BindingFlags.Instance)
				.First(m => m.Name == "Build" && m.GetParameters()[0].ParameterType == typeof(string));
			build.Invoke(modCompObj, new object[] { path });
		}

		public static void reloadMods()
		{
			Utils.log("[RELOADING]");
			Main.menuMode = 10006;
			typeof(Terraria.ModLoader.ModLoader).GetMethod("Reload", BindingFlags.NonPublic | BindingFlags.Static)
				.Invoke(typeof(ModLoader), new object[] { });
		}

		public static void saveConfig(ModConfig config)
        {
			typeof(ConfigManager).GetMethod("Save", BindingFlags.Static | BindingFlags.NonPublic)
				.Invoke(null, new object[] { config });

			typeof(ConfigManager).GetMethod("Load", BindingFlags.Static | BindingFlags.NonPublic)
				.Invoke(null, new object[] { config });
		}

        public static bool isEnabled(string modName)
        {
            bool enabled = (bool)typeof(Terraria.ModLoader.ModLoader)
				.GetMethod("IsEnabled", BindingFlags.Static | BindingFlags.NonPublic)
				.Invoke(null, new object[] { modName });
            return enabled;
        }

		public static HashSet<string> getEnabledMods()
        {
			HashSet<string> mods = new HashSet<string>();
			string json = File.ReadAllText(Path.Combine(PATH.MODS_DIR, "enabled.json"));
			Newtonsoft.Json.JsonConvert.PopulateObject(json, mods);
			return mods;
        }

		public static void showMessage(string msg, int gotoMenu, Terraria.UI.UIState state = null, string altButtonText = "", Action altButtonAction = null)
        {
			var infoMessage = Interface.GetField("infoMessage", BindingFlags.Static | BindingFlags.NonPublic).GetValue(null);
			var Show = TerrariaAssembly.GetType("Terraria.ModLoader.UI.UIInfoMessage")
				.GetMethod("Show", BindingFlags.NonPublic | BindingFlags.Instance);
			
			Show.Invoke(infoMessage, new object[] { msg, gotoMenu, state, altButtonText, altButtonAction });
		}

		public static void showMessage(MessageOptions msg)
        {
			showMessage(msg._message, msg._gotoMenu, msg._state, msg._altButtonText, msg._altButtonAction);
        }

		public static Terraria.UI.UIState getModLoaderState(string name)
        {
			return (Terraria.UI.UIState)Interface.GetField(name, BindingFlags.NonPublic | BindingFlags.Static).GetValue(null);
        }
    }
}


namespace BetterMusic.Conf
{
    public class ConfigClient : ModConfig
	{
		public override ConfigScope Mode => ConfigScope.ClientSide;

		[Label("[ADD YOUR MUSIC]")]
		[Tooltip("Click here & save whenever you add, remove or modify files in the \"Your Music\" folder.\n" +
			"The files from \"Your Music\" will be copied to the game, and tModLoader will reload." +
			"\nThis will allow you to use those files in this config to replace game music.")]
		[BackgroundColor(255, 0, 0, 0)]
		[ReloadRequired]
		[DefaultValue(false)]
		public bool doReload = false;

		[Header("Your Music")]
		[Label("NPC Music")]
		[Tooltip("Modify music for individual NPCs (modded and vanilla)")]
		[BackgroundColor(80, 100, 20, 100)]
		[SeparatePage]
		public Dictionary<NPCDefinition, NPCMusicConfigs> npcOpts 
			= new Dictionary<NPCDefinition, NPCMusicConfigs>();
		
		[Label("Vanilla Environment Music")]
		[Tooltip("Modify vanilla music by music name")]
		[BackgroundColor(100, 80, 20, 100)]
		[SeparatePage]
		public Dictionary<MusicIDSelect, MusicIDConfigs> idOpts 
			= new Dictionary<MusicIDSelect, MusicIDConfigs>();

		[Label("Modded Environment Music")]
		[Tooltip("Replace any modded track, such as biome or event music. If you want to modify enemy songs, use the NPC music list (it also contains modded NPCs).\n" +
			"HOW TO USE: When the modded music you want to replace is playing in-game, click on the plus button in this list to add it." +
			"You can then replace it as usual." +
			"\nNote that sometimes modders change the names of the music files, in which case this process must be repeated.")]
		[BackgroundColor(10, 0, 50, 100)]
		[SeparatePage]
		public Dictionary<MusicName, MusicIDConfigs> modMusicOpts
			= new Dictionary<MusicName, MusicIDConfigs>();

		[Label("Drip Lord")]
		[Tooltip("Enables Drip Lord theme.")]
		[DefaultValue(false)]
		private bool dripLord = false;

        public override void OnChanged()
        {
			if (BetterMusic.songsPrepared && BetterMusic.postSetupContent && GetInstance<ConfigClient>() != null)
			{
				Utils.log("OnChanged");
				MusicManager.prepareCustomSongs();
			}
        }

		public override bool NeedsReload(ModConfig pendingConfig)
        {
			return base.NeedsReload(pendingConfig);
        }

        public override bool AcceptClientChanges(ModConfig pendingConfig, int whoAmI, ref string message)
        {
			Utils.log("AcceptClientChanges");
			return base.AcceptClientChanges(pendingConfig, whoAmI, ref message);
        } 
    }

	public struct Music
    {
		[JsonIgnore]
		public string path;
		[JsonIgnore]
		public int priority;
    }
	public enum BossPhase { Always, Phase1, Phase2 }
	public enum WorldTime { Always, Night, Day }
	public enum WorldCompletion { Always, PreHardmode, Hardmode, PostMoonlord }
	public enum PlayerHeight { Always, Overground, Underground }
	public enum RainOption { Always, NoRain, Rain }
	public enum MusicPriorityOption
	{ Auto=-1, BiomeLow=1, BiomeMedium=2, BiomeHigh=3, Environment=4, Event=5, BossLow=6, BossMedium=7, BossHigh=8, Max=99 }

	[TypeConverter(typeof(ToFromStringConverter<MusicIDSelect>))]
	public class MusicIDSelect : TagSerializable
    {
		[OptionStrings(new string[] { "AltOverworldDay", "AltUnderground", "Boss1", "Boss2",
			"Boss3", "Boss4", "Boss5", "Corruption", "Crimson", "Desert", "Dungeon", "Eclipse",
			"Eerie", "FrostMoon", "GoblinInvasion", "Hell", "Ice", "Jungle", "LunarBoss",
			"MartianMadness", "Mushrooms", "Night", "Ocean", "OldOnesArmy", "OverworldDay",
			"PirateInvasion", "Plantera", "PumpkinMoon", "Rain", "RainSoundEffect", "Sandstorm",
			"Snow", "Space", "Temple", "TheHallow", "TheTowers", "Title", "Underground",
			"UndergroundCorruption", "UndergroundCrimson", "UndergroundHallow" })]
		[DefaultValue("Choose the music to replace")]
		[DrawTicks]
		public string name = "";

		public static readonly Func<TagCompound, MusicIDSelect> DESERIALIZER = Load;

		[JsonIgnore]
		public int Type
		{
			get
			{
				var fields = typeof(MusicID)
					.GetFields().Where(f => f.Name == name);
				if (fields.Count() == 0)
					return -1;
				else 
					return (short)fields.First().GetValue(null);
			}
		}

		[JsonIgnore]
		public int valid =  -1;

		public MusicIDSelect()
		{
			name = "Enter music name";
		}

		public MusicIDSelect(string name)
		{
			this.name = name;
		}

		public static MusicIDSelect FromString(string s)
		{
			return new MusicIDSelect(s);
		}

		public override string ToString()
		{
			return $"{name}";
		}

        public TagCompound SerializeData()
		{
			return new TagCompound
			{
				["name"] = name
			};
		}

		public static MusicIDSelect Load(TagCompound tag)
		{
			return new MusicIDSelect(tag.GetString("name"));
		}
	}

	[SeparatePage]
	[BackgroundColor(40, 40, 80, 200)]
	[TypeConverter(typeof(ToFromStringConverter<NPCMusicConfigs>))]
	public class NPCMusicConfigs
	{
		[SeparatePage]
		[BackgroundColor(40, 40, 80, 200)]
		[Label("Replaced music for this enemy")]
		[Tooltip("Here you can add your custom music. Click on the plus sign and enter the name of the file to start." +
			"\n\nYou can add multiple different tracks to the same enemy. Once in game, the file that meets the conditions you specify will play." +
			"\nIf there are multiple files whose conditions are met, the one with the lowest health % value will be chosen (provided the enemy's HP is below that mark)." +
			"\nMake sure you're not adding multiple files with overlapping conditions and the same health value.")]
		public List<NPCMusicConfig> ls;

		public NPCMusicConfigs()
		{
			ls = new List<NPCMusicConfig>();
		}
		public void Add(NPCMusicConfig item)
		{
			ls.Add(item);
		}

		public override string ToString()
		{
			if (ls.Count == 0)
				return "Click me";
			else if (ls.Count == 1)
				return ls[0].ToString();
			return String.Join(", ", ls.Select(f =>
			{
				string fstr = f.ToString();
				if (fstr.Contains("File not found") || fstr.Contains("File not added"))
					return fstr;
				return f.fileName;
			}));
		}
	}

	[SeparatePage]
	public class NPCMusicConfig
	{
		[Label("Music file name (with extension)")]
		[Tooltip("Input the name of the music file that should be played for this enemy.\n" +
			"The file must first be added with the [ADD YOUR MUSIC] button.")]
		[DefaultValue("file.mp3")]
		public string fileName = "";

		[Label("Only play after NPC health drops to (percent)")]
		[Tooltip("Custom music will only play after boss health drops to specified percent of total HP." +
			"\n\nThis can also be used to only trigger the song once the boss enters second form." +
			"\nFor example, if a boss has an invulnerable part that only gets exposed in his second phase, you can set this parameter to 99.9% for that part." +
			"\nThat way, once this part becomes vulnerable, its HP will likely already drop below 99.9%, and custom music will play. (Example: Golem Body)" +
			"\nAlternatively, if a boss enters phase 2 once its HP reaches 0, you can set this parameter to 0.5%. " +
			"\n(NOTE: This last method can be unreliable. It's possible that you will deal more damage in a single strike than 0.5% of the boss HP, instantly activating phase 2 and skipping this check." +
			"\nIf you expect to deal more than 0.5% damage in a single hit, set it to a bigger number than 0.5. The bigger the number, the more reliable it becomes.)" +
			"\n\nYou can edit this value more precisely at \\Modloader\\Mod Configs\\BetterMusic_ClientConfiguration.json")]
		[Range(0.25f, 100)]
		[Increment(0.25f)]
		[DefaultValue(100)]
		[DrawTicks]
		public float hp = 100;
		
		[Label("Boss phase (Vanilla only)")]
		[Tooltip("Phase 2 is defined as follows:" +
			"\nEye of Cthulhu: HP below 50% (65% in Expert Mode)" +
			"\nEater of Worlds: Not yet implemented" +
			"\nBrain of Cthulhu: All creepers dead" +
			"\nSkeletron: Both hands destroyed" +
			"\nThe Twins: One twin dead" + 
			"\nSkeletron Prime: All hands destroyed" +
			"\nGolem: Head detached" +
			"\nMoon Lord: Core exposed" +
			"\n\nFor all other bosses, phase 2 is defined as 50% of total HP.")]
		[DefaultValue(BossPhase.Always)]
		[DrawTicks]
		public BossPhase phase = BossPhase.Always;

		[Label("Time")]
		[Tooltip("Only play at day or at night")]
		[DefaultValue(WorldTime.Always)]
		[DrawTicks]
		public WorldTime time = WorldTime.Always;

		[Label("Height")]
		[Tooltip("Only play underground or overground")]
		[DefaultValue(PlayerHeight.Always)]
		[DrawTicks]
		public PlayerHeight height = PlayerHeight.Always;

		[Label("Rain")]
		[Tooltip("Only play when it is raining or when it is clear")]
		[DefaultValue(RainOption.Always)]
		[DrawTicks]
		public RainOption rain = RainOption.Always;

		[Label("Game progression")]
		[DrawTicks]
		[DefaultValue(WorldCompletion.Always)]
		[Tooltip("Only play in pre Hardmode, Hardmode or post Moon Lord")]
		public WorldCompletion compl = WorldCompletion.Always;

		[Header("Advanced")]
		[DrawTicks]
		[Label("Music priority")]
		[Tooltip("This should probably stay on Auto, unless you're replacing music for an NPC which doesn't otherwise have music (in which case Auto simply defaults to BossLow).\n" +
			"\nExample use: You want to add music to the Martian Saucer enemy, which appears during the Martian Madness event. Martian Madness music has a priority of 7." +
			"\nSince \"Auto\" defaults to BossLow (6), you'd have to increase the priority to BossMedium (7) in this case.")]
		[DefaultValue(MusicPriorityOption.Auto)]
		public MusicPriorityOption priority = MusicPriorityOption.Auto;

		//[Label("Enable continous HP check")]
		//[Tooltip("With this option on, the music will play if and ONLY if the npc HP is below the specifed percentage, rather than always playing after the HP has been reduced to that point." +
		//	"\nEnable this if you're having reliability issues with the HP option.")]
		//[DefaultValue(false)]
		//public bool continousHpCheck = false;

		[JsonIgnore]
		public int hpBelow = 0;

		[JsonIgnore]
		public Music music;

		public bool phase2Active(NPC npc)
        {
			if (npc.boss)
            {
				switch (npc.type)
                {
					case 4:
						if (Main.expertMode)
							return (float)(npc.life * 100) / npc.lifeMax <= 65;
						else 
							return (float)(npc.life * 100) / npc.lifeMax <= 50;
					case 266:
						return !NPC.AnyNPCs(267);
					case 35:
						return !NPC.AnyNPCs(36);
					case 125:
						return !NPC.AnyNPCs(126);
					case 126:
						return !NPC.AnyNPCs(125);
					case 127:
						return !NPC.AnyNPCs(128) && !NPC.AnyNPCs(129)
							&& !NPC.AnyNPCs(130) && !NPC.AnyNPCs(131);
					case 245:
						return NPC.AnyNPCs(249);
					case 398:
						return npc.life < npc.lifeMax;
				}
            }

			return (float)(npc.life * 100) / npc.lifeMax <= 50;
		}

		public void resetConditionChecks()
        {
			hpBelow = 0;
        }

		public int healthBelowThreshold(NPC npc)
        {
			if (hp >= 100)
				return 1;
			if (100 * ((double)npc.life / npc.lifeMax) <= hp)
				return 1;
			return 0;
		}

		public bool conditionsMet(NPC npc)
		{
			if (rain == RainOption.Rain && !Main.raining)
				return false;
			if (rain == RainOption.NoRain && Main.raining)
				return false;
			if (compl == WorldCompletion.PreHardmode && Main.hardMode)
				return false;
			if (compl == WorldCompletion.Hardmode && !Main.hardMode)
				return false;
			if (compl == WorldCompletion.PostMoonlord && !NPC.downedMoonlord)
				return false;
			if (time == WorldTime.Day && !Main.dayTime)
				return false;
			if (time == WorldTime.Night && Main.dayTime)
				return false;
			if (height == PlayerHeight.Underground && !Main.LocalPlayer.ZoneRockLayerHeight && !Main.LocalPlayer.ZoneUnderworldHeight)
				return false;
			if (height == PlayerHeight.Overground && !Main.LocalPlayer.ZoneDirtLayerHeight && !Main.LocalPlayer.ZoneOverworldHeight && !Main.LocalPlayer.ZoneSkyHeight)
				return false;
			if (phase == BossPhase.Phase2 && !phase2Active(npc))
				return false;
			if (phase == BossPhase.Phase1 && phase2Active(npc))
				return false;
			return true;
		}

		public override string ToString()
		{
			if (!Config.Instance.addedFiles.Contains(fileName))
            {
				if (!File.Exists(Path.Combine(PATH.YOURMUSIC_DIR, fileName)))
					return $"File not found: {fileName}";
				return $"File not added: {fileName}";
			}

			string done = $"{fileName} (";

			if (hp < 100)
				done += $"HP:{ hp}%";

			if (phase == BossPhase.Phase2)
				done += ", phase 2";
			else if (phase == BossPhase.Phase1)
				done += ", phase 1";

			if (time == WorldTime.Day)
				done += ", day";
			else if (time == WorldTime.Night)
				done += ", night";

			if (compl == WorldCompletion.PreHardmode)
				done += ", pre Hardmode";
			else if (compl == WorldCompletion.Hardmode)
				done += ", Hardmode";
			else if (compl == WorldCompletion.PostMoonlord)
				done += ", post ML";

			if (height == PlayerHeight.Overground)
				done += ", overground";
			else if (height == PlayerHeight.Underground)
				done += ", underground"; 

			if (rain == RainOption.Rain)
				done += ", rain";
			else if (rain == RainOption.NoRain)
				done += ", no rain";

			if (done[done.Length - 1] == '(')
				done = fileName;
			else
			{
				done += ')';
				if (done.Contains("(, "))
				{
					var split = done.Split(new string[] { "(, " }, StringSplitOptions.None);
					done = split[0] + '(' + split[1];
				}
			}

			return done;
		}
	}

	[SeparatePage]
	[BackgroundColor(40, 40, 80, 200)]
	[TypeConverter(typeof(ToFromStringConverter<MusicIDConfigs>))]
	public class MusicIDConfigs //: IEnumerable<MusicIDConfig>
    {
		[SeparatePage]
		[Label("Replaced music")]
		[Tooltip("Here you can add your custom music. Click on the plus sign and enter the name of the file to start." +
			"\n\nYou can add multiple different tracks to replace the same vanilla track. Once in game, the file that meets the conditions you specify will play." +
			"\nIf there are multiple files whose conditions are met, they get randomly chosen based on the chance you specify." +
			"\nMake sure you're not adding multiple files with overlapping conditions and 100% chance.")]
		[BackgroundColor(40, 40, 80, 200)]
		public List<MusicIDConfig> ls; //{ get; private set; }

        public MusicIDConfigs()
        {
            ls = new List<MusicIDConfig>();
        }
        public void Add(MusicIDConfig item)
        {
            ls.Add(item);
        }

		public override string ToString()
        {
			if (ls.Count == 0)
				return "Click me";
			else if (ls.Count == 1)
				return ls[0].ToString();
			return String.Join(", ", ls.Select(f => 
			{
				string fstr = f.ToString();
				if (fstr.Contains("File not found") || fstr.Contains("File not added"))
					return fstr;
				return f.fileName;
			}));
        }
    }

	[SeparatePage]
	public class MusicIDConfig
	{
		[Label("Music file name (with extension)")]
		[Tooltip("Input the name of the music file that should be played for this enemy.\n" +
			"The file must first be added with the [ADD YOUR MUSIC] button.")]
		[DefaultValue("file.mp3")]
		public string fileName = "";

		[Label("Probability of playing")]
		[Tooltip("When replacing environment music, the order of the files matters:" +
			"\nIf you add two files with the same conditions, the first one will be played with the specified probability." +
			"\nIf the roll of the first one wasn't successful, the second file will be played with its probability." +
			"\nSo for example, if you wanted to replace a vanilla track with two alternates, and if you wanted both to be equally likely to play," +
			"\nyou would add the first file with 50%, and the second with 100%, NOT 50%." +
			"\nIf you wanted to add two songs as alternates to an existing vanilla song (for a total of 3 alternates with the same probability)," +
			"\nyou would need to add the first file with 33%, the second with 50%.")]
		[Range(0, 100)]
		[DefaultValue(100)]
		public int probability = 100;

		//[Label("Chance to play")]
		//[Tooltip("The chance that custom music will play instead of the original track." +
		//	"\nIf you select \"Add as alternate\" here, the chance will be determined automatically based on the amount of tracks that play in the same conditions." +
		//	"")]
		//[DefaultValue("100%")]
		//[DrawTicks]
		//[OptionStrings(new string[] { "Add as alternate", 
		//	"1%", "2%", "3%", "4%", "5%", "6%", "7%", "8%", "9%", "10%", // No, I will not make any custom mod UI
		//	"11%", "12%", "12.5%", "13%", "14%", "15%", "16%", "16.6%", "17%", "18%", "19%", "20%", 
		//	"21%", "22%", "23%", "24%", "25%", "26%", "27%", "28%", "29%", "30%", 
		//	"31%", "32%", "33%", "33.3%", "34%", "35%", "36%", "37%", "38%", "39%", "40%",                       
		//	"41%", "42%", "43%", "44%", "45%", "46%", "47%", "48%", "49%", "50%", 
		//	"51%", "52%", "53%", "54%", "55%", "56%", "57%", "58%", "59%", "60%", 
		//	"61%", "62%", "63%", "64%", "65%", "66%", "67%", "68%", "69%", "70%", 
		//	"71%", "72%", "73%", "74%", "75%", "76%", "77%", "78%", "79%", "80%", 
		//	"81%", "82%", "83%", "84%", "85%", "86%", "87%", "88%", "89%", "90%", 
		//	"91%", "92%", "93%", "94%", "95%", "96%", "97%", "98%", "99%", "100%" })]
		//public string chancestr = "100%";

		[Label("Time")]
		[Tooltip("Only play at day or at night")]
		[DefaultValue(WorldTime.Always)]
		[DrawTicks]
		public WorldTime time = WorldTime.Always;

		[Label("Height")]
		[Tooltip("Only play underground or overground")]
		[DefaultValue(PlayerHeight.Always)]
		[DrawTicks]
		public PlayerHeight height = PlayerHeight.Always;

		[Label("Rain")]
		[Tooltip("Only play when it is raining or when it is clear")]
		[DefaultValue(RainOption.Always)]
		[DrawTicks]
		public RainOption rain = RainOption.Always;

		[Label("Game progression")]
		[DrawTicks]
		[DefaultValue(WorldCompletion.Always)]
		[Tooltip("Only play in pre Hardmode, Hardmode or post Moon Lord")]
		public WorldCompletion compl = WorldCompletion.Always;

		[JsonIgnore]
		public int checkedRoll = 0;

		[JsonIgnore]
		public Music music;

		public void resetConditionChecks()
		{
			checkedRoll = 0;
		}

		public bool randomSuccess()
        {
			if (Main.rand.Next(99) >= probability)
				return false;
			return true;
		}

        public bool isEqual(MusicIDConfig opts)
        {
			return fileName == opts.fileName
				&& probability == opts.probability
				&& rain == opts.rain
				&& compl == opts.compl
				&& time == opts.time
				&& height == opts.height;
		}

        public bool conditionsMet()
        {
			if (rain == RainOption.Rain && !Main.raining)
				return false;
			if (rain == RainOption.NoRain && Main.raining)
				return false;
			if (compl == WorldCompletion.PreHardmode && Main.hardMode)
				return false;
			if (compl == WorldCompletion.Hardmode && !Main.hardMode)
				return false;
			if (compl == WorldCompletion.PostMoonlord && !NPC.downedMoonlord)
				return false;
			if (time == WorldTime.Day && !Main.dayTime)
				return false;
			if (time == WorldTime.Night && Main.dayTime)
				return false;
			if (height == PlayerHeight.Underground && !BetterMusic.undergroundAudio)
				return false;
			if (height == PlayerHeight.Overground && BetterMusic.undergroundAudio)
				return false;
			return true;
        }

		public override string ToString()
		{
			if (!Config.Instance.addedFiles.Contains(fileName))
			{
				if (!File.Exists(Path.Combine(PATH.YOURMUSIC_DIR, fileName)))
					return $"File not found: {fileName}";
				return $"File not added: {fileName}";
			}

			string done = $"{fileName} ("; 

			if (probability < 100)
				done += $"chance: {probability}%";

			if (time == WorldTime.Day)

				done += ", day";
			else if (time == WorldTime.Night)
				done += ", night";

			if (compl == WorldCompletion.PreHardmode)
				done += ", pre Hardmode";
			else if (compl == WorldCompletion.Hardmode)
				done += ", Hardmode";
			else if (compl == WorldCompletion.PostMoonlord)
				done += ", post ML";

			if (height == PlayerHeight.Overground)
				done += ", overground";
			else if (height == PlayerHeight.Underground)
				done += ", underground";

			if (rain == RainOption.Rain)
				done += ", rain";
			else if (rain == RainOption.NoRain)
				done += ", no rain";

			if (done[done.Length - 1] == '(')
				done = fileName;
			else
            {
				done += ')';
				if (done.Contains("(, "))
				{
					var split = done.Split(new string[] { "(, " }, StringSplitOptions.None);
					done = split[0] + '(' + split[1];
				}
			}

			return done;
		}

		// A constructor here breaks everything for some reason. bleargh.
        public static MusicIDConfig Create(MusicIDConfig m)
        {
            MusicIDConfig x = new MusicIDConfig();
            x.fileName = m.fileName;
            x.probability = m.probability;
            x.height = m.height;
            x.music = m.music;
            x.rain = m.rain;
            x.time = m.time;
            x.compl = m.compl;
            return x;
        }
	}

	[TypeConverter(typeof(ToFromStringConverter<MusicName>))]
	public class MusicName
	{
		internal string name;
		internal string displayName = "";
		public MusicName()
        {
			name = BetterMusic.getCurrentMusicName();
			name = name == null ? "" : name;

			if (name.Contains('/'))
			{
				var split = name.Split('/');
				if (split.Length == 4 && split[1] == "Sounds" && split[2] == "Music")
					displayName = $"{split[split.Length - 1]} ({split[0]})";
				else
					displayName = $"{split[split.Length - 1]} ({name.Substring(0, name.LastIndexOf('/')) + 1})";
			}
		}
		public MusicName(string musicName)
		{
			name = musicName;

			if (name.Contains('/'))
			{
				var split = name.Split('/');
				if (split.Length == 4 && split[1] == "Sounds" && split[2] == "Music")
					displayName = $"{split[split.Length - 1]} ({split[0]})";
				else
					displayName = $"{split[split.Length - 1]} ({name.Substring(0, name.LastIndexOf('/')) + 1})";
			}
		}
		public override string ToString()
        {
			return displayName;
        }
		public static MusicName FromString(string s)
        {
			if (s.Contains('('))
            {
				int p1 = s.IndexOf('(');
				int p2 = s.IndexOf(')');
				string path = s.Substring(p1 + 1, s.Length - p1 - 2);
				if (!path.Contains("/"))
					path += "/Sounds/Music/";
				s = path + s.Substring(0, s.IndexOf('(') - 1);
			}
			return new MusicName(s);
        }
    }

	public class Config
	{
		public bool xxX_Enable_Drip_Lord_Theme_WARNING_USE_AT_OWN_RISK_Xxx = false;
		public bool firstEverEnter = true;
		public bool showWelcomeMessage = true;
		public bool doneFirstTimeSetup = false;
		public bool doSecondReload = false;
		public string lastVersion = "";

		public HashSet<string> addedFiles = new HashSet<string>();

		private Config() 
		{
			addedFiles = new HashSet<string>();
			firstEverEnter = true;
			showWelcomeMessage = true;
			doneFirstTimeSetup = false;
			doSecondReload = false;
			lastVersion = "";
			xxX_Enable_Drip_Lord_Theme_WARNING_USE_AT_OWN_RISK_Xxx = false;
		}
		private static Config instance;
		public static Config Instance
		{
			get
			{
				if (instance == null)
					instance = new Config();
				return instance;
			}
			private set { instance = value; }
		}

		public string Save(string path="")
		{
			if (String.IsNullOrEmpty(path))
				path = PATH.CONFIG_FILE;
			string json = Newtonsoft.Json.JsonConvert.SerializeObject(Config.Instance, Formatting.Indented);
			File.WriteAllText(path, json);
			return json;
		}
		public string Load(string path = "")
		{
			if (String.IsNullOrEmpty(path))
				path = PATH.CONFIG_FILE;
			string json = "";
			try
			{
				json = File.ReadAllText(path);
				Newtonsoft.Json.JsonConvert.PopulateObject(json, Config.Instance);
			}
			catch
			{
				Config.Instance = null;
				json = Newtonsoft.Json.JsonConvert.SerializeObject(Config.Instance);
				Directory.CreateDirectory(Directory.GetParent(path).FullName);
				File.WriteAllText(path, json);
			}
			return json;
		}
	}
}


namespace BetterMusic
{
    public class BetterMusic : Mod
	{
		#region Fields

		public static BetterMusic betterMusic;
		public string logPath = ""; //@"C:\Users\sokfi\Documents\11RandomSmall\BetterMusic.log";
		public string changeLog = "";

		int prevMenuMode = -1;
		int prevMusic = -1;
		bool musicSwitched = false;
		int curModNpcMusic = -1;
		int framecount = 0;
		bool inBossRange = false;
		bool forceReplaceTitle = false;
		bool enteredWorldOnce = false;
		public static bool songsPrepared = false;
		public static bool postSetupContent = false;
		internal static ModLoad.ModLoader.MessageOptions message = null;

		public static bool undergroundAudio { get { 
				return (double)Main.LocalPlayer.position.Y > Main.worldSurface * 16.0 + (double)(Main.screenHeight / 2);  
			} }

		public static bool replaceMusicID = false;
		public static bool replaceModBossMusic = false;
		public static bool replaceVanillaOrNonBossMusic = false;

		public static List<string> musicIds { get; private set; }

		public static Dictionary<int, List<MusicIDConfig>> musicIdToMusicInfo
			= new Dictionary<int, List<MusicIDConfig>>();

		public static Dictionary<int, List<NPCMusicConfig>> npcToMusicInfo 
			= new Dictionary<int, List<NPCMusicConfig>>();

		public static Dictionary<Mod, Dictionary<string, List<NPCMusicConfig>>> modNpcToMusicInfo
			= new Dictionary<Mod, Dictionary<string, List<NPCMusicConfig>>>();

		public static Dictionary<int, NPCMusicConfig> nonBossNpcToMusicInfo
			= new Dictionary<int, NPCMusicConfig>();

		public static IDictionary<SoundType, IDictionary<string, int>> sounds;

		#endregion Fields

		public override void Load()
		{
			betterMusic = this;
			musicIds = typeof(MusicID).GetFields()
				.Where(f => f.IsPublic && f.FieldType == typeof(short))
				.Select(f => f.Name).OrderBy(x => x).ToList();

			Config.Instance.Load(PATH.CONFIG_FILE);
			//Utils.show($"{Config.Instance.lastVersion}, {ModLoader.GetMod("BetterMusic").Version}");
			
			GetInstance<ConfigClient>().doReload = false;

            ModLoad.ModLoader.saveConfig(GetInstance<ConfigClient>());

			ReplacerSetup.setupReplacer();
			Utils.log($"[LOAD COMPLETE]");
		}

        public override void PostSetupContent()
        {
			sounds = (IDictionary<SoundType, IDictionary<string, int>>)typeof(SoundLoader)
				.GetField("sounds", BindingFlags.NonPublic | BindingFlags.Static)
				.GetValue(null);

			bool removedFiles = MusicManager.updateAddedFileList(false);
			if (Config.Instance.firstEverEnter)
            {
				Config.Instance.firstEverEnter = false;
				ReplacerSetup.setDefaults();
			}
			else if (Config.Instance.lastVersion != ModLoader.GetMod("BetterMusic").Version.ToString())
			{
				string msg = "Music Replacer just updated. ";
				if (!String.IsNullOrEmpty(changeLog))
					msg += $"\n{changeLog}\n\n";
				if (removedFiles)
                {
					message = new ModLoad.ModLoader.MessageOptions(msg + "Custom files must be readded. Add now?" +
						"\nNOTE: TmodLoader should reload when you click Add. Check the README in Your Music if it doesn't.", 888,
						ModLoad.ModLoader.getModLoaderState("modsMenu"),
						altButtonText: "Add",
						altButtonAction: (() => {
							GetInstance<ConfigClient>().doReload = true;
						}));
				}
				else if (!String.IsNullOrEmpty(changeLog))
				{
					message = new ModLoad.ModLoader.MessageOptions(msg, 888,
						ModLoad.ModLoader.getModLoaderState("modsMenu"),
						altButtonText: "Add",
						altButtonAction: (() => {
							GetInstance<ConfigClient>().doReload = true;
						}));
				}
			}
			else if (removedFiles)
            {
				BetterMusic.message = new ModLoad.ModLoader.MessageOptions("Music Replacer: One or more files has not been added. Add now?" +
					"\nNOTE: TmodLoader should reload when you click Add. Check the README in Your Music if it doesn't.",
					altButtonText: "Add",
					altButtonAction: (() => { GetInstance<ConfigClient>().doReload = true; }));
			}

			Config.Instance.lastVersion = ModLoader.GetMod("BetterMusic").Version.ToString();
			Config.Instance.Save(PATH.CONFIG_FILE);

			IL.Terraria.Main.UpdateAudio += UpdateAudioHook;

			postSetupContent = true;

			Utils.log("[POST SETUP COMPLETE]");
		}

        public override void Close()
		{
			closeMusic();
			IL.Terraria.Main.UpdateAudio -= UpdateAudioHook;
			closeMusic();

			songsPrepared = false;
			postSetupContent = false;
			Config.Instance.Save(PATH.CONFIG_FILE);

			base.Close();
		}

        public override void PostUpdateInput()
        {
            base.PostUpdateInput();

			if (postSetupContent && GetInstance<ConfigClient>() != null)
            {
				if (Main.gameMenu)
				{
					if (GetInstance<ConfigClient>().doReload)
					{
						reloadAddMusic();
					}

					if (Main.menuMode == 0)
					{
						if (!songsPrepared)
						{
							Utils.log("[FIRST ENTER]");

							if (Config.Instance.doSecondReload)
							{
								postSetupContent = false;
								songsPrepared = false;
								Config.Instance.doSecondReload = false;
								Config.Instance.Save();

								closeMusic();

								ModLoad.ModLoader.reloadMods();
							}

							MusicManager.prepareCustomSongs();
							songsPrepared = true;
						}
						else
                        {
							//ModLoad.ModLoader.showMessage("TEST TEST TEST", Main.menuMode);
						}

						if (Config.Instance.showWelcomeMessage)
                        {
							Config.Instance.showWelcomeMessage = false;
							string txt = "Better Music & Music Replacer\n\n"
								+ File.ReadAllText(Path.Combine(PATH.YOURMUSIC_DIR, "# README #.txt"));
							ModLoad.ModLoader.showMessage(txt, Main.menuMode,
								altButtonText: "Open Your Music",
								altButtonAction: (() => { 
									Config.Instance.showWelcomeMessage = true; 
									Terraria.Utils.OpenFolder(PATH.YOURMUSIC_DIR); 
								}));
						}
						else if (message != null)
						{
							var msg = new ModLoad.ModLoader.MessageOptions(message);
							message = null;
							ModLoad.ModLoader.showMessage(msg);
						}
					}

					prevMenuMode = Main.menuMode;
				}
				else if (!Main.gameMenu && songsPrepared)
                {
					enteredWorldOnce = true;
				}
			}
		}

		public override void UpdateMusic(ref int music, ref MusicPriority priority)
		{
			base.UpdateMusic(ref music, ref priority);

			if ((framecount++ % 300) == 0 && !Main.gameMenu)
            {
				string logtxt = $"currently playing: {Main.curMusic}, in menu: {Main.menuMode}";
				if (Main.raining)
					logtxt += $", rain: {Main.maxRaining}";
				if (undergroundAudio)
					logtxt += $", ug audio";
				Utils.log(logtxt);
			}

            musicSwitched = prevMusic != Main.curMusic;

			if (Main.gameMenu && songsPrepared && !forceReplaceTitle && !enteredWorldOnce)
			{
				if (Main.curMusic != 6 && musicIdToMusicInfo.TryGetValue(6, out var minfo))
				{
					var slots = minfo.Select(x => GetSoundSlot(SoundType.Music, x.music.path));
					if (!slots.Contains(Main.curMusic))
                    {
						Utils.log("Forcing Title replace");
						forceReplaceTitle = true;
					}
				}
			}

			if (replaceVanillaOrNonBossMusic)
			{
				List<int> ignoreResetHpCheck = new List<int>();

				for (int i = 0; i < Main.npc.Length; i++)
				{
					if (npcToMusicInfo.ContainsKey(Main.npc[i].type) && Main.npc[i].active)
					{
						var musicls = npcToMusicInfo[Main.npc[i].type];
						var valid = musicls.Where(m => m.conditionsMet(Main.npc[i])).ToList();
						var validAndBelowThreshold = new List<NPCMusicConfig>();

						foreach (var musicConf in valid)
						{
							if (musicConf.hpBelow == 1 || musicConf.healthBelowThreshold(Main.npc[i]) == 1)
							{
								ignoreResetHpCheck.Add(Main.npc[i].type);
								musicConf.hpBelow = 1;
								validAndBelowThreshold.Add(musicConf);
							}
						}

						if (npcIsClose(Main.npc[i]))
						{
							if (validAndBelowThreshold.Count == 1)
							{
								music = GetSoundSlot(SoundType.Music, validAndBelowThreshold[0].music.path);
								priority = (MusicPriority)validAndBelowThreshold[0].music.priority;
							}
							else if (validAndBelowThreshold.Count > 1)
							{
								float min = validAndBelowThreshold.Select(m => m.hp).Min();
								var minConfig = validAndBelowThreshold.Where(m => m.hp == min).First();
								music = GetSoundSlot(SoundType.Music, minConfig.music.path);
								priority = (MusicPriority)minConfig.music.priority;
							}
						}
					}
				}

				foreach (var key in npcToMusicInfo.Keys.Except(ignoreResetHpCheck))
				{
					foreach (var item in npcToMusicInfo[key])
						item.hpBelow = 0;
				}
			}

			if (Config.Instance.xxX_Enable_Drip_Lord_Theme_WARNING_USE_AT_OWN_RISK_Xxx)//GetInstance<ConfigClient>()?.dripLord == true)
			{
				if (NPC.AnyNPCs(398))
				{
					music = GetSoundSlot(SoundType.Music, "Sounds/Music/zx2aSKAMasxoaLS7AK6d4axcoP23ASasYALsdmykayiiefo-drip");
					priority = (MusicPriority)999999;
				}
			}

			prevMusic = Main.curMusic;
		}

		List<string> modIgnoreResetHpCheck = new List<string>();
		public void UpdateAudioHook(ILContext il)
        {
            ILCursor c = new ILCursor(il);

			c.EmitDelegate<Action>(() => 
			{ 
				inBossRange = false; 
				modIgnoreResetHpCheck.Clear(); 
			});

			c.TryGotoNext(i => i.MatchCall("Microsoft.Xna.Framework.Rectangle", "Intersects"));
			c.Index++;

			c.EmitDelegate<Func<bool, bool>>((x) =>
			{
				inBossRange = x;
				return x;
			});

			c.TryGotoNext(i => i.MatchLdfld("Terraria.ModLoader.ModNPC", "musicPriority"));
			c.TryGotoNext(i => i.MatchCallvirt("Terraria.NPC", "get_modNPC"));
            c.TryGotoNext(i => i.MatchLdfld("Terraria.ModLoader.ModNPC", "music"));

            c.EmitDelegate<Func<ModNPC, ModNPC>>(npc =>
            {
				curModNpcMusic = -1;

				if (replaceModBossMusic)
                {
					if (modNpcToMusicInfo.TryGetValue(npc.mod, out var dic))
					{
						if (dic.TryGetValue(npc.Name, out var musicls))
						{
							var valid = musicls.Where(m => m.conditionsMet(npc.npc)).ToList();
							var validAndBelowThreshold = new List<NPCMusicConfig>();

							foreach (var music in valid)
							{
								if (music.hpBelow == 1 || music.healthBelowThreshold(npc.npc) == 1)
								{
									modIgnoreResetHpCheck.Add(npc.Name);
									music.hpBelow = 1;
									validAndBelowThreshold.Add(music);
								}
							}

							if (validAndBelowThreshold.Count == 1)
							{
								curModNpcMusic = GetSoundSlot(SoundType.Music, validAndBelowThreshold[0].music.path);
							}
							else if (validAndBelowThreshold.Count > 1)
							{
								float min = validAndBelowThreshold.Select(m => m.hp).Min();
								var minConfig = validAndBelowThreshold.Where(m => m.hp == min).First();
								curModNpcMusic = GetSoundSlot(SoundType.Music, minConfig.music.path);
							}
						}
					}
				}

				return npc;
            });

            c.Index++;

            c.EmitDelegate<Func<int, int>>((x) =>
            {
                if (curModNpcMusic == -1)
					return x;
                return curModNpcMusic;
            });

			c.TryGotoNext(i => i.MatchLdfld("Terraria.Main", "newMusic"));
			c.TryGotoNext(i => i.MatchStsfld("Terraria.Main", "curMusic"));

			c.EmitDelegate<Func<int, int>>((x) =>
			{
				int newID = x;

				if (replaceMusicID)
                {
					if (Main.gameMenu && forceReplaceTitle)
                    {
                        if (musicIdToMusicInfo.TryGetValue(6, out var minfo))
                        {
							var slot = GetSoundSlot(SoundType.Music, minfo[0].music.path);
							newID = slot;
						}
                    }
                    else if (musicIdToMusicInfo.TryGetValue(x, out var musicls))
                    {
                        var valid = musicls.Where(m => m.conditionsMet()).ToList();

                        int i; bool breakNext = false;
                        for (i = 0; i < valid.Count; i++)
                        {
                            if (breakNext) break;

                            if (valid[i].checkedRoll != -1)
                            {
                                if (valid[i].checkedRoll == 0)
                                {
                                    bool success = valid[i].randomSuccess();
                                    valid[i].checkedRoll = success ? 1 : -1;
                                }

                                if (valid[i].checkedRoll == 1)
                                {
                                    newID = GetSoundSlot(SoundType.Music, valid[i].music.path);
                                    valid[i].checkedRoll = 1;
                                    breakNext = true;
                                }
                            }
                        }

                        foreach (var item in musicls.Except(valid.GetRange(0, i)))
                            item.checkedRoll = 0;

                        foreach (var key in musicIdToMusicInfo.Keys)
                        {
                            if (key == x)
                                continue;
                            foreach (var item in musicIdToMusicInfo[key])
                                item.checkedRoll = 0;
                        }
                    }
                    else
                    {
                        foreach (var key in musicIdToMusicInfo.Keys)
                        {
                            foreach (var item in musicIdToMusicInfo[key])
                                item.checkedRoll = 0;
                        }
                    }
                }

                if (replaceModBossMusic)
                {
                    foreach (var modkey in modNpcToMusicInfo.Keys)
                    {
                        foreach (var key in modNpcToMusicInfo[modkey].Keys.Except(modIgnoreResetHpCheck))
                        {
                            foreach (var music in modNpcToMusicInfo[modkey][key])
                                music.hpBelow = 0;
                        }
                    }
                }

                return newID;
			});
        }

		private bool npcIsClose(NPC npc)
        { //TODO: Bad
			return (inBossRange && npc.boss) 
				|| (!npc.boss && (npc.position - Main.LocalPlayer.position).Length() < 2000);
		}

		private void reloadAddMusic()
        {
			GetInstance<ConfigClient>().doReload = false;
			ModLoad.ModLoader.saveConfig(GetInstance<ConfigClient>());

			var oldFiles = new HashSet<string>(Config.Instance.addedFiles);

			if (!ReplacerSetup.devModeReady())
				ReplacerSetup.setupDevMode();

			MusicManager.addFiles();

			bool cont = true;

			try {
				ModLoad.ModLoader.buildMod(PATH.EXTRACT_DIR);
			}
			catch (Exception ex)
            {
				cont = false;
				Config.Instance.addedFiles = oldFiles;
				Config.Instance.Save(PATH.CONFIG_FILE);
				string msg = "Adding files failed. Check the README in Your Music.\n\n" + ex.Message + "\n" + ex.StackTrace + 
					"\n\n" + ex.GetBaseException().Message + "\n" + ex.GetBaseException().StackTrace;
				BetterMusic.message = new ModLoad.ModLoader.MessageOptions(msg, 888,
						ModLoad.ModLoader.getModLoaderState("modsMenu"),
						altButtonText: "README",
						altButtonAction: (() => {
							Process.Start(Path.Combine(PATH.YOURMUSIC_DIR, "# README #.txt"));
						}));
				throw new Exception(msg);
            }

			if (cont)
            {
				Config.Instance.doSecondReload = true;
				Config.Instance.Save(PATH.CONFIG_FILE);
				closeMusic();
				ModLoad.ModLoader.reloadMods();
			}
		}

		public static string getCurrentMusicName()
        {
			string s = null;
			if (sounds != null && sounds[SoundType.Music] != null)
				s = sounds[SoundType.Music].FirstOrDefault(x => x.Value == Main.curMusic).Key;
			return s;

			//if (modSounds != null && modSounds[SoundType.Music] != null)
			//	if (modSounds[SoundType.Music].ContainsKey(Main.curMusic))
			//		return modSounds[SoundType.Music][Main.curMusic].sound.Name;

			//return null;
		}

		private void closeMusic()
        {
			if (Directory.Exists(PATH.MUSIC_EXTRACT_DIR))
			{
				DirectoryInfo dir = new DirectoryInfo(PATH.MUSIC_EXTRACT_DIR);
				foreach (FileInfo file in dir.GetFiles())
				{
					try
					{
						var slot = GetSoundSlot(SoundType.Music, "Sounds/Music/" + file.NameNoExt());
						if (Main.music.IndexInRange(slot) && Main.music[slot]?.IsPlaying == true)
							Main.music[slot].Stop(Microsoft.Xna.Framework.Audio.AudioStopOptions.Immediate);
					}
					catch { }
				}
			}

			foreach (var name in ReplacerSetup.defaultMusic)
			{
				try
				{
					var slot = GetSoundSlot(SoundType.Music, "Sounds/Music/" + name.NameNoExt());
					if (Main.music.IndexInRange(slot) && Main.music[slot]?.IsPlaying == true)
						Main.music[slot].Stop(Microsoft.Xna.Framework.Audio.AudioStopOptions.Immediate);
				}
				catch { }
			}

            foreach (var name in Config.Instance.addedFiles)
            {
				try
				{
					var slot = GetSoundSlot(SoundType.Music, "Sounds/Music/" + name.NameNoExt());
					if (Main.music.IndexInRange(slot) && Main.music[slot]?.IsPlaying == true)
						Main.music[slot].Stop(Microsoft.Xna.Framework.Audio.AudioStopOptions.Immediate);
				}
				catch { }
			}

			foreach (var kvp in musicIdToMusicInfo)
			{
                foreach (var item in kvp.Value)
                {
					try
					{
						var slot = GetSoundSlot(SoundType.Music, "Sounds/Music/" + item.fileName.NameNoExt());
						if (Main.music.IndexInRange(slot) && Main.music[slot]?.IsPlaying == true)
							Main.music[slot].Stop(Microsoft.Xna.Framework.Audio.AudioStopOptions.Immediate);
					}
					catch { }
				}	
			}

			foreach (var kvp in npcToMusicInfo)
			{
				foreach (var item in kvp.Value)
				{
					try
					{
						var slot = GetSoundSlot(SoundType.Music, "Sounds/Music/" + item.fileName.NameNoExt());
						if (Main.music.IndexInRange(slot) && Main.music[slot]?.IsPlaying == true)
							Main.music[slot].Stop(Microsoft.Xna.Framework.Audio.AudioStopOptions.Immediate);
					}
					catch { }
				}
			}

			foreach (var dic in modNpcToMusicInfo.Values)
			{
				foreach (var kvp in dic)
				{
                    foreach (var item in kvp.Value)
                    {
						try
						{
							var slot = GetSoundSlot(SoundType.Music, "Sounds/Music/" + item.fileName.NameNoExt());
							if (Main.music.IndexInRange(slot) && Main.music[slot]?.IsPlaying == true)
								Main.music[slot].Stop(Microsoft.Xna.Framework.Audio.AudioStopOptions.Immediate);
						}
						catch { }
					}
				}
			}

			string s = getCurrentMusicName();
			if (!String.IsNullOrEmpty(s) && s.Contains("BetterMusic"))
            {
				try {
					Main.music[Main.curMusic].Stop(Microsoft.Xna.Framework.Audio.AudioStopOptions.Immediate);
				}
				catch { }
			}
		}
	}


	internal static class MusicManager
	{
		public static readonly string[] allowedFileTypes
			= new string[] { ".mp3", ".wav", ".ogg" };

		public static void addFiles()
		{
			Config.Instance.addedFiles.Clear();

			DirectoryInfo dir = new DirectoryInfo(PATH.YOURMUSIC_DIR);

            foreach (var file in dir.GetFiles())
            {
				if (!allowedFileTypes.Contains(file.Extension))
					continue;

				File.Copy(file.FullName, Path.Combine(PATH.EXTRACT_DIR, "Sounds", "Music", file.Name), true);
				Config.Instance.addedFiles.Add(file.Name);
			}

			Utils.log("[ADDED FILES]");
		}

		public static void prepareCustomSongs(bool vanillaOnly = false)
		{
			updateAddedFileList();

			BetterMusic.replaceMusicID = false;
			BetterMusic.replaceModBossMusic = false;
			BetterMusic.replaceVanillaOrNonBossMusic = false;

			BetterMusic.npcToMusicInfo.Clear();
			BetterMusic.modNpcToMusicInfo.Clear();
			BetterMusic.musicIdToMusicInfo.Clear();

			var envMusicNames = new HashSet<string>();
			var modMusicNames = new HashSet<string>();

			var removeNpcOpts = new List<NPCDefinition>();
			var removeMIdOpts = new List<MusicIDSelect>();
			var removeModOpts = new List<MusicName>();

            foreach (var def in GetInstance<ConfigClient>().npcOpts)
            {
				bool added = false;
                if (def.Key.mod == "Terraria")
                    added = prepareNpcMusic(def);
                else if (!vanillaOnly)
                    added = prepareModNpcMusic(def);
				if (!added)
					removeNpcOpts.Add(def.Key);
            }

			foreach (var def in GetInstance<ConfigClient>().idOpts)
            {
				if (def.Key.Type != -1)
				{
					if (envMusicNames.Contains(def.Key.name))
					{
						if (Main.gameMenu)
							BetterMusic.message = new ModLoad.ModLoader.MessageOptions(
								$"Music Replacer: The environment music list already contains an entry for \"{def.Key.name}\", add your music there.",
								888, ModLoad.ModLoader.getModLoaderState("modsMenu"));
					}
					else if (prepareIdMusic(def))
					{
						envMusicNames.Add(def.Key.name);
						continue;
					}
				}
				removeMIdOpts.Add(def.Key);
			}

			foreach (var def in GetInstance<ConfigClient>().modMusicOpts)
			{
				if (!String.IsNullOrEmpty(def.Key.name))
                {
					if (modMusicNames.Contains(def.Key.name))
                    {
						if (Main.gameMenu)
							BetterMusic.message = new ModLoad.ModLoader.MessageOptions(
								$"Music Replacer: The mod music list already contains an entry for \"{def.Key.displayName}\", add your music there.", 
								888, ModLoad.ModLoader.getModLoaderState("modsMenu"));
                    }
					else if (prepareModIdMusic(def))
					{
						modMusicNames.Add(def.Key.name);
						continue;
					}
				}
				removeModOpts.Add(def.Key);
			}

			foreach (var key in removeNpcOpts)
			{
				if (GetInstance<ConfigClient>().npcOpts.ContainsKey(key))
					GetInstance<ConfigClient>().npcOpts.Remove(key);
			}

			foreach (var key in removeMIdOpts)
			{
				if (GetInstance<ConfigClient>().idOpts.ContainsKey(key))
					GetInstance<ConfigClient>().idOpts.Remove(key);
			}

			foreach (var key in removeModOpts)
			{
				if (GetInstance<ConfigClient>().modMusicOpts.ContainsKey(key))
					GetInstance<ConfigClient>().modMusicOpts.Remove(key);
			}
			
			if (removeNpcOpts.Count > 0 || removeMIdOpts.Count > 0 || removeModOpts.Count > 0)
				ModLoad.ModLoader.saveConfig(GetInstance<ConfigClient>());

			string txt = $"[SONGS PREPARED]";
			if (vanillaOnly) txt += " (vanilla only)";
			Utils.log(txt);
		}

		private static bool prepareNpcMusic(KeyValuePair<NPCDefinition, NPCMusicConfigs> kvp)
		{
			if (kvp.Value == null || kvp.Value.ls == null)
				return false;

			int count = 0;
            foreach (var item in kvp.Value.ls)
            {
				if (item == null)
					continue;
				count++;
				if (!Config.Instance.addedFiles.Contains(item.fileName))
					continue;

				var file = new FileInfo(Path.Combine(PATH.YOURMUSIC_DIR, item.fileName));

				item.music.path = "Sounds/Music/" + item.fileName.NameNoExt();
				if ((int)item.priority < 1)
					item.music.priority = getVanillaPriority(kvp.Key.name);
				else
					item.music.priority = (int)item.priority;

				if (!BetterMusic.npcToMusicInfo.ContainsKey(kvp.Key.Type))
					BetterMusic.npcToMusicInfo.Add(kvp.Key.Type, new List<NPCMusicConfig>());
				if (BetterMusic.npcToMusicInfo[kvp.Key.Type] == null)
					BetterMusic.npcToMusicInfo[kvp.Key.Type] = new List<NPCMusicConfig>();

				BetterMusic.npcToMusicInfo[kvp.Key.Type].Add(item);
				BetterMusic.replaceVanillaOrNonBossMusic = true;
			}

			return count > 0;
		}

		private static bool prepareModNpcMusic(KeyValuePair<NPCDefinition, NPCMusicConfigs> kvp)
		{
			Mod mod = ModLoader.GetMod(kvp.Key.mod);
			if (mod == null)
				return true;

			int count = 0;
			foreach (var item in kvp.Value.ls)
            {
				if (item == null)
					continue;
				count++;
				if (!Config.Instance.addedFiles.Contains(item.fileName))
					continue;

				var file = new FileInfo(Path.Combine(PATH.YOURMUSIC_DIR, item.fileName));

				item.music.path = "Sounds/Music/" + item.fileName.NameNoExt();
				item.music.priority = (int)item.priority;

				if (mod.GetNPC(kvp.Key.name).music < 0)
				{
					if (item.music.priority < 1)
						item.music.priority = (int)MusicPriorityOption.BossLow;

					if (!BetterMusic.npcToMusicInfo.ContainsKey(kvp.Key.Type))
						BetterMusic.npcToMusicInfo.Add(kvp.Key.Type, new List<NPCMusicConfig>());
					if (BetterMusic.npcToMusicInfo[kvp.Key.Type] == null)
						BetterMusic.npcToMusicInfo[kvp.Key.Type] = new List<NPCMusicConfig>();

					BetterMusic.npcToMusicInfo[kvp.Key.Type].Add(item);
				}
				else
				{
					if (!BetterMusic.modNpcToMusicInfo.ContainsKey(mod))
						BetterMusic.modNpcToMusicInfo.Add(mod, new Dictionary<string, List<NPCMusicConfig>>());
					if (!BetterMusic.modNpcToMusicInfo[mod].ContainsKey(kvp.Key.name))
						BetterMusic.modNpcToMusicInfo[mod].Add(kvp.Key.name, new List<NPCMusicConfig>());
					if (BetterMusic.modNpcToMusicInfo[mod][kvp.Key.name] == null)
						BetterMusic.modNpcToMusicInfo[mod][kvp.Key.name] = new List<NPCMusicConfig>();

					BetterMusic.modNpcToMusicInfo[mod][kvp.Key.name].Add(item);
				}

				BetterMusic.replaceModBossMusic = true;
			}

			return count > 0;
		}

		private static bool prepareIdMusic(KeyValuePair<MusicIDSelect, MusicIDConfigs> kvp)
		{
			if (kvp.Value == null || kvp.Value.ls == null)
				return false;

			int count = 0;
            foreach (var item in kvp.Value.ls)
            {
				if (item == null)
					continue;
				count++;
				if (!Config.Instance.addedFiles.Contains(item.fileName))
					continue;

				var file = new FileInfo(Path.Combine(PATH.YOURMUSIC_DIR, item.fileName));

				item.music.path = "Sounds/Music/" + item.fileName.NameNoExt();
				item.music.priority = -1;

				if (!BetterMusic.musicIdToMusicInfo.ContainsKey(kvp.Key.Type))
					BetterMusic.musicIdToMusicInfo.Add(kvp.Key.Type, new List<MusicIDConfig>());
				if (BetterMusic.musicIdToMusicInfo[kvp.Key.Type] == null)
					BetterMusic.musicIdToMusicInfo[kvp.Key.Type] = new List<MusicIDConfig>();

				BetterMusic.musicIdToMusicInfo[kvp.Key.Type].Add(item);
				BetterMusic.replaceMusicID = true;
			}

			return count > 0;
		}

		private static bool prepareModIdMusic(KeyValuePair<MusicName, MusicIDConfigs> kvp)
		{
			if (kvp.Value == null || kvp.Value.ls == null)
				return false;
			if (ModLoader.GetMod(kvp.Key.name.Substring(0, kvp.Key.name.IndexOf('/'))) == null)
				return true;

			int count = 0;
			foreach (var item in kvp.Value.ls)
			{
				if (item == null)
					continue;
				count++;
				if (!Config.Instance.addedFiles.Contains(item.fileName))
					continue;

				var file = new FileInfo(Path.Combine(PATH.YOURMUSIC_DIR, item.fileName));

				item.music.path = "Sounds/Music/" + item.fileName.NameNoExt();
				item.music.priority = -1;

				int mId = SoundLoader.GetSoundSlot(SoundType.Music, kvp.Key.name);
				if (!BetterMusic.musicIdToMusicInfo.ContainsKey(mId))
					BetterMusic.musicIdToMusicInfo.Add(mId, new List<MusicIDConfig>());
				if (BetterMusic.musicIdToMusicInfo[mId] == null)
					BetterMusic.musicIdToMusicInfo[mId] = new List<MusicIDConfig>();

				BetterMusic.musicIdToMusicInfo[mId].Add(item);
				BetterMusic.replaceMusicID = true;
			}

			return count > 0;
		}

		private static int getVanillaPriority(string name)
		{
			int priority;
			name = name.ToLower();
			switch (name)
			{
				case "moonlordcore":
					priority = 8;
					break;

				case "plantera":
				case "lunartowersolar":
				case "lunartowernebula":
				case "lunartowerstardust":
				case "lunartowervortex":
				case "lunarpillar":
				case "lunarpillars":
				case "celestialpillar":
				case "celestialpillars":
				case "martiansaucercore":
				case "martiansaucer":
					priority = 7;
					break;

				default:
					priority = 6;
					break;
			}
			return priority;
		}

		public static bool updateAddedFileList(bool message = true)
		{
			int count = 0;

			Config.Instance.addedFiles.RemoveWhere(s => {
				if (!BetterMusic.sounds[SoundType.Music].ContainsKey("BetterMusic/Sounds/Music/" + s.NameNoExt())) {
					if (File.Exists(Path.Combine(PATH.YOURMUSIC_DIR, s)))
						count++;
					return true;
				}
				return false;
			});
			if (count > 0 && message)
            {
				BetterMusic.message = new ModLoad.ModLoader.MessageOptions("Music Replacer: One or more files has not been added. Add now?" +
					"\nNOTE: TmodLoader should reload when you click Add. Check the README in Your Music if it doesn't.",
					altButtonText: "Add",
					altButtonAction: (() => { GetInstance<ConfigClient>().doReload = true; }));
            }
			return count > 0;
		}
	}


	internal static class ReplacerSetup
    {
		public static readonly string[] defaultMusic = new string[] {
			"Alt Rain.ogg",
			"Blizzard.ogg",
			"The Destroyer.ogg",
			"Duke Fishron.ogg",
			"Eye of Cthulhu.ogg",
			"King Slime.ogg",
			"Lunatic Cultist.ogg",
			"Ocean Night.ogg",
			"Plantera P2.ogg",
			"Queen Bee.ogg",
			"Skeletron Prime.ogg",
			"Jungle Night.ogg",
			"Space Night.ogg",
			"Title.ogg",
			"The Twins.ogg",
			"Ug Desert.ogg",
			"Ug Hallow.ogg",
			"Ug Jungle.ogg",
			"Wall of Flesh.ogg",
			"Plantera P1.ogg"
		};
		 
		public static void setupReplacer()
		{
			Directory.CreateDirectory(PATH.EXTRACT_DIR);
			Directory.CreateDirectory(PATH.MODCOMPILE_DIR);
			Directory.CreateDirectory(PATH.YOURMUSIC_DIR);

			bool devMode = (bool)ModLoad.ModLoader.ModCompile
				.GetProperty("DeveloperMode", BindingFlags.Static | BindingFlags.Public).GetValue(null);

			if (!devModeReady())
				setupDevMode();

			if (Terraria.Utilities.PlatformUtilities.IsXNA == File.Exists(Path.Combine(PATH.MODCOMPILE_DIR, "tModLoader.XNA.exe")))
			{
				Utils.ForceEmptyInsanoMode(PATH.MODCOMPILE_DIR, filesOnly: true);
				setupModCompile();
			}

			ModLoad.ModLoader.extractMod(BetterMusic.betterMusic);

			using (StreamWriter sw = File.CreateText(Path.Combine(PATH.YOURMUSIC_DIR, "# README #.txt")))
				sw.WriteLine(File.ReadAllText(Path.Combine(PATH.EXTRACT_DIR, "# README #.txt")));

			if (!Config.Instance.doneFirstTimeSetup)
				firstTimeSetup();

			DirectoryInfo dir = new DirectoryInfo(PATH.MUSIC_EXTRACT_DIR);
			try { 
				Utils.ForceEmptyInsanoMode(dir.FullName, new string[] { "zx2aSKAMasxoaLS7AK6d4axcoP23ASasYALsdmykayiiefo-drip.ogg" }); }
            catch { }

			if (!devMode)
			{ 
				// TODO: How to remove Mod Sources button if user hasn't activated dev mode?
			}

            Utils.log("[REPLACER SETUP]");
		}

		public static void firstTimeSetup()
        {
			try
			{
				foreach (var name in defaultMusic)
				{
					File.Copy(Path.Combine(PATH.MUSIC_EXTRACT_DIR, name),
							Path.Combine(PATH.YOURMUSIC_DIR, name), true);
				}
			}
			catch
			{
				throw new WarningException("Default music files are missing, first time setup cannot complete.\n" +
					"Redownload Better Music from the mod browser.");
			}			

			Config.Instance.doneFirstTimeSetup = true;
			Config.Instance.Save(PATH.CONFIG_FILE);

			Utils.log("[FIRST TIME SETUP]");
		}

		public static void setDefaults()
        {
			GetInstance<ConfigClient>().idOpts.Clear();
			GetInstance<ConfigClient>().npcOpts.Clear();

			Config.Instance.addedFiles.Clear();
			Config.Instance.addedFiles.UnionWith(defaultMusic);

			foreach (var name in defaultMusic)
				defaultOption(name);

			Config.Instance.Save(PATH.CONFIG_FILE);
			ModLoad.ModLoader.saveConfig(GetInstance<ConfigClient>());

			Utils.log("[DEFAULTS SET]");
		}

		private static void defaultOption(string name)
        {
			if (name == defaultMusic[0])
			{
				var key = MusicIDSelect.FromString("Rain");
				var val = new MusicIDConfig();
				val.time = WorldTime.Day;
				val.probability = 50;
				val.fileName = name;
				GetInstance<ConfigClient>().idOpts.Add(key, new MusicIDConfigs());
				GetInstance<ConfigClient>().idOpts[key].Add(val);
			} // rain alt
			else if (name == defaultMusic[1])
			{
				var key = MusicIDSelect.FromString("Snow");
				var val = new MusicIDConfig();
				val.rain = RainOption.Rain;
				val.fileName= name;
				GetInstance<ConfigClient>().idOpts.Add(key, new MusicIDConfigs());
				GetInstance<ConfigClient>().idOpts[key].Add(val);
			} // blizzard
			else if (name == defaultMusic[2])
			{
				var key = NPCDefinition.FromString("Terraria TheDestroyer");
				var val = new NPCMusicConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key].Add(val);
			} // destroyer
			else if (name == defaultMusic[3])
			{
				var key = NPCDefinition.FromString("Terraria DukeFishron");
				var val = new NPCMusicConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key].Add(val);
			} // duke
			else if (name == defaultMusic[4])
			{
				var key = NPCDefinition.FromString("Terraria EyeofCthulhu");
				var val = new NPCMusicConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key].Add(val);
			} // eye
			else if (name == defaultMusic[5])
			{
				var key = NPCDefinition.FromString("Terraria KingSlime");
				var val = new NPCMusicConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key].Add(val);
			} // king slime
			else if (name == defaultMusic[6])
			{
				var key = NPCDefinition.FromString("Terraria CultistBoss");
				var val = new NPCMusicConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key].Add(val);
			} // cultist
			else if (name == defaultMusic[7])
			{
				var key = MusicIDSelect.FromString("Ocean");
				var val = new MusicIDConfig();
				val.time = WorldTime.Night;
				val.fileName = name;
				GetInstance<ConfigClient>().idOpts.Add(key, new MusicIDConfigs());
				GetInstance<ConfigClient>().idOpts[key].Add(val);
			} // ocean night
			else if (name == defaultMusic[8])
			{
				var key = NPCDefinition.FromString("Terraria Plantera");
				var val = new NPCMusicConfig();
				val.hp = 50;
				val.fileName = name;

				var keys = GetInstance<ConfigClient>().npcOpts.Keys.Where(f => f.Type == 262);
				if (keys.Count() > 0)
					GetInstance<ConfigClient>().npcOpts[keys.First()].Add(val);
				else
				{
					GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
					GetInstance<ConfigClient>().npcOpts[key].Add(val);
				}
			} // plant p2
			else if (name == defaultMusic[9])
			{
				var key = NPCDefinition.FromString("Terraria QueenBee");
				var val = new NPCMusicConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key].Add(val);
			} // queen bee
			else if (name == defaultMusic[10])
			{
				var key = NPCDefinition.FromString("Terraria SkeletronPrime");
				var val = new NPCMusicConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key].Add(val);
			} // skelly prime
			else if (name == defaultMusic[11])
			{
				var key = MusicIDSelect.FromString("Jungle");
				var val = new MusicIDConfig();
				val.height = PlayerHeight.Overground;
				val.time = WorldTime.Night;
				val.fileName = name;
				
				var keys = GetInstance<ConfigClient>().idOpts.Keys.Where(f => f.name == "Jungle");
				if (keys.Count() > 0)
					GetInstance<ConfigClient>().idOpts[keys.First()].Add(val);
				else
                {
					GetInstance<ConfigClient>().idOpts.Add(key, new MusicIDConfigs());
					GetInstance<ConfigClient>().idOpts[key].Add(val);
				}
			} // jungle night
			else if (name == defaultMusic[12])
			{
				var key = MusicIDSelect.FromString("Space");
				var val = new MusicIDConfig();
				val.time = WorldTime.Night;
				val.fileName = name;
				GetInstance<ConfigClient>().idOpts.Add(key, new MusicIDConfigs());
				GetInstance<ConfigClient>().idOpts[key].Add(val);
			} // space night
			else if (name == defaultMusic[13])
			{
				var key = MusicIDSelect.FromString("Title");
				var val = new MusicIDConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().idOpts.Add(key, new MusicIDConfigs());
				GetInstance<ConfigClient>().idOpts[key].Add(val);
			} // title
			else if (name == defaultMusic[14])
			{
				var key = NPCDefinition.FromString("Terraria Retinazer");
				var val = new NPCMusicConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key].Add(val);
				var key2 = NPCDefinition.FromString("Terraria Spazmatism");
				var val2 = new NPCMusicConfig();
				val2.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key2, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key2].Add(val2);
			} // twins
			else if (name == defaultMusic[15])
			{
				var key = MusicIDSelect.FromString("Desert");
				var val = new MusicIDConfig();
				val.height = PlayerHeight.Underground;
				val.fileName = name;
				GetInstance<ConfigClient>().idOpts.Add(key, new MusicIDConfigs());
				GetInstance<ConfigClient>().idOpts[key].Add(val);
			} // ug desert
			else if (name == defaultMusic[16])
			{
				var key = MusicIDSelect.FromString("UndergroundHallow");
				var val = new MusicIDConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().idOpts.Add(key, new MusicIDConfigs());
				GetInstance<ConfigClient>().idOpts[key].Add(val);
			} // ug hallow
			else if (name == defaultMusic[17])
			{
				var key = MusicIDSelect.FromString("Jungle");
				var val = new MusicIDConfig();
				val.height = PlayerHeight.Underground;
				val.fileName = name;

				var keys = GetInstance<ConfigClient>().idOpts.Keys.Where(f => f.name == "Jungle");
				if (keys.Count() > 0)
					GetInstance<ConfigClient>().idOpts[keys.First()].Add(val);
				else
				{
					GetInstance<ConfigClient>().idOpts.Add(key, new MusicIDConfigs());
					GetInstance<ConfigClient>().idOpts[key].Add(val);
				}
			} // ug jungle
			else if (name == defaultMusic[18])
			{
				var key = NPCDefinition.FromString("Terraria WallofFlesh");
				var val = new NPCMusicConfig();
				val.fileName = name;
				GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
				GetInstance<ConfigClient>().npcOpts[key].Add(val);
			} // wall
			else if (name == defaultMusic[19])
			{
				var key = NPCDefinition.FromString("Terraria Plantera");
				var val = new NPCMusicConfig();
				val.hp = 100;
				val.fileName = name;

				var keys = GetInstance<ConfigClient>().npcOpts.Keys.Where(f => f.Type == 262);
				if (keys.Count() > 0)
					GetInstance<ConfigClient>().npcOpts[keys.First()].Add(val);
				else
				{
					GetInstance<ConfigClient>().npcOpts.Add(key, new NPCMusicConfigs());
					GetInstance<ConfigClient>().npcOpts[key].Add(val);
				}
			} // plant p1
		}

		public static bool devModeReady()
        {
			MethodInfo devReadyCheck = ModLoad.ModLoader.ModCompile
				.GetMethod("DeveloperModeReady", BindingFlags.NonPublic | BindingFlags.Static);
			return (bool)devReadyCheck.Invoke(ModLoad.ModLoader.ModCompile, new object[] { "" });
		}

		public static void setupDevMode()
        {
			object[] parameters = new object[] { "" };

			MethodInfo assCheck = ModLoad.ModLoader.ModCompile
				.GetMethod("ReferenceAssembliesCheck", BindingFlags.NonPublic | BindingFlags.Static);
			bool refAssemblies = (bool)assCheck.Invoke(ModLoad.ModLoader.ModCompile, parameters);

			MethodInfo modCompileVersionCheck = ModLoad.ModLoader.ModCompile
					.GetMethod("ModCompileVersionCheck", BindingFlags.NonPublic | BindingFlags.Static);
			bool modCompileVersion = (bool)modCompileVersionCheck.Invoke(ModLoad.ModLoader.ModCompile, parameters);

			try
            {
				if (!refAssemblies)
					setupReferenceAssemblies();
				if (!modCompileVersion)
					setupModCompile();
			}
			catch (Exception ex)
            {
				if (ex is System.Net.WebException)
				{
					string error = "Automatic install failed:";
					error += "\nDevice doesn't seem to be connected to the internet. Connect and try again.";
					throw new WarningException(error);
					//Why the hell does tml sometimes randomly decide to silently catch exceptions instead of just showing them???
					Main.MenuUI.SetState(ModLoad.ModLoader.getModLoaderState("errorMessage"));
				}
				else throw ex;
			}

			if (!devModeReady())
			{
				string error = "Automatic install failed\n";

				if (!Utils.checkInternet())
				{
					error += "\nDevice doesn't seem to be connected to the internet. Connect and try again." +
						"\n(An internet connection is needed to download ModCompile and .NET 4.5 reference assemblies to build the mod.)";
					throw new WarningException(error);
					Main.MenuUI.SetState(ModLoad.ModLoader.getModLoaderState("errorMessage"));
				}
				else
                {
					error += "Music Replacer is not working. Select \"Fix\" and follow the requirements there.";
					BetterMusic.message = new ModLoad.ModLoader.MessageOptions(error, 888, 
						ModLoad.ModLoader.getModLoaderState("modsMenu"), 
						altButtonText: "Fix", 
						altButtonAction: (() => {
							Main.MenuUI.SetState(ModLoad.ModLoader.getModLoaderState("developerModeHelp"));
                        }));
				}
			}
		}

		private static void setupReferenceAssemblies()
		{
			if (!Directory.Exists(PATH.MODCOMPILE_DIR))
				Directory.CreateDirectory(PATH.MODCOMPILE_DIR);

			ServicePointManager.Expect100Continue = true;
			ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;
			string url = "https://github.com/fiso64/BetterMusic/raw/master/v4.5%20Reference%20Assemblies.zip";
			using (WebClient client = new WebClient())
				client.DownloadFile(url, Path.Combine(PATH.MODCOMPILE_DIR, "v4.5 Reference Assemblies.zip"));

			System.IO.Compression.ZipFile.ExtractToDirectory(Path.Combine(PATH.MODCOMPILE_DIR, "v4.5 Reference Assemblies.zip"), PATH.MODCOMPILE_DIR);

			MethodInfo assCheck = ModLoad.ModLoader.ModCompile.GetMethod("ReferenceAssembliesCheck", BindingFlags.NonPublic | BindingFlags.Static);
			object[] parameters = new object[] { "" };

            int count = 0;
            while (!(bool)assCheck.Invoke(ModLoad.ModLoader.ModCompile, parameters) && count++ < 60)
            {
                System.Threading.Thread.Sleep(500);
            }

            if (File.Exists(Path.Combine(PATH.MODCOMPILE_DIR, "v4.5 Reference Assemblies.zip")))
				File.Delete(Path.Combine(PATH.MODCOMPILE_DIR, "v4.5 Reference Assemblies.zip"));
		}

		private static void setupModCompile()
		{
			Type devModeHelp = ModLoad.ModLoader.TerrariaAssembly.GetType("Terraria.ModLoader.UI.UIDeveloperModeHelp");
			object devModeHelpInstance = Activator.CreateInstance(devModeHelp);
			MethodInfo downloadModCompile = devModeHelpInstance.GetType()
				.GetMethod("DownloadModCompile", BindingFlags.NonPublic | BindingFlags.Instance);
			downloadModCompile.Invoke(devModeHelpInstance, null);

			MethodInfo modCompileVersionCheck = ModLoad.ModLoader.ModCompile
				.GetMethod("ModCompileVersionCheck", BindingFlags.NonPublic | BindingFlags.Static);
			object[] parameters = new object[] { "" };

            //TODO use timers instead, add progress bar (all of this is not necessary but I don't care)
            int count = 0;
            while (!(bool)modCompileVersionCheck.Invoke(ModLoad.ModLoader.ModCompile, parameters) && count++ < 60)
            {
                System.Threading.Thread.Sleep(500);
            }

            devModeHelpInstance = null;
		}
	}


	internal static class Utils
    {
		public static void log(params object[] stuff)
        {
			if (BetterMusic.betterMusic == null)
				return;
			if (String.IsNullOrEmpty(BetterMusic.betterMusic.logPath))
				return;
			if (!Directory.Exists(Directory.GetParent(BetterMusic.betterMusic.logPath).FullName))
				return;

			string logtxt = $"[{DateTime.Now.ToString("HH:mm:ss")}] ";
			int count = 0;
			foreach (var o in stuff)
            {
				if (count > 0)
					logtxt += "\n";
				if (o is string)
					logtxt += o;
				else if (o is System.Collections.IEnumerable)
                {
					var ls = o as System.Collections.IEnumerable;
					foreach (var oo in ls)
						logtxt += $"{oo}, ";
                }
				else
                {
					logtxt += o.ToString();
				}

				count++;
            }
			if (!File.Exists(BetterMusic.betterMusic.logPath)) 
				File.Create(BetterMusic.betterMusic.logPath);
			File.AppendAllText(BetterMusic.betterMusic.logPath, logtxt + "\n");
        }

		public static string stringify<T, V>(this Dictionary<T, V> dic, bool newLines = false)
        {
			var lines = dic.Select(kvp => kvp.Key.ToString() + ": " + kvp.Value.ToString());
			if (newLines)
				return String.Join("\n", lines);
			else
				return String.Join(", ", lines);
		}

		public static void extractNpcNames()
		{
			bool notepad = false;
			var values = Environment.GetEnvironmentVariable("PATH");
			foreach (var path in values.Split(Path.PathSeparator))
				if (File.Exists(Path.Combine(path, "notepad.exe"))) { notepad = true; break; }

			string txt;
			if (notepad)
				txt = "DISPLAY NAME".PadRight(55) + " REAL NAME\n\n";
			else
				txt = "DISPLAY NAME, REAL NAME\n\n";

			foreach (var mod in ModLoader.Mods)
			{
				if (mod.Name == "BetterMusic" || mod.Name == "ModLoader")
					continue;

				var npcs = (Dictionary<string, ModNPC>)typeof(Mod)
					.GetField("npcs", BindingFlags.NonPublic | BindingFlags.Instance)
					.GetValue(mod);

				if (notepad)
				{
					txt += $"### {mod.DisplayName}".PadRight(55) + $" {mod.Name}\n";
					foreach (ModNPC npc in npcs.Values)
						txt += $"{npc.DisplayName.GetDefault()}".PadRight(55) + $" {npc.Name}\n";
					txt += $"\n\n";
				}
				else
				{
					txt += $"### {mod.DisplayName}, {mod.Name}\n";
					foreach (ModNPC npc in npcs.Values)
						txt += $"{npc.DisplayName.GetDefault()}, {npc.Name}\n";
					txt += $"\n\n";
				}
			}
			File.WriteAllText(Path.Combine(PATH.YOURMUSIC_DIR, "# Extracted Names #.txt"), txt);

			if (notepad)
				Process.Start("notepad.exe", Path.Combine(PATH.YOURMUSIC_DIR, "# Extracted Names #.txt"));
			else
				Process.Start(Path.Combine(PATH.YOURMUSIC_DIR, "# Extracted Names #.txt"));
		}

		public static void show(string text)
		{
			string path = Path.Combine(Main.SavePath, "zMOD_TEST_DELETE_" + DateTime.Now.Ticks + ".txt");
			using (StreamWriter sw = File.CreateText(path))
				sw.WriteLine(text);
			System.Diagnostics.Process.Start(path);
		}

		public static void showMessage(string text)
		{
			string path = Path.Combine(PATH.EXTRACT_DIR, "IMPORTANT.txt");
			using (StreamWriter sw = File.CreateText(path))
				sw.WriteLine(text);
			System.Diagnostics.Process.Start(path);
		}

		public static void showOpCodes(string typeName, string methodName)
		{
			string path = ModLoad.ModLoader.TerrariaAssembly.Location;
			if (typeName == "BetterMusic")
				path = @"D:\Users\sokfi\Documents\My Games\Terraria\ModLoader\Mod Specific Data\BetterMusic\BetterMusic.XNA.dll";
			var assembly = AssemblyDefinition.ReadAssembly(path);
			var rp = new ReaderParameters();
			rp.ReadingMode = ReadingMode.Immediate;
			rp.ReadWrite = true;
			rp.InMemory = true;

			var toInspect = assembly.MainModule
			  .GetTypes()
			  .SelectMany(t => t.Methods
				  .Where(m => m.HasBody)
				  .Select(m => new { t, m }));
			toInspect = toInspect.Where(x => x.t.Name.EndsWith(typeName) && x.m.Name == methodName);

			string opcodesStr = "";
			foreach (var method in toInspect)
				foreach (var instruction in method.m.Body.Instructions)
					opcodesStr += $"{instruction.OpCode} \"{instruction.Operand}\"\n";

			string path2 = Path.Combine(Main.SavePath, "zMOD_TEST_DELETE_" + DateTime.Now.Ticks + ".txt");
			using (StreamWriter sw = File.CreateText(path))
				sw.WriteLine(opcodesStr);
			System.Diagnostics.Process.Start(path);
		}

		public static bool IsInt(this string text, out int i) 
		{ 
			return int.TryParse(text, out i); 
		}

		public static string NameNoExt(this FileInfo file)
        {
			string name = file.Name;
			if (name.Contains('.'))
				name = name.Substring(0, name.LastIndexOf('.'));
			return name;
        }

		public static string NameNoExt(this string name)
		{
			if (name.Contains('.'))
				name = name.Substring(0, name.LastIndexOf('.'));
			return name;
		}

		public static void Empty(this DirectoryInfo directory, string[] exclude=null)
		{
			foreach (System.IO.FileInfo file in directory.GetFiles())
            {
				if (exclude != null && exclude.Contains(file.Name))
					continue;
				File.SetAttributes(file.FullName, FileAttributes.Normal);
				file.Delete();
			}
			foreach (System.IO.DirectoryInfo subDirectory in directory.GetDirectories())
				subDirectory.Delete(true);
		}

        public static void Empty(string path, string[] exclude = null)
        {
            var directory = new DirectoryInfo(path);
            directory.Empty(exclude);
        }

        public static bool Compare<K, V>(this Dictionary<K, V> dic1, Dictionary<K, V> dic2)
			where V : IComparable
        {
            if (dic1.Count != dic2.Count)
                return false;
            foreach (var pair in dic1)
            {
                V value;
                if (!dic2.TryGetValue(pair.Key, out value))
                    return false;
                if (value.CompareTo(pair.Value) != 0)
                    return false;
            }
            return true;
        }

		public static string GetMD5Checksum(this FileInfo file)
		{
			using (var md5 = System.Security.Cryptography.MD5.Create())
			{
				using (var stream = System.IO.File.OpenRead(file.FullName))
				{
					var hash = md5.ComputeHash(stream);
					return BitConverter.ToString(hash).Replace("-", "");
				}
			}
		}

		public static bool checkInternet()
        {
			try
			{
				System.Net.NetworkInformation.Ping myPing = new System.Net.NetworkInformation.Ping();
				String host = "google.com";
				byte[] buffer = new byte[32];
				int timeout = 1000;
				System.Net.NetworkInformation.PingOptions pingOptions = new System.Net.NetworkInformation.PingOptions();
				System.Net.NetworkInformation.PingReply reply = myPing.Send(host, timeout, buffer, pingOptions);
				return (reply.Status == System.Net.NetworkInformation.IPStatus.Success);
			}
			catch (Exception)
			{
				return false;
			}
		}


		public static void DeleteDirectoryInsanoMode(string target_dir)
		{
			for (int i = 0; i < 10; i++)
			{
				try
				{
					string[] files = Directory.GetFiles(target_dir);
					string[] dirs = Directory.GetDirectories(target_dir);


					foreach (string file in files)
					{
						File.SetAttributes(file, FileAttributes.Normal);
						File.Delete(file);
					}

					foreach (string dir in dirs)
						DeleteDirectoryInsanoMode(dir);

					Directory.Delete(target_dir, false);
				}
				catch (DirectoryNotFoundException)
				{
					return;
				}
				catch (IOException)
				{
					System.Threading.Thread.Sleep(10);
					continue;
				}
				return;
			}
		}

		public static void ForceEmptyInsanoMode(string targetDir, string[] exclude = null, bool filesOnly = false)
        {
			for (int i = 0; i < 10; i++)
			{
				try {
					string[] files = Directory.GetFiles(targetDir);
					string[] dirs = Directory.GetDirectories(targetDir);

					bool todo = false;
					foreach (string file in files)
					{
						if (exclude != null && exclude.Contains(new FileInfo(file).Name))
							continue;
						todo = true; 
						File.SetAttributes(file, FileAttributes.Normal);
						File.Delete(file);
					}

					if (!filesOnly)
                    {
						foreach (string dir in dirs)
						{
							if (exclude != null && exclude.Contains(new DirectoryInfo(dir).Name))
								continue;
							DeleteDirectoryInsanoMode(dir);
							todo = true;
						}
					}

					if (!todo)
						return;
				}
				catch (DirectoryNotFoundException) {
					return;
				}
				catch (IOException) {
					System.Threading.Thread.Sleep(50);
					continue;
				}
				return;
			}
		}
	}


	internal static class PATH
	{
		public readonly static string CONFIG_FILE;
		public readonly static string YOURMUSIC_DIR;
		public readonly static string EXTRACT_DIR;
		public readonly static string MUSIC_EXTRACT_DIR;
		public readonly static string MODCOMPILE_DIR;
		public readonly static string MODS_DIR;

		static PATH()
		{
			CONFIG_FILE = Path.Combine(Main.SavePath, "Mod Configs", "BetterMusic.json");
			YOURMUSIC_DIR = Path.Combine(Main.SavePath, "Your Music");
			EXTRACT_DIR = Path.Combine(Main.SavePath, "Mod Specific Data", "BetterMusic");
			MUSIC_EXTRACT_DIR = Path.Combine(EXTRACT_DIR, "Sounds", "Music");
			MODS_DIR = Path.Combine(Main.SavePath, "Mods");
			MODCOMPILE_DIR = (string)ModLoad.ModLoader.ModCompile
				.GetField("modCompileDir", BindingFlags.Static | BindingFlags.NonPublic)
				.GetValue(ModLoad.ModLoader.ModCompile);
		}
	}
}