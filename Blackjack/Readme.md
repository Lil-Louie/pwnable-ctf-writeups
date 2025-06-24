#  Pwnable CTF: Blackjack  
**Category:** Logic / Application Security  
**Difficulty:** Beginner–Intermediate  
**Author:** Luis Galvez  
**Date:** June 2025  

---

## Challenge Summary  
The goal was to become a millionaire within the game. To do that, we needed to exploit a logic flaw in the betting system of a simple C-based Blackjack game.

---

## Files / Reference  
- Original C code:  
  http://cboard.cprogramming.com/c-programming/114023-simple-blackjack-program.html

---

## Description (from CTF prompt)  
> Hey! Check out this C implementation of a Blackjack game!  
> I found it online:  
> http://cboard.cprogramming.com/c-programming/114023-simple-blackjack-program.html  
>   
> I like to give my flags to millionaires.  
> How much money you got?  
>   
> `ssh blackjack@pwnable.kr -p2222` (password: `guest`)

---

## Vulnerability Analysis  

The program allows the user to input **any integer** for the bet — including **negative values**. The vulnerability lies in how losses are handled:

### Vulnerable Snippet:
```c
int betting() {
    printf("\n\nEnter Bet: $");
    scanf("%d", &bet);
 
    if (bet > cash) {
        printf("\nYou cannot bet more money than you have.");
        printf("\nEnter Bet: ");
        scanf("%d", &bet);
        return bet;
    }
    else return bet;
}
