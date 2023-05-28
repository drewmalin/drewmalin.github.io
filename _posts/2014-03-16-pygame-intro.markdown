---
layout: post
title: "Game Development with Pygame and Pyganim"
date: 2014-03-16 21:00:00
categories: [python]
permalink: :year-:month-:day-:title
---

Another weekend, another project!

I read an article recently about [Pyganim](http://inventwithpython.com/pyganim/), a Python sprite animation module meant for use with [Pygame](http://pygame.org/news.html). After browsing through the API and some example code snippets, I figured I would give it a try myself.

### Step 1: Install Pygame

The [downloads](http://pygame.org/download.shtml) section of the Pygame website includes instructions for installing the library on various operating systems. Although the installation instructions are well documented, several attempts to follow them on my Mac went nowhere. In the end I was finally able to install Pygame using homebrew:

```
brew install pygame
brew install sdl sdl_image sdl_mixer sdl_ttf portmidi
```

### Step 2: Grab Pyganim

The 'installation' of pyganim is as simple as putting pyganim.py (located [here](http://inventwithpython.com/pyganim/index.html#download)) in the appropriate location. Here is what my filestructure looks like for this project:

```
|- .
|- main.py
|- pyganim.py
|- resources/
```

### Step 3: Hello, world!

With everything set to start coding, here's a basic snippet to hit the ground running:

```python
import pygame, sys
from pygame.locals import *

pygame.init()
window = width, height = (500, 400)
windowSurface = pygame.display.set_mode(window)
BLUE = (0, 0, 255)

while True:
    for event in pygame.event.get():
        if event.type == QUIT:
            pygame.quit()
            sys.exit    
                                                    
    windowSurface.fill((0, 0, 0))
    pygame.draw.circle(windowSurface, BLUE, (width/2, height/2), 20, 0)
    pygame.display.flip()
```

Boom! A blue circle in a sea of blackness!

With that, we now need a task... a direction... Seeing as one of the most basic things a player would expect from a game is to be able to move the main character around, let's make a main character! I present to you, the aptly named, Dude:

![Idle Dude]({{ site.url }}/assets/2014-03-16-pygame-intro/dudestill.gif)

I will spare you the several hours my very-much-non-artistic-brain spent hammering away at making this little guy mobile. Behold, Dude has been given life:

![Run Forward Dude]({{ site.url }}/assets/2014-03-16-pygame-intro/duderunfront.gif) ![Run Back Dude]({{ site.url }}/assets/2014-03-16-pygame-intro/duderunback.gif) ![Run Right Dude]({{ site.url }}/assets/2014-03-16-pygame-intro/duderunright.gif) ![Run Left Dude]({{ site.url }}/assets/2014-03-16-pygame-intro/duderunleft.gif)

Now it's time to throw these frames into pyganim to see what it can do:

```python
import pygame, sys
import pyganim
import math
from pygame.locals import *

pygame.init()
window = width, height = (320, 240)
windowSurface = pygame.display.set_mode(window)

walkLeftAnim = pyganim.PygAnimation(
		[('resources/duderunleft/duderunleft0.tiff', 0.1),
		 ('resources/duderunleft/duderunleft1.tiff', 0.1),
		 ('resources/duderunleft/duderunleft2.tiff', 0.1),
		 ('resources/duderunleft/duderunleft3.tiff', 0.1),
		 ('resources/duderunleft/duderunleft4.tiff', 0.1),
		 ('resources/duderunleft/duderunleft5.tiff', 0.1),
		 ('resources/duderunleft/duderunleft6.tiff', 0.1),
		 ('resources/duderunleft/duderunleft7.tiff', 0.1)])

walkRightAnim = pyganim.PygAnimation(
		[('resources/duderunright/duderunright0.tiff', 0.1),
		 ('resources/duderunright/duderunright1.tiff', 0.1),
		 ('resources/duderunright/duderunright2.tiff', 0.1),
		 ('resources/duderunright/duderunright3.tiff', 0.1),
		 ('resources/duderunright/duderunright4.tiff', 0.1),
		 ('resources/duderunright/duderunright5.tiff', 0.1),
		 ('resources/duderunright/duderunright6.tiff', 0.1),
		 ('resources/duderunright/duderunright7.tiff', 0.1)])

walkFrontAnim = pyganim.PygAnimation(
		[('resources/duderunfront/duderunfront0.tiff', 0.1),
		 ('resources/duderunfront/duderunfront1.tiff', 0.1),
		 ('resources/duderunfront/duderunfront2.tiff', 0.1),
		 ('resources/duderunfront/duderunfront3.tiff', 0.1),
		 ('resources/duderunfront/duderunfront4.tiff', 0.1),
		 ('resources/duderunfront/duderunfront5.tiff', 0.1),
		 ('resources/duderunfront/duderunfront6.tiff', 0.1),
		 ('resources/duderunfront/duderunfront7.tiff', 0.1)])

walkBackAnim = pyganim.PygAnimation(
		[('resources/duderunback/duderunback0.tiff', 0.1),
		 ('resources/duderunback/duderunback1.tiff', 0.1),
		 ('resources/duderunback/duderunback2.tiff', 0.1),
		 ('resources/duderunback/duderunback3.tiff', 0.1),
		 ('resources/duderunback/duderunback4.tiff', 0.1),
		 ('resources/duderunback/duderunback5.tiff', 0.1),
		 ('resources/duderunback/duderunback6.tiff', 0.1),
		 ('resources/duderunback/duderunback7.tiff', 0.1)])

walkLeftAnim.play()
walkRightAnim.play()
walkFrontAnim.play()
walkBackAnim.play()

target = [0, 0]
current = [0, 0]
anim = walkFrontAnim

while True:
	for event in pygame.event.get():
		if event.type == QUIT:
			pygame.quit()
			sys.exit
		elif event.type == MOUSEBUTTONDOWN and event.button == 1:
			target = event.pos

	# Clear screen 
	windowSurface.fill((0, 0, 0))
 
 	# Perform character movement
 	if math.fabs(current[0] - target[0]) > 5 or math.fabs(current[1] - target[1]) > 5:
 		sqrtdist = math.sqrt(((target[0] - current[0]) ** 2) + ((target[1] - current[1]) ** 2))
 		moveX = (target[0] - current[0]) / sqrtdist
 		moveY = (target[1] - current[1]) / sqrtdist
 		if math.fabs(moveX) > math.fabs(moveY):
 			if moveX > 0:
 				anim = walkRightAnim
 			else:
 				anim = walkLeftAnim
 		else:
 			if moveY > 0:
 				anim = walkFrontAnim
 			else:
				anim = walkBackAnim
 		current[0] += moveX
 		current[1] += moveY
 		anim.blit(windowSurface, current)
 	# If no movment, idle the animation at the first frame
 	else:
 		anim.blitFrameNum(0, windowSurface, current)

 	# Swap the buffers
	pygame.display.flip()
```

And the result?

![Dude Play]({{ site.url }}/assets/2014-03-16-pygame-intro/dudeplay.gif)

[Full project](https://github.com/drewmalin/pygame_and_pyganim)