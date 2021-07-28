+++
title = "Reverse engineering Age of empire II - Part I - Text Helpers"
date = 2021-07-28
+++

## Introduction 

// TODO 

### Memories

Before going further I would like to share some reminiscence. 

I've been playing age of empire for two decades now. Actually Age of Empire I is the first game I ever played. 
In 1998 we had no computer at home so my father took me to is office from time to time to play AoE for a couple of hour. 

In the early 2000s I used to play at one of my friend's home with his elder browser.
We were playing in separate room so no one can cheat looking at is opponent screen. 
Later during high school we orginazed 4vs4 AoE2 LANs with friends.
I was in charge of settings up the network for every one and bringing a cracked version
of Ao2 the conquerors (don't tell Microsoft please). 
We drank beer, and smoked marijuana, spammed **14** and played the best game ever made. 

Evoking this memory feel me with nostalgia and I am pretty sure most AoE2 players of my generation feel the same about the game.

### Age of empire versions 

Since the first release of Age of empire II in 1999 many extention and new version as been published :
- The conqueror : the first expansion pack release in 2000 
- HD Edition and its many addons (2013) 
    - The Forgotten
    - The African Kingdoms
    - Rise of the Rajas
- Definitive Edition
    - Lords of the West
    - Dawn of the Dukes

Along this blog post we will focus on the Definitive editions. Its the latest and most popular version. 

## Siege Engineers

One day, my friend Alex told me he was working on [aoe2techtree.net](https://aoe2techtree.net) clone.
He started this has a hobby project to learn a new javascript framework. I was quite interested and 
when he told me how hard it was for him to collect game data I started to look at aoe2techtree codes 
to understand where the data came from. 

Turns out there is a discord server dedicated to Age of Empire reverse engineers called Siege Engineer.
The amount of amazig project going on there is just incredible. Game record analysis, techtree, game replay, modding ...
It would take an entire book to list every Ao2 related project made by those people, 
you can check their repository [here](https://github.com/SiegeEngineers/).  

After asking around where I could get the techtree data I found some usefull tools : 
- [aoe2dat](https://github.com/HSZemi/aoe2dat) is a small C++ program to extract raw data from the game and convert
    them to a giant json file (about 300Mb)
- [genieutils](https://github.com/Tapsa/genieutils) is the backend library for ao2dat, in charge of deserializing the binary file
    containing all the game data (we will talk about this later). 
- [genie-rs](https://github.com/SiegeEngineers/genie-rs) a rewrite of genieutils in rust ❤️ (Doesn't support Definitive Edition yet).

## Exploring the data

The age of Empire II De instalation folder contain a file called `empires2_x2_p1.dat`, this is the grall of Ao2 reverse engineers. 
It's a binary containing about every usefull information on the game : unit stats, buildings, technologies, sound, sprite location etc.

If you are a linux user like me and installed the game via steam you can find this file in  `$HOME"/.steam/steam/steamapps/common/AoE2DE/resources/_common/dat/empires2_x2_p1.dat`. 


### Extrating data via aoe2dat

I installed [aoe2dat](https://github.com/HSZemi/aoe2dat) and followed the build instruction. 
After a bit of struggle with the needed library I manage to extract the game data : 

```shell
./aoe2dat "$HOME"/.steam/steam/steamapps/common/AoE2DE/resources/_common/dat/empires2_x2_p1.dat
```

aoe2dat create two file from the parsed binary : 
- `full.json` : A 258Mb json file containing all the game data
- `units_buildings_techs.json` : A digest version of the full.json file containing only 
    useful information on techs, units and buildings.

```json
	"47" : {
		"cost": {
			"wood": 0,
			"food": 300,
			"gold": 200,
			"stone": 0
		},
		"help_converter":28047,
		"language_file_name":7047,
		"language_file_help":107047,
		"name":"Chemistry"
	},
```

The above sample represent the data for the [Chemistry](https://ageofempires.fandom.com/wiki/Chemistry) technology, 
It is quite self explanatory appart from the `help_converter` and languages fields, 
these are actually mapping to some internationalized text in some other game files. 

### Getting help texts

Going on with the chemistry example, here are the acual game data we are looking for :

![chemistry](../images/chemistry_techtree.png)

This is a screenshot of the game with the help text displayed when hoovering chemistry on the techtree. 


To find this value in the game files, we need to list files in `"$HOME"/.steam/steam/steamapps/common/AoE2DE/resources`
```shell
❯ ls "$HOME"/.steam/steam/steamapps/common/AoE2DE/resources
br  _common  de  en  es  fr  hi  it  jp  ko  _launcher  ms  mx  _packages  ru  tr  tw  vi  zh
```

Now let's explore english game content : 
```
❯ tree"$HOME"/.steam/steam/steamapps/common/AoE2DE/resources/en
en
├── campaign
│   └── movies
│       ├── cm1.wmv
│       ├── ... 
└── strings
    ├── history
    ├── ... 
    └── key-value
        ├── key-value-modded-strings-utf8.txt
        └── key-value-strings-utf8.txt
```

The data we are looking for are located in `"$HOME"/.steam/steam/steamapps/common/AoE2DE/resources/en/strings/key-value/key-value-strings-utf8.txt`.

Taking a look at the file content we can see some numeric values mapping to a string content :
```
1150 "Victory!"
1151 "Defeat!"
1152 "was defeated"
1200 "Ready"
1201 "Computer"
1202 "Empty"
1203 "Loading"
```

Now we will look for value we previously got for Chemistry in the json file produced by aoe2dat (I am using the french text here to match with my installed game). 

**7047** :  

```
"Chimie"
```

**28047** :

```
"Développer <b>Chimie<b> (<cost>) \nLes unités à projectiles (excepté les unités à poudre à canon) gagnent 
+1 de force d'attaque. <b><i> Nécessaire pour les unités poudre à canon (Canonnier à main, galion à canon, 
canon à\nbombarde, tour de bombarde).<b><i>"
```

It's a match ! 

`107047` does not seem to map to anything in the text helper file, after digging the language file and comparing with other technology and unit data I found out there is actually an offset to get some other usefull help text : `107047 - 99000 = 8047`.

**8047** :

```
"Développer Chimie (+1 d'attaque avec projectiles, excepté pour les unités à poudre à canon)"
```


Now we have the first bits and pieces to build a techtree with some textual information.

This concludes the first part on this serie, in the next post we will create and populate a database
from ao2edat/genie extracted game data. 


