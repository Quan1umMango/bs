---
date:  2026-03-04
title: Fitting a snake in 512 bytes 
description: "Bootloader asm pain in the ass lol"
layout: page 
collections: ["post", "Assembly", "Bootloader"]
---

So a while back I started reading about Operating Systems using the lovely free book called [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/)\
\
Naturally I wanted to take a shot at writing my own OS.\
That idea lasted until I realised that I was first stuck inside a 512 byte bootloader. So, I decided to see what I can do in just those byte and nothing but the BIOS\
\
I will spare you the details of how the bootloader works, but the reason why it exists is because when your computer gets powered on it the BIOS tests all its components (in a process called POST). It will then look for a bootable device, and when it finds one, loads the first sector (called the boot sector) into memory and starts executing it.\
\
The first sector contains the Bootloader which is a tiny (and I do mean tiny) program whose job it is to load the operating system.
It is a whopping 512 bytes long. Jeez what bloat!\
All of the bootloader code must fit within these bytes. Well it doesn't really need to be big, because what its supposed to do can be achieved in just a couple of lines of assembly code\
\
In this post, I will just be going through what I did and how it all turned out. I will be making the classic snake game, although not really making the complete version of it for the reasons I'll soon explain. I will be sharing a bit of assembly too, but I'll try to explain most of it

