---
layout: post
title:  "Writing minimalistic games"
date:   2020-03-07 13:00:00 +0000
categories: [dotnet, fun]
tags: [dotnet, games, console, minimax]
---

How to use game development for self-satisfaction. When you work for a long time on the same project with a team, it's nice to finish something quickly on your own.

## Motivation

If you have been working on the same code for many months or even years, you can sometimes feel like you never finish a project. Yes, you add new features, you complete your sprints, but this is not the end. After one code change there will be another, and this is good, it means you are working on something useful and it pays your bills. But sometimes it would be nice to do something different. Maybe use an unfamiliar language, challenge yourself, try something new and get it done quickly.

## MVP games

Only small subset of developers write games as their day-to-day job. For many it was a reason to start coding, but most developers are employed to do something else. If you are not a game developer, how can you build a game quickly? When I say quickly, I mean literally one evening. And here comes an idea I had couple years ago, what would be the most minimalistic UI for any game? A console application. You might have even played some games in console, especially if you have been using computers for a long time. There are many games you could create this way. Tic-tac-toe, Minesweeper, Connect Four they can use column and row as an input, but you can use arrows to move on a board and create Snake, Sokoban or Sliding Puzzle. You may think about many other games to build.

## How to run code examples

All of the games are working with .Net Core 3.1, when I implemented them it was on 1.1, so older versions should work too. You don't need any external libraries. To run them you will have to create a console application. You can do it by running this command:

{% highlight shell %}
dotnet new console -n Snake && \
rm Snake/Program.cs
{% endhighlight %}

Paste files from a specific gist into Snake directory and then run:

{% highlight shell %}
dotnet build && \
dotnet run
{% endhighlight %}

## Games with typing a location

When you work with a Console application, you may need some response from a user, your code reads what a user typed in. This is what I decided to use for first few games.

### Minesweeper

When I started, my first idea was to write a Minesweeper, it's a single player game so you can play it alone. Generating a board requires height, width and number of bombs. You allocate bombs randomly and calculate numbers for all empty spaces. Player can select a tile by providing two integers. If you want, you can add an option to flag a spot where you expect a bomb. More challenging part is when a player picks a square, that is not surrounded by any bombs, then you have to reveal all the surrounding squares. If another square is not surrounded by bombs you have to carry on, for a game like that, an recursion will be good enough. While this game is not easy to read, you can play it. Not bad for a code with 241 lines.

![Minesweeper screen shot](/assets/2020-03-07/Minesweeper.png)

You could use different colour for the column and row ids, and for numbers on the board, to make it easier to see. You can make sure the first revealed field doesn't contain a bomb, by excluding it when generating the board.

You can find my code [here](https://gist.github.com/tomaszbartoszewski/414c6560baec4626cc6308b206c82ceb)

### Tic-tac-toe

We have a single player game, to do something different you can implement a game for two players and add some AI. Tic-tac-toe doesn't need advanced algorithms to make AI unbeatable. You can create a first version for two human players and then add single player mode. I used an IPlayer interface which gives next move, this way I can switch player type easily between Human and AI without changing core engine. In my case I implemented a Minimax algorithm which considers moves until the end of the game, because search space for it is not big, it doesn't take a long time for a computer to pick a sensible move.

![Tic-tac-toe screen shot](/assets/2020-03-07/TicTacToe.png)

If you are interested, have a look [here](https://gist.github.com/tomaszbartoszewski/09e02706b79d0c66874d23574a3c6c9e)

### Connect Four

A game which has similar way of judging players is Connect Four. This game is more difficult, total number of moves is a lot higher, so your AI can't check every possible game tree. We can as well make it more visual, with using background colours instead of characters. In this case I implemented first human players, added a random player and then implemented Minimax algorithm. Because of too many possible moves, I had to limit searching to a certain depth. Other issue, early moves were in places not giving an advantage, middle column can be used with more winning lines than columns on the edges. That's why I added a version which invades middle part of the board first. There is a certain pleasure, when you lose a game against an algorithm you implemented.

![Connect Four screen shot](/assets/2020-03-07/ConnectFour.png)

Code for this game is [here](https://gist.github.com/tomaszbartoszewski/79bf151dd69872cf6111780beb0bac05)

## Moving on the screen

While you can play all the games above, previous interaction method only works for turn-based games. What if you would like to create some Arcade game, or just make interactions easier. Using arrows would be ideal.

### Snake

You don't need a fancy graphic for a Snake, simplicity of this game is one of the reasons it is so popular. Going back to basics is a good way to write it. Things you have to handle are changing directions, increasing size after eating food, hitting walls or it's own body and making sure different parts of body move in right direction. When a snake eats food you can think about it, as leaving a snake in place and replacing food with snake's head. I decided to store directions on different parts of the board, until snake passes by it. My first idea was to generate new board after every move. That caused unnecessary refreshing which didn't look nice. I changed the code to refresh only tiles where snake travels in specific move. You could probably simplify it, by adding extra square before snake's head and cleaning tile where tail's end has been, leaving the rest of the body in place.

![Snake screen shot](/assets/2020-03-07/Snake.png)

If you want to play it, code is [here](https://gist.github.com/tomaszbartoszewski/04aba19fb2b55be7caa3bc2dd22dd983)

### Sliding Puzzle

Have you ever seen a sliding puzzle? It has usually 16 squares, one is empty and remaining 15 either have numbers or a picture. The idea is to slide squares, to put numbers in order or complete a picture. Because I had to display two digit numbers, I decided to draw tiles 4x3, moving with arrows we already solved with the Snake game, you can reuse the code.

![Sliding puzzle screen shot](/assets/2020-03-07/SlidingPuzzle.png)

Look at [this](https://gist.github.com/tomaszbartoszewski/20eae51c6763cece407082ca87ae8c23) code, it only has 191 lines!

### Sokoban

I really like puzzles, and I wanted to make a game with different levels. My last entry here is Sokoban. Have you ever tried to put boxes on their destinations? Some maps are challenging. Sokoban boards are usually defined with this set of characters:

* \# is a wall
* @ is a player
* $ is a box
* . is a goal
* \+ is a played on a goal
* \* is a box on a goal
* space is an empty square

I store level definitions in separated file and added codes for different levels, so a player can write a code down, close a game and come back to it later. It helped with checking, if everything works too.

![Sokoban screen shot](/assets/2020-03-07/Sokoban.png)

While it doesn't look spectacular it is fun to play. Code is [here](https://gist.github.com/tomaszbartoszewski/c2606a233bf11cd2c922791c496a0752).
You have to put both files in project root directory. Try to add your own levels.

## Victory!

All of the games above have in total 1874 lines of code. I've seen in the past some commercial applications, with a single class using more lines and here we have 6 working games. You could definitely write them more concise, but the small size was not my goal, it's rather a side effect of using terminal as a UI. What games would you like to implement? Try to find something you can finish in one evening, hopefully it will give you as much satisfaction as it gave me.
