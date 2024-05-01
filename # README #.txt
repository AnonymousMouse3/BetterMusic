___________________________________________
          HOW TO ADD CUSTOM MUSIC          
___________________________________________

1.
Place your music files inside the "Your Music" folder. (ogg or wav files)

2.
Go to Better Music config and click [ADD YOUR MUSIC], save config
NOTE: tModLoader should reload automatically as soon as you hit save. Check the notes if it doesn't.

3. 
Once done reloading, you can select the enemy or biome music to replace and enter the name of the file.



___________________________________________
                   NOTES          
___________________________________________

- For multiplayer: If multiplayer does not work, try sharing the BetterMusic.tmod file found in 
ModLoader\Mods\ and the BetterMusic config files in ModLoader\Mod Configs\

- Your custom songs should be about as loud as the default ones

- You only need to [ADD YOUR MUSIC] when modifying, adding or removing files in the "Your Music" folder.
Other than that, you can manipulate added music however you want (even ingame).

- When choosing an enemy, if you see multiple parts that apply, use common sense. 
(Moon Lord Core instead of head, Eater of Worlds head instead of body, etc..)

- If you're adding multiple different files for the same enemy and the same HP value, make sure they 
don't have overlapping conditions. Even if the conditions aren't the exact same, it doesn't mean they 
don't overlap: If file A is has no conditions and file B is set to only play underground, and if both 
are set to play at 100% of the enemy's HP, file A will always play, since it's the first on the list and 
its conditions already apply. In that case you should select the overground condition for A. If A and B 
overlap, but are set to play at different HP levels, the one with the lower HP value will always play 
once the enemy's HP drops to that point.

- The same applies for biome music. Make sure you're not adding multiple files with overlapping 
conditions and 100% chance to play, as the second file will never play in that case.

- TmodLoader should reload automatically after you click on the "add your music" button and save. If it 
doesn't, the installation process has failed somehow.
If the [ADD YOUR MUSIC] button doesn't work: Delete the mod, delete all BetterMusic config files located 
at Terraria\ModLoader\Mod Configs\ and redownload the mod from the Mod Browser to try again.
It's also worth giving it a try in the other tmodloader version (64 or 32bit). Once the files are added, 
they become part of the mod file, and you can switch to the other tml version you've been using (you can also transfer the .tmod file to another machine without readding the music).



___________________________________________
                KNOWN BUGS          
___________________________________________

- Replacements for Overworldday and AltOverworldDay only work if if you add your music to both with 100% 
chance of playing