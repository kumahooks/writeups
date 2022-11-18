The problem's text is as follows:
```
Welcome to Flare-On 9!

You probably won't win. Maybe you're like us and spent the year playing Wordle. We made our own version that is too hard to beat without cheating.

Play it live at: http://flare-on.com/flaredle/
```

With the introduction above and the link, Flare-On also gives us a zip called flaredle.7z

The files we are given are:
* index.html
* script.js
* words.js

Opening words.js we stumble with the following words list:
```js
export const WORDS = ['acetylphenylhydrazine',
	'aerobacteriologically',
	'alkylbenzenesulfonate',
	'aminoacetophenetidine',
	'anatomicopathological',
	'anemometrographically',
	'anthropoclimatologist',
	'anthropomorphological',
	'anticonstitutionalism',
	'anticonstitutionalist',
	'antienvironmentalists',
	'antiinstitutionalists',
	'antimaterialistically',
	'antinationalistically',
	'antisupernaturalistic',
	...
	]
```

Checking what script.js is about:
```js
import { WORDS } from "./words.js";

const NUMBER_OF_GUESSES = 6;
const WORD_LENGTH = 21;
const CORRECT_GUESS = 57;
let guessesRemaining = NUMBER_OF_GUESSES;
let currentGuess = [];
let nextLetter = 0;
let rightGuessString = WORDS[CORRECT_GUESS];

function initBoard() {
    let board = document.getElementById("game-board");

    ...
}
```

We can already see the variables "CORRECT_GUESS" and "rightGuessString".
Following "rightGuessString" we have a very big function called "checkGuess":

```js
function checkGuess () {
    let row = document.getElementsByClassName("letter-row")[NUMBER_OF_GUESSES - guessesRemaining]
    let guessString = ''
    let rightGuess = Array.from(rightGuessString)
    
    ...
}
```

With no further peeking, We can check which word is the #57 word of the list and it is: flareonisallaboutcats

Upon submition of that word in the wordle game, we are greeted with the flag: 

![image](https://user-images.githubusercontent.com/69819027/201485717-e7e4688d-b792-44bd-bf8a-5b01772a783a.png)
