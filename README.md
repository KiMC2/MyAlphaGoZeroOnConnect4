**My Simple Implementation of AlphaGo Zero on Connect4**

**What is AlphaGo Zero?**

AlphaGo is a reinforcement learning program invented by DeepMind. It is the first program to defeat a world champion in the game of Go. AlphaGo Zero is their latest version of the program and the best part of this version is that it learns how to play the game without needing any human data, guidance or domain knowledge except the rules of the game. It starts from zero and becomes a master all on its own!

**AlphaGo Zero Algorithm**

The algorithm of AlphaGo Zero is so simple and elegant. In general, it is a Monte Carlo Tree Search algorithm blessing by a modern neural network.

Monte Carlo Tree Search

In computer science, Monte Carlo tree search (MCTS) is a heuristic search algorithm for some kinds of decision processes. The focus of MCTS is on the analysis of the most promising moves, expanding the search tree based on random sampling of the search space.

Take Connect4 (or any other discrete, deterministic games with perfect information) as example, the input of a MCTS is the current state **s\_0** of the game - the positions of the colored discs in the suspended grid. The nodes on the Monte Carlo tree are the states of the game. If it is the first time the current node is being visited, the node will be expanded by trying every possible move **a**  on the state and appending the corresponding subsequent state of each move as its children.

 ![1. mcts - expanding](/images/1.%20mcts%20-%20expanding.png)

And then, the children must be valued. The value **W** is determined by randomly playing the remaining of the game from that sub state -- this is called _Rollout_. If the winner is the same player who performed the previous expanding move, **W** will be +1. Otherwise, **W** will be -1. In case of tied game, it will be 0. Since the rollout is a random play, it does not necessarily represent the optimal result.

 ![2. mcts - rollout](/images/2.%20mcts%20-%20rollout.png)

Perform above random rollout on each child node to determine their values. Besides the rollout value, the number of the rollout times **N** is another important attribute needed to be captured for each node. These two attributes need to be accumulated for every rollout on each individual node, and they also need to be propagated back up the tree by increasing the parent&#39;s **W** and **N** , which means the parent&#39;s values are its own accumulated values plus the total accumulated values of its children.

 ![3. mcts - accumulate w n](/images/3.%20mcts%20-%20accumulate%20w%20n.png)

Once the node is expanded, a child node shall be chosen to transition the search to the next state because it is too expensive or even impossible to try out all branches of the tree.  Similar to the other reinforcement learning algorithms, the main difficulty in selecting child nodes is maintaining some balance between the exploitation of deep variants after moves with high average win rate and the exploration of the moves with few rollouts. The formula as below called UCT (Upper Confidence Bound 1 applied to tree) can be used for balancing exploitation and exploration:

 ![4. mcts - uct](/images/4.%20mcts%20-%20uct.png)

in which, **W\_i** is the accumulated W value of **i** -th move, **N\_i** is the accumulated rollout count of **i** -th move and **N\_b** represents every child node in the same level, including **i** -th node.

The first component of above formula corresponds to exploitation; it is high for moves with high average win ratio. The second component corresponds to exploration; it is high for moves with few rollouts. The parameter **c** ≥ 0 controls the tradeoff between choosing exploitation (low **c** ) and exploration (high **c** ). It is often set empirically. Therefore, the child node with the highest **U** value will be chosen for the state transition.

 ![5. mcts - transition](/images/5.%20mcts%20-%20transition.png)

Starting from the initial input state, transitioning to the next state based on the **U** value and ending up when reaching a new state and expending (rollout and propagate back) it, this process is one iteration of MCTS.

 ![6. mcts - iterations](/images/6.%20mcts%20-%20iterations.png)

Run the search for multiple iterations (i.e. 50, 100 or even thousands of times depending on the time and the computing capacity, etc.), the tree should gradually build up its branches selecting the relatively good moves. Until now, all the searches are just simulations. The game state in the real world is still in the initial input state of the tree search, hasn&#39;t been changed yet. Then, the bot finally stops its thinking and make the move by choosing the child node with the highest **U** value.

 ![7. mcts - make real move](/images/7.%20mcts%20-%20make%20real%20move.png)

Two Heads Neural Network

The Monte Carlo Tree Search is great, but it is not efficient enough for dealing with large branching games, such as chess, Go and even Connect4 (For classic Connect4 played on 6 high, 7 wide grid, there are 4,531,985,219,092 positions for all game boards populated with 0 to 42 pieces). Thus, the neural network needs to be leveraged to enhance MCTS algorithm, given the nature of neural network is like a magic box: finite samples go in, infinite predictions come out.

First, the neural network can improve the exploitation of MCTS. The original rollout method needs to randomly simulate a lot of moves to value the nodes. It is very inefficient and inaccurate. It would be much better, if a network **v** could directly approximate the true rollout value for a given state **s** in the range of [-1, 1]:

 ![8. value network formula](/images/8.%20value%20network%20formula.png)

To prepare the training data for this network, one can play a game based on MCTS. At the end of the game, the final result **z** is back filled in every step per players. For example, all winner&#39;s steps are scored as 1, all loser&#39;s steps are scored as -1. In case of tied game, all steps are scored as 0. Since this is a regression problem, the loss function can be MSE (Mean Square Error).

 ![9. value network loss](/images/9.%20value%20network%20loss.png)

The network **v** is so called Value Network.

The neural network can also improve the exploration of MCTS. Imaging a network **p** which can directly compute the probability **P** of choosing each following move **a** on the given state **s** in the MCTS, it can be used for enforcing/adjusting the original exploration component.

 ![10. policy network formula](/images/10.%20policy%20network%20formula.png)

In term of training, the actual search probabilities  **π** from the previous MCTS iterations can be used for computing the target labels in the data.

 ![11. policy network training label](/images/11.%20policy%20network%20training%20label.png)

where **π\_s,a** represents the actual search probability of taking move **a** on state **s** , **N\_s,a** is the accumulated visit count of taking move **a** on state **s** , **s,b** represents each possible move **b** on state **s**. Apparently, **π** is the proportion of the visit count. Since this is more like a classification task, the categorical cross entropy loss function can be used for training this network.

 ![12. policy network loss](/images/12.%20policy%20network%20loss.png)

The network **p** is so called Policy Network.

In fact, AlphaGo Zero actually combined above two networks into one Two Headed Monster.

 ![13. combined network](/images/13.%20combined%20network.png)

In order to apply this network onto MCTS, several useful arrays shall be defined:  **N** , **Q** and **P**. These arrays are indexed by the combination of state **s** and its possible following move **a** , that is, **N(s,a)**, **Q(s,a)** and **P(s,a)**. And **s** and **a** can make the following transition:

 ![14. state transition](/images/14.%20state%20transition.png)

As shown above, taking action **a** on state **s** can transition state **s** to its child state. Hence, above arrarys can also be considered as being indexed by the child state **s\_a**  (transitioned from state **s** by taking move **a** ).

Therefore, Array **N(s,a)** stores the visit count on state **s\_a** during the previous MCTS. Array **Q(s,a)** stores the average rollout approximation of state **s\_a** :

 ![15. Q(s,a)](/images/15.%20Q(s,a).png)

Array **P(s,a)** stores the predicted probability of choosing state **s\_a** :

 ![16. P(s,a)](/images/16.%20P(s,a).png)

_Note that, the network outputs for calculating_ **Q(s,a)** _and_ **P(s,a)** _need to be predicted separately as they use different state as input._

Having above definition, the adapted UCT for the Two Headed network can be denoted as below:

 ![17. updated uct](/images/17.%20updated%20uct.png)

In addition, AlphaGo Zero leverages the Residual Network technology to optimize its network model. The Residual Network was developed by the researchers from Microsoft Research to overcome the degradation problem of the deep neural network (with network depth increasing, accuracy gets saturated and then degrades rapidly).

Given a neural network and denote its input is **x** and its expected output is **H(x)**, if one passes the input **x** directly to the output as the initial value, what the network is supposed to learn now will become **F(x)=H(x)-x**. As depicted by the diagram below, it is a typical Residual Unit of Residual Network.

 ![18. residual unit](/images/18.%20residual%20unit.png)

