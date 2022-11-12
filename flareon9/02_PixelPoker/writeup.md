The problem's text is as follows:
```
I said you wouldn't win that last one. I lied. The last challenge was basically a captcha. Now the real work begins. Shall we play another game?
```

This challenge gives us a zip called "02_PixelPoker".
Upon further inspection, the readme.txt in that zip says:
```
Your goal is simple: find the correct pixel and click it
```

Nice! Then our goal is to simply find (or find a way around it) a coordinate in the binary.

Opening the binary with Ollydbg, we can start to test what the binary does.
So from playing the game quickly, we realize that once we click any place around the window, the game adds 1 to the total amount of wrong attempts. Once we reach 10 wrong attempts, we are greeted with the following message:

[image pixelpoker1.png]

My first instinct once I read this, is to dump all string references in Olly and find that message.
This is what greets me:

[image pixelpoker2.png]

From a quick look of this part of the code, we realize it's checking if the value stored in eax is 10(0A), if it isn't, it takes the jump:
```
JNZ SHORT 0040146F
```

From here onward, it's time to check if the clicked pixel's position was correct.
This is the following code greeting us immediately after the jump:
```
0040146F   40                             INC EAX
00401470   33D2                           XOR EDX,EDX
00401472   A3 98324100                    MOV DWORD PTR DS:[413298],EAX
00401477   A1 04204100                    MOV EAX,DWORD PTR DS:[412004]
0040147C   8B35 80324100                  MOV ESI,DWORD PTR DS:[413280]
00401482   F7F6                           DIV ESI
00401484   3BFA                           CMP EDI,EDX
00401486   0F85 CA000000                  JNZ PixelPok.00401556
0040148C   A1 08204100                    MOV EAX,DWORD PTR DS:[412008]
00401491   33D2                           XOR EDX,EDX
00401493   8B0D 84324100                  MOV ECX,DWORD PTR DS:[413284]
00401499   F7F1                           DIV ECX
0040149B   3BDA                           CMP EBX,EDX
0040149D   0F85 AB000000                  JNZ PixelPok.0040154E
004014A3   33FF                           XOR EDI,EDI
```

Noticeably, both jumps in this piece of code succeeds a check. Both of those are checking the coordinates, and if they are not correct, they jump out and execute the "wrong attempt" code.
The easiest way to see if this is the case, is filling both jumps with NOPs.

Once we do that, we are greeted with:

[image pixelpoker3.png]