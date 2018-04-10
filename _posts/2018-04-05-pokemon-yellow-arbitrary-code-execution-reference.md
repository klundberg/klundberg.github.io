---
category: gaming
tags: pokemon glitches
---

I played the first and second generations of Pokemon a ton back in the mid-late nineties, and I've recently gotten back into it with Pokemon Go and Pokemon Moon/Ultra Sun pretty hard. Nintendo sells the original gameboy games via their virtual console on the 3ds, and I've been playing a ton of Pokemon Yellow as well.

There's a ton of fun glitches you can pull off in Yellow to basically do anything you want in the game via arbitrary code execution (like obtain any pokemon, alter any pokemon's stats, among other things). I'm very interested in using these to get pokemon with perfect stats or to get shiny pokemon into the latest games. However, these glitches require you to do things like have certain quantities of certain items in your inventory in a certain order (since those items can represent z80 assembly instructions that you can then execute). Remembering all the different item codes for the glitches I want to execute is hard, and there's no single comprehensive reference for all of the ones I want that I've found, so I'm going to aggregate and document them here.

**Disclaimer:** this is for the English version of Pokemon Yellow only. Red/Blue and other languanges have different item code requirements, and red/blue requires an entirely different setup to get the glitch item that will let you execute these codes (yellow uses a glitch item named `ws# #m#`, red/blue use a glitch item named `8F`).

[This guide](https://www.reddit.com/r/pokemon/comments/5q8zlg/getting_gen_1_mew_in_yellow_guide_does_not_work/) describes how to do a lot of the initial setup and lists some of the initial codes you can execute, and I'll be re-iterating a lot of what this post says, but also adding some other useful information that I've gathered from other places, but use it to learn how to get `ws# #m#` and enough `Revive` to get started with these codes.

When you get the `ws# #m#` item, you need to designate a box in your PC to contain only specific pokemon in a specific order. This is because using `ws# #m#` will by default treat your current box as executable code, but reordering and editing pokemon in the PC is hard, so the following pokemon list will tell `ws# #m#` to redirect execution to your inventory instead which is much easier to handle. In your box you must have these 11 pokemon in this order (and not one more):

1. Seel **with exactly 233 HP** (nothing else has any stat requirements)
2. Parasect
3. Growlithe
4. Magikarp
5. Psyduck
6. Flareon
7. Tentacool
8. Female Nidoran
9. Any Pokemon
10. Any Pokemon
11. Any Pokemon

Whenever you use `ws# #m#`, make sure this box is selected, and make sure if any pokemon you get are sent to this box, that you take it out so that this list is preserved.

Now you can set up your inventory to make things happen. Lots of the item codes I've seen require you to have nothing in your inventory aside from the items listed, but that gets pretty annoying when you need to change item codes around a lot to do something complex. To solve this, after every inventory list here you can add `TM01` to the end (in any amount). `TM01` corresponds to the return instruction in z80 assembly, which means that any items after it won't be executed as code, and you can keep enough items for multiple codes in your inventory at once if needed.

### To change any item into any other item:

1. `ws# #m#`
2. The item you want to change x1
3. `Burn Heal x43`
4. `Ice Heal x43`
5. `Revive x201`, or `Full Heal x201`

This will change the item in slot 2 by decrementing the item's ID by 1 if you use Revives, or incrementing the item's ID by 1 if you use Full Heals. You can consult [this list of item IDs](https://glitchcity.info/biglist.htm) to see where the item you want is in relation to the item you have. If you try to go below ID 0, you'll loop around to item ID 255 and vice versa.

### To increase or decrease the quantity of an item's stack by one:

1. `ws# #m#`
2. The item you want to change quantity of
3. `Burn Heal x43`, or `Ice Heal x43`
4. `Revive x201`, or `Full Heal x201`

This code which is similar to the prior one will decrease (with revives) or increase (with full heals) the number of items you have in slot 2 by 1. If you want to get more than 99 of an item (which isn't possible otherwise without finding missingno), you can take one item, have `Revive x201` in slot 4, then use `ws# #m#` twice, once to reduce the stack to 0, then once to overflow and loop around to 255. From there you can opt to just toss the items in order to get down to the exact count you need for other codes. Choosing `Burn Heal` or `Ice Heal` makes no difference, it just needs to be one of those.

### To get any pokemon immediately as a gift

1. `ws# #m#`
2. Any item (you may also have `ws# #m#` be in slot 2)
3. `Repel x[SpeciesIndex]`
4. `X Speed x64`
5. `Awakening x[Desired Level]`
6. `TM05 x89`
7. `Lemonade x201`

This is brilliant. When you use the right number of `Repel` (for instance, `Repel x21` for Mew), and you have the number of `Awakening` corresponding to the level you want it to be, you're instantly gifted the pokemon you wanted. To find the right number of `Repel`, use [the same link as the item index](https://glitchcity.info/biglist.htm) which lists the pokemon's species index as well. Don't assume that the pokedex number is what you need, it isn't the same at all.

If you want to get a Mew that can be transferred to the pokemon bank however, there are some restrictions. It must be **at least level 5**, the original trainer's name must be **GF**, and it must have an original trainer ID of **22768**. To manipulate your trainer ID and name if yours don't match use the next set of codes before using this to get Mew.

### To change your trainer's name

First, use the pokemon name rater in Lavender Town to change your pokemon's nickname to the name you want, and put it in the first slot of your party. Then use this inventory:

1. `ws# #m#`
2. Any item (you may also have `ws# #m#` be in slot 2)
3. `TM50 x180`
4. `TM10 x64`
5. `TM34 x87`
6. `TM09 x46`
7. `Carbos x52`
8. `X Accuracy x34`
9. `Full Heal x201`

Use `ws# #m#` as many times as the length of your desired name, plus 1. If you're renaming yourself to `GF`, you need to use `ws# #m#` three times, since you need to copy over the string terminator character from the nicknamed pokemon as well. If you're renaming yourself to something 6 characters long, you should use it 7 times. You can change it back to your old name if you want by nicknaming another pokemon to your old name and repeating the process.

This code will increase the quantity of `TM10` by 1 each time you use it as well, which represents pointing to the next character of your name. You'll need to toss back down to 64 in order to do change your trainer's name from scratch again.

### To change your trainer's ID

1. `ws# #m#`
2. Any item (you may also have `ws# #m#` be in slot 2)
3. `Lemonade x[xx]`
4. `Repel x[yy]`
5. `Carbos x211`
6. `X Accuracy x88`
7. `Water Stone x115`

The ID you want controls what `Lemonade` and `Repel`'s quantities must be. For a transferrable mew, you want `Lemonade x89` and `Repel x 12`. To change it back, consult [the reddit thread I linked earler](https://www.reddit.com/r/pokemon/comments/5q8zlg/getting_gen_1_mew_in_yellow_guide_does_not_work/).

Once you do this, then you can use the gift pokeon code to get a legitimate Mew.

### To edit the DV stats of a pokemon
This will let you make a pokemon "perfect" by ensuring its individual stat adjustments (called IVs in later games, but DVs in generation 1/2) are the best ones possible, or are the precise values needed for a pokemon to be considered shiny upon transfer.

Put the pokemon you want to edit first in your party, then execute this code:

1. `ws# #m#`
2. Any item (you may also have `ws# #m#` be in slot 2)
3. `Lemonade x255`, or `Lemonade x170`
4. `X Accuracy x 134`
5. `Carbos x209`
6. `Poke Ball x199`
7. `Fresh Water x201`

255 Lemonade will set your pokemon's DVs all to 15 (the highest possible, for a perfect stats pokemon), while 170 Lemonade will set it's DVs all to 10 to make it shiny when you transfer. It's possible for your pokemon to have 15 DV in attack with 10 everywhere else and still be shiny, but this code doesn't do that. Other more complicated codes might be able to modify attack to make it perfect but I'm not familiar with them.

## That's it!

If there's any other pokemon enthusiasts out there who know of other item codes (or who want me to add red/blue or european/japanese codes) let me know!