_Above diagram come from _ [_https://arxiv.org/pdf/1512.03385.pdf_](mhtml:file://C:%5C%5CUsers%5C%5Caku5551983%5C%5COneDrive%5C%5C%E6%96%87%E6%A1%A3%5C%5CEvernote.enex.mht!https://arxiv.org/pdf/1512.03385.pdf)

Residual Network has changed the learning objective, which is no longer the complete expected output **H(x)**, but the gap between input and output **H(x)-x**, that is so called Residual.

Different from the traditional Convolutional layer or Full Connection layer which would more or less cause information loss, Residual Network resolved this problem in some degree by &#39;shortcutting&#39; the input information to the output, which protects the integrity of the information and also simplifies the learning objective and learning complexity as the entire network just needs to learn the difference between input and output.

Training

The details about training data generation and loss function selection have been discussion in the previous sections. Here, just put the combined loss function provided in the AlphaGo Zero paper as below:

 ![19. combined loss](/images/19.%20combined%20loss.png)

in which, c||θ||^2 is the L2 regulation of the network parameters.

The following diagram provides a high level conceptual picture of the training process of AlphaGo Zero.

 ![20. alphago zero](/images/20.%20alphago%20zero.png)

The training starts from many rounds of games of self-play. The moves during self-play are selected based on MCTS reinforced by the &#39;best&#39; network by far. The self-play steps and results are captured along with MCTS stats to form the training data. Fit the &#39;current&#39; network with the training data. And then, let the &#39;best&#39; network and the &#39;current&#39; network to fight with each other for a number of games. If the &#39;current&#39; network won a certain percentage of the games (e.g. 60%), it will become the new king to reign MCTS. Repeat above steps to keep fitting the &#39;current&#39; network.

_Note that, the hardware running AlphaGo Zero has very powerful computing capacity, which can execute large iterations of MCTS, deep Residual Network fitting and network competition in parallel. Its MCTS algorithm is also a variant that can perform multiple tree searches simultaneously._

Since MCTS is better than random play, the training data is being improved along the training, so is the network accordingly, and meanwhile the improved network is also optimizing MCTS. This virtuous circle is the key of why AlphaGo Zero can learn from itself!

Other Technologies

Besides above key technologies used in AlphaGo Zero, there are also some additional details. Some of them are:

- Symmetry

The board of most games are symmetry. They are invariant to reflection and/or rotation. For example, Connect4 grid is invariant to reflection, the training data can be doubled by flipping the state and the probabilities data horizontally.

- Temperature

The way to calculate the MCTS probabilities for the moves has been discussed in the previous section:

 ![21. policy network training label](/images/21.%20policy%20network%20training%20label.png)

However, above is just a simplified version for the sake of clarity. The actual formula used by AlphaGo Zero is as below:

 ![22. policy network real training label](/images/22.%20policy%20network%20real%20training%20label.png)

where **τ** is the temperature which controls the degree of exploration. The higher the temperature the wider the exploration, meaning every move may get a chance to be selected for tree exploration. When it is cool down (**τ** -&gt; 0), only the move with the most visit count will be selected. The temperature should be high (=1) for the first several turns of moves as there are more uncertainties at the beginning of the game. And then, it should be an infinitesimal value in order to select the &#39;best&#39; move leading to the victory.

- Dirichlet Noise

Dirichlet Noise has also been added onto the predicted probabilities of the root node (only) to introduce additional exploration in MCTS:

 ![23. dirichlet noise](/images/23.%20dirichlet%20noise.png)

**ε**=0.25 makes sure that the Dirichlet Noise only affects a small portion (25%) of the probabilities.

**Dir(α)** is the function of (Symmetric) Dirichlet Distribution, which can be considered as the probabilities of the (prior) probabilities. **α** is an array containing the parameters called Concentration Parameter. The larger the value of the concentration parameter, the more evenly distributed is the resulting distribution. The smaller the value of the concentration parameter, the more sparsely distributed is the resulting distribution, that is, the distribution more tends to concentrate on a single point. The values of the Concentration Parameter for the probabilities of all moves are same, because there is no prior knowledge favoring one move over another. The value of the Concentration Parameter should be scaled in inverse proportion to the approximate number of valid moves in a typical state. As many valid moves as Go, this parameter is about 0.03. In term of Connect4, a value around 0.8 should be alright.

**My Implementation on Connect4**

My version of AlphaGo Zero was written in Python with Keras APIs (Tensorflow as back end).

The hardware I used for training was terribly bad. It was a 4 vCPU (no GPU) 8GB RAM VM running in my laptop. But it was OK. I saved the best and the current networks as well as the training data periodically so that I could stop and resume the training easily. Since the foundation of AlphaGo Zero algorithm is MCTS, it doesn&#39;t really sensitive to the interruption of training either. So far, the training has taken me about 600 hours in the last 2 months (consider my hardware T\_T). But I am quite happy with the result. The bot can beat human (my wife and I) quite often (~50% winning rate) now.

Below are some more details about my implementation. First, it executes the self-play (data generation), network fitting and network comparison in different threads, so that the network fitting process does not have to wait for the time-consuming self-play and network comparison. Second, I found that the iteration number of MCTS is one of the key parameters to improve the quality of the training data and the network accordingly. With high MCTS iteration number, it is like playing with a master who can think further than others. I set this value to 100 when I was training the network although it took longer time for MCTS, but I felt it&#39;s worth it. Third, the training data will be cleaned when the best network was replaced by the current network. The network fitting and comparison threads would be suspended until a sufficient amount of training data were generated by the new best network. The network comparison thread would be suspended for an extra-long period (e.g. 1 hour) to allow the current network be well-trained by the new data and also avoid the best network being replace too often. Below is a typical training summary visualized by TensorBoard:

 ![24. tensorboard diagram](/images/24.%20tensorboard%20diagram.png)

Hope you like this post and please provide your feedback.


References:

https://medium.com/applied-data-science/how-to-build-your-own-alphazero-ai-using-python-and-keras-7f664945c188

http://tim.hibal.org/blog/alpha-zero-how-and-why-it-works/

https://web.stanford.edu/~surag/posts/alphazero.html

https://www.nature.com/articles/nature24270.epdf

https://en.m.wikipedia.org/wiki/Connect_Four

https://en.m.wikipedia.org/wiki/Monte_Carlo_tree_search

https://en.m.wikipedia.org/wiki/Dirichlet_distribution

