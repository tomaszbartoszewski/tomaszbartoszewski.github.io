---
layout: post
title:  "Writing minimalistic games"
date:   2020-03-03 20:00:00 +0000
categories: [dotnet, fun]
tags: [dotnet, games, console, minmax]
---

How to use game development for self satisfaction. When you work for a long time on the same project with a team, it's nice to finish something quickly on your own.

## Motivation

When you work for a long time in the same place, you can sometimes feel like you never finish a project. Yes, you add new features, you complete your sprints, but this is not the end. After one code change there will be another, and this is good, it pays your bills. But sometimes it would be nice to do something different. Maybe use an unfamiliar language, challenge yourself, try something new and get it done quickly.

## MVP games

Only small subset of developers write games as their day to day job. For many it was a motivation to start coding, but most developers are employed to do something else. If you are not a game developer, how can you build a game quickly? When I say quickly, I mean literally one evening. And here comes an idea I had couple years ago, what would be the most minimalistic UI for any game, it would be a console application. You might have even played some games in console, especially if you have used computers for a long time. There are many games you could implement this way. TicTacToe, Minesweeper, ConnectFour they can use column and row as an input, but you can use arrows to move on a board and create Snake, Sokoban or Sliding Puzzle. You may think about many other games to implement.

## How to run code examples

All of the games are working with .Net Core 3.1 but I implemented it on 1.1, you don't need any external libraries. To run them you will have to create a console application. You can do it by running this command:

{% highlight shell %}
dotnet new console -n Snake && \
rm Snake/Program.cs
{% endhighlight %}

Paste files from specific gist into Snake directory and then run:

{% highlight shell %}
dotnet build && \
dotnet run
{% endhighlight %}

## Games with typing a location

When you work with a Console application, you may require some input from a user, your code reads what a user typed in. This is what I decided to use for first few games.

### Minesweeper

When I started with it, my first idea was to implement Minesweeper, it's a single player game so you can play it alone. Generating a board requires height and width and number of bombs. You allocate bombs randomly and calculate numbers for all empty spaces. Player can select location by providing two integers. If you want, you can add an option to flag a spot where you expect a bomb to be. More challenging part is when you reveale a square not surrounded by any bombs, then you want to reveal all the surrounding squares. If any is not surrounded by bombs you have to carry on, for a game like that, an recursion will be good enough. While this game is not easy to read, you can play it. Not bad for a code with 241 lines.

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

## Moving on the screen

While you can play all the games above, it only works for turn-based games. What if you would like to implement some Arcade game, or just make interactions easier. Using arrows would be ideal.

### Snake

You don't need a fancy graphic for a Snake, simplicity of this game is one of reasons for its popularity. So going back to basics you can implement it. Things you have to handle are changing directions, increasing size after eating food, hitting walls or it's own body and making sure different parts of body move in right direction. When a snake eats food you can think about it as leaving a snake in place and swapping food to be snake's head. I decided to store directions on different parts of the board, until snake passes by it. My first idea was to generate new board after every move. That caused unnecessary refreshing which didn't look nice. I changed the code to refresh only tiles where snake travels in specific move. You could probably simplify it, by adding extra squer before snake's head and cleaning tile where tail's end has been.

![Snake screen shot](/assets/2020-03-06/Snake.png)

If you want to play it, code is [here](https://gist.github.com/tomaszbartoszewski/04aba19fb2b55be7caa3bc2dd22dd983)

### Sliding Puzzle

Have you ever seen a sliding puzzle? It has usually 16 squares, one is empty and remaining 15 either have numbers or a picture. The idea is to slide squers, to put them in order or complete a picture. Because I had to display two digit numbers, I decided to draw tiles 4x3, moving with arrows we already solved with the Snake game, you can reuse the code.

![Sliding puzzle screen shot](/assets/2020-03-06/SlidingPuzzle.png)

Look at [this](https://gist.github.com/tomaszbartoszewski/20eae51c6763cece407082ca87ae8c23) code, it only has 191 lines!

### Sokoban

I really like puzzles, so my last entry here is Sokoban. Have you ever tried to put boxes on their destinations? It can be sometimes challenging. What I wanted to do, was to implement a game with different levels. Sokoban boards are usually defined with this set of characters:

* \# is a wall
* @ is a player
* $ is a box
* . is a goal
* \+ is a played on a goal
* \* is a box on a goal
* space is an empty square

I store level definitions in separated file and added codes for different levels, so a player can write a code down, close a game and come back to it later. It helped with checking, if everything works too.

![Sokoban screen shot](/assets/2020-03-06/Sokoban.png)

While it doesn't look spectacular it is fun to play. Code is [here](https://gist.github.com/tomaszbartoszewski/c2606a233bf11cd2c922791c496a0752).
You have to put both files in project root directory. Try to add your own levels.

## Victory!

All of the games above have in total 1874 lines of code. I've seen in the past some commercial applications, with a single class using more lines of code and here we have 6 working games. You could definitely write them more concise, but the small size was not my goal, it's rather a side effect of using terminal as a UI. What games would you like to implement? Try to find something you can finish in one evening, hopefully it will give you as much sattisfaction as it gave me.