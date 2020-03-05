---
layout: post
title:  "Writing minimalistic games"
date:   2020-03-03 20:00:00 +0000
categories: [dotnet, fun]
tags: [dotnet, games, console, minmax]
---

How to use game development for self satisfaction. When you work for a long time on the same project with a team, it's nice to finish something quickly on your own.

## Motivation

When you work for a long time in same place, you can sometimes feel like you never finish a project. Yes, you add new features, you finish your sprints, but this is not the end. After one code change there will be another, and this is good, it pays your bills. But sometimes it would be nice to do something different. Maybe use a different language, challenge yourself, try something new and get it done quickly.

## MVP games

Not many developers write games as their day to day job. For many it was a motivation to start coding, but most developers are employed to do something else. If you are not a game developer, how can you build a game quickly? When I say quickly, I mean literally one evening. And here comes an idea I had couple years ago, what would be the most minimalistic UI for any game, it would be a console application. You might have even played some games in console, especially if you have used computers for a long time. There are many games you could implement this way. TicTacToe, Minesweeper, ConnectFour they can use column and row as an input, but you can use arrows to move on a board and create Snake, Sokoban or Sliding Puzzle. You may think about other games to implement.

## Games with inserting position

When you work with a Console application, you may require some input from a user, your code reads what a user typed in. This is what I decided to use for first few games.

### Minesweeper

When I started with it, my first idea was to implement Minesweeper, it's a single player game so you can play it alone. Generating a board requires height and width and number of bombs. You allocate bombs randomy and calculate numbers for all empty spaces. Player can select location by providing two integers. If you want, you can add an option to flag a spot where you expect a bomb to be. More challenging part is when you reveale a square not surrounded by any bombs, then you want to reveal all the surrounding squares, if any is empty you have to carry on. For a game like that an recursion will be good enough. While this game is not easy to read, you can play it. Not bad for a code with 241 lines.

![Minesweeper screen shot](/assets/2020-03-06/Minesweeper.png)

You could use different colour for the column and row ids, and for numbers on the board, to make it easier to see. You can make sure the first revealed field doesn't contain a bomb, by excluding it when generating the board.

You can find my code [here](https://gist.github.com/tomaszbartoszewski/414c6560baec4626cc6308b206c82ceb)

### Tic-tac-toe

We have a single player game, to do something different you can implement a game for two players and add some AI. Tic-tac-toe doesn't require advanced algorithms to make AI unbeatable. You can implement first version for two human players and then add single player mode. I used an IPlayer interface which gives next move, this way I was able to switch player type easily. In my case I implemented a Minimax algorithm which considers moves until the end of the game, because search space for it is not big, it doesn't take a long time for a computer to pick a sensible move. 

![Tic-tac-toe screen shot](/assets/2020-03-06/TicTacToe.png)

If you are interested have a look [here](https://gist.github.com/tomaszbartoszewski/09e02706b79d0c66874d23574a3c6c9e)

### Connect Four

A game which has similar way of judging players is Connect Four. This game is more difficult, total number of moves is a lot higher, so your AI can't check every possible game. We can as well make it more visual with using background colours instead of characters. In this case I implemented first human players, added a random player and then implemented Minimax algorithm. Because of too many possible moves, I had to limit searching to a certain depth. Because of that, early moves were in places not giving an advantage, middle column can be used with more winning groups than columns on the edges. That's why I added a version which invades middle part of the board first. There is a certain pleasure, when you lose a game against an algorithm you implemented.

![Connect Four screen shot](/assets/2020-03-06/ConnectFour.png)

Code for this game is [here](https://gist.github.com/tomaszbartoszewski/79bf151dd69872cf6111780beb0bac05)