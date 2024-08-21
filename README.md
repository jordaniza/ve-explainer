# Understanding Checkpoints in veGovernance

## Intro

Vote-Escrowed Tokens ("veTokens") represent a form of token design where token holders must lock/stake governance tokens for periods of time in order to vote in a Decentralized Autonomous Organization ("DAO").

The veToken mode was popularised by DeFi protocols such as Curve finance, and has seen widespread popularity over the years in protocols like Angle, Origin, Velodrome, Solidly, PieDAO and others.

In many veToken implementations, dynamic token balances are computed based on stakers' remaining lock times. In this document we explain how such systems work, using the Aerodrome AMM and Solidity as an example. We show in particular how the checkpointing system and logic functions, and how such logic can be generalised to create different curve implementations.

## Aerodrome and Mechanics

Before jumping into the logic it's worth grabbing some further context about Aerodrome and how it works.

Aerodrome is a popular DeFi protocol that is heavily inspired by Curve finance. Users can lock $AERO tokens for periods of up to 4 years and in return receive voting power in the Aerodrome protocol.

Voters in Aerodrome can direct future $AERO emissions by allocating their vote weight across _gauges_. A gauge in Aerodrome typically points to a particular liqudity pool (such as WETH-USDC). Users who provide liquidity in such a pool receive standard Liqudity pool fees + emissions dictated by the % of votes that pool/gauge receives.

As an example: 100 $AERO are being distributed as incentives.

- WETH/USDC gauge receives 60% of votes (60 $AERO)
- USDC/DAI gauge receives 40% of votes (40 $AERO)

Liquidity providers who provide liquidity in the WETH/USDC pool will share the 60 $AERO rewards, in addition to trading fees.

## Voting power

You might be asking then - how is voting power calculated?

In Aerodrome, when you lock \$AERO, you receive a veNFT (Vote Escrowed NFT) which contains information about your locked \$AERO, such as:

- Lock End Date
- Lock Amount
- Whether the lock should automatically renew until you manually choose to unlock ("Permanent")

Assuming you don't opt for a permanent lock, you can choose to lock \$AERO for between 1 week and 4 years. You gain the maximum amount of voting power if you lock for 4 years.

As the end date of your lock approaches, your voting power decreases. This can be visualised in the graph below:

![./images/decay-curve-linear.png]

## Fetching Historic Voting Power

In most voting systems, it's important to have snapshots of a user's voting power at a point in time. This helps prevent scenarios where users purchase (or borrow) large amounts of voting power _after_ a particularly significant vote begins. Even in a ve system, where tokens are typically non-transferrable for the duration of the lock, it's important that users cannot backrun voting.

Consequently, we need a function to fetch the historical voting power of a user at given point in time.

In Aerodrome, this function exists and is called `balanceOfNFTAt`.

We might think of a simple way to implement it:

```sol
/// Not real code: this is just an example
function balanceOfNFTAt(
  uint256 _tokenId,
  uint256 timestamp
) public view returns (uint256 votingPower) {
  // fetch the lock details using the token Id
  (uint256 amount, uint256 endDate) = locks[_tokenId];

  // compute the voting power using the decay function
  uint256 votingPowerPerSecondRemaining = amount / MAX_LOCK_TIME;
  uint256 secondsRemainingAtTimestamp = endDate - timestamp;

  return votingPowerPerSecondRemaining * secondsRemainingAtTimestamp;
}
```

The above is nice and simple and easy to understand: To find the user's voting power, we simply grab the amount locked and their end date, and evaulate the curve at the passed timestamp.

The challenge here then becomes: what to do if a user wants to _change their lock_?

By changing a lock we typically mean 1 of the following options:

- Increasing the _amount_ locked
- Increasing the _duration_ locked (renewing)

This is a case where the above code will fail.

Why? Consider the very simple example where Alice locks 10k \$AERO at t = 1000, the maximum lock time is 5000 and she locks for the full 5000 seconds.

At t = 2000, Alice's voting power can be calculated by:

```c
votingPowerPerSecondRemaining = 10,000 / 5000 = 2
secondsRemainingAtTimestamp = 5000 - 2000 = 3000
votingPower = 2 * 3000 = 6000
```

At t = 3000 she locks 10k more \$AERO.

We calculate Alice's new voting power:

```c
votingPowerPerSecondRemaining = 20,000 / 5000 = 4
secondsRemainingAtTimestamp = 5000 - 3000 = 2000
votingPower = 4 * 2000 = 8000
```

But what happens when we now query Alice's voting power at t = 2000?

```c
votingPowerPerSecondRemaining = 20,000 / 5000 = 4
secondsRemainingAtTimestamp = 5000 - 2000 = 3000
votingPower = 4 * 3000 = 12,000
```

_Because we have not kept track of historical balances, we incorrectly record Alice's past voting power as higher than it was at the time_.

> Note as well, we haven't said anything yet about computing the total voting power across all users. This is important if we need to compute things like totalSupply, quorums etc. We will go into this in depth later.

## Solving our historical balances for the user

So we need to fix the above code, and have some way of preserving history, so we can freeze voting power and prevent backrunning attacks.

One option might be to mint a new veNFT for each lock. This is certainly possible but requires that locks become immutable - once you've locked tokens, that lock cannot be changed or voting history will be unreliable again.

For some protocols, this has worked. Especially in the case of low numbers of locks, this is a simple approach that can be quite effective. On the other hand, for users who are frequently engaging with the protocol, managing large numbers of locks can get unwieldy, and aggregating voting power can be extremely expensive.

What if, instead, we ensured we kept an historical record of locks.

## Checkpoints: Approach 1

Checkpoints are absolutely critical for understanding the escrow mechanism in protocols like Curve and Aerodrome.

Put simply: a checkpoint is written to the ledger every time a user makes a change to their lock. This records any changes in the users lock at that time, so that historical queries can return an accurate view of past state.

> If you're familar with vote delegation in ERC20Votes, the motivation is exactly the same: we store historic checkpoints of voting power every times a user transfers tokens or changes delegation. The implementation in veTokens is more advanced but shares many of the same principles.

Our basic information that we need for a checkpoint for a user deposit is:

1. How many governance tokens did the user have locked at that point?
2. When does the user's lock end?
3. When was the point recorded?

So we _could_ define a struct `UserPoint` with fields `amount`, `endDate` and `timestamp`. Every time a user would modify a lock, we would record a new `UserPoint`.

For a given timestamp `t` in the past, we would then find the nearest `UserPoint` _before_ `t`, and then recompute the voting power using the formula we used to calculate Alice's balance, above.

```sol

struct UserPoint {
    uint256 amount;
    uint256 endDate;
    uint256 timestamp;
}

// store historic user checkpoints
// we don't yet show checkpointing logic but you get the idea
mapping (uint256 timestamp => UserPoint) userPointHistory;


function balanceOfNftAt(uint256 _tokenId, uint256 t) public view returns (uint256 votingPower) {
    // find the nearest user point before time t - we will come back to this part
    // but it involves binary searching through past points
    UserPoint memory checkpoint = getNearestUserPointBefore(_tokenId, t);

    votingPowerPerSecondRemaining = checkpoint.amount / MAX_LOCK_TIME;
    secondsRemainingAtTimestamp = checkpoint.endDate - t;
    return votingPowerPerSecondRemaining * secondsRemainingAtTimestamp;
}
```

## Checkpoints: Another way

The above is okay but we can do better. Rather than recompute `votingPowerPerSecondRemaining`, why not just store it directly?

In fact, we should talk a bit about the mathematics of `votingPowerPerSecondRemaining`, as it's crucial to understanding how voting power decays.

`votingPowerPerSecondRemaining` is another way to express a _rate of change_, in our case, it's the _rate of change in voting power, with respect to time_.

Visually, you can see this as the `slope` of the User's voting power curve:

![slopes.png]

A steeper slope = the user's voting power is decaying faster.
A shallower slope = the user's voting power is decaying slower.

> You can write this as a simple derivative $ slope = \frac{d{VotingPower}}{dt} $
