---
layout: post
title:  "Count-Min Sketch and Sparse Recovery"
date:   2022-04-20 00:51:00 -0400
categories: jekyll update
---
*(This blog post was produced by Edward Song and Andrew Wei as the final project for EECS 586 at the University of Michigan)*

## Introduction
A Count-Min Sketch, or CM sketch, is a data structure that makes clever use of hash functions and randomization. Using a CM sketch, we can provide much of the same functionality as a hash table with a fraction of the space required. This comes with the caveat that our results will only be approximate, not exact, but CM sketches provide nice guarantees on both the probability and margin of error. 

## Background
Before proceeding with any further explanation, let’s back up a bit and briefly review the topics of hash functions and randomized algorithms. 

### Hash Functions
A hash function is a function that maps inputs from an arbitrarily large space to a finite, smaller space. Put simply, a hash function takes inputs and compresses them into smaller, usually easier-to-work-with values. For instance, we might consider a hash function like $$h(x)=x\mod 5$$. In this case, $$h$$ is a hash function that maps from $$\mathbb{R}$$ to $$\{0,1,2,3,4\}$$. Since the number of values in $$\mathbb{R}$$ is larger than the number of possible outcomes, $$h$$ is guaranteed to eventually map two inputs to the same output. This event is known as a *collision*. To avoid collisions, we ideally want our hash functions to produce values uniformly across the space of outputs.

### Universal Hash Families
A universal hash family is a finite set of hash functions with certain properties. Specifically, say we have a set of hash functions $$H$$ that use a number of bins $$M$$. If $$H$$ is a universal hash family, then for any distinct input pair $$x, y$$ and for each individual hash function $$h \in H$$, we want the probability of $$x$$ and $$y$$ colliding to be at most inversely proportional to $$M$$. More formally, $$H$$ is a universal hash family if:

$$\forall x,y;x\neq y:P_{h \in H}[h(x)=h(y)] \leq 1/M$$

For the CM sketch we will actually require a stronger condition, that $$H$$ be pairwise independent. For a hash family to be pairwise independent, it must be that for a pair of distinct inputs $$x_1$$ and $$x_2$$, the probability that they map to an arbitrary pair of outputs $$y_1$$ and $$y_2$$ is exactly $$1/M^2$$. In other words:

$$\forall x_1,x_2;x_1 \neq x_2:P_{h \in H}[h(x_1)=y_1 \wedge h(x_2)=y_2]=1/M^2$$

Notice that pairwise independence implies universality of the hash family. 

### Randomized Algorithms
Many problems in computer science are very expensive to solve with purely deterministic algorithms. In many of these cases, we can introduce some randomness into the algorithmic solution to dramatically improve the algorithm’s time or space requirements. A classic example of this paradigm is randomized quicksort. Using Blum et. al.’s deterministic algorithm for finding a median, we can select a pivot in $$O(n)$$ time. The drawbacks, however, are that this $$O(n)$$ hides a non-negligible coefficient and that the median algorithm can be complicated to implement. We can instead choose the pivot uniformly at random. This new, randomized quicksort can be shown to have $$O(n\log n)$$ expected runtime with a high, ex. $$1-n^{-10}$$ probability. 

## Motivation
Now, to motivate the conception of the CM sketch, imagine that you are a market researcher who is working for a major online retailer. 

<img src="/images/count-min-sketch/online_retailer.jpeg" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 60%;"/>

Every month, a small subset of the products listed on your company’s site dominate the overall number of sales. You would like to identify if there are any trends or patterns in this subset of products. To do so, you first need a list of the top selling products per month. Luckily, this list doesn’t need to be perfectly accurate. After all, you’re more interested in finding any overarching patterns as opposed to those on a product-level. As a first attempt, you consider building a hash table that maps product IDs to the number of times they were purchased.

<img src="/images/count-min-sketch/hash_table.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

Unfortunately, you quickly realize that the size requirements for this table would make any kind of data analysis far too inconvenient to perform offline. Say, for instance, that you represented each product ID as a 4-byte integer. Then in the worst case, your table could contain up to 4,294,967,295 unique product ID keys. With each key taking up 4 bytes, the table would require 16 gigabytes of storage just to store the keys. If we also store the count of purchases as 4-byte integers, we would have to add on another 16 gigabytes. Lastly, if we consider space overhead and load factor optimization (ex. load factor of 1/2), we might require yet another 16 to 32 gigabytes. The total worst-case space requirement comes out to anywhere from 48 to 64 gigabytes— a value much larger than what most consumer computers will be able to handle in working memory. 

