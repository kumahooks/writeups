The challenge's text is as follows:
```
FLARE FACT #823: Studies show that C++ Reversers have fewer friends on average than normal people do. That's why you're here, reversing this, instead of with them, because they don't exist.

We’ve found an unknown executable on one of our hosts. The file has been there for a while, but our networking logs only show suspicious traffic on one day. Can you tell us what happened?
```

Okay, so immediately we can see we have an executable and a pcap file. Quickly analyzing the pcap, we notice the first POST the executable made was with an encrypted message `ydN8BXq16RE=`. The response it received was a big encrypted blob right after.
Knowing these and with not much more to peek in the pcap, let's analyze the executable.

The first behavior we can notice is that the software opens and nothing seems to happen. Searching for the reason why this is happening, we end up a bit after the start of the main, falling in a seemingly infinite sleep routine.
The reason for that is the comparison of whatever was returned in the `CALL 0x00294570`. If that is not 0xF, we fall into the sleep.

Now, the first thing I ever did with this was simply setting eax to 0xF and proceeding my analysis. While this wasn't absolutely wrong, I had problems further down the road because understanding this value is very important later on. So let's try to at least point out how eax is being set.

If we get further in the `0x00294570` call, we can quickly see it's doing some date operations. At [EBP+8] we have the start of a struct that has a specific date saved. At this moment I did some kind of thought exercise: thinking how the threat actor acted in this case. We can see a function doing a bunch of operations using date values and then returning a value that decides whether the attack should be run or not. If we read the information the challenge gives us, this threat only ran one specific day. So I thought it was good to assume that's what it's checking, if we are running at the exact target day. Luckily, that wasn't very hard to poke more at: checking for Intermodular Calls, we quickly find how it's gathering the information: GetLocalTime.

This function returns the following struct:
```cpp
typedef struct _SYSTEMTIME {
	WORD wYear;
	WORD wMonth;
	WORD wDayOfWeek;
	WORD wDay;
	WORD wHour;
	WORD wMinute;
	WORD wSecond;
	WORD wMilliseconds;
} SYSTEMTIME, *PSYSTEMTIME, *LPSYSTEMTIME;
```

