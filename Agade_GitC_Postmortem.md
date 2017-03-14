#Agade Ghost in the Cell Postmortem

##Introduction

I will try to explain everything that my final code did, without wasting too much time talking about the evolutions of my AI. But first, a bit about my plan for the contest.

##Developement methodology

I chose C++ because it is the language I know best.

One of the first things I did was code an Arena program([github link](https://github.com/Agade09/CG-GitC-Arena)) that could play games between two of my AIs. The Arena runs the binaries of the two competing AIs, feeds them inputs and receives the outputs just like Codingame must do it on their servers. After every game the Arena prints the win rate of the first AI against the second AI with an error bar. Error bars on win rate can be calculated using [Bernouilli confidence interval](http://www.sigmazone.com/binomial_confidence_interval.htm). The same thing [cgstats](http://cgstats.magusgeek.com/app) does to give you an error bar on the win rate.

A new version of my program would beat the previous version, even by 0.5%, or would be thrown away. Being able to know that an AI is 0.5% better than another one requires a lot of games so I was glad to have the performance of C++ to run games as fast as possible. During the first half of the contest my AI was answering in ~0.01ms and I was able to run ~40 thousand games in ~10 minutes.

Trying to guess that an AI is slightly better by looking at a few games in the IDE and/or doing some submits, which only have ~100 games, is an attack on my sanity. I was therefore very happy to have one easy number to look at.

One might argue that judging AIs against the previous version has risks of specialisation. And it's true. But it was that or spending time looking at IDE games and trying to guess.

In order to avoid specialisation against myself, I sometimes disabled guessing bomb targets in my tests, because I'm obviously really good at guessing my own bombs.

Sometimes I took my latest version and played it against the code from a few versions ago. It was interesting to see that sometimes, playing better against a good AI makes you not as dominating against a weaker AI. But I decided to always take the version that beat the previous version, thinking that performance against top AIs was the only thing that mattered. I'm not trying to get 100% win rate against bronze players, I'm trying to get 60% win rate against top players.

It's the first contest where I used an Arena program and I think it payed off. 

In the last hour of the competition I found an improvement to my bomb sending logic which had a ~55% win rate against my previous version. I trusted my Arena games to the end and submitted the code even though it had never been in the online arena. 

##AI Hiding

At the top, at every contest, there are players who submit inferior versions of their code to the online arena and only release their real code at the end. You might call it competitive behavior, but I call it anti-competitive behavior. If everyone does it, there is no more competition. I am happy that I won without such tactics. In the arena you could always find my latest version.

##Heuristics

The core of my AI was based on heuristics I developed until saturday afternoon. I chose to go for heuristics because this game has such a huge branching factor (possible moves at every turn) that I couldn't imagine anything else. Also, the heuristics were so fast that I could run games really quickly to test my ideas.

In this game the smallest advantage over your opponent could allow you to get an extra point of production, gain even more advantage and so forth exponentially until your enemy's position is hopeless. So a lot of my thinking was in terms of investment, cost and time to return on investment.

###Best Path

In every map there were ways to go from A to B by going through intermediate factories in <= time. This doesn't only save time, it also obscures your troop movements to the enemy, allows you to redirect units elsewhere should the situation has changed by the time the units get to the intermediate factory and helps your AI defend/increase with minimal effort.

I compute a BestPath from every factory to every factory. A path is considered the best if its cost in turns is <= to every other path and the amount of intermediate factories is >= to every other path. In other words, favor paths with as many intermediaries as possible without taking detours. 

Be mindful that the time lenth of a path is different to the space length of that path. You have to add an extra turn per factory in the path because it takes a turn to send a troop. For example if Distance(A,B)==1 Turns(A,B)=2 and Turns(A,B,C)=Distance(A,B,C)+2.

In my code, whenever I want to send troops from A to B, I send it to the next intermediary in the pre-computed best path, which is B itself if the path is direct.

In my best paths I was not only using my factories but also neutral and enemy factories as intermediaries. This feels like a bit of a hack but it was simple and worked quite well.

###Factory Value

How much is factory i worth? In pseudo code:

```
Value[i]=Production[i]+0.01*pow(Paths_Through[i],0.5)+0.1
```

Where Paths_Through is the number of "Best Paths" that go through it from any factory to any other factory. A factory that has a lot of paths going through it has a bit more strategic value. A factory with no paths going through it and no production is still worth something because its production could be increased.

###Move Selection

My move selection followed an arbitrary pattern of, for every factory I had I would chose moves for the units that weren't needed for defense in the next 5 turns which is computed using known troop movements and production. In pseudocode:

```
for(int i in My_Factories){
	int sparable_forces{units[i]-Needed_For_Defense(i,next 5 turns)};
	vector<int> Candidate_Moves={1,2,...,N_Factories};
	sort(Candidate_Moves by score);
	for(int j:Candidate_Moves){
		Send appropriate amount of forces to factory j;
		if(sparable_forces==0){
			break;
		}
	}
}
```

A move is the integer id of the other factory. An id corresponding to one of my factories means "The move of sending defensive reinforcements to that factory", an id corresponding to an enemy/neutral factory means "The move of attacking that factory" and an id corresponding to the factory itself means "Increase". The question is now, how to score the moves.

####Neutral Attack Score

```
Value[id]/(pow(Turns to get there,2)*Required_To_Take_Neutral(id,Turns to get there))
```

In this formula losing units isn't penalised as heavily as wasting time travelling. It is generally better to use your units as quickly as possible, idle units might as well not exist.

Required_To_Take_Neutral() computes the units necessary to take a neutral factory on the turn when the units would arrive, taking into account all presently known troop movements towards that factory and future production should an enemy acquire the factory. It didn't take into account bombs that might be going towards that factory, maybe a possible improvement there.

####Enemy Attack Score

```
Value[id]/(pow(Turns to get there,2)*8)
```

Same formula as Neutral Attack Score but with troop requirement=8. It is probably possible to do better than a constant 8, but it is not a trivial problem because the enemy will move troops to defend that factory.

####Defense Score

```
Value[id]/(pow(Turns to get there,2)*Needed_Reinforcements(id,Turns to get there+5))
```

The same formula again but with troop requirement=needed reinforcements of that factory for the next Turns_To_Get_There+5 turns. In the Needed_Reinforcements() function it was important to take into account incoming bombs. This was to counter the bomb + 1 unit strategy. If I had guessed the bomb target correctly, my AI would send 1 unit to counter the incoming unit because keeping a factory for 1 unit scores really high.

I did not use the bomb + 1 unit strategy myself because of how I felt it was easily countered and required some kind of hardcoding complications I wanted to avoid in my AI. On top of things sending that 1 unit on its long path along with the bomb guarantees that that unit is now useless for a lot of turns. In my thought process, and as you see above in my evaluations, travel time is heavily penalised. If you take 10 turns to reach my factory and I send 1 unit which took 3 turns to counter your unit, I am winning slightly.

####Increase Score

```
1/pow(10,1.6)
```

The 1 represents the 1 gained production. The 10 represents the 10 lost units. The reason for the 1.6 instead of the 2 in the other scores is that increase is an investment which starts to pay itself back immediately, whereas when I am traveling 5 turns to get a neutral, I am not getting anything for 5 turns.

###Move Action

Above I presented my scoring of the possible actions of a factory. The actions are then sorted by score and done from best to worst until all sparable_units have been used. I now describe the actions.

####Attack Neutral
Send no more than the required forces calculated by the Required_To_Take_Neutral() function used in the score evaluation. But do send even if you don't have all the required units. I believe this helped mostly in situations where no factory could send enough units, but altogether there were enough separable units to take the factory. My AI does not think globally. It thinks in a greedy way from the point of view of every factory, so factories don't coordinate intentionally to send forces to a neutral.

####Attack Enemy
Send all available units, you never know what the enemy might be able to defend with.

####Defensive Reinforcements
Send no more than the required reinforcements as calculated by Needed_Reinforcements(id,Turns to get there+5) like in the Defense Score.

####Increase
If the factory has a production of 3, send at most 10-current_units to the closest of my factories with production <3.

If the factory has a production <3 and has >=10 sparable_units and no bomb is hitting it next turn, increase. Otherwise, keep your units by breaking out of the loop over decreasingly attractive actions. 

I thought maybe I shouldn't allow increase if bombs were coming in 2 or 3 turns but it didn't seem to play better.

###Details

When I send a troop from A to B, I actually send it to some intermediate factory C, as I explained when I talked about best paths. I disregard any troop moving action in which the troops would arrive at the same time as a bomb at C.

When I decide on an action like a bomb or a troop movement I update the state of the game to reflect that decision so that my next decisions can take this into account.

If a bomb is coming next turn on a factory then sparable_units=all_units.

###Bombs

Bombs are a very important early game tool that can easily make or break the game. My goal with bombs is to deny production, the killing of units being too uncertain. Bombing a factory with a production of 2/3 denies 10/15 early game units which is huge.

####Sending

I send both my bombs on the first turn to any factory I believe the enemy will own within 10 turns. I determine future owners by self playing 10 turns with no bombs and no increase. This might seem like a risky move, I could even help my opponent by killing neutral units for him. But it seemed to be well worth the risk as it gives an early game advantage that can often lead to victory. If the enemy tries to wait for the bomb to hit and kill off the neutrals for him, he might lose out on production by waiting anyway and/or guess wrong.

The two targetted factories are chosen by the following formula:

```
pow(production,3)/Time_To_Bomb
```

This mostly aims at the highest producing factories but is willing to make a compromise between denying more units later and denying less units now.

####Guessing

I guess the enemy bombs very much like I send them. But I consider a factory a potential target if it is in my first ten turns future owners list or if I will be the owner according to current troop movements. When I guess a bomb I store a sorted list of candidates. Should the guess turn out to be wrong, I pick the next candidate on the list.

##To Increase or not to Increase?

The different version of the above heuristics performed quite well, they weren't making a lot of blatant mistakes and my AI was ranked quite well. But the AIs kept getting a lot better very quickly and it was apparent to me that increasing was a huge weakness of my AI. The problem is my AI has a local factory level greedy approach but the question of whether or not to increase requires seeing the big picture. I might have a safe, far from the frontline factory which greedily thinks it can increase without any problems, whereas actually those 10 troops were needed for battle, I then lose a factory and subsequently the game. I couldn't think of any good heuristics for this so I decided on the following scheme: I play 20 turns of my AI with increasing enabled/disabled against my AI with increasing disabled. If my production is higher after 20 turns of increasing I play my AI in increasing mode, otherwise I disable increasing.

This led to a huge gain in win rate. The AI would increase whenever it could and, to its knowledge, whenever it couldn't get punished for it.

To be more specific I don't simulate 20 turns with increasing on, I simulate 20 turns of increasing until 1 increase is done. Then the question is, is this a winning or losing position? So I continue the self play with increasing disabled to check this.

##Conclusion

In the end, I had 700 lines of C++ and the winning code was version 79. There was some bloat, some obsolete code and some very similar functions that should have probably been golfed together but as you can tell from the lines of code and the above description of my AI, I tried to keep things rather simple/manageable/robust. I suspect that when the multiplayer comes out my AI will get beaten quite quickly.

The game rules were nice and simple, but I slightly disliked the hint of imperfect information of enemy bomb targets not being given. And while the game is good, I wouldn't say that it's my absolute favorite programming game because when all you are doing are heuristics, you aren't learning algorithms like minimax and simulated annealing that can be applied to other problems. In any case you're always learning something but I wanted to make that point. Maybe a machine learning approach could have been taken?

This competition was crazy, the top 10 was constantly changing and AIs felt like they were getting twice as strong every day. People were still improving a lot until the last hours of sunday. I could see that it was possible for me to win, but felt like it was anybody's game until the last few hours when I started to think I might have actually done it. By comparison in contests like Smash The Code and Fantastic Bits you could guess the winner in the first days. I started on this website about two years ago, asking myself how I could find the minimum of N numbers on Onboarding, I can't believe that I won... in front of 3500 people. I am very grateful.
