# Understanding Checkpoints in veGovernance

# Table of Contents

- [Understanding Checkpoints in veGovernance](#understanding-checkpoints-in-vegovernance)
  - [Intro](#intro)
  - [Aerodrome and Mechanics](#aerodrome-and-mechanics)
  - [Voting power](#voting-power)
  - [Fetching Historic Voting Power](#fetching-historic-voting-power)
  - [Solving our historical balances for the user](#solving-our-historical-balances-for-the-user)
  - [Checkpoints: Approach 1](#checkpoints-approach-1)
  - [Checkpoints: Another way](#checkpoints-another-way)
  - [Global Checkpoints & Total Supply](#global-checkpoints--total-supply)
    - [Solution: a series of changes](#solution-a-series-of-changes)
    - [Scheduled curve changes and `dslope`](#scheduled-curve-changes-and-dslope)
    - [Advanced: Computing Total Supply for Higher Order Polynomials](#advanced-computing-total-supply-for-higher-order-polynomials)
    - [Example](#example)
  - [TODO](#todo)
  - [Writing checkpoints](#writing-checkpoints)
  - [Reading via binary search](#reading-via-binary-search)
- [Other Curves](#other-curves)
  - [Increasing voting power](#increasing-voting-power)
  - [Nonlinear curves](#nonlinear-curves)
    - [Do you need total supply?](#do-you-need-total-supply)
    - [Linear approximations](#linear-approximations)
- [Extracting curves to separate modules](#extracting-curves-to-separate-modules)
  - [IEscrowCurve](#iescrowcurve)


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

![image](https://github.com/user-attachments/assets/01474031-5168-4549-b1d2-92bc59be613f)

## Fetching Historic Voting Power

In most voting systems, it's important to have snapshots of a user's voting power at a point in time. This helps prevent scenarios where users purchase (or borrow) large amounts of voting power _after_ a particularly significant vote begins. Even in a ve system, where tokens are typically non-transferrable for the duration of the lock, it's important that users cannot backrun voting.

Consequently, we need a function to fetch the historical voting power of a user at given point in time.

In Aerodrome, this function exists and is called `balanceOfNFTAt`.

We might think of a simple way to implement it:

```solidity
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

```solidity

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

A steeper slope = the user's voting power is decaying faster.
A shallower slope = the user's voting power is decaying slower.

Note that the `slope` here is _purely_ a function of the amount of $AERO tokens a user locks, because every user will go from 100% to 0% voting power per $AERO over a 4 year period:

![image](https://github.com/user-attachments/assets/54791691-edc4-4523-b3c9-77253c4de4cf)

> You can write this as a simple derivative $`slope = \frac{d{VotingPower}}{dt}`$. We're not trying to be clever here, this is important to understand when thinking about total supply.

We can therefore express a user's voting power at time t as a function of the `slope`.

To keep things clean, we need to also make sure we use the correct `amount` at a given time (people may increase their amounts with new deposits).

Aerodrome uses some further mathematical conventions here, we've already discussed `slope`, but they introduce the term `bias` to refer to voting power at a point in time.

> Why bias? Bias / intercept are terms borrowed from mathematics (bias was used in the original Curve finance implementation). If we think of our user's historical voting power as series of curves, then at every point we define a new curve with a starting point of `bias`, and a rate of change of `slope`:

![image](https://github.com/user-attachments/assets/e704ddd3-93a6-433d-b9d5-122209315136)

In the above chart you can see that, at 2 years, we add a new deposit without increasing the lock duration. This defines a new, steeper `slope` (a higher rate of decay in voting power), and a new bias - the new starting point of this "new" curve.

With this info, we can rewrite our `UserPoint` struct. We move the user's _current_ balance and endDate into a separate struct `LockedBalance`, and then defined the `UserPoint` struct only to contain information about the user's _decay curve_ at that point in time. We also add in block numbers along with timestamps to ensure clocks are synchronised in the past - here is the full `UserPoint` struct:

```solidity
struct LockedBalance {
    int128 amount;
    uint256 end;
   // ignore for now: permanent is used for locks that are auto-renewed, non-decaying
    bool isPermanent;
}

struct UserPoint {
   int128 bias;
   int128 slope; // # -dweight / dt
   uint256 ts;
   uint256 blk; // block
   // ignore for now: permanent is used for locks that are auto-renewed, non-decaying
   uint256 permanent;
}
```

And our evaluation of a user's historical voting power at `timestamp` is:

```solidity
UserPoint memory lastPoint = getUserPointBefore(_tokenId, timestamp);
votingPower = lastPoint.bias lastPoint.slope * (timestamp - lastPoint.timestamp);
```

## Global Checkpoints & Total Supply

So far we've covered _user_ checkpointing - that is to say, recording historical balances for one user.

What we haven't covered is "Global" checkpoints - that is capturing the historical snapshot of voting power _across all users at any point in history_.

Having a global view of the entire system allows us to compute certain key metrics like total voting power - which is absolutely key in certain governance systems, for example:

- A system that require minimum voting quorums for proposals to pass.
- Systems that need to implicitly re-allocate unused voting power.
- Systems that need to normalise vote weights over total voting power (where 'not voting' is still a vote)
- etc.

> It's worth stressing here: _not all governance systems need to know total supply_. Computing the global state of these systems onchain is significantly more challenging than computing the voting power for a single user, so it's worth asking ourselves if our particular governance system needs such a view.

It's of course possible to evaluate every User Point at a given time, and add the results. This could be feasible offchain even for moderate numbers of voters, but it rapidly becomes untenable onchain, simply due to the nature of the EVM and the high cost of onchain compute and storage reads.

### Solution: a series of changes

What can we do to solve this problem?

Well, it helps to remind ourselves of 3 facts:

1. Global voting power at time `t` is just the aggregate of all User voting power at time `t`
2. Every time we make a change to User Voting power, we record it as a `UserPoint`
3. The only time global voting power will change, is when User Voting power changes

The consequence of 2 & 3 is that we can just update _global voting power_ every time we update _user voting power_. We can use the same parameters as above for user points: bias, slope and timestamp/block to record a `GlobalPoint`, much like a `UserPoint`:

```solidity
struct GlobalPoint {
    int128 bias;
    int128 slope; // # -dweight / dt
    uint256 ts;
    uint256 blk; // block
    // again - ignore this for now
    uint256 permanentLockBalance;
}
```

You can see an example of the global curve vs. the user curve in the chart below, in this example:

- User A deposits 100 $AERO at t = 0, for the full 4 years. The slope of the _global_ supply is exactly the same as the slope of User A
- At t = 2 years, user B deposits 50 $AERO. You see the curve jump as it now represents the _combination_ of users A & B
- At t = 4 years, user A's tokens fully unlock and _their curve is removed from the global curve_
- For the years t = 4 -> t = 6, the curve is now simply the curve of user B

![image](https://github.com/user-attachments/assets/7b604cfc-1d6d-4b13-831b-804f4ea3bb08)

The important point here is that we have all the information to build the _global_ curve if we adjust it when setting every _user_ curve.

You should now be asking: what about that last part?

The final 2 years after User A's stake unlocks changes the shape of the curve, and this doesn't appear to be stored anywhere in `slope` or `bias`.

This is entirely correct, and constitutes one of the more tricky parts of building out a global supply - how do we "schedule" changes to the slope _in the future_.

### Scheduled curve changes and `dslope`

What we need to do, is have a way to log, at the point of deposit:

- When the user's lock expires
- How should we adjust the global curve when said lock expires

Fortunately for us, we know all the above information - we store the `endDate` inside the `LockedBalance`, and we know the slope of the user's decay curve from `UserPoint`.

What this means is that we can store a list of `slopeChanges`: times in the future when we need to _decrease_ the global `slope`, by removing the user's `slope`.

Every time we create or alter a lock, we store a value `dslope` (change in slope) that will be scheduled at a point in time. We store this in a mapping:

```solidity

// timestamp => dslope
mapping(uint256 => int128) public slopeChanges;
```

> `dslope` follows conventions from calculus where you might see $`\frac{dSlope}{dt}`$ aka: the rate of change of the slope over time. It might be helpful to think of `dslope` as a analgous to a second derivative - `slope` is already a measure of how fast voting power decays, and so you can think of `dslope` as something like $`\frac{d^2VotingPower}{dt^2}`$. What's interesting here is that, by scheduling slope changes into discrete intervals, the Global curve avoids having an actual second derivative above zero, which makes things a lot easier for us.

### Advanced: Computing Total Supply for Higher Order Polynomials

> This section is more advanced & speculative - so consider it a WIP but I thought it was interesting.

Polynomial decay functions with orders higher than 1 (aka: quadratic and beyond) are challenging in the EVM to compute in closed form. This is because we run into the same problem as with the linear curve: evaulating at time t in the naive way would require recomputing voting power for all users.

That said, you can in theory use an extremely similar approach as Curve/Aerodrome use in the linear case to compute higher order functions (although this is experimental and I would advise being careful with the implementation). 

> I'm not 100% sure if this generalises to all slopes, but we can show a worked example for a simple curve in $`x^2`$ - I'd argue that cubic and beyond curves most likely introduce needless complexity with minimal benefit, at least in the voting escrow case.

Specifically, you can migrate from storing just `dslope` in a schedule of `slopeChanges`, to storing a pairwise set of coefficients, every time a user changes a lock.

Look at the below graph for 2 users, we have 4 curves:

![image](https://github.com/user-attachments/assets/bfbd36d7-dcec-4749-826c-b7a5006bf66a)

- Curve 1 is just the curve of user A
- Curve 2 represents the aggregate curve of users A & B
- Curve 3 is just the curve of user B
- Curve 4 is zero

What we can do then:

1. Grab the coefficients of the _global curve_ and compute the coefficients for the _user curve_
2. Combine the coefficients of the _global curve_ with the new _user curve_ to compute a new _global curve_
3. Schedule a time t = endDate when we will adjust the _global curve_ by removing the _user curve_ -> this is a curve `delta` that can be aggregated across multiple users.

### Example

## Scenario

- **User 1** deposits at $` t_1 = 0 `$ with an initial voting power $` V_{0,1} = 100 `$
- **User 2** deposits at $` t_2 = 2 `$ (2 years later) with an initial voting power $` V_{0,2} = 80 `$

## Coefficients

Given the function:

$` V_i(t) = V_{0,i} \cdot \left(1 - \frac{(t - t_i)^2}{16}\right) `$

The quadratic coefficients for each user are:

- $` a_i = -\frac{V_{0,i}}{16} `$
- $` c_i = V_{0,i} `$

## Process

1. **At `t = 0`**: User 1 deposits.

   - **User 1's Coefficients**: $` \left(-\frac{100}{16}, 100\right) = (-6.25, 100) `$
   - **Entry Delta**: Add $` (-6.25, 100) `$ to the aggregate coefficients.
   - **Aggregate Coefficients After User 1's Deposit**: $` (-6.25, 100) `$
   - **Exit Delta**: Schedule to subtract $` (-6.25, 100) `$ at $` t = 4 `$ (4 years after deposit).

2. **At `t = 2`**: User 2 deposits.

   - **Current Aggregate Coefficients**: $` (-6.25, 100) `$(from User 1).
   - **User 2's Coefficients**: $` (-5, 80) `$
   - **Entry Delta**: Add $` (-5, 80) `$ to the aggregate coefficients.
   - **Aggregate Coefficients After User 2's Deposit**: $` (-6.25 - 5, 100 + 80) = (-11.25, 180) `$
   - **Exit Delta**: Schedule to subtract $` (-5, 80) `$ at $` t = 6 `$(4 years after User 2's deposit).

3. **At `t = 4`**: User 1’s voting power expires.

   - **Apply Exit Delta**: Subtract $` (-6.25, 100) `$ from the aggregate coefficients.
   - **Aggregate Coefficients After User 1's Exit**: $` (-11.25 + 6.25, 180 - 100) = (-5, 80) `$

4. **At `t = 6`**: User 2’s voting power expires.
   - **Apply Exit Delta**: Subtract $` (-5, 80) `$ from the aggregate coefficients.
   - **Aggregate Coefficients After User 2's Exit**: $` (-5 + 5, 80 - 80) = (0, 0) `$

## Example Table

| Time $` t `$ | Action                 | Aggregate Coefficients Before | Entry/Exit Delta       | Aggregate Coefficients After |
| ------------ | ---------------------- | ----------------------------- | ---------------------- | ---------------------------- |
| 0            | User 1 deposits        | (0, 0)                        | (+) $` (-6.25, 100) `$ | $` (-6.25, 100) `$           |
| 2            | User 2 deposits        | $` (-6.25, 100) `$            | (+) $` (-5, 80) `$     | $` (-11.25, 180) `$          |
| 4            | User 1's power expires | $` (-11.25, 180) `$           | (-) $` (-6.25, 100) `$ | $` (-5, 80) `$               |
| 6            | User 2's power expires | $` (-5, 80) `$                | (-) $` (-5, 80) `$     | $` (0, 0) `$                 |

## Summary

- **Initial Deposit (User 1)**: Starts with the curve $` V(t) = 100 \cdot \left(1 - \frac{t^2}{16}\right) `$ with coefficients $` (-6.25, 100) `$
- **Second Deposit (User 2)**: Adds to the existing curve, resulting in new aggregate coefficients $` (-11.25, 180) `$
- **Expiry of User 1's Voting Power**: Adjusts the aggregate coefficients to reflect the removal of User 1's influence, leaving only User 2's curve.
- **Expiry of User 2's Voting Power**: Brings the aggregate coefficients back to $` (0, 0) `$, indicating no remaining voting power.

## TODO

## Writing checkpoints

## Reading via binary search

# Other Curves

## Increasing voting power

## Nonlinear curves

### Do you need total supply?

### Linear approximations

# extracting curves to separate modules

A generalized escrow contract needs to allow for different curve system. In the Aerodrome implementation, all logic lives inside `VotingEscrow.sol`. What we want is the following architecture:

![image](https://github.com/user-attachments/assets/9824a33d-f9b5-4378-9780-c8381947c591)

What we want is that the curve logic, including the checkpointing state, is safely encapsulated into a single Escrow Curve contract, which is both parameterized for different curve types, and swappable depending on what implentation someone wants. The example curve types include:

- Linear decreasing (default), used in Aerodrome, Velodrome, Curve etc.

  - Vote decreases linearly over time
  - Parameters:
    - MAX_LOCK
    - MIN_LOCK
    - Epoch Length

- Linear increasing: TBC

  - Vote increases linearly over time
  - Parameters:
    - MAX_INCREASE
    - MIN_LOCK
    - epoch length

- Non-linear Increasing:
  - No totalSupply

We first look at the extraction process.

## IEscrowCurve

Smart contracts work especially well in a Service Oriented Architecture (SOA) due to the EVM being a global, standardised runtime. This is a fancy way of saying: "we can split things up quite nicely".

Our plan here is to move the relevant state involved in the escrow curve calculations _out_ of the main `VotingEscrow.sol` contract. This includes:

- Checkpointing logic, in particular the internal checkpoint function
- Structs concerning Global and User points
- Mappings storing historical points
- Epochs (TODO: ??????)

We note also that the `BalanceLogicLibrary` from Aerodrome is convenient to inline into the curve, for the simple reason that the library is effectively achieving the same goal of logic abstraction, albeit while writing to the state of the calling contract.

We instead create an entirely separate state. This does introduce some runtime overhead due to using the `CALL` opcode, but it means that different curves can be defined.
