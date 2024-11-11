---
layout: post
title: Flareon11 2024 CTF Challenge 1 - Frog
---
Welcome to Flare-On 11! Your mission is get the frog to the \"11\" statue, and the game will display the flag. All flags in this event are formatted as email addresses ending with the @flare-on.com domain.
[![Challenge 1 - Frog](https://lh3.googleusercontent.com/pw/AP1GczMc4Z0pvk-NzMtaw5InM5ZaRdvQYiryVJoQVvTHvZS4EmjlQoJZjRfkJxt10g2bByPFa2TXohKlsNdNF86Ji3nwKEQ-oqhvsqnsdqHHdMdQ-J3Ddd9YheV0-nzMOsZZpOOW3Zg-V7n2jHtrxNOps-jA=w734-h596-s-no-gm?authuser=2)](https://lh3.googleusercontent.com/pw/AP1GczMc4Z0pvk-NzMtaw5InM5ZaRdvQYiryVJoQVvTHvZS4EmjlQoJZjRfkJxt10g2bByPFa2TXohKlsNdNF86Ji3nwKEQ-oqhvsqnsdqHHdMdQ-J3Ddd9YheV0-nzMOsZZpOOW3Zg-V7n2jHtrxNOps-jA=w734-h596-s-no-gm?authuser=2){:target="_blank"} <br/>**Figure: 1 - Frog** <br/><br/>


# Overview
The Flare-On Challenge is the [FLARE team's](https://cloud.google.com/security/mandiant-services) annual Capture-the-Flag (CTF) contest. It is a single-player series of Reverse Engineering puzzles that runs for 6 weeks every fall. [#flareon11](flare-on11.ctfd.io) is launching Sept. 27th 2024 at 8pm EST.

This is the very first challenge and here is the description from the page.
> 1 - frog
>
>Welcome to Flare-On 11! Download this 7zip package, unzip it with the password 'flare', and read the README.txt file for launching instructions. It is written in PyGame so it may be runnable under many architectures, but also includes a pyinstaller created EXE file for easy execution on Windows.
>
>Your mission is get the frog to the \"11\" statue, and the game will display the flag. Enter the flag on this page to advance to the next stage. All flags in this event are formatted as email addresses ending with the @flare-on.com domain.

Download and unzip the file with password `flare` we can see a few files that are good to start

```bashscript
$ ls
README.txt
fonts
frog.exe
frog.py
img
```

As we have the source code `frog.py`, we can directly jump into the codes for  understanding the game logic to find the flag, but what was the fun without seeing how does the game look like? There will be many approaches to solve this challenge, let start with normal one first.

# Install dependencies
*This step is optional for Windows users and for ones who don't need to run the game.*

The `frog.py` is importing `pygame`, hence we need to install this page. We can use pip3 to install `pygame` package:
```bashscript
$ pip3 install pygame

error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try brew install
    xyz, where xyz is the package you are trying to
    install.

    If you wish to install a Python library that isn't in Homebrew,
    use a virtual environment:

    python3 -m venv path/to/venv
    source path/to/venv/bin/activate
    python3 -m pip install xyz

    If you wish to install a Python application that isn't in Homebrew,
    it may be easiest to use 'pipx install xyz', which will manage a
    virtual environment for you. You can install pipx with

    brew install pipx

    You may restore the old behavior of pip by passing
    the '--break-system-packages' flag to pip, or by adding
    'break-system-packages = true' to your pip.conf file. The latter
    will permanently disable this error.

    If you disable this error, we STRONGLY recommend that you additionally
    pass the '--user' flag to pip, or set 'user = true' in your pip.conf
    file. Failure to do this can result in a broken Homebrew installation.

    Read more about this behavior here: <https://peps.python.org/pep-0668/>

note: If you believe this is a mistake, please contact your Python installation or OS distribution provider. You can override this, at the risk of breaking your Python installation or OS, by passing --break-system-packages.
hint: See PEP 668 for the detailed specification.
```
As you can see, it recommends using a virtual environment instead of a system-wide installation to avoid risking issues with our Python setup or operating system. Let's follow the instructions to create a virtual environment instead.

```bashscript
$ python3 -m venv path/to/venv
$ source path/to/venv/bin/activate
(venv) $ python3 -m pip install pygame
Collecting pygame
  Using cached pygame-2.6.1-cp313-cp313-macosx_11_0_arm64.whl.metadata (12 kB)
Using cached pygame-2.6.1-cp313-cp313-macosx_11_0_arm64.whl (12.4 MB)
Installing collected packages: pygame
Successfully installed pygame-2.6.1
```

We just ran three commands to create, activate, and install `pygame` in the virtual environment (you’ll notice the virtual environment is activated by the `(venv)` prefix).

# Start the game - Run `frog.py`
```bashscript
(venv) $ python3 frog.py
pygame 2.6.1 (SDL 2.28.4, Python 3.13.0)
Hello from the pygame community. https://www.pygame.org/contribute.html
```

After starting `frog.py` from the command line, you can see the game window appears with a frog, a Flareon statue labeled `11` (the target where we need to move the frog), and walls (blocks) that protect the statue from the frog.

[![Starting frog position](https://lh3.googleusercontent.com/pw/AP1GczMc4Z0pvk-NzMtaw5InM5ZaRdvQYiryVJoQVvTHvZS4EmjlQoJZjRfkJxt10g2bByPFa2TXohKlsNdNF86Ji3nwKEQ-oqhvsqnsdqHHdMdQ-J3Ddd9YheV0-nzMOsZZpOOW3Zg-V7n2jHtrxNOps-jA=w734-h596-s-no-gm?authuser=2)](https://lh3.googleusercontent.com/pw/AP1GczMc4Z0pvk-NzMtaw5InM5ZaRdvQYiryVJoQVvTHvZS4EmjlQoJZjRfkJxt10g2bByPFa2TXohKlsNdNF86Ji3nwKEQ-oqhvsqnsdqHHdMdQ-J3Ddd9YheV0-nzMOsZZpOOW3Zg-V7n2jHtrxNOps-jA=w734-h596-s-no-gm?authuser=2){:target="_blank"} <br/>**Figure: 2 - Starting frog position** <br/><br/>

We can use the arrow keys to move the frog step by step, but there is no way to get the frog close to the statue because the wall blocks our path.

# Explore the source code - `frog.py`
```python
import pygame

pygame.init()
pygame.font.init()
screen_width = 800
screen_height = 600
tile_size = 40
tiles_width = screen_width // tile_size
tiles_height = screen_height // tile_size
screen = pygame.display.set_mode((screen_width, screen_height))
clock = pygame.time.Clock()
victory_tile = pygame.Vector2(10, 10)

pygame.key.set_repeat(500, 100)
pygame.display.set_caption('Non-Trademarked Yellow Frog Adventure Game: Chapter 0: Prelude')
dt = 0

floorimage = pygame.image.load("img/floor.png")
blockimage = pygame.image.load("img/block.png")
frogimage = pygame.image.load("img/frog.png")
statueimage = pygame.image.load("img/f11_statue.png")
winimage = pygame.image.load("img/win.png")

gamefont = pygame.font.Font("fonts/VT323-Regular.ttf", 24)
text_surface = gamefont.render("instruct: Use arrow keys or wasd to move frog. Get to statue. Win game.",
                               False, pygame.Color('gray'))
flagfont = pygame.font.Font("fonts/VT323-Regular.ttf", 32)
flag_text_surface = flagfont.render("nope@nope.nope", False, pygame.Color('black'))

class Block(pygame.sprite.Sprite):
    def __init__(self, x, y, passable):
        super().__init__()
        self.image = blockimage
        self.rect = self.image.get_rect()
        self.x = x
        self.y = y
        self.passable = passable
        self.rect.top = self.y * tile_size
        self.rect.left = self.x * tile_size

    def draw(self, surface):
        surface.blit(self.image, self.rect)

class Frog(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        self.image = frogimage
        self.rect = self.image.get_rect()
        self.x = x
        self.y = y
        self.rect.top = self.y * tile_size
        self.rect.left = self.x * tile_size

    def draw(self, surface):
        surface.blit(self.image, self.rect)

    def move(self, dx, dy):
        self.x += dx
        self.y += dy
        self.rect.top = self.y * tile_size
        self.rect.left = self.x * tile_size

blocks = []



player = Frog(0, 1)

def AttemptPlayerMove(dx, dy):
    newx = player.x + dx
    newy = player.y + dy

    # Can only move within screen bounds
    if newx < 0 or newx >= tiles_width or newy < 0 or newy >= tiles_height:
        return False

    # # See if it is moving in to a NON-PASSABLE block.  hint hint.
    for block in blocks:
        if newx == block.x and newy == block.y and not block.passable:
            return False

    player.move(dx, dy)
    return True


def GenerateFlagText(x, y):
    key = x + y*20
    encoded = "\xa5\xb7\xbe\xb1\xbd\xbf\xb7\x8d\xa6\xbd\x8d\xe3\xe3\x92\xb4\xbe\xb3\xa0\xb7\xff\xbd\xbc\xfc\xb1\xbd\xbf"
    return ''.join([chr(ord(c) ^ key) for c in encoded])

def main():
    global blocks
    blocks = BuildBlocks()
    victory_mode = False
    running = True
    while running:
        # poll for events
        # pygame.QUIT event means the user clicked X to close your window
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_w or event.key == pygame.K_UP:
                    AttemptPlayerMove(0, -1)
                elif event.key == pygame.K_s or event.key == pygame.K_DOWN:
                    AttemptPlayerMove(0, 1)
                elif event.key == pygame.K_a or event.key == pygame.K_LEFT:
                    AttemptPlayerMove(-1, 0)
                elif event.key == pygame.K_d or event.key == pygame.K_RIGHT:
                    AttemptPlayerMove(1, 0)

        # draw the ground
        for i in range(tiles_width):
            for j in range(tiles_height):
                screen.blit(floorimage, (i*tile_size, j*tile_size))

        # display the instructions
        screen.blit(text_surface, (0, 0))

        # draw the blocks
        for block in blocks:
            block.draw(screen)

        # draw the statue
        screen.blit(statueimage, (240, 240))

        # draw the frog
        player.draw(screen)

        if not victory_mode:
            # are they on the victory tile? if so do victory
            if player.x == victory_tile.x and player.y == victory_tile.y:
                victory_mode = True
                flag_text = GenerateFlagText(player.x, player.y)
                flag_text_surface = flagfont.render(flag_text, False, pygame.Color('black'))
                print("%s" % flag_text)
        else:
            screen.blit(winimage, (150, 50))
            screen.blit(flag_text_surface, (239, 320))

        # flip() the display to put your work on screen
        pygame.display.flip()

        # limits FPS to 60
        # dt is delta time in seconds since last frame, used for framerate-
        # independent physics.
        dt = clock.tick(60) / 1000

    pygame.quit()
    return

def BuildBlocks():
    blockset = [
        Block(3, 2, False),
        Block(4, 2, False),
        Block(5, 2, False),
        Block(6, 2, False),
        Block(7, 2, False),
        Block(8, 2, False),
        Block(9, 2, False),
        Block(10, 2, False),
        Block(11, 2, False),
        Block(12, 2, False),
        Block(13, 2, False),
        Block(14, 2, False),
        Block(15, 2, False),
        Block(16, 2, False),
        Block(17, 2, False),
        Block(3, 3, False),
        Block(17, 3, False),
        Block(3, 4, False),
        Block(5, 4, False),
        Block(6, 4, False),
        Block(7, 4, False),
        Block(8, 4, False),
        Block(9, 4, False),
        Block(10, 4, False),
        Block(11, 4, False),
        Block(14, 4, False),
        Block(15, 4, True),
        Block(16, 4, False),
        Block(17, 4, False),
        Block(3, 5, False),
        Block(5, 5, False),
        Block(11, 5, False),
        Block(14, 5, False),
        Block(3, 6, False),
        Block(5, 6, False),
        Block(11, 6, False),
        Block(14, 6, False),
        Block(15, 6, False),
        Block(16, 6, False),
        Block(17, 6, False),
        Block(3, 7, False),
        Block(5, 7, False),
        Block(11, 7, False),
        Block(17, 7, False),
        Block(3, 8, False),
        Block(5, 8, False),
        Block(11, 8, False),
        Block(15, 8, False),
        Block(16, 8, False),
        Block(17, 8, False),
        Block(3, 9, False),
        Block(5, 9, False),
        Block(11, 9, False),
        Block(12, 9, False),
        Block(13, 9, False),
        Block(15, 9, False),
        Block(3, 10, False),
        Block(5, 10, False),
        Block(13, 10, True),
        Block(15, 10, False),
        Block(16, 10, False),
        Block(17, 10, False),
        Block(3, 11, False),
        Block(5, 11, False),
        Block(6, 11, False),
        Block(7, 11, False),
        Block(8, 11, False),
        Block(9, 11, False),
        Block(10, 11, False),
        Block(11, 11, False),
        Block(12, 11, False),
        Block(13, 11, False),
        Block(17, 11, False),
        Block(3, 12, False),
        Block(17, 12, False),
        Block(3, 13, False),
        Block(4, 13, False),
        Block(5, 13, False),
        Block(6, 13, False),
        Block(7, 13, False),
        Block(8, 13, False),
        Block(9, 13, False),
        Block(10, 13, False),
        Block(11, 13, False),
        Block(12, 13, False),
        Block(13, 13, False),
        Block(14, 13, False),
        Block(15, 13, False),
        Block(16, 13, False),
        Block(17, 13, False)
    ]
    return blockset


if __name__ == '__main__':
    main()
```

The game screen has a size of `800 x 600`, and each tile is `40 x 40`, which means there are `800 / 40 = 20` columns and `600 / 40 = 15` rows.

The code also includes a hard-coded `victory_tile = pygame.Vector2(10, 10)`, meaning that if the frog reaches the tile position `(10, 10)`, we win the game and can get the flag. Let’s note this for reference.

There is a `Block` class that stores each block's location and indicates if it's passable (allowing the frog to move over it). Additionally, there is a `Frog` class, which contains the frog's current location, an array of `blocks` containing `Block` instances (representing the wall we see on the game screen that blocks the frog from reaching the statue), an `AttemptPlayerMove` function to validate if the new position is a valid move (checking if the frog is out of screen boundaries or hitting the blocks), a `main` function to handle user keyboard inputs for moving the frog, and finally, a `GenerateFlagText` function, which decodes the encoded flag and displays it on the screen. With this information, we have enough to solve this challenge. There are several approaches to solve it, both with and without running the game.


# 1st approach - Modify player starting position

In the source code, the player starts at column 1 and row 0 with `player = Frog(0, 1)`. Since we know the hard-coded `victory_tile = pygame.Vector2(10, 10)` position, we can change the player’s starting position to `player = Frog(10, 10)` instead and re-run the game. Kaboom!


[![Modify player starting position - Kaboom!!!](https://lh3.googleusercontent.com/pw/AP1GczMIs7QqluXrsZsdvMiF5VqkPJzKfRqKAw-MIJO6D4Ye-EVQ1guXVF0R0GL8mglrkGjh4b97khymo6J2GSPCdmSE3azDaWhoD0UGf7AAiW3x436NfH-m6BKNHX25NkFobrHPTjUwF9ZPAmbJmIAENRXx=w734-h596-s-no-gm?authuser=2)](https://lh3.googleusercontent.com/pw/AP1GczMIs7QqluXrsZsdvMiF5VqkPJzKfRqKAw-MIJO6D4Ye-EVQ1guXVF0R0GL8mglrkGjh4b97khymo6J2GSPCdmSE3azDaWhoD0UGf7AAiW3x436NfH-m6BKNHX25NkFobrHPTjUwF9ZPAmbJmIAENRXx=w734-h596-s-no-gm?authuser=2){:target="_blank"} <br/>**Figure: 3 - Modify player starting position - Kaboom!!!** <br/><br/>

# 2nd approach - Make blocks passable

Now let’s have a bit of fun and allow the frog to leap over the blocks! To do this, we can force `passable` to always be `True` regardless of the value passed into the `Block` constructor, like this:

```python
class Block(pygame.sprite.Sprite):
    def __init__(self, x, y, passable):
        super().__init__()
        self.image = blockimage
        self.rect = self.image.get_rect()
        self.x = x
        self.y = y
        # self.passable = passable # commented out this original code
        self.passable = True # force all blocks are passable, which means it's nolonger a block :D
        self.rect.top = self.y * tile_size
        self.rect.left = self.x * tile_size
```

Re-run the game, and let the kids have fun! ^_^
[![Do you see the frog can leap?](https://lh3.googleusercontent.com/pw/AP1GczOrPT009IfwzZK7JbwHQ1kCtyBiC6Mk0gHd-CCHifs-RrerQPrlQJza_gZRhht4mFIasWH1DczPd8aCpu3P8drOymkpyWjkSKQA5KoydA7Jv-s-VIS2dgKQShJB6yifKWLpEwVE4CCemaAcajedHya-=w1350-h1096-s-no-gm?authuser=2)](https://lh3.googleusercontent.com/pw/AP1GczOrPT009IfwzZK7JbwHQ1kCtyBiC6Mk0gHd-CCHifs-RrerQPrlQJza_gZRhht4mFIasWH1DczPd8aCpu3P8drOymkpyWjkSKQA5KoydA7Jv-s-VIS2dgKQShJB6yifKWLpEwVE4CCemaAcajedHya-=w1350-h1096-s-no-gm?authuser=2){:target="_blank"} <br/>**Figure: 4 - Do you see the frog can leap?** <br/><br/>

# 3rd approach - Find the XOR key

The above approaches require patching and running the game, which means setting up a Python environment. But what if we don’t want that and prefer to just use static analysis?

Since we know that the `GenerateFlagText(x, y)` method is responsible for decoding the flag, let’s try to understand what it does in this context.

```python
def GenerateFlagText(x, y):
    key = x + y*20
    encoded = "\xa5\xb7\xbe\xb1\xbd\xbf\xb7\x8d\xa6\xbd\x8d\xe3\xe3\x92\xb4\xbe\xb3\xa0\xb7\xff\xbd\xbc\xfc\xb1\xbd\xbf"
    return ''.join([chr(ord(c) ^ key) for c in encoded])
```

This method contains a hardcoded encoded Unicode string with characters represented by escape sequences like `\xa5`, which are hexadecimal byte values. It iterates over each character `c` in the `encoded` string, converts `c` to its integer ASCII (or byte) representation using `ord(c)`, and then `XOR`s it with the key to decode the character. The result is converted back into a character using `chr()`.

Since we know the `victory_tile` earlier is `(10, 10)`, we can calculate the `XOR` key: `key = 10 + 10 * 20 = 210`.

Now, let’s assume the challenge didn’t hardcode the plain `victory_tile` value, and we don’t know the `key` value. How could we determine the `key` to decode the flag?

Looking again at the `GenerateFlagText` function, it uses a single `key` to `XOR` with each encoded character. From the challenge description, we also know that **"All flags in this event are formatted as email addresses ending with the @flare-on.com domain"**. This allows us to use a known-plaintext attack to recover the `key` value and decode the entire `encoded` flag.

Here’s the formula we have:
`ord(encrypted_c) ^ key = ord(decrypted_c)`, and we can recover the key using `key = ord(encrypted_c) ^ ord(decrypted_c)`.

We know some `decrypted_c` characters from the plain text email. Let’s take the last character, `decrypted_c = 'm'`, and the last byte character from encoded string is `ord(encrypted_c) = 0xbf`. Plugging this into the formula, we get:

`key = ord(encrypted_c) ^ ord(decrypted_c) = 0xbf ^ ord('m') = 210`

Now we have the `key` value recovered and can use this key to decode the flag.
```bashscript
$ python3
Python 3.13.0 (main, Oct  7 2024, 05:02:14) [Clang 15.0.0 (clang-1500.3.9.4)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> encoded = "\xa5\xb7\xbe\xb1\xbd\xbf\xb7\x8d\xa6\xbd\x8d\xe3\xe3\x92\xb4\xbe\xb3\xa0\xb7\xff\xbd\xbc\xfc\xb1\x\
bd\xbf"
>>> flag = ''.join([chr(ord(c) ^ 210) for c in encoded])
>>> print(flag)
welcome_to_11@flare-on.com
>>>
```

# Conclusion

The first challenge is usually an easy warm-up, and we’ve explored a few approaches to tackle it. In any case, this has been a fun game to play with the kids ^_^
