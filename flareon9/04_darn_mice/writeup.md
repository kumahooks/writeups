The challenge's text is as follows:
```
"If it crashes its user error." -Flare Team
```

This challenge gives us a zip called "04_darn_mice".
Opening the .exe inside the zip, it immediately crashes.

Okay, fine, that's how it is. Let's Open Olly already and debug it. Our first goal is finding out what is happening to the crashes. Flare clearly states it's an user error, so I wonder, what kind of input are we doing that causes it to crash?

If we search for string references, we can already see some strings that were supposed to show us.

![image](https://user-images.githubusercontent.com/69819027/201487734-e95f20a6-34ed-4029-bf1e-5d630b09dec8.png)


Let's follow the first one and see where it leads.
We immediately fall in a piece of code right after (what seems to be) a gigantic array initialization:

![image](https://user-images.githubusercontent.com/69819027/201487738-68b19bf0-72b9-4402-b94d-9bb4aea4a535.png)


Okay, first steps first. Let's trace-back and see what is calling this function:

![image](https://user-images.githubusercontent.com/69819027/201487759-7d082597-87cd-4d68-b8f2-b2199672d424.png)


This looks very interesting, right? If the user is causing the crash, then an input is somehow causing it. What kind of input are we doing by just opening the software? Well, duh! It's arguments.
And that CMP instruction from the code above is doing exactly that: checking if we have 2 arguments. If we run it with only one argument (just the file name), it immediately exits!

Okay, let's run it with a big memorable argument then: "wearedoingflareonctf".
Once ran with such an argument, we get past the initial argument check into the function that has that big array we saw before.

The very first thing I did when I got to this point, was after breaking at that said function initialization, I just ran(F9) the code again.
Well, that didn't work out very well for me.
Actually, it wasn't that bad:

![image](https://user-images.githubusercontent.com/69819027/201487769-d0dc1e25-39e2-4921-b2a5-111ac4e57811.png)


As you can see, two strings were shown to us in the console (and it didn't crash before showing them!!), but it immediately followed to an access violation code, trying to write at 0x00000077.
Well, that didn't work out very well. Let's get back to the BP at the start of that function and learn how to walk before we run.

As you can see, the function starts defining some values in what I assume is a struct.
We can more or less define that piece of code as:
```C
struct bytes {
	DWORD var1 = eax; 				// EBP-4
	const char data[37] = 			// EBP-28
		"\x50\x5E\x5E\xA3
		 \x4F\x5B\x51\x5E
		 \x5E\x97\xA3\x80
		 \x90\xA3\x80\x90
		 \xA3\x80\x90\xA3
		 \x80\x90\xA3\x80
		 \x90\xA3\x80\x90
		 \xA3\x80\x90\xA2
		 \xA3\x6B\x7F\x00";
	DWORD var2 = 0; 				// EBP-2C
	DWORD var3 = 0;					// EBP-30
	DWORD var4 = 0;					// EBP-34
}
```

So it's filling what seems to be a char array with 36 bytes.
Tracing forward, we see a function being called right after pushing the texts we saw on screen. We didn't crash nor had any violation when running the two first strings, so I stepped over then.
The following code is ran in a seemingly error-free manner and continues until we enter in a loop (with that JMP instruction down there):

![image](https://user-images.githubusercontent.com/69819027/201487812-80e3343b-fdcc-426d-af07-2f8b3f23e82d.png)


Now we are inside a loop and we haven't crashed yet. So far so good.
Until now, all we did was checking if we had two arguments, checking if the argument is the correct size and showed some texts on screen.

Tracing forward inside that loop, we can write a pseudo-code of that first part of the loop as:
```C
while ((my_struct->counter)++ != 24) {
	if (my_struct->data[my_struct->counter] == '\0') break;
	if (argument[my_struct->counter] == '\0') break;

	...
}
```
So it's looping through both the argument and the data array. With that information, we can already assume it's gonna do some kind of comparison or operation between both datas.

![image](https://user-images.githubusercontent.com/69819027/201487839-ac03c8ec-671e-4ab0-9d46-7cb670ceffdc.png)


After going through checking if both arrays positions are valid, it calls VirtualAlloc with a PAGE_EXECUTE_READWRITE argument, storing the allocated address at EBP-34(the struct's var4!).

What does this mean, exactly?
Well, instead of just allocating for read/write, it's also gonna supposedly use execute. Does this mean this piece of code is being called later? (spoiler: yes!)
You can see a few instructions below a CALL instruction for DWORD PTR SS:[EBP-34].

And not coincidentally, it's also the part where a violation is occurring, because the code is calling a part of the code that has weird instructions. Let's find out what's exactly happening here:
1. It gets the allocated address in memory (EBP-34)
2. Gets the current counter of the loop (EBP-2C)
3. Gets the byte of the position relative to the counter in the data array (EBP+EAX-28)
4. Gets the byte of the position relative to the counter in the argument
5. Sums data array character and argument character
6. Writes the result sum at the memory page (in EBP-34)
7. Calls EBP-34

The violation is at step 7. The reason for that violation is because (arg[index] + data[index]) results in an assembly instruction that violates on execution. What kind of result is desired to resume the execution of the code normally?
Well that answer is obvious: the RET instruction!
What that means is, once we sum a byte from our argument with a byte of the encoded data, we must result in a return instruction at the given address. Which only means that all we have to do to find out the correct argument for the desired execution is by subtracting 0xC3(RET) by the encoded data!

We can easier visualize all that in here:

![image](https://user-images.githubusercontent.com/69819027/201487874-423f915c-64f3-4e58-8414-0e6a074b1583.png)


A quick C++ solution:
```C++
#include <iostream>

int main(int argc, char* argv[]) 
{
    const char encoded_data[37] = "\x50\x5E\x5E\xA3\x4F\x5B\x51\x5E\x5E\x97\xA3\x80\x90\xA3\x80\x90\xA3\x80\x90\xA3\x80\x90\xA3\x80\x90\xA3\x80\x90\xA3\x80\x90\xA2\xA3\x6B\x7F\x00";
    char translated_data[37];

    for (int i = 0; encoded_data[i] != '\0'; ++i) {
        translated_data[i] = 0xC3 - encoded_data[i];
    }

    std::cout << translated_data;
    return 0;
}
```
The resulting string is: "see three, C3 C3 C3 C3 C3 C3 C3! XD"

Once we use that string as an argument for the program, we then obtain the flag:

![image](https://user-images.githubusercontent.com/69819027/201487899-7d39bb72-3e34-45b4-8aba-a49a5fe3e5b5.png)


I really laughed at this flag! It was a very refreshing solution :)
