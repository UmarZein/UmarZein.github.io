+++
title = 'Audience Programming'
date = 2024-04-14T05:32:16+07:00
draft = false
description = "Let the audience do the job"
+++
The idea was fully inspired by **[ThePrimeagen](https://www.youtube.com/@ThePrimeagen)'s** [**2300 Devs vs 1 Dev**](https://www.youtube.com/watch?v=W3zhFtSXXG0) VOD in which his twitch's chat competed against him in a coding race wherein they had to sum an array `value` whilst he had to code binary search and hello world in expressjs. The twist is that chat's messages had to be somehow combined and piped into *his* editor.

With half of his twitch chat having more than 3 seconds of video delay and a quarter with more than 5 --- which means they didn't get to play --- it took around 8 minutes for chat to write `values.reduce((x,a)=>x+a)`. 

Now, of course, chat's speed is bound by their stupidity, but I see an opportunity for this to become a fair(er) game.

# The Voice of the Hivemind

The aggregation method used to select an action was to elect the most frequent 1-character message (or `<backspace>` and other special characters) in chat over a 5-second period (which we call a round). While it's certainly intuitive and functional, we can think of a better way.

***

Throughout this article, different approach will be applied to the following sample of messages to illustrate its effect.

```yaml
chatter#23: "const sum = arr"
chatter#27: "const sum = arr.re"
chatter#67: "const tot = arr.reduce"
chatter#68: "const x = 0"
chatter#69: "const sum = "
chatter#61: "const total"
chatter#65: "let sum = 0; arr.forEach(n => sum += n);"
chatter#34: "let tot ="
chatter#38: "lol"
chatter#31: "let sum = 0; for(let i = 0"
chatter#22: "const sum = arr.reduce((n, {Amount})"
chatter#98: "lol"
chatter#21: "const x = arr.reduce"
chatter#33: "arr arr arr arr arr"
```

It will also be tested against a bigger dataset accross a spectrum of entropy, on the low end of which would be a dataset with low number of unique messages w.r.t. the number of messages and the opposite on the other end

## Beyond one character

Looking at the example above, it's reasonable to say that a few of the possible aggregation results would be `const sum =` or `let sum = 0` or just `const`. But it's not as simple as "the most common x" since it shouldn't be `lol`, even though it's the most common _whole_ message, and it _definitely_ shouldn't be ` ` (space), the most common substring. The most common word? That would be `arr`. There's more factor in the game

As guidelines, the aggregated message should:

* be commonly found in the beginnings of messages

* not lean towards very short messages 

## The Proportional Falloff Algorithm

> **TODO** this is not exactly how it works: it needs to be reworded

This algorithm needs two real-number parameters $T_a,T_b \in [0,1]$. It starts off by identifying the most common first letter (in our case, `c`) and appending it into the result buffer. Then, it goes into a loop:

1. Filter only messages that start with the result buffer (in our first iteration, that would be 8 messages coming from chatter # 23, 27, 67, etc.)

1. Find the most common next letter (in our first iteration, the `o` in `const` among those 8 messages)

1. Calculate the proportion $P_a$ of the number of messages that continue with the next letter w.r.t. the number of messages that start with the result buffer (100% in our first iteration since all of the messages that start with `c` also starts with `co`)

1. Calculate the proportion $P_b$ of the number of messages that start with only the result buffer w.r.t. the number of total messages (57% in our case)

1. If the proportion $P_a$ is less than or equal to a threshold $T_a$ which we decide, return the result buffer

1. If the proportion $P_b$ is less than or equal to a threshold $T_b$ which we decide, return the result buffer

1. Append the next letter to the result buffer 

1. Repeat the cycle

> The lower $T_a$, the more permissive the algorithm will be in accepting longer and biased messages. $T_b$ is used to mitigate an issue where the algorithm slides into a hyper-specific result when all the falloffs are smooth

Applying the algorithm with our example, here some results:

* Having $T_a \gt 0.5$ results in `const `, unless $T_b \gt 0.5$ in which case it will only output `c` 

* When $T_a \le 0.5$, $T_b=0$ yields the entirity of chatter#22's message, `const sum = arr.reduce((n, {Amount})`. $T_b=0.1$ yields `const sum = arr.re`. $T_b=0.2$ yields `const sum = arr`. $T_b \in\\{0.3, 0.4, 0.5\\}$ yields `const ` 

Even though it depends in one's case, I recommend $0.3 \lt T_a \le 0.5$ and $0.1 \le T_b \le 0.5$

___

Since we have only introduced 1 algorithm, it is not yet appropriate to benchmark it. Instead, we will compare it with other algorithms later on

## Formula based approach

Since the aim is to capture subprefixes which are as long and common as possible, we can remodel this as a mathematical optimization problem. If $P$ is the subprefix, $L$ is the subprefix length, and $N$ is the subprefix frequency, this problem then becomes 

$$
\mathrm{max}_P [LN]
$$

> basically, this is saying, we are to change $P$ such that $LN$ is maximized

Here is an example score function, $LN$, which we will try to maximize w.r.t. $P$, the subprefix. With this formula, if the prefix length$L$ doubles,  so does the score. However, that will usually reduce its occurrence/frequency $N$ and thus also affecting the score. Moreover, this function is naive
and might contain subtle pitfalls such as equally honoring $L$ and $N$. We will try different variations such as $L(N-1), LN^2, LN(N-1), \dots$

___ 

Unfortunately, with the example, not a lot can be said about their differences because they all returned `const ` except when $L$ has a higher power than $N$ like in $L^2N$ or $L^3N^2$ which will return chatter#22's full message

## Trie Optimization

Since we are searching for prefixes, the trie data structure may prove to be effecient in some cases. The trie that can be implemented in this case is visualized as follows:

![image](/assets/trie_dark.avif)

> **TODO** Continue this

> **TODO** Tf-Idf but opposite? Df-Itf?