## The Actual Count-Min Sketch Data Structure
This is where our Count-Min Sketch comes to the rescue. Instead of a 1D-array like that used in a hash table, visualize a 2D-array with width $$w$$ and depth $$d$$. Each cell in this 2D-array is initialized to 0 and is used as a numerical counter. We refer to this entire array as a *sketch*.

<img src="/images/count-min-sketch/width_w_depth_d.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

Additionally, we require a pairwise independent, universal hash family of size $$d$$. Make sure to check the background section for more information on universal hash families.

## Incrementing on a Key
To increment the value of a key $$k$$ by some amount $$c$$, we first think about assigning each of the $$d$$ rows of the sketch one of the $$d$$ hash functions. Then, we hash $$k$$ using each of these hash functions to produce a set of outputs $$j$$. That is, $$1 \leq i \leq d:j_i=h_i(k)$$. Once we calculate these indices, we simply increment the cells at positions $$(i,j_i)$$ by $$c$$. 

Let’s illustrate this process with an example. Say we have a CM sketch with $$w = 5$$ and $$d = 4$$. Then our sketch will initially look like this:

<img src="/images/count-min-sketch/initial_0.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

Of course, we also have a universal hash family of size $$d$$ to go with the sketch. Now say we want to increment the value of the key `"abc"` by 2. The first step is to use each of the $$d$$ hash functions to hash `"abc"`. For the sake of this example, let’s say $$h_1(\text{“abc”}) = 1$$, $$h_2(\text{“abc”}) = 3$$, $$h_3(\text{“abc”}) = 1$$, $$h_4(\text{“abc”}) = 4$$. Then $$j=[1,3,1,4]$$ and our $$(i, j)$$ pairs are $$(1, 1), (2, 3), (3, 1)$$, and $$(4, 4)$$. We increment the corresponding cells and end up with the following sketch:

<img src="/images/count-min-sketch/first_add.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

Let’s additionally increment the value of the key `"qwe"` by 3. Say $$h_1(\text{“qwe”}) = 2$$, $$h_2(\text{“qwe”}) = 3$$, $$h_3(\text{“qwe”}) = 3$$, $$h_4(\text{“qwe”}) = 5$$. After incrementing, our sketch finally looks like this:

<img src="/images/count-min-sketch/second_add.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

## Query
The process of querying a key is very similar to incrementing on a key. We again start by computing the $$d$$ hash values of the key $$k$$ then forming coordinate pairs. Once we have the pairs, we can calculate their corresponding cells’ minimum value. This min value is the answer to our query. 

We can illustrate this process by continuing our example from the previous section. Recall that after incrementing the key `"abc"` by 2 and `"qwe"` by 3, we ended up with the following sketch:

<img src="/images/count-min-sketch/second_add.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

Now let’s say we query on the key `"abc"`. We know that for `"abc"`, our hash values are $$j=[1,3,1,4]$$ and our coordinate pairs are therefore $$(1, 1), (2, 3), (3, 1)$$, and $$(4, 4)$$. Looking at the sketch, we can see that the corresponding cell values at these coordinates are $$[2, 5, 2, 2]$$. Since we take the minimum of these values, we return 2 as the answer to the query. This correctly answers our query in this case, as we only ever incremented `"abc"` by 2 during the lifetime of this sketch. 

Can our query go wrong? Well yes, it can when there are hash collisions. Let’s say we do a couple more increment operations, like incrementing `"dog"` by 1, `"cat"` by 4, and `"byz"` by 2. After these operations, we end up with the following sketches:

After adding `"dog"`:
<img src="/images/count-min-sketch/add_dog.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

After adding `"cat"`:
<img src="/images/count-min-sketch/add_cat.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

After adding `"byz"`:
<img src="/images/count-min-sketch/add_byz.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

Let’s again query on the key `"abc"`. 

<img src="/images/count-min-sketch/query_abc.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

This time, the values at the coordinates are $$[3, 5, 6, 4]$$. We get an answer of 3 when we take the minimum. Unfortunately, since we only ever incremented `"abc"` by 2, our answer of 3 overestimates the true answer by 1 and is not entirely accurate.

As another example, imagine querying on the key `"yop"`. Notice that we never encountered this key before. We get $$h_1(\text{“yop”}) = 4$$, $$h_2(\text{“yop”}) = 5$$, $$h_3(\text{“yop”}) = 1$$, $$h_4(\text{“yop”}) = 3$$. 

<img src="/images/count-min-sketch/query_yop.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 80%;"/>

Looking at $$(1, 4), (2, 5), (3, 1)$$ and $$(4, 3)$$, we can see that the corresponding cell values are $$[4, 4, 6, 1]$$. Again, we overestimate our true expected answer of 0 because of hash collisions.

