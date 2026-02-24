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



# Lets pull out GDB


# Game Plan


# Trial and Error


# The computer playing for us 


# Win!




