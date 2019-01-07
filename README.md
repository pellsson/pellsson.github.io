# SWAGGINZZZ

In what was her first game of NetHack ever, SWAGGINZZZ struggled. She constantly bumped into walls, and oftentimes found herself with critically low HP. The setbacks would not deter her however, and after 7 minutes and 15 seconds she ended up ascending – having taken 2087 in-game turns to do so.

Some will argue that SWAGGINZZZ cheated. She did, in fact, have help. Her friend clumsily played for the first 6 minutes of the game looking for a fountain, making the effective ascension time for SWAGGINZZZ ~1 minute. Other than that though, SWAGGINZZZ with her RNG-predicting AWS-cluster of monster computers ascended all on her own, completely legitimately, without any help 😉

[Dumplog](https://s3.amazonaws.com/altorg/dumplog/SWAGGINZZZ/1546732576.nh361.txt)

[TTYREC](https://alt.org/nethack/trd/?file=https://alt.org/nethack/userdata/S/SWAGGINZZZ/ttyrec/2019-01-05.23:56:16.ttyrechttps://alt.org/nethack/trd/?file=https://alt.org/nethack/userdata/S/SWAGGINZZZ/ttyrec/2019-01-05.23:56:16.ttyrec)

[NAO Scoreboard](https://alt.org/nethack/fastasc-current.html)

# Goals for SWAGGINZZZ

We set out to have a single run getting the #1 spot in all three "major-categories": highest score, lowest turn count, and fastest realtime on NAO. We needed an RNG-predicting bot.

Lowest turn count and fastest realtime were achieved, but we abandoned the high score. Getting a sufficient score with an infinite amount of wishes would not have been terribly difficult: `Air Elemental`-wishing 2,000 times for stacks of 4-5 dilithium crystals over ~700 turns – job done. It did however feel like a waste of NAO resources to spam the massive amount of additional inputs required for the extra 2,000 wishes (as compared to the 90 wishes we wound up using for the other highscores).

As an added bonus, it’s probably the first [“lowest scoring”-ascension](https://alt.org/nethack/lowscoreasc361.html) record with all three quest artifacts still in the inventory 🙂

# RNG in NetHack

To be able to predict the RNG in NetHack, we first had to find the RNG seed used in the game we were playing on NAO. On Linux (and thus on NAO) NetHack uses `srandom()` & `random()` for RNG. The seed is 32-bit, allowing for 2^32 (4,294,967,296) different starting values.

Locally, retrieving the seed can be done by looking at the internal memory of NetHack, but to do it on a public server where the game instance is on a different computer, you need a different approach.

The easiest way to find which of the 2^32 different starting values that NAO assigned to you, is to simply restart start the game locally on an identical system until you see the same game. Finding it within a reasonable time frame would require significantly more luck than SWAGGINZZZ had in her run though 🙂

# Comparing games

How do you determine if you are in the same game in two different instances of NetHack?

The most obvious way is to just look at the dungeon layout and player position. Modifying the NetHack code to examine the internal dungeon layout given at the start of the game, and compare it to the text output on NAO is pretty complicated though.

We found another approach. As some people in #nethack already figured out, the Tourist was chosen as she starts with a lot of random stuff. 1-1000 gold, a dart stack of varying size, 6 different food objects, a camera with random charges and the initial starting attributes. This is enough input data to accurately and uniquely determine if two games are identical.

# Quickly finding the seed

Our first idea was very basic. Modify NetHack to iterate over all seeds, and for each seed compare the inventory and starting attributes with the game on NAO. If they match, we have the seed.

Unfortunately, this method would have taken weeks on average. Weeks won't do for fastest realtime ascension.

We went back and forth between several methods, but ultimately settled on building a gigantic database. Create a hash of the inventory and starting attributes for **all** `Tou Hum Fem Neu` games possible, and save them in a binary sorted lookup table.

Using every computer at hand, including a couple of 72-core AWS instances, we soon had everything we needed.

![72-core AWS instance playing a cleverly `fork()`-ed NetHack version to generate hashes.](https://i.imgur.com/ktYKkl8.jpg "72-core AWS instance playing a cleverly fork()-ed NetHack version to generate hashes.")

Sorting the ~100 GB database took a few hours 🙂

With the our sorted database in place, we could now go from starting inventory -> seed. Binary searching for an arbitrary starting inventory would in worst case be 32 lookups, absolutely free and zero time cost.

# Ascending NetHack

So, we could now go from starting inventory to RNG-seed in no time, but we still had to ascend. While we could write a simple "wish"-bot, get +127 Magicbane and then ascend by hand from there, achieving both fastest realtime and lowest turn count in the same game would not be easy.

First idea was to start a NAO game, fetch the seed, then saving and perfecting the seed offline. It was quickly ruled out though as the game is reseeded each time it is started (i.e. when you load your game).

Loooong story short, we wrote a bot. You had to play the first turns (offline) and move her to a non-magic fountain located next to a wall. If you died, no big deal, just retry on the same seed. This is why SWAGGINZZZ stood still for 6 minutes, we had absolutely horrible RNG when trying to get the specific fountain needed on dlvl2.

The fountain is required for wishes. The wall is required to be able to offset the random state without advancing the game state – every time the character attempts to walk into a wall, it calls random() without wasting any in-game time. From the fountain, the bot ascends completely on her own.

Note that we had a lot of names that were better than SWAGGINZZZ. They all succumbed when bugs in our bot logic would cause the NAO and local state to drift apart, so we simply save-quit those games as to not raise suspicion on the IRC channel. After a while, coming up with good names gets pretty hard. 😉

## Manipulating RNG

We decided to write the bot as a modified version of NetHack (instead of reading the screen for instance), since it gives you access to all internal structures which greatly simplifies the effort. ESP on steroids.

While the code in the bot is probably some of the worst code ever written (~3k lines of if-statements), the underlying design is pretty clever.

NetHack would start and run to the first `getchar()` and then `fork()` from underneath there. The child would try to solve a single phase (say, get 1 wish). If it failed (which it would a lot) the child process simply exit. The parent process is notified, advances RNG once, then calls `fork()` again for a new attempt at solving the phase.

Whenever a phase was solved (say, get a wish and the fountain is still there) the child process would tell the parent what it did to solve the phase, and the parent process would catch up. Essentially creating a new "save state".

This technique allowed for extremely simple and stupid bot logic. If she died or failed, just retry the phase again after advancing RNG. For instance, solving the Sanctum is primarily a bunch of static #jump commands in sequence. In that phase, the bot doesn't see, hear or feel anything – she just jumps – and if at the end of it she has the amulet, she is happy. Blindly jumping to the altar and one-shotting the priest does of course take **A LOT** of attempts, but modern CPUs are pretty fast and NetHack is a pretty low-end game 🙂

Now this is a huge simplification. Even with RNG manipulation, writing a bot that 99% ascends NetHack is **extremely** complicated. So much stuff can go wrong, and there is no shortage of corner cases.

# Horrible nightmarish bugs & obstacles

Piggy-backing on existing NetHack functions. If you want to research RNG-manipulation in NetHack, never call existing nethack functions. We pretty soon established it as a forbidden practice. 🙂 I recall a specific occasion when the bot was equipping rings. You need to answer *Left* or *Right* if no rings are worn. We figured that since NetHack already implements this function, let us just call it to determine what finger to use, less work for us. Turns out,  **every** function in NetHack seems to find a way to call `random()`, causing the RNG state to drift.
*Edit: Or was it armor slot it was trying to deduce? Something along those lines.*

Antholes. Truly hate antholes. At seemingly complete random, when replaying the ascension the replay would suddenly deviate from the ascended run and collapse like a supernova. Restart NetHack and paste the exact same structure again and it works. After much code digging and debugging it turns out that what populates an anthole depends on your `ubirthday`, not `random()`. As a result, if the bot had encountered an anthole the replay would only work every third second.

In the Valley of the Dead, the game would consistently deviate – our bot build would get a different state than NAO, as well as another locally-ran vanilla build. This turned out to be caused by nethack attempting to name the randomly generated corpses found on that level from old player names, even with bones disabled. The bot didn’t have any old records due to it always being run in a clean environment, making it consume one less `random()`.

To provoke the Wizard of Yendor, we polymorph into a Master Mind Flayer, using its psychic blast monster ability to damage him. On rare occasion, this wouldn’t work – he just took damage and wouldn’t teleport out of his tower, no matter how many times we blasted him. It turns out that there’s only a low chance of him spawning asleep (when we investigated his flags, we assumed he always was) and that ability doesn’t wake up sleeping enemies.

# Ascension Summary

- Find non-magic fountain next to a wall (manually)
- 90 wishes. Eyes to phase-jump through walls. 60 c!oGL. ~+100 MB
- Genocide LcPUn (due to lazy bot code ;))
- Tele Vlad, phase-jump tower, get candelabrum.
- `h`-strats to provoke Rodney (who is not allowed to return by RNG)
- Wait for quest
- Level-tele to jumping distance from quest leader (save realtime over landing next to).
- Go to end
- Land jumping distance from square
- Phase-jump to priest
- c!oGL to top (Mysterious force not allowed. Lovely.)
- Hallu for RNG manip without walls (just press space)
- Force portals relatively close on Earth, Air, Fire.
- Ascend.

I would like to end this YAAP with the eloquently phrased first reaction in #nethack 🙂

```
00:04:01 Megaman3300: holy fuck
00:04:05 Megaman3300: That's a fast fucking ascension
```

# Credits
SWAGGINZZZ
Aransentin
Breggan
Hampe
Pellsson