## Probability Analysis
*(This section heavily draws on the original CM Sketch paper by Cormode and Muthukrishnan)*

Consider creating a series of indicator variables like so:

$$I_{x,y,i}=\begin{cases}
1&\text{if }x\neq y \wedge h_i(x)=h_i(y) \\
0&\text{otherwise}
\end{cases}$$

Since our family of hash functions $$H$$ is pairwise independent, we know:

$$E[I_{x,y,i}]=P[h_i(x)=h_i(y)]\leq \frac{1}{\text{range}(h_i)}=\frac{\epsilon}{e}$$

We want to be able to quantify how much we overestimated for each entry. Define a random variable $$Q_{x,i}$$ over the choice of $$h$$ to be $$Q_{x,i}=\sum_{y=1}^{n}I_{x,y,i}a_y$$. If we denote the true count of an input $$x$$ to be $$a_x$$, then by construction, $$\text{count}[y, h_y(x)] = a_x+Q_{x,i}$$. Then we can use linearity of expectation and the pairwise independent property to get:

$$E(Q_{x,i})=
E\left[\sum_{y=1}^{n}I_{x,y,i}a_y\right]=
\sum_{y=1}^{n}E[I_{x,y,i}a_y] \leq 
\frac{\epsilon}{e}\sum_{y=1}^{n}a_y=
\frac{\epsilon}{e}\left\Vert a\right\Vert_1$$

Here, the term  refers to the sum of all counts in the entire data structure.

How common is this guarantee? We can derive this answer by first trying to bound the probability that this behavior does not happen. Denote our estimate of a key’s value as $$\hat{a}_x$$. Then we can use the Markov inequality to derive the following:

$$P[\hat{a_x}>a_x+\epsilon\left\Vert a\right\Vert_1]$$

$$=P[\forall i:\text{count}[i,h_i(x)]>a_x+\epsilon \left\Vert a\right\Vert_1]$$

$$=P[\forall i:a_x+Q_{x,i}>a_x+\epsilon \left\Vert a\right\Vert_1]$$

$$=P[\forall i:Q_{x,i}>eE[Q_{x,i}]]<e^{-d}\leq \delta $$

So we see that with probability $$1-\delta $$, our estimate is an overestimate by at most $$\epsilon \left\Vert a\right\Vert_1$$. If we set $$w=\left\lceil \frac{e}{\epsilon} \right\rceil$$ and $$d=\left\lceil\ln \frac{1}{\delta}\right\rceil$$, then we can control how much error we wish to tolerate by modifying $$w$$ and $$d$$ appropriately. These formulae for $$w$$ and $$d$$ can also be interpreted intuitively. As we increase the width of the 2D-array, we increase the number of bins that can be hashed into. For a universal hash family, the number of bins $$M$$ affects the probability in a inversely proportional way, i.e. $$1/M$$. Similarly, more rows means more hash functions are applied to an input. Since we take the minimum cell value across all rows, every hash function has to encounter a collision for the returned count to be inaccurate. Fortunately, we specified that our hash functions are part of a pairwise independent, universal hash family. Think about the probability of a certain event happening for every independent random variable as we add more variables. In exactly the same way, the probability of all hash functions having a collision decreases exponentially with more hash functions.

## Space Analysis
There are two components that take up space, the 2D array and the pairwise independent, universal hash family. The space requirement for the 2D array is pretty easy to figure out. There are $$d$$ rows and $$w$$ columns, so the total space requirement is $$O(dw)$$. The space requirement for the hash family is a bit trickier, but still not too bad. We claim that storing a singular hash function from this family can take as little as constant space. One such example is $$h(k)=(ak+b\mod p)\mod w$$. For this function, we only ever need to store a constant 3 integer values. Further, changing $$a$$, $$b$$, and $$p$$ to create more functions gives us a pairwise independent, universal hash family. For $$d$$ hash functions, then, the total space requirement is $$O(d)$$. Putting the two components together, our final space requirement comes out to $$O(dw+d)=O(dw)$$.

## Back to the Motivating Scenario
To get a more concrete perspective on the benefits of our CM sketch, let’s go back to the scenario from the motivations section. We will still store the counts as 4-byte values within each cell. Assume $$w = 10,000$$ and $$d = 10$$. Then the array can consume at most $$10,000\times 10\times 4=400,000$$ bytes. Let’s say we store three 8-byte integers per hash function. With 10 hash functions, we need a total of $$3\times 8\times 10=240$$ bytes. Adding the two together, we get 400,240 bytes total, or about 0.4 megabytes. This is a minuscule amount of storage compared to the 48 gigabytes we were contemplating earlier. 