At the moment I'm writing this, it's 26/10/2022. Looking at where the date is stored (it's not static!), I gather:

![image](https://user-images.githubusercontent.com/69819027/201500800-b1e541ef-7118-45bc-8bac-51eec9608b28.png)


I decided to do a quick check. I changed my day and month to the same as the pcap (Jun 14). And poof! The executable skipped the sleep and we are good to start analyzing the rest. This behavior matches the original attack's situation, where it ran once one day and never again.

Proceeding with the analysis, we know the threat at some point sends an encrypted message. Our goal now would be finding where that message is encrypted, so let's get back to the function that checked for the Jun 14 day.

Further down the code, we see it sends the first encrypted data `ydN8BXq16RE=` at:

![image](https://user-images.githubusercontent.com/69819027/201500801-b5ccf337-abb5-4eb6-a71f-9e1206d05ae2.png)


What caught my attention at this specific moment is that at ESI we have four numbers appended with "ahoy". I let it be for now until I understood what was up with it, but I'm starting to think it's either the encoded data or some seed for the encryption.
Deeper in the function above we can see where it exactly encodes the data and returns the result string at EAX:

![image](https://user-images.githubusercontent.com/69819027/201500809-3ea97068-5817-44e7-b634-189182db971d.png)


Every restart we notice a different encoded string, thus we can assume some kind of pseudo-randomness is at game. A bad malware developer would use time as a seed for the encoding, so I bet all my money on that. Back to the pcap file, the arrival time we had was as follows:

![image](https://user-images.githubusercontent.com/69819027/201500810-770c96aa-7a02-461d-a911-d6c949d8d29e.png)


Finding the correct time in this situation is a bit tricky, because there is latency. But we can safely assume 14:36 is the correct values or at least hope they are (worst case it will be a second or two difference). The milliseconds though are a different story, it will probably be something close to 649. The correct time after a few attempts is 14:36:637.

While I was trying to find the correct milliseconds, I noticed something interesting. Remember the four digits+ahoy that we thought it would be either a seed or the decoded data? Well seems like the four digits are set by the second call right after the GetLocalTime. The digits respective to our correct date is 11950, where 1195ahoy is the string used later on. Changing eax to 11950 seems to be enough (instead of having to change the minutes, second and milliseconds every run). My quick patch-solution for a better quality of life was as follows:

Inside this call:

![image](https://user-images.githubusercontent.com/69819027/201500817-a493244e-69fd-4588-a50b-381bebf893c0.png)


I changed the code to:

![image](https://user-images.githubusercontent.com/69819027/201500820-0437729e-8f85-4831-a618-7c099da5729e.png)


The binary code is:
```
B8 AE 2E 00 00 3E 66 C7 45 E6 0E 00 3E 66 C7 45 E2 06 00 C3 CC CC CC CC CC CC CC CC CC CC CC CC CC
```

This quick patch makes sure our date is correct while also setting our EAX to 11950, which fixes the encoding later on.

Okay, we managed to find the day we had to be in order to run the attack and the exact time it used to seed the data with. What now?
The next thing that happened once the threat sent the `ydN8BXq16RE=` data is receive a response from the server with an also encrypted data. Since we are mimicking step-by-step of what happened at the attack, we will probably also need to do that.

Considering all this, we need to find where the attack handles the server response. One-stepping through the execution, there are some Http function calls, more specifically:

![image](https://user-images.githubusercontent.com/69819027/201500824-f580abf1-cb25-494f-a670-e1f257f0d54d.png)


As you can see, the response we get is:

![image](https://user-images.githubusercontent.com/69819027/201500825-ca0015f8-858a-4834-927e-00398e5f3e0e.png)


This seems a bit odd, since it's a different response to what we were expecting to receive. But the thing is, Flare-On already caught this threat. It already patched whatever response it was supposed to be sending. So we will obviously not receive the exact response of that time, because it's not exploitable anymore. But we have the exploited-response! It's in our pcap file. Considering this, instead of giving our malware the fixed response, we can just feed it the response it was expecting beforehand. Maybe it decrypts it for us?

The expected data from the pcap file is (don't forget the null in the end!):
```
54 64 51 64 42 52 61 31 6E 78 47 55 30 36 64 62 42 32 37 45 37 53 51 37 54 4A 32 2B 63 64 37 7A
73 74 4C 58 52 51 63 4C 62 6D 68 32 6E 54 76 44 6D 31 70 35 49 66 54 2F 43 75 30 4A 78 53 68 6B
36 74 48 51 42 52 57 77 50 6C 6F 39 7A 41 31 64 49 53 66 73 6C 6B 4C 67 47 44 73 34 31 57 4B 31
32 69 62 57 49 66 6C 71 4C 45 34 59 71 33 4F 59 49 45 6E 4C 4E 6A 77 56 48 72 6A 4C 32 55 34 4C
75 33 6D 73 2B 48 51 63 34 6E 66 4D 57 58 50 67 63 4F 48 62 34 66 68 6F 6B 6B 39 33 2F 41 4A 64
35 47 54 75 43 35 7A 2B 34 59 73 6D 67 52 68 31 5A 39 30 79 69 6E 4C 42 4B 42 2B 66 6D 47 55 79
61 67 54 36 67 6F 6E 2F 4B 48 6D 4A 64 76 41 4F 51 38 6E 41 6E 6C 38 4B 2F 30 58 47 2B 38 7A 59
51 62 5A 52 77 67 59 36 74 48 76 76 70 66 79 6E 39 4F 58 43 79 75 63 74 35 2F 63 4F 69 38 4B 57

67 41 4C 76 56 48 51 57 61 66 72 70 38 71 42 2F 4A 74 54 2B 74 35 7A 6D 6E 65 7A 51 6C 70 33 7A
50 4C 34 73 6A 32 43 4A 66 63 55 54 4B 35 63 6F 70 62 5A 43 79 48 65 78 56 44 34 6A 4A 4E 2B 4C
65 7A 4A 45 74 72 44 58 50 31 44 4A 4E 67 3D 3D 00
```

One-stepping the execution after the data is fed, we are then greeted with the flag right after the decryption is done:

![image](https://user-images.githubusercontent.com/69819027/201500829-0de286c2-bfc8-4ba0-9679-14e736ec1195.png)




Also, as an extra note, one can notice in the pcap file that two POSTs are done in the attack, but we retrieved the flag with just half of it. What happens if we do the whole execution? :)
