The problem's text is as follows:
```
You got a question? Ask the 8 ball!
```

This challenge gives us a zip called "03_magic8ball".
When we unzip it, we are greeted with a .exe and a bunch of libraries.
Opening the .exe, we stumble upon this:

![image](https://user-images.githubusercontent.com/69819027/201486975-b19c0ab7-5e0b-45be-acf4-43967eefd976.png)


Ok, sure. It is exactly what the title says. A magic 8 ball. Noticeably, we don't only ask questions, but can also shake it. They surely wouldn't do a shake mechanic if it wasn't important... right?

Proceeding, let's open it with Olly and start our thing.

First of all, when we ask it a question, a text appear inside the ball. At first, it seems like it's a random text:

![image](https://user-images.githubusercontent.com/69819027/201486980-dc51861d-1d5f-4334-8ec2-3658c5fb0eaa.png)


That gives us two informations:
1. It has a list of those strings randomly appearing on your screen
2. It invokes some kind of random function

I used the second information for simplicity, and it ended up being correct. Let's search for intermodular calls in Olly and find the random function it's using. The function is rand(). Following it, it takes us here:

![image](https://user-images.githubusercontent.com/69819027/201486992-7a192219-884c-4c66-9686-79d09a0165fc.png)


To be completely honest, that piece of code didn't give me at first much information about what I wanted. I wanted a way to find out what the software required from me. A text? A key sequence for the ball? Both?
Something in that code caught my curiosity, though:
It was accessing variables in a struct-like pattern, in the address at ESI. Once I checked that address in my dump to see if there was anything of value in there, I stumbled upon this:

![image](https://user-images.githubusercontent.com/69819027/201486999-3a2d3ede-8740-4bbc-86e0-8e46ce935ca7.png)


That must be a static string. Now I knew something I wanted to search. My First step was to trace-back all the way from the start of the software execution, to find out exactly when it wrote that text.

Tracing back all the way to the start of that function, we can see it comes from a static pointer at 0xEA6090. Searching for a quick references of that pointer, we get to:

![image](https://user-images.githubusercontent.com/69819027/201487007-b37ba665-326c-43cb-8304-24e5701a97a7.png)


So this is where everything starts. Upon restart of the program, we notice that string isn't written there yet.
Tracing forward the execution of the program, we find the function:

![image](https://user-images.githubusercontent.com/69819027/201487021-71839cca-e890-4c9a-a52b-ca80180fd0f0.png)


Reading all the way down of this, you can spot something very clearly:

![image](https://user-images.githubusercontent.com/69819027/201487023-894eed45-0795-4ab5-b2c6-419108406c75.png)


Recognizing the pattern, we found when it writes the text.
Right clicking the first MOV instruction and searching for references to the address constant, the only other place it's referencing it is:

![image](https://user-images.githubusercontent.com/69819027/201487037-4a01a086-308f-46dc-8d10-d2c495e49a6d.png)


Reading the code at question and the code right above it, we can reach the conclusion that this is the part of the assembly tasked to check the user's given text AND the part above it is checking some kind of movement sequence for the ball. When we randomly press the arrow keys and then send a text, it's registered at ECX the sequence of the keys we did, e.g (LDUL)

The code then compares byte by byte, to this sequence: 4C, 4C, 55, 52, 55, 4C, 44, 55, 4C
Or translated, we have: L, L, U, R, U, L, D, U, L

Given all that information, we can just enter the sequence above with the text "gimme flag pls?". The key will be shown as:

![image](https://user-images.githubusercontent.com/69819027/201487039-64f663c5-9524-446d-a4fb-f5edb6280004.png)