Earlier, we said that the error can be controlled by $$w$$ and $$d$$. For $$w = 10,000$$ and $$d = 10$$, the chance of error is $$\delta =\frac{1}{e^{10}} \approx 0.00005$$ and the estimate is at most an overestimate by $$\epsilon \left\Vert a\right\Vert_1=\frac{e}{10,000}\left\Vert a\right\Vert_1\approx 0.0003\left\Vert a\right\Vert_1$$. Those are pretty nice bounds.

## The Sparse Recovery Problem

Using a CM sketch, we explored how to count the number of times an element appeared in a data stream. The natural progression to this problem is to also determine how many nonzero elements we counted. 
 
Now let's return once again to the running example of the market researcher. Using a CM sketch, we are able to make queries about number of times inidivual prodcuts have sold, but what we ultimately want is a list of the top selling products, not just individual product sales alone.

<!-- ![image info](/images/sparse-recovery/excel.jpg) -->
<img src="/images/sparse-recovery/excel.jpg" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 60%;"/>
 
This is where the sparse recovery problem comes in. Suppose we have a vector $x$, which can be arbitrarily large. If $x$ contains $s$ nonzero elements or less, we want to return all those nonzero entries. If $x$ has more than s nonzero entries, we want to be able to recognize this as well. In other words, we only want to return the few “important” elements of a vector $x$. Beyond E-commerce research, this problem also has extensive applications in signal processing, such as speech recognition, imaging, and data collection. 

## Building Blocks of the Algorithm

It was mentioned above that $x$ should have “$s$ nonzero elements or less”. We refer to this property as being $x$ is s-sparse if this is the case. So, we say that $x$ is 1-sparse if there is only one nonzero element in $x$. This forms the core building block for our sparse recovery algorithm: the 1-sparse recovery sketch. The core idea for the 1-sparse recovery sketch is that we want to maintain three numbers:
- $w_1 = \sum_{i=1}^{n} x_i$ 
- $w_2 = \sum_{i=1}^{n} x_i \times i$
- $w_3 = \sum_{i=1}^{n} x_i \times r^i \mod p$, where $p >= n^c$ is a prime, and $r$ is randomly sampled from $\{ 1, \dots, p - 1 \}$. 

Given any update $\Delta$ to the $i$th entry of x, we update $w_1 \leftarrow w_1 + \Delta$, $w_2 \leftarrow w_2 + \Delta \times i$, $w_3 \leftarrow w_3 + \Delta \times r^i \mod p$.

In fact, three numbers are all we need for our 1-sparse recovery sketch $sk(x)$. Given a query, use our sketch to report our answer as follows:
- We report that $x = 0$ if $w_1 = w_2 = w_3 = 0$,  as there are never any updates to any entries of $x$.
- We report that $x$ is not 1-sparse if $\frac{w_2}{w_1} \notin \{1, \dots, n \}$ or $w_3 \neq w_1 \times r^{w_2 / w_1} \mod p$.
- Otherwise, we report that $x$ is 1-sparse, and return the index and count of the desired element using $(i, x_i) = (\frac{w_1}{w_2}, w_1)$.

## Correctness and Probability Analysis

Following the definitions of $w_1, w_2,$ and $w_3$, we can build an intuition that this works simply by plugging in values under scenarios where $x$ is 1-sparse or not. However by plugging in numbers alone, we'll soon realize that there are instances where this doesn't always work. Nonetheless, we can show that this still holds with high probability, that is dependent on the prime $p$.

We first assume that $\frac{w_2}{w_1} \in \{1, \dots, n \}$. Then, we want to find the probability that $w_3 - w_1 \times r^{w_2 / w_1} = 0 \mod p$. By substituting in the definitions of $w_3$ and $w_1$, we notice that we obtain a degree $n$ polynomial in $r$ over $\mathbb{F}_p$. This probability is simply the probability that our polynomial is $0 \mod p$. In other words, the probability of sampling a root of our polynomaial, over a set of size $p$. This is at most $\frac{n}{p}$, where we remember that $p = n^c$.

This is great! As we have just found that any query to our sketch is correct with probability $1 - n^{c - 1}$. We can easily adjust this probability to be arbitrarily high by adjusting the constant $c$. 

## The Sparse Recovery Algorithm
Using our 1-sparse recovery sketch, we are almost ready to complete our full sparse recovery algorithm. Conceptually, the sparse recovery algorithm is composed of $k = O(s)$ many 1-sparse vcectors and 1-sparse recovery sketches for each of $s$ nonzero entries. But we're lacking one core detail: How do we make sure that our vector can be maintained by a series of $k$ 1-sparse vectors? The answer is in something we have already covered: Hash functions and universal hash families. 

