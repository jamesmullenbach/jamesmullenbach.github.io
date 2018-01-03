---
layout: post
title: Word vectors are (not) magic - a holiday experiment with Codenames
---

(This post has an accompanying Jupyter notebook that you can play around with [here](https://github.com/jamesmullenbach/codenames)).

Over the break between semesters, I've spent a lot of time with family playing a popular board game called [Codenames](https://czechgames.com/en/codenames/).

If you haven't played, the gist is that one player from each team, the 'spymaster', tries to get their team members to select their team's assigned words from a group of 25 while avoiding the other team's words and a game-ending 'assassin' word, using one word clues. It's like the game show Password, except clues can apply to any number of words. 

It's a fun language based game and makes for an interesting testbed for simple experiments like the one I'm about to talk about.

Naturally, I thought about how a computer might play this game. Let's look at a simple way to suggest clues for the spymaster!

## Spymaster as a machine learner

First, we'll need some semantic representation for each word. For this, I chose Google's [pre-trained word2vec vectors](https://code.google.com/archive/p/word2vec/), but you could use the word vector or other representation of your choosing. 

The problem for the spymaster is a really tidy analogy for classification! Specifically, we have a dataset of 25 words, and on any turn we pick a few that we want our team to guess, while avoiding the other team's words. We can think of our words as positive examples, and the other team's words and the assassin word as negative examples. This plays right into the word analogy strengths of word2vec. A simple visual of an ideal clue:

<img src='/assets/codenames.png'/>


## An example board

As an initial check that this can work, let's generate some clues that match to just one of our team's words. We'll get an example board with [this resource](https://horsepaste.com).

<img src='/assets/board.png'/>

Let's play blue, then our opponents are red. There are some 'bystander' words (the black ones) which belong to neither team but are helpful to avoid. Let's ignore these for simplicity. Time to get coding! First, load the word2vec vectors into `gensim` (I limit the vocab for memory reasons, and to avoid esoteric clues).  We can also make lists for the relevant words.

```python
import gensim
model = gensim.models.KeyedVectors.load_word2vec_format('GoogleNews-vectors-negative300.bin', binary=True, limit=200000)
ours = ['watch', 'star', 'mole', 'Berlin', 'limousine', 'day', 'wind', 'cap', 'thumb']                                  
theirs = ['smuggler', 'crown', 'cotton', 'palm', 'pumpkin', 'giant', 'link', 'dog']                                     
assassin = 'tie'
```

## Proof of concept: one- and two-word clues

There's several ways to get good one-word clues. Let's start by just finding the most similar words to each of our team's words:

```python
clues = {}
for our_word in ours:
    #get similar words from google vocab, and lowercase them
    candidates = [(c.lower(), s) for c,s in model.most_similar(positive=[our_word], topn=50)]
    #per game rules, we should exclude multi-word results and words that use part of the clue word.
    #we could use stemmers for this maybe, but let's keep it simple
    clues[our_word] = [(c,s) for c,s in candidates if '_' not in c and c not in our_word and our_word not in c][:5]
```

This gives (top result only for brevity):
```
{'Berlin': ('munich', 0.6743212938308716),
 'cap': ('hat', 0.4268711507320404),
 'day': ('week', 0.65529865026474),
 'limousine': ('limos', 0.6517890095710754),
 'mole': ('birthmark', 0.46605658531188965),
 'star': ('heartthrob', 0.543801486492157),
 'thumb': ('pinkie', 0.6484930515289307),
 'watch': ('see', 0.5326846837997437),
 'wind': ('gusts', 0.5962637662887573)}
```

Most of these seem pretty good! I'm not sure if `'limos'` would fly by the rules, but the next best result, `'chauffered'`, is valid. It's also interesting to observe how the words with more senses `('cap', 'mole', 'watch')` have lower scores. Let's try doing the same for two-word clues. First, let's generalize the rule-keeping a bit:

```python
def verify(candidate, word_list):
    if '_' in candidate:
        return False
    for word in word_list:
        if word in candidate or candidate in word:
            return False
```

Now, get candidate clues for all combinations of two words with `itertools`:

```python
import itertools
clues_2 = {}
for our1, our2 in itertools.combinations(ours, r=2):
    our_list = [our1, our2]
    candidates = [(c.lower(), s) for c,s in model.most_similar(positive=our_list, topn=50)]
    clues_2[(our1, our2)] = [(c,s) for c,s in candidates if verify(c, our_list)][:5]
```

I won't post the whole results here. Some interesting suggestions are: `'standout'` and `'legend'` for `('watch', 'star')`, `'spies'` for `('watch', 'mole')`, and `'weather'` for `('watch', 'wind')`. One impressive result is `'stasi'` for `('mole', 'Berlin')`, which -- I had to Wikipedia this -- is the [former German secret police during the Cold War](https://en.wikipedia.org/wiki/Stasi). A couple good clues (at least, I think so) that it misses are `'solar'` for `('star', 'wind')` and `'celebrity'` for `('star', 'limousine')`. Overall, many suggested clues really only pertain to one of the two words. So, we can improve.

## Avoiding opponent's words

So far, we have not tried to avoid the 'bad' words: the other team's words and the assassin. For instance, `'wrist'` is not a *great* clue for `('watch', 'thumb')`, as it may confuse teammates with the opposing team's word `'palm'`. All we have to do is add the argument `negative=theirs+[assassin]` to the penultimate line of the last code block. Unfortunately, these results are poor and seem to be an artifact of `gensim`'s similarity computing method - the top result for many pairs of words is `'portfolios'`. I suspect that the method places too much emphasis on the negative words, as there are more negative than positive words.

To mitigate this, we can just use our entire team's words as the positive examples. 

```python
candidates = model.most_similar(positive=ours, negative=theirs+[assassin], topn=100)
```

Then, since these clues are really trying to capture all nine of our words, we can filter down the results and find out which of our team's words each clue is best for.

```python
import operator
cand_words = []
for cand,_ in candidates:
    if verify(cand, ours):
        scores = {}
        for our_word in ours:
            scores[our_word] = model.similarity(cand, our_word)
        cand_words.append((cand, sorted(scores.items(), key=operator.itemgetter(1), reverse=True)[:5]))
```

This gives a few promising results that can target more than one of our teams words. Here are the 10 best clues, in order, and 5 corresponding team words, that a computer thinks we should use:

```
[{'flashbulbs': ['star', 'limousine', 'wind', 'watch', 'day']},
 {'eve': ['day', 'star', 'Berlin', 'watch', 'limousine']},
 {'rookies': ['star', 'cap', 'day', 'watch', 'wind']},
 {'VIPs': ['limousine', 'watch', 'star', 'mole', 'Berlin']},
 {'Tomczyk': ['Berlin', 'cap', 'wind', 'watch', 'day']},
 {'Stasi': ['Berlin', 'mole', 'wind', 'watch', 'thumb']},
 {'countdown': ['day', 'watch', 'mole', 'star', 'wind']},
 {'##/#-hour': ['day', 'watch', 'limousine', 'wind', 'thumb']},
 {'invitees': ['limousine', 'star', 'watch', 'day', 'Berlin']},
 {'hour': ['day', 'watch', 'limousine', 'wind', 'star']}]
```

`'Flashbulbs'` really seems great, though `'paparazzi'` may be even more direct for `('star', 'limousine', 'watch')`. I also like the suggestion `'night'` (ranked 15), for `('day', 'watch', 'star')`.

## Clue giving by a classifier

Let's try a whole new approach based on machine learning. We can easily find a linear classifier as illustrated before, then just project that to our vocabulary!

```python
import numpy as np
from sklearn import linear_model
X = np.array([model.word_vec(word) for word in ours + theirs + [assassin]])
Y = np.array([1 for i in range(len(ours))] + [-1 for i in range(len(theirs)+1)])
#this is an SVM by default
clf = linear_model.SGDClassifier(max_iter=1000, fit_intercept=False, penalty='none')
clf.fit(X,Y)
vec = clf.coef_[0]
```

Note how we don't want to learn a bias term, as we're really looking for pure similarity to the hyperplane. We can also afford no regularization, as there's not really a concept of generalization here. The resulting vector is not normalized the same way as google's word2vec vectors are, but this won't affect the relative differences in similarity which are important. 

How do we find the closest word vector to the learned hyperplane? Luckily, `gensim` has `similar_by_vector`, which does exactly what we want:

```python
candidates = model.similar_by_vector(vec, topn=100)
```

Finding the words for these candidates the same way as before gives us:

```
[{'flashbulbs': ['star', 'limousine', 'wind', 'watch', 'day']},
 {'wattage': ['star', 'wind', 'Berlin', 'limousine', 'watch']},
 {'VIPs': ['limousine', 'watch', 'star', 'mole', 'Berlin']},
 {'radar': ['watch', 'wind', 'cap', 'star', 'mole']},
 {'invitees': ['limousine', 'star', 'watch', 'day', 'Berlin']},
 {'limos': ['limousine', 'star', 'watch', 'Berlin', 'mole']},
 {'earpieces': ['limousine', 'cap', 'watch', 'mole', 'thumb']},
 {'hotel': ['limousine', 'star', 'Berlin', 'day', 'watch']},
 {'supernovas': ['star', 'wind', 'cap', 'mole', 'day']},
 {'visor': ['cap', 'thumb', 'mole', 'watch', 'wind']}]
```

The results are very similar, which is a great sanity check. Superficially, they do also seem more "accessible" as there are fewer proper nouns. It also has the advantage of being more flexible, as the problem of over-focusing on negative examples is gone, so we can get good results for just a couple of our team's words:

```python
ours_sub = ['watch', 'star']
X = np.array([model.word_vec(word) for word in ours_sub + theirs + [assassin]])
Y = np.array([1 for i in range(len(ours_sub))] + [-1 for i in range(len(theirs)+1)])
clf = linear_model.SGDClassifier(max_iter=1000, fit_intercept=False, penalty='none')
clf.fit(X,Y)
vec = clf.coef_[0]
```

Filtering to valid results gives a top 5 of:

```
['weeknights', 'broadcasts', 'RADAR', 'viewing', 'forewarned']
``` 

Pretty reasonable!

Finally, a machine learning question: Is our projection-based approach the best way to utilize the finite hypothesis class? There might be legitimate research on this.

## Extensions

Overall, we have seen that a word vector approach can be quite powerful! However, my opinion is that I can usually still come up with clues that are at least as good, with a few exceptions. The computer is certainly faster than me at it. There's some practical and hypothetical ways to extend this! In decreasing order of practicality:

- I ignored 'bystander' words, which don't belong to any team, but are useful to avoid. We could introduce a weighting system for negative examples and work that into our algorithm somehow.
- Augment word vectors with knowledge of common colloquialisms from some corpus, and common sense relationships from WordNet and ConceptNet that are not captured by proximity-based word vectors. This will help us use clues such as "green" for "thumb" or "night" for "day" and "cap" that are obvious to people.
- The lesson of the game is to consider all senses of a word. We've used word2vec here, which just has a single vector that is meant to represent all senses. Taking an approach that allows for modeling multiple senses would likely be fruitful here.
- (pure hypothetical) When we play the game in real life, we know something about our teammates' internal representations of words. One clue that was given when I played related to Doctor Who, and applied to four words! Goal: use personalized word vectors for each of your teammates :)

Thanks for reading! Test out these ideas in a game and decide for yourself if word vectors are magic.

