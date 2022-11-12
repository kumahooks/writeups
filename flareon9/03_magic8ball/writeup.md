The problem's text is as follows:
```
You got a question? Ask the 8 ball!
```

This challenge gives us a zip called "03_magic8ball".
When we unzip it, we are greeted with a .exe and a bunch of libraries.
Opening the .exe, we stumble upon this:
[image magic8ball1.png]

Ok, sure. It is exactly what the title says. A magic 8 ball. Noticeably, we don't only ask questions, but can also shake it. They surely wouldn't do a shake mechanic if it wasn't important... right?

Proceeding, let's open it with Olly and start our thing.

First of all, when we ask it a question, a text appear inside the ball. At first, it seems like it's a random text:
[image magic8ball2.png]

That gives us two informations:
1. It has a list of those strings randomly appearing on your screen
2. It invokes some kind of random function

I used the second information for simplicity, and it ended up being correct. Let's search for intermodular calls in Olly and find the random function it's using. The function is rand(). Following it, it takes us here:
[image magic8ball3.png]

To be completely honest, that piece of code didn't give me at first much information about what I wanted. I wanted a way to find out what the software required from me. A text? A key sequence for the ball? Both?
Something in that code caught my curiosity, though:
It was accessing variables in a struct-like pattern, in the address at ESI. Once I checked that address in my dump to see if there was anything of value in there, I stumbled upon this:
[image magic8ball4.png]

That must be a static string. Now I knew something I wanted to search. My First step was to trace-back all the way from the start of the software execution, to find out exactly when it wrote that text.

Tracing back all the way to the start of that function, we can see it comes from a static pointer at 0xEA6090. Searching for a quick references of that pointer, we get to:
[image magic8ball5.png]

So this is where everything starts. Upon restart of the program, we notice that string isn't written there yet.
Tracing forward the execution of the program, we find the function:
[image magic8ball6.png]

Reading all the way down of this, you can spot something very clearly:
[image magic8ball7.png]

Recognizing the pattern, we found when it writes the text.
Right clicking the first MOV instruction and searching for references to the address constant, the only other place it's referencing it is:
[image magic8ball8.png]

Reading the code at question and the code right above it, we can reach the conclusion that this is the part of the assembly tasked to check the user's given text AND the part above it is checking some kind of movement sequence for the ball. When we randomly press the arrow keys and then send a text, it's registered at ECX the sequence of the keys we did, e.g (LDUL)

The code then compares byte by byte, to this sequence: 4C, 4C, 55, 52, 55, 4C, 44, 55, 4C
Or translated, we have: L, L, U, R, U, L, D, U, L

Given all that information, we can just enter the sequence above with the text "gimme flag pls?". The key will be shown as:
[image magic8ball9.png]
