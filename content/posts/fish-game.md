+++
title = "Kaspersky CTF 2025 - Fish Game"
date = "2025-07-31T23:48:38+05:30"
author = ""
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["game", "rev","godot"]
keywords = ["", ""]
description = "writeup for Fish Game from Kaspersky CTF 2025"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

# Fish Game

Domain: Reverse Engineering

Points: 50

## Given Information
> I want to play a game together. I also like fish.
>
> https://storage.yandexcloud.net/kasperskyctf/game-9ae73bf428610fbc.exe

## Solution

We are given a game made with Godot where our score increases by attacking smaller fish but the game ends if a fish touches us. Once dead, the game submits the score to a server and it checks if its higher than 5000. If the check passes, the flag is displayed.

![fish game](/images/fishgame.png)

We can use a tool called [gdsdecomp](https://github.com/GDRETools/gdsdecomp/) that can decompile the game to give godot project files.

On opening the project in godot, we can see the following code in `score.gd`.
![score.gd](/images/scoregd.png)

```swift
func _ready() -> void :
    $Control / VBoxContainer / Label.text = "Score: " + str(Registry.asd / 123)
```
This function forms the score text which is displayed on death. It is seen that score is stored in `Registry.asd` as 123 times the actual score.

The rest of `score.gd` contains code that sends and receives http requests that validate the score and check if its above the requirement to get a flag. It is not important for solving the challenge as we can just increase the score we get per kill.

In `enemy.gd` the following function seems to actually modify the score variable on kill:
```swift
func _on_body_entered(body: Node2D) -> void :
    if body is Player:
        var x = 0
        for xx in body.get_children():
            if xx is Enemy:
                x += 1
        Registry.asd = x * 123
        get_tree().change_scene_to_file("res://lose.tscn")
```
It increments score by 1 each time we kill a smaller fish, then stores it in `Registry.asd` as `score*123`.

So if we change the score to increment by 10000 with `x+=10000`, all we need is 1 kill to meet the requirement. So I made this change and ran the game in the engine.

![score](/images/score.png)


![flag](/images/flag.png)

Flag: `kaspersky{g1v3_m4n_4_f1sh1ng_r0d_h3_w0u1d_b3_f4d}`