<!-- ![image info](/images/sparse-recovery/1rev.png) -->
<img src="/images/sparse-recovery/1rev.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 60%;"/>

Let's try out the following algorithm:
We start by initializing a hash function $h: [n] \rightarrow [2s]$, as well as $2s$ many 1-sparse recovery data strutures. For any update $(i, \Delta)$, we update the $h(i)^{\text{th}}$ 1-sparse recovery data structure and $w_3 \leftarrow w_3 + \Delta \times r^i \mod p$. Now, to process a query, we compute the two items:
1. The set of all $(i, x_i)$ reported by any of the 1-sparse recovery data structures, known as $S_{report}$. 
2. An estimated $\tilde{w_3} = \sum_{(i, x_i) \in S_{report}} x_i r^i \mod p$.
Now if, $|{S_{report}}| > s$ or $w_3 \neq \tilde{w_3}$, we report that $x$ is not s-sparse. Otherwise, we return $S_{report}$ as our answer. 

Since we are working with hash functions here, we recognize that there is a possibility of a collision. In this case, we might fail to report a nonzero element $i$. However, this creates an issue with our exisiting algorithm. The probability of a collision is high, as there are roughly $s$ many terms and $2s$ possible 1-sparse recovery data structures. This means there at most a $\frac{1}{2}$ probability of a collision, and therefore failure. 

To remedy this, we need to make $c\log{n}$ independent copies of the hash function and as well as the $2s$ 1-sparse recovery data strcutures. Now, the probability of a collision is only $\frac{1}{n^{c}}$, where we can make $c$ as large as we want.

## Space Complexity

Our sparse recovery data strcuture accomplishes the goal that wanted to achieve. But how much space does it require? We saw that we had to make many copies of hash functions and 1-sparse recovery data structures. But how much space does each one need? This is where universal hash families come in. We know that each 1-sparse recovery data structure only needs 3 numbers, $w_1, w_2, w_3$, to represent it. We also know that there is a universal hash family, $h_{(a,b)} = ((ax + b) \mod p) \mod M$ which only needs two numbers $a$ and $b$ to represent it. With that, the space complexity for our sparse recovery data structure is $O(s\log n)$. That's pretty managable!

## Applications of Sparse Recovery 

We motivated the use of sparse recovery algorithms in previous sections with our running example of the market researcher. Now, we will dicsuss more examples for applications of sparse recovery. 

### Electroencephalograms (EEGs)
Electroencephalogram (EEGs) are used to detect electrical activity in the brain given response to different stimuli. At any given moment, only a few sources in the brain are active, out of countless regions and parts. 

<!-- ![image info](/images/eeg.png) -->
<img src="/images/eeg.png" alt="" style=" display: block; margin-left: auto; margin-right: auto; width: 60%;"/>

EEGs produce large amounts of data, and like the other examples and applications we've seen, this requires large amounts of storage that can quickly become too expensive. This problem can be solved by using sparse recovery. By treating the EEG inputs as the vector $x$, neuroscientists can recover the electrical brain signals generated by the patient with only $O(s\log n)$ space! With hundreds or thousands of patients, sparse recovery is extremely effective and valuable, allowing more computational and hardware resources to be used for the analysis of the data itself. 

## References
1. Prof. Thatchaphol’s EECS 586 Lecture Notes
2. https://florian.github.io/count-min-sketch
3. https://www.cse.unsw.edu.au/~cs9314/07s1/lectures/Lin_CS9314_References/cm-latin.pdf
4. https://towardsdatascience.com/big-data-with-sketchy-structures-part-1-the-count-min-sketch-b73fb3a33e2a
5. http://people.csail.mit.edu/rivest/pubs/BFPRT73.pdf
6. https://en.wikipedia.org/wiki/Hash_function
7. https://en.wikipedia.org/wiki/Universal_hashing
8. https://en.wikipedia.org/wiki/Randomized_algorithm
9. https://mathworld.wolfram.com/HashFunction.html
10. https://mathworld.wolfram.com/UniversalHashFunction.html
11. https://en.wikipedia.org/wiki/Quicksort
12. https://barnasahadotcom.files.wordpress.com/2016/01/lec3-haritha-1.pdf
13. http://dimacs.rutgers.edu/~graham/pubs/papers/cmencyc.pdf
14. https://tjn-blog-images.s3.amazonaws.com/wp-content/uploads/2017/07/18164223/online-retail-810x380.jpg