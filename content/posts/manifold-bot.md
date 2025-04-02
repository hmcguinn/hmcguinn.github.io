---
title: Building a diamond Manifold prediction bot in a weekend
date: 2025-04-01T21:51:02-07:00
draft: false
---

I built a prediction market bot in a few hours that ranked fourth in [Manifold's Platinum Sage Gremlins league for March](https://manifold.markets/leagues/23/platinum/sage-gremlins/Te00Mp9cXobRVXjKbYIgn8DPhah2). The bot has 10x'd it's mana using nothing but free tools and simple LLM prompting over the last 2 months.

### Why prediction markets matter

Prediction markets like Manifold let people "bet" on future events and reward accuracy. Metaculus and Manifold are two sites that offer play money, while Kalshi and Polymarket let you wager real money.  

[I'm not particularly well-calibrated at predictions](https://manifold.markets/HoultonMcGuinn), so I wanted to see if I could build a bot to do better. Building it took just a few hours, it runs on [Retool Workflows](https://retool.com/workflows) (which lets us take advantage of free OpenAI credits!), and it has been surprisingly effective.
###  LLM consensus

The bot's logic is straightforward:
- Check our balance
- Get the 15 most recent markets on Manifold
- Ask both GPT-4o and o1 to predict the probability
- Only place bets when:
    - Both models agree within 10% of each other
    - Their consensus differs from the market price by at least 30%

Adding instructions to return 0.0 if the model was not confident, and using the consensus of two models meant when both agreed on a significantly different probability than the market, they were usually right.

The bot currently only works for [Yes/No markets](https://manifoldmarkets.notion.site/Types-of-markets-399d8d2eb2ba4569aaa28a9952eafb23?pvs=25), which are the simplest markets on Manifold.

The market selection logic is pretty crude. I limited it to the 15 most recent markets because of Retool's rate limiting for OpenAI tokens. Since Manifold's API returns markets in chronological order, this approach focuses on newly created markets—which turns out to be where most of the alpha is.

An example API response with market's information [looks like this](https://docs.manifold.markets/api#get-v0markets):

```
[
	{
	  "id": "A2NuS86yps",
	  "creatorId": "b1QnZhu3AfSelAPwOuQ8hzJQ43v1",
	  "creatorUsername": "BJensen",
	  "creatorName": "B Jensen",
	  "createdTime": 1741574763971,
	  "creatorAvatarUrl": "https://lh3.googleusercontent.com/a/ACg8ocLkmEIpPNEXJGXYIL9w8GW89gVYLWGSB3qbzjKS1fdVr5xhAw=s96-c",
	  "closeTime": 1742187540000,
	  "question": "Will the Machines rise up against us?",
	  "slug": "will-the-machines-rise-up-against-u",
	  "url": "https://manifold.markets/BJensen/will-the-machines-rise-up-against-u",
	  "pool": {
	    "NO": 588.2792760938454,
	    "YES": 1699.872901591853
	  },
	  "probability": 0.25709796832169085,
	  "p": 0.5000000000000001,
	  "totalLiquidity": 1000,
	  "outcomeType": "BINARY",
	  "mechanism": "cpmm-1",
	  "volume": 2320.127098408147,
	  "volume24Hours": 0,
	  "isResolved": false,
	  "uniqueBettorCount": 2,
	  "lastUpdatedTime": 1741575274325,
	  "lastBetTime": 1741575274325,
	  "lastCommentTime": 1741574943821
	}
]
```


We'll extract the `question`, `description`, `probability`, `totalLiquidity`, `uniqueBettorCount`, and `pool` fields for more analysis.
### Decision making logic

The core of the bot is passing each market's information to `o1` / `gpt-4o` to generate probability estimates. We pass it to these two models, to try and get some consensus on what the expected probability should be. 

For each market, we pass the relevant information to both models with this prompt:

```
Analyze this prediction market question and estimate the probability of it resolving to YES.
Consider all available information and provide a numerical probability between 0 and 1.

Market title: {{value.question}}
Description: {{value.description }} 

Current market probability: {{ value.probability }} 
Total liquidity: {{value.totalLiquidity }}
Unique bettors: {{value.uniqueBettorCount }}
Unique bettors: {{value.pool }}

If you are not confident in a prediction, return 0.0. 

Provide ONLY the numerical probability estimate with 2 decimal places, nothing else.
```

I originally did not include the current market probability and liquidity, but found it helpful to give additional context to the models. 

The decision algorithm has two criterion:

- The prediction probability for `gpt-4o` and `o1` is within 10%, and
- The consensus prediction is different from the market's current probability by at least 30%

This double-check system helps filter out noisy predictions and focuses on markets with significant mispricing:

```
MIN_PROBABILITY_DIFFERENCE = 0.30  # Only bet when 30% different from market
MAX_PROB_DIFFERENCE = 0.1  # Models must agree within 10%

successful_bets = []

for i, market in enumerate(markets):
    # Skip resolved markets or markets without probability
    if market.get('isResolved', False) or 'probability' not in market:
        continue
    
    # Skip if no o1 probability available
    if not get_o1_probabilities.data[i] or not get_4o_probabilities.data[i]:
        continue
        
    current_prob = market['probability']
    o1_prob = float(get_probabilities.data[i])
    4o_prob = float(get_probabilities_raw.data[i])
    
    # Check for large discrepancies between probabilities
    if abs(o1_prob - 4o_prob) > MAX_PROB_DIFFERENCE:
        print(f"Large difference on {market['question']}: {o1_prob} vs {4o_prob}")
		continue
    
    # Skip markets where AI is not confident
    if o1_prob == 0 and 4o_prob == 0:
        print(f"NOT CONFIDENT: {market['question']}")
        continue
    
    # Determine trade direction based on probability difference
    if o1_prob > current_prob + MIN_PROBABILITY_DIFFERENCE:
        outcome = "YES"
    elif o1_prob < current_prob - MIN_PROBABILITY_DIFFERENCE:
        outcome = "NO"
    else:
        print(f"{market['question']}. Expected probability {o1_prob}. Difference {current_prob - o1_prob}")
        continue
    
    # Place bet and track results
    print(f"Placing {outcome} bet on {market['question']}. Expected probability {o1_prob}")
    success = place_bet(market['id'], INITIAL_BET_AMOUNT, outcome)
    
    if success:
        successful_bets.append(
            f"Placed {outcome} bet on {market['question']}. Expected probability {o1_prob}. Current probability {current_prob}"
        )
    else:
        print("Failed to place bet")

return successful_bets
```


### Results: A slow but steady climb

This [bot](https://manifold.markets/SlicerBot) has been running for ~2 months and has grown its mana from the initial 100 to 1000- a 10x increase. While not amazing, seeing it generate consistent mana has been pretty cool.

![mana](/img/manifold-mana.png)

#### Most profitable bets 

- [Liang Wenfeng in Forbes' Top 10 Billionaires List by December 31, 2035](https://manifold.markets/lakshaytalkstocomputer/liang-wenfeng-in-forbes-top-10-bill) - 38 mana (+110%) with NO bet at 80% probability
- [Will Neymar Jr. be announced by Santos FC by Wednesday?](https://manifold.markets/Dtm999992/will-neymar-jr-be-announced-by-sant) - 24 mana (+81%) with NO bet at 50% probability
- [Will Buloke shire council grant a planning permit for Esoteric Festival in time for the March 2025 event?](https://manifold.markets/BenKloester/will-buloke-shire-council-grant-a-p) - 19 mana (+396%) with YES bet at 20% probability
- [Will Trump announce mass layoffs on Feb 7?](https://manifold.markets/LittleBunnyvzsw/will-trump-announce-mass-layoffs-on) - 18 mana (+125%) with NO bet at 74% probability
- [Space agency announces mission to divert asteroid 2024 YR4 before 22 Dec 2032](https://manifold.markets/7sB/space-agency-announces-mission-to-d) - 15 mana (+50%) with NO bet at 47% probability
#### Least profitable bets 
- [Gulf of America is renamed again by mid 2035 (in Google maps)](https://manifold.markets/Ernie/gulf-of-america-is-renamed-again-by) - 23 mana (-51%) with NO bet at 55% probability
- [Will Trump sign a bill into law in his first week?](https://manifold.markets/Manifest/will-trump-sign-a-bill-into-law-in) - 20 mana (-100%) with YES bet at 31% probability 
- [Bitcoin below $75K before March 15?](https://manifold.markets/predyx_markets/bitcoin-below-75k-before-march-14) - 16 mana (-47%) with YES bet at 39% probability
- [Will Andrew Cuomo announce he is running for NYC mayor by the end of March?](https://manifold.markets/strutheo/will-andrew-cuomo-announce-he-is-ru-hZLqNplOEd) - 14 mana (-98%) with NO bet at 50% probability
- [Will the S&P 500 stock index close higher on Feb 3 than it closed on Jan 31?](https://manifold.markets/MarnixV/will-the-sp-500-stock-index-close-h-NtOsA5C2AS) - 10 mana (-100%) with YES bet at 33% probability

### What it does well

The bot excels at catching newly created markets where the initial probability of 50% is misaligned. By running on a cron job every 2 hours, it can react relatively quickly to new questions. The limiting factor for run frequency is Retool's limit of 500 free workflow runs per month.

The bot bets a very small amount- 5 mana. This doesn't move the market much, even on questions with little liquidity. This is definitely below the threshold that people would choose to manually trade at. 

### What it doesn't do well 

The bot struggles with questions about recent news events. Since `o1` and `gpt-4o` have a [knowledge cutoff of October 2023](https://help.openai.com/en/articles/9824965-using-openai-o1-models-and-gpt-4o-models-on-chatgpt#h_586632b812) (and a lot has happened since then!), it can't accurately assess probabilities for events that depend on post-cutoff information. 3 of the bots worst trades are politics related questions that it has no knowledge of.

### Future improvements 

There's a ton of things that could improve this bot: 
####  Dynamic bet sizing
The bot currently places a fixed bet of 5 mana regardless of confidence or mana balance. This was a quick initial value I chose and never revisited. Now that the mana balance has grown 10x, using Kelly criterion or another position sizing method would make more sense.

#### Position management
This is probably the #1 thing I want to add. Right now, I manually review and sell the most profitable positions every few days to free up Mana. Automating this process would free up my time and potentially improve results through more disciplined exits.

#### Web search integration
The largest losses come from questions about recent events due to the models' knowledge cutoff. Adding tool calling or web search capabilities would help address this blindspot, though this would increase complexity and potentially API costs.

### Lessons learned 
Building this was a lot of fun! It's been able to make a slow, but steady profit.

If you found this interesting, Metaculus also recently put out a [video on how to build a Metaculus bot in 30 minutes](https://www.loom.com/share/fc3c1a643b984a15b510647d8f760685) uses a [similar approach](https://github.com/Metaculus/metac-bot-template/blob/main/main.py). 
