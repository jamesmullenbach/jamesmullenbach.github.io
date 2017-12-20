---
layout: post
title: Review of "Learning Cooperative Visual Dialog Agents with Deep Reinforcement Learning" (ICCV 2017)
---

[Paper link](https://arxiv.org/abs/1703.06585)

The visual dialog task and dataset introduced in the previous paper does not easily lend itself to supervised learning; it seems unnatural to model each dialog output as a function of the dialog history and image input. This paper addresses this by re-framing visual dialog as a reinforcement learning task, and introducing a new evaluation that is useful for training in this regime.

On a high level, the authors conceptualize a cooperative game for two agents, based on visual data and using natural language to communicate. One agent, the Q-bot, reads a generated caption for an MSCOCO image, which the other agent, the A-bot, can see. In ten rounds, it then asks questions, that the A-bot answers, to learn more about the image and develop its internal representation. At the end of dialog, the Q-bot selects which image is most likely being discussed from the entire set of images. This is clearly a difficult task for many reasons, including the necessity for long term memory, communication in natural (human) language, and visual processing. 

The main strength of the paper is in its careful framing of the visual dialog task they seek to solve. The methodological motivation is a synthesis of many new ideas from recent breakthroughs in AI, including learning by self-play, which influences the idea to create a game-like environment in which agents can learn from a theoretically infinite space of interaction, and adversarial learning, which influences the decision to let the game shape the dialog into a ‘good’ one, by making the game objective require informative dialog. It was interesting to see that this is perhaps the first dialog system of any kind, let alone visually-grounded, trained in this way, without manually defining well-formed language. They are even able to generalize the game to other, perhaps non-visual objectives, as the reward function is constructed like a general loss function for regression between a target and a predictor. 

One of the weaknesses of the paper is in the quantitative evaluation. While the results for the guessing game evaluation are strong and show a significant improvement over the supervisedly-trained baseline, the other visual dialog task of answer ranking has a much smaller improvement. It is good that the authors chose to include a model variant that trains the A-bot jointly with ground truth answer supervision to enforce human-like language. Another weakness is that the authors don’t seem to specifically show with evidence how the Q-bot learns to ask questions the A-bot is better at answering. Some of the examples show this, but it wasn’t addressed quantitatively. Also, The idea of using a human-as-questioner or human-as-answerer is intriguing and the results could be illuminating as to the successes and failures of this model. It would be a technical challenge, but great to see results for this in the future using the developed AMT chat interface.

Overall, this paper introduces a novel task and training paradigm for visually-grounded dialog between intelligent agents. The quantitative results may be a little lacking, but this could be due to the partially contrived nature of the evaluation, and the direction shows promise for future research, which could improve with more robust policy networks, better pre-training, or other approaches.

I would be interested to see the conditions under which natural language communication between the agents starts to break down. Since the use of English mostly comes from the supervised learning pre-training, and there is not anything in the reward function to explicitly motivate ‘natural’ language (except in the Frozen-Q-multi case), are there cases where the bots will enter into un-natural or non-human interpretable communication? What are they, and does adding back the ground truth answer supervision help reduce this?

