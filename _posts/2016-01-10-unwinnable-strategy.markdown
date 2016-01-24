---
layout: post
title:  "Unwinnable Strategy"
date:   2016-01-10
permalink: /unwinnable_strategy
---

There are many strategies to implement an unbeatable AI in a zero-sum game like tic-tac-toe, including using the <a href="https://en.wikipedia.org/wiki/Minimax" target="_blank">minimax algorithm</a>. Solely for entertainment, I will explain an alternative strategy that might not be as fool-proof, but will get the job done.

The minimax algorithm involves making a move by calculating the best payoff for every move. Subsequently, the best move for the second player is the opposite of the first move. In tic-tac-toe, the most important move that must be calculated correctly is *when there are two spots of a winning combination taken by the same player*. How must the potentially winning player react? How must the potentially losing player react? What does this look like in code?

First, let's start from the beginning and examine all of the possible moves for the AI. This <a href="https://www.quora.com/Is-there-a-way-to-never-lose-at-Tic-Tac-Toe" target="_blank">Quora question on tic-tac-toe strategies</a> does an excellent job of explaining this.

Through it, we are able to deduce the following possible moves of an AI:

Board:<br>
1 | 2 | 3<br>
~~~~~~~<br>
4 | 5 | 6<br>
~~~~~~~<br>
7 | 8 | 9

<ol>
  <li>Center move (5)</li>
  <li>Corner move (1, 3, 7, 9)</li>
  <li>Edge move (2, 4, 6, 8)</li>
</ol>

Let's write the code for these moves! 

**Center of the universe.**
---

A center move is as simple as writing a method that determines if the center spot is taken and if it isn't, to take that spot.

{% highlight ruby linenos%}
  def move(board)
    5 if !board.taken?("5")  
  end
{% endhighlight %}

\* Just FYI, I'm utilizing code from other parts of my program and passing in a *board* argument that is an array of empty cells and utilizing a method #taken that determines if the spot is already filled. I will cover the entirety of my tic-tac-toe game in a subsequent post.

**Boxed into a corner.**

This method will choose a corner spot at random, as long as it hasn't been taken.

{% highlight ruby linenos%}
  def corner_move(board)
    corners = [0,2,6,8] #remember, arrays are zero based
    corners.shuffle.detect { |spot| !board.taken?(spot+1) } #the +1 is necessary as valid inputs are numbers 1-9
  end
{% endhighlight %}

**On the edge.**

An edge is always bordered by two corners and thus, this method chooses an edge spot at random.

{% highlight ruby linenos%}
  def edge_move(board)
    [1,3,5,7].shuffle.detect { |spot| !board.taken?(spot+1) }
  end
{% endhighlight %}

And just like that, we have declared all possible moves on the board for the AI. The hard part is instilling logic into making the best move.

**One more move to win!**

Going back to the original discussion, the AI must determine if there is a winning combination that must be utilized or blocked. A winning combination can be any one of these arrays:

{% highlight ruby %}
WIN_COMBINATIONS = [
    [0,1,2],
    [3,4,5],
    [6,7,8],
    [0,3,6],
    [1,4,7],
    [2,5,8],
    [0,4,8],
    [6,4,2]
  ]
{% endhighlight %}

So ideally, we would like the AI to check each potentially winning combination and determine if two of the three spots are already filled.

The pseduo-code (more code than 'pseduo') would look like this:

{% highlight ruby linenos %}
  def open_spot
    Game::WIN_COMBINATIONS.detect do |spot|
      (spot[1] == taken && spot[2] == taken && spot[3] == free) ||
      (spot[2] == taken && spot[3] == taken && spot[1] == free) ||
      ((spot[3] == taken && spot[1] == taken && spot[2] == free)) 
    end
  end
{% endhighlight %}

In this method, we are checking each winning combination to see if any two out of the three spots are taken. If no such combination exists, the method will return `nil`.

However, we must bear in mind that it actually matters whether the spots are taken both by "X" or taken both by "O" (in other words, a winning combination is defined by all "X"s or all "O"s). So we must first assign a 'token' to each player:

{% highlight ruby linenos %}
  def opp_token  #this method assigns the correct token for the AI's opponent based on the AI's token
    ( self.token == "X" ? "O" : "X")
  end
{% endhighlight %}

Taking into account this new information, we produce this marvel of coding:

{% highlight ruby linenos %}
  def open_spot(board, token)  
    Game::WIN_COMBINATIONS.detect do |spot|
      ((board.cells[spot[0]] == token && board.cells[spot[1]] == token) && !board.taken?(spot[2]+1)) ||
      ((board.cells[spot[1]] == token && board.cells[spot[2]] == token) && !board.taken?(spot[0]+1)) ||
      ((board.cells[spot[2]] == token && board.cells[spot[0]] == token) && !board.taken?(spot[1]+1))
    end
  end
{% endhighlight %}

So now we have a method that determines if there are two "X"s (or "O"s) in a potentially winning combination on the current board.

**For the win! For the block!**
---

Here is the kicker: *it doesn't matter if the two spots are both "X"s or both "O"s, the next move of the AI is to take that empty third spot.*

Huh? Let's see an example.

Board:<br>
&nbsp;&nbsp;&nbsp;| X | O<br>
~~~~~~~~<br>
&nbsp;&nbsp;&nbsp;| O |  <br>
~~~~~~~~<br>
&nbsp;&nbsp;&nbsp;| X |  

It is X's turn and it will move into that bottom left spot to **block** the win for O.

Board:<br>
X |&nbsp;&nbsp;&nbsp;&nbsp;|  <br>
~~~~~~~~<br>
&nbsp;&nbsp;&nbsp;| X | O <br>
~~~~~~~~<br>
O |&nbsp;&nbsp;&nbsp;&nbsp;|

 It is X's turn and it will move into that bottom left spot to <strong>win</strong> the game. No matter whose tokens are in the winning combination, the AI must put their token in that third spot.

 Makes sense? We can now create a method that will utilize the previous methods to find the third spot of the winning combination that isn't yet taken:

{% highlight ruby linenos %}
  def win_or_block(board)
    if open_spot(board, self.token)  # if the winning combination is the AI's tokens, it will go for the win
      open_spot(board, self.token).detect { |index| !board.taken?(index+1) }
    elsif open_spot(board, self.opp_token) # else, if the winning combination is the AI's opponent's token, it will go for the block
      open_spot(board, self.opp_token).detect { |index| !board.taken?(index+1) }
    end 
  end
{% endhighlight %}

This method will return a number between 1 and 9 that will either win the game or block an opponent's win.

**Putting it all together now.**
---

Now we have methods for a center move, corner move, edge move, and a move to win/block a winning combination.

Based on the above strategy provided by the Quora question, we can implement something like this:

{% highlight ruby linenos %}
  def possible_move(board)
    5 || win_or_block(board) || corner_move(board) || edge_move(board) 
  end
{% endhighlight %}

So the AI will always choose this conditional sequence of moves and be truly unbeatable!


