# Manic Miner (BBC)

This is a disassembly/reassembly of Manic Miner for the BBC Micro.
I thought I would write up some thoughts on it while I'm here.

I should mention that this is the version from http://bbcmicro.co.uk/game.php?id=188 so it differs slightly from the original in that it has instructions before the game loads, and the copy protection has been circumvented.

The original asks for a four digit code from a random grid location found on a physical sheet of paper that was supplied with the game. The paper has groups of four colours laid out in a grid, each colour corresponding to a number 1-4 that must be entered correctly to continue. This system has previously been circumvented, but the original code to handle this is still present.

# Loading
After the instructions, there three more title screens! The first shows the large dancing letters of MANIC MINER and a scrolling message. The cassette version of the game animated this screen while loading, which I think was a fairly uncommon feature at the time.

The second was used for the entry code, but here is displayed for a brief time before the third title screen with a 'Penrose triangle' (the triangular 3d optical illusion) drawn via OSWRCH PLOT commands. At this point the game is fully loaded, and RETURN starts a new game.

# Obfuscation
*FX 200,3 is sprinkled a few times while loading / initialising to ensure memory is cleared on BREAK.

Code and data are split between files, and moved around in memory, sometimes EOR'd with &55 (code) or &AA (data).

A couple of short routines are stashed in zero page, perhaps to make them harder to find, to avoid cheating. The first is the code that triggers a switch, and the second is the code that resets your score at the end of the game.

A single byte of the level data is changed (a key position on level 20) during initialisation. Perhaps to make it harder to make a completely perfect copy of the game, or perhaps it's just a last minute 'fix'. I don't think it makes a significant difference to the difficulty of the last level.

# Implementation issues
The games runs slowly compared to the Spectrum original. This is partly down to the method of plotting. Horizontal guardians are drawn using OSWRCH to write user defined characters to the screen. The keys are animated using the same method. The second major reason for being slow is that there is no collision map. When the game needs to know what is on the screen (e.g. the squares surrounding the player) it reads the character from the screen using OSBYTE 135. The OS has to read the 8x8 pixels off the screen and then compare them against each character 32-255 in the current character set until if finds a match. This is understandably not efficient.

For example, each key is animated (one at a time round robin style) by moving the text cursor to the X,Y position of a key (3 calls to OSWRCH), reading the character from the screen (OSBYTE 135) and if it's still a key (not taken) then plot the key in a new colour (three more calls to OSWRCH).

The player and vertical guardians are drawing using a custom plot routine (which is also not greatly efficient).

There is an explicit short delay in the main loop, but this is much less significant a factor in the speed.

# Variations from the Spectrum version
* Only four colours are available instead of the Spectrum's 16

* The BBC version only occupies a fraction of the screen

* The title screen is basic on the BBC. The Spectrum version has a visual scene, a piano keyboard and The Blue Danube playing.

* The screen layout is different, with the Air bar at the side rather than below, and the Spectrum version shows the lives visually as player sprites animating, but it's just a number on the BBC.

* The individual graphics are sometimes not perfectly identical

* The final two levels are different. The Spectrum's "Solar Power Generator" has been replaced with a new design "The Meteor Storm", and the final level is a different design too (but still called "The Final Barrier"). They both include a new feature that I am calling 'energy fields', barriers that turn on / off at regular intervals and are deadly when on.

# The level teleport cheat
Type 'A SECRET' on the pause screen. After resuming play, the fn keys teleport you to the different levels, and SHIFT+fn keys go to later levels. The code is a little hidden since it doesn't use one of the regular methods of reading keys. It doesn't call the OS, nor does it read the keyboard directly. Instead it reads memory location $ec, which the OS uses to hold the code of the key currently pressed. I wonder if this was deliberate obfuscation.

# Level format

## Strips
The strips of level data are stored at 'levelDefinitions', which is memory address $6c00, or offset $1680 into MINER4 binary file (all bytes EORd with $55 in the file).

The separator between each level is $ff, $ff which occurs before and after every level.
This is followed by header information:

    byte offset     meaning
         0          two nybbles, high is palette colour 0, low is palette colour 1
         1          low nybble is palette colour 2
         2          high nybble is the regular floor sprite, low nybble is the crumble floor
         3          lower 3 bits are the conveyor sprite and the side wall sprite
         4          lower nybble is the key sprite
         5          the exit sprite - top two bits are the type:

           00 = 8x8 exit sprite repeated four times (lower 3 bits are sprite number)
           10 = 8x16 exit sprite mirrored about the Y axis (lower 3 bits are sprite number)
           01 = }
           11 = } 16x16 exit sprite (lower 6 bits are the sprite number)

    Then the level data starts:

    command         description
        $ff         Increments the levelFeatureIndex. Once it reaches 5, the level is done.
        $fe         The next four bytes determine a rectangle to draw:
            <x_min> <y_min> <width> <y_max>
        $fd         Sets the levelFeatureIndex to the next byte value.
        else        Next three bytes determine a horizontal strip to draw:
            <x_min> <y_min> <width>

## Single Items
Individual items are stored at 'levelSingleItemDefinitions', which is memory address $7170, or offset $1bf0 into MINER4 binary file (all bytes EORd with $55 in the file)

The separator between each level is $ff which occurs before and after every level.

    command         description
        $ff         end of level
        $fe         all X coordinates have 15 added to them from now on until the next change of type
        $fd         increment type
        else        top nybble is X coordinate, bottom nybble is Y coordinate. Plot.

At plot time, the current type determines the outcome:

        $eb         switch to colour 1 and draw <something>
        $ee         spider and thread (next byte is length of thread)
        $f0         key (colour 3)
        else        sprite number (see below)

## Standard Sprites

    Sprite numbers start at $80 and repeat. $80 is the same as $a0 $c0 and $e0.
    Alternative sprite pages are swapped in and out as needed e.g. to draw horizontal guardians.

      sprite        description
        ($5f         underscore, which is identical to the most crumbled floor sprite)
        $80         platform
        $81         alternative platform
        $82         crumble floor
        $83         crumble floor
        $84         crumble floor
        $85         crumble floor
        $86         crumble floor
        $87         crumble floor
        $88         crumble floor
        $89         crumble floor
        $8a         (empty)
        $8b         conveyor
        $8c         conveyor
        $8d         conveyor
        $8e         conveyor
        $8f         wall
        $90         key
        $91         exit top left
        $92         exit top right
        $93         exit bottom left
        $94         exit bottom right
        $95         (empty)
        $96         (empty)
        $97         (empty)
        $98         unswitched switch
        $99         switched switch
        $9a         (empty)
        $9b         spike 1
        $9c         spike 2
        $9e         thread
        $9f         (empty)


# Issues
* Slow
* Player movement is flickery
* Collision detection is poor, not pixel based
* Music is basic (similar to the Spectrum) and a note can be missed when overridden by playing a sound effect
* Doesn't work on the either the Master or Electron