## BIOS, what can we do with that?
At this point, we're still inside the BIOS. So what can we actually do?\
Surprisingly enough to make a game. You can draw things to the screen and take in input.\
But the way you program is in assembly. You will be working with registers, stacks and interrupts\
it would be a crime (okay maybe not a *crime* per-se but you get the point) to not mention [Ralf Brown's Interrupt List](http://www.ctyme.com/rbrown.htm)\
The interrupts are how you signal the BIOS to do things for you. Remembering all this isn't realistic, which is where RBIL comes in. \
\
This website will be your best friend when trying to program the BIOS. The reason it's so big is because it documents decades of BIOS, DOS and vendor specific interrupt interfaces all in one place. 

## Bootloading some stuff
Now I told you that the program must fit in 512 bytes. I lied\
Well, technically, it is 512 bytes. But the last two bytes are reserved for the boot signature ``0xaa55``. So you're really left with 510 bytes to work with.\
\
Now the simplest boot sector is the one that simply exists. But that's no fun.\
\
So here I state with no proof: The simplest boot sector is one which loops forever. \
I don't believe this will be controversial, because almost all "making your own OS" resources do start with the infinitely-looping boot sector as one of the first things. \
Doing this is pretty simple:

```x86asm
loop:
    jmp loop

times 510-($-$$) db 0; fills the remaining space with 0, except for the last two bytes 
dw 0xaa55
```

The assembly is not that scary. It creates a label called loop, and just keeps jumping to it, running the same jump instruction again and again and again...you get the point.\
\
The ``times 510-($-$$) db 0;`` simply fills the remaining space with 0.\
The ``$`` represents the current assembly location, and ``$$`` for the start address of the current section. You can think of it as the start of the bootsector.\
So doing ``$-$$`` tells you how many bytes you have used so far (including code and data). We subtract that value from 510, giving the number of padding bytes we need before putting the boot signature.\
The last line ``dw 0xaa55`` **d**efines a **w**ord (which is 2 bytes on x86 environments) with the value 0xaa55.\
\
The semicolon acts like a single line comment.

Without the infinite loop, the cpu continues to execute whatever bytes come after it. Since those are just padding bytes, they would be interpreted as instructions, which probably won't end well\
It also cool to note that the infinite jump itself is just 2 bytes!
\
Assembling this with NASM:
```
nasm -f bin -o ./bin/bl.bin bl.asm 
```
And running using an emulator like qemu:
```
qemu-system-x86_64 -drive file=./bin/bl.bin,format=raw
```
Gives us a screen which does nothing!!

## Bootloaders first words
Printing onto the screen is fairly easy too. We do this by requesting the bios (using interrupts) and filling in specific registers (which act as parameters). The BIOS interrupt routine runs, draws things on the screen, and returns control back to our program.\
\
The interrupt number for video services output is ``0x10`` (it isn't a base-10 but rather a hexadecimal number. The RBIL page uses the h suffix to indicate it's a hexadecimal number. 10h == 0x10).\
Looking at RBIL:\
![RBIL giving so many results](../static/images/posts/snek/rbilwoa.png)\
\
Woah, so much stuff. You'll see that some of the entries are just Int 0x10, while others have a slash followed by other information.\
These other values tell us what to set the registers (like AH, AL, AX, BL etc.) when we want to call that particular routine.\
Depending on the values they contain, the same instruction ``int 0x10`` will do different things\
\
We want to print things to the screen, the entry corresponding to it is the last one ``int 0x10/AH=0Eh`` (0Eh is another way of writing 0x0E)\
\
Clicking the entry, we get this:\
![RBIL page for int 0x10/ AH=0Eh](../static/images/posts/snek/rbil0eh.png)\
(truncated)\
So to get a teletype output, we need to:
- Set AH to 0Eh or 0x0E
- AL to the character to write
- BH is the page number (forget about this for now, its always 0 in our case)
- BL is the foreground color 
##
This is pretty good info, we can now go ahead and write a simple "hello":
```x86asm

mov ah, 0x0E
mov al, 'H'
mov bh, 0
mov bl, 0
int 0x10
mov al, 'E'
int 0x10
mov al, 'L'
int 0x10
int 0x10 ; why do you think we do the same call twice?
mov al, 'O'
int 0x10

loop:
    jmp loop

; And the rest of the code is the same...
```
Pretty nice, we now have an "HELLO" on our screens!!:\
![output for the above code](../static/images/posts/snek/hello.png)

Notice the back to back interrupts after moving 'L' to register ``al``. Why is that? (waiting for you to think...and done!)
Thats because we want to print 'L' twice into the screen. Since ``al`` already contains that letter (and also because we know that this particular interrupt will not change that register), we can simply call the interrupt again!

## Let there be White! And Black! And Green! And...
The astute reader might notice the BL parameter, for the foreground color. This however only works in graphics mode, and we are currently in text mode\
\
Graphics mode allows you to operate on individual pixels, while text mode works with characters placed on a grid. Text mode also let's the BIOS manage the cursor for us.\
I simply went with it because it just looked simpler\
Now then, how do we print colors?? We use  different ``AH`` values:\
![RBIL page for int 0x10/AH=09h](../static/images/posts/snek/rbil09h.png)
![RBIL page for int 0x10/AH=09h but more info](../static/images/posts/snek/rbil09hbig.png)\
Using these, we are pretty much set to draw with different colors. We will use a simplified example just to keep it small:
```x86asm
mov ah, 0x09
mov al, 'H'
mov cx, 1 ; refer the above picture 
mov bh, 0
mov bl, 2
int 0x10
```
And we see\
![output for the above code](../static/images/posts/snek/colors.png)\
Well one thing you might notice is the cursor is at the start of the line, but shouldn't it have been at the end? Well turns out that this interrupt call doesn't handle cursors\
(you might now see why I did not create a complete "hello" example, setting the cursor would be a pain for every letter)\
\
Speaking of cursors.Yes there is one for setting the cursors too, ``int 0x10/AH=02h``, but we will avoid it for now. Read it up for yourself if you're interested!!\
\
Now, let's fill the whole screen with a color. To do this, we simply set the cursor position to be at (0,0) (which is the top left corner) and the ``CX`` argument as number of characters, or equivalently, the screen width multiplied screen height. So we are printing``SCR_W*SCR_H`` amount of characters:
```x86asm
%define SCR_W 80
%define SCR_H 25 

mov dh, 0
mov dl, 0
mov bh, 0
mov ah, 0x02 ; setting the cursor
int 0x10

mov al, 'H'
mov cx, SCR_W * SCR_H 
mov bl, 2 ; green foreground
mov bh, 0 ; black background
mov ah, 0x09 ; typing with color
int 0x10
```
Giving us:\
![output for the above code](../static/images/posts/snek/fillscrh.png)\
Well we can also make it be just green, simply do this in the lower half of the code:
```x86asm
mov al, ' '
mov cx, SCR_W * SCR_H 
mov bx, 0xF2 ; green background, white foreground. Note that bl and bh (both one byte registers) are the lower and higher bits of the
             ; two byte register bx. So Setting bx to 0xF2 would really set bl to be 0x2 and bh to be 0xF
             ; This higher and lower convention is followed in assembly languages to other registers too (ah, al, ax, and so on)
mov ah, 0x09 ; typing with color
int 0x10
```
Giving us:\
![output for the above code](../static/images/posts/snek/fillscrbg.png)\
Pretty cool if you ask me, that you can do all of this at such a low level.

## Snake and friends
We will first define two bytes, just for the snakes x and y position:
```x86asm
player_x db SCR_W/2
player_y db SCR_H/2
```
This will define a byte with value ``SCR_W/2``, called ``player_x``, and other called ``player_y`` to ``SCR_H/2`` which stores the position of the head of the player. The head is now at the center of screen.\
I created a main loop, where we just color the screen green repeatedly for now.\
The code now also moves the cursor to the above defined values and colors them\
I also added a small wait (using ``int 0x15/AH=86h``):\
![head only :D](../static/images/posts/snek/headonly.png)\
Aww look at it!!\
\
The next step would be detecting user input and updating the position according to that. User input is checked by using:\
![RBIL user input](../static/images/posts/snek/rbilui.png)\
Either of these two work, but the top one is synchronous, pauses the program until there is a keystroke to read and clears the keystroke buffer.\
The second one is async, does not pause the program, but does not clear the keystroke buffer.\
So heres what I did:\
I checked for user input using the second interrupt. If there is a user input, I clear the buffer using the first interrupt. If I did not clear the buffer, then I would keep reading the same keystroke over and over\
Lets peek into one of them:\
![result of peeking at the RBIl page](../static/images/posts/snek/keystrokepeek.png)\
You can see that ``AH`` stores the BIOS scan code, and ``AL`` stores the ASCII code.\
We will be using the BIOS scan code instead of the ASCII code. I'll let you figure it out why it may not be the best to use ASCII codes (hint: would our game really need to distinguish between 'A' and 'a'?)\
The scan code represent the physical key pressed, whereas the ASCII code represents the character produced.\
Well now we can detect when we press keys, and depending on those keys I'd want to update the position of the head.\
\
To make this be more akin to the snake game, I created a byte to store the directions, along with defining constants for each direction:
```
%define MOVIN_UP 0b00
%define MOVIN_DOWN 0b01
%define MOVIN_LEFT 0b10
%define MOVIN_RIGHT 0b11

...

dir db MOVIN_UP
``` 
The ``dir`` byte stores the current direction\
Notice how this is a size optimization. We can store which direction we are moving, by using only a byte (really it's just 2 bits, but accessing individual bits will take a lot more instructions, negating our size optimization)\
\
Now depending on the keys pressed, we can change the ``dir`` value. But we must also check for improper moving. For example, if you are currently moving up, you should be allowed to just move down directly. This part again is pretty simple to implement. The function does three things:
- Check if any key is pressed. Return immediately if no keys are pressed.
- Clear the keyboard buffer (so as to not read the same input over and over)
- Compare the scan code constants for keys 
- Change the direction based on the keys pressed (and the current direction of the snake)

Do note that labels do not create blocks or functions by themselves. They are simply addresses. Unless we jump somewhere else, execution continues into the next instruction.
```x86asm
handle_input:
	mov ah, 0x01
	int 0x16
	jz hi_end

	mov ah, 0x0
	int 0x16

	cmp ah, KEY_A ; is A pressed?
	je hi_on_A ; handle for A key

	cmp ah, KEY_W
	je hi_on_W

\cmp ah, KEY_S
	je hi_on_S

    cmp ah, KEY_D
    jne hi_end
hi_on_D:
	cmp byte [dir], MOVIN_LEFT ; are we moving left?
	je hi_end ; if yes, then return. Because we cannot move right if the head is already moving to the left
	mov byte [dir], MOVIN_RIGHT
	jmp hi_end

hi_on_A:
	cmp byte [dir], MOVIN_RIGHT
	je hi_end
	mov byte [dir], MOVIN_LEFT
	jmp hi_end

hi_on_W:
	cmp byte [dir], MOVIN_DOWN
	je hi_end
	mov byte [dir], MOVIN_UP
	jmp hi_end

hi_on_S:
	cmp byte [dir], MOVIN_UP
	je hi_end
	mov byte [dir], MOVIN_DOWN
hi_end:
	ret
```

Assuming all the constants are defined (look at the source code for the program)\
\
We also want to update the snake's position depending on the direction. We do that by simply checking what the ``dir`` value is, and then updating the x or y values accordingly:\
<video src="../static/videos/posts/snek/controllable_guy.mp4" width="500" height="500" controls></video>
\
\
Okay it isn't the prettiest but, seeing it moving is still pretty satisfying\
## Making the Body
To make body we need to have the length of the snake, as well as the x and y coordinates of the body parts themselves:
```x86asm
%define MAX_LEN 4 ; bear with me

player_x times (MAX_LEN+1) db 0
player_y times (MAX_LEN+1) db 0  
len dw 0
```
Here we just modified our previous variables ``player_x`` and ``player_y`` to instead be an array of bytes with ``MAX_LEN+1`` number of elements.\
The way we store it would look something like this in memory:
```
player_x:
[head_x][body_1_x][body_2_x][body_3_x]..

player_y:
[head_y][body_1_y][body_2_y][body_3_y]..
```
So the first element denotes the x or y coordinate of the head, while the other denotes the x and y coordinates of a particular body part. 
The implementation is mostly bookkeeping, moving body parts around, shifting head positions. I will skip over it, but the full code is available in the repo. 

## Apple
Next I added in the apple. Checking if the snake eats the apple was easy, because its just comparing if two 2 1-byte values are the same. That is, checking if ``(x,y)`` of the apple is equal to the ``(x,y)`` of the snake head. Adding the body had to do a bit more than just increasing the length. It had to put set the coordinates of the new body parts just behind the end body part, keeping in the mind the direction it was travelling\
<video src="../static/videos/posts/snek/snake_gets_cancer_lol.mp4" width="500" height="500" controls></video>\
Ummm... wtf happend?

## Oh no...
Ahh we've run out of space. Basically the max length that we decided was too small. Simple fix though, just increase ``MAX_LEN`` to be bigger!\
![no lmao](../static/images/posts/snek/yeahnolmao.png)\
Yeah that won't work. The error comes from this line:
```x86asm
times 510-($-$$) db 0
```
It pretty much tells us that our program is more than 510 bytes big.\
Increasing ``MAX_LEN`` increases the amount of data we use.\
\
The way to solve this would be to optimize our instructions.\
Take a look at this:
```x86asm
mov ax, 0
```
It takes a total of 3 bytes.\
Consider this other snippet, which does the same thing:
```x86asm
xor ax, ax
```
This instead takes only 2 bytes.\
Or say when you have to increment a value by 1. Instead of doing:
```x86asm
add word [val], 1
```
You could instead do:
```x86asm
inc word [val]
```
And save another 1 byte.\
(These instructions are not always interchangeable though, because they can affect CPU flags differently).
These are some of the optimizations that your compilers do for you during machine code generation to reduce code size, and when we only have effectively 510 bytes to work with, these are even more crucial!\
Applying such optimizations to my written code as well as refactoring it to avoid duplicates, I went from having my binary size to be 474, from the 510+ bytes I was using. That's a lott of savings.

## The Incomplete End
I never really finished snake.\
There isn't a win/lose screen.\
The snake also can go out of bounds, in which case it behaves weirdly.
The snake can go inside itself\
Apples can spawn inside the snake.\
\
Most of these aren't difficult to solve. The issue however is that we just don't have enough space for this logic. I had to choose between adding features or allowing my snake to grow longer. Unfortunately, I cannot have both. 

Despite all these limitations, seeing a game run from the boot sector was still pretty cool. It can be a good reminder of how much can be done with such little space. I do plan on moving away from just the boot sector into a real mode environment, which will allow me to do a lot more without fighting for every single byte. Maybe there's a future project idea hidden in there somewhere

## Other Misc Questions 
### Q. Why don't I use the stack to store the snake's body positions?
A. Because I got the idea in the middle of writing the post lol. It could be done though, and it would save a lot of bytes. Another improvement would be to store the player x and y in a single 16-bit value.

### Q. Why is the game flickering?
A. I think it's got to do with me printing into the screen a lot of times per second, and because there isn't any frame buffer.\
There is however another anomaly that needs explaining. When moving to the top of the screen, the flicker gets really bad. But anywhere below it's not really noticeable. Why that happens, I have no idea.

### Q. Why does the snake look like its moving vertically at a much faster rate than moving horizontally?
A. This is due to the fact that the text mode characters are taller than they are wide. So it's drawing a lot more physical distance when it draws the snake vertically rather than when it's drawing horizontally. I reckon that equal distances will be covered in equal amounts of time, if you decide to measure it.




