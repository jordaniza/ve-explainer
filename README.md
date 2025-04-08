# VE Explainer

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

Aerodrome is a popular DeFi protocol, forked from Solidly which in turn drew heavy inspiration from Curve Finance. Users can lock $AERO tokens for periods of up to 4 years and in return receive voting power in the Aerodrome protocol.

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

At any given moment then, a user may hold a veNFT with some voting power. We need a way to model the dynamic voting power on chain.

In Aerodrome, this function exists and is called `balanceOfNFTAt`.

We might think of a simple way to implement it:

```solidity
/// UNAUDITED CODE: do not use in production
// define a simple data structure to collect veNFT data
struct Lock {
    uint256 amount;
    uint256 endDate;
}
mapping(uint256 tokenId => Lock) locks;

// an example function for fetching the balance of a veNFT identified by _tokenId
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

The challenge here then becomes: **what to do if a user wants to _change their lock_?**

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
/// UNAUDITED CODE: do not use in production
struct UserPoint {
    uint256 amount;
    uint256 endDate;
    uint256 timestamp;
}
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

## The full balance function

We now have most of the conceptual pieces to understand the full balance querying function used inside Aerodrome, let's first take a look at it, then evaluate line by line.

```solidity
    function balanceOfNFTAt(
        mapping(uint256 => uint256) storage _userPointEpoch,
        mapping(uint256 => IVotingEscrow.UserPoint[1000000000]) storage _userPointHistory,
        uint256 _tokenId,
        uint256 _t
    ) external view returns (uint256) {
        uint256 _epoch = getPastUserPointIndex(_userPointEpoch, _userPointHistory, _tokenId, _t);
        // epoch 0 is an empty point
        if (_epoch == 0) return 0;
        IVotingEscrow.UserPoint memory lastPoint = _userPointHistory[_tokenId][_epoch];
        if (lastPoint.permanent != 0) {
            return lastPoint.permanent;
        } else {
            lastPoint.bias -= lastPoint.slope * (_t - lastPoint.ts).toInt128();
            if (lastPoint.bias < 0) {
                lastPoint.bias = 0;
            }
            return lastPoint.bias.toUint256();
        }
    }

    function getPastUserPointIndex(
        mapping(uint256 => uint256) storage _userPointEpoch,
        mapping(uint256 => IVotingEscrow.UserPoint[1000000000]) storage _userPointHistory,
        uint256 _tokenId,
        uint256 _timestamp
    ) internal view returns (uint256) {
        uint256 _userEpoch = _userPointEpoch[_tokenId];
        if (_userEpoch == 0) return 0;
        // First check most recent balance
        if (_userPointHistory[_tokenId][_userEpoch].ts <= _timestamp) return (_userEpoch);
        // Next check implicit zero balance
        if (_userPointHistory[_tokenId][1].ts > _timestamp) return 0;

        uint256 lower = 0;
        uint256 upper = _userEpoch;
        while (upper > lower) {
            uint256 center = upper - (upper - lower) / 2; // ceil, avoiding overflow
            IVotingEscrow.UserPoint storage userPoint = _userPointHistory[_tokenId][center];
            if (userPoint.ts == _timestamp) {
                return center;
            } else if (userPoint.ts < _timestamp) {
                lower = center;
            } else {
                upper = center - 1;
            }
        }
        return lower;
    }
```

### Arguments

Let's start with the variables passed to the function.

```solidity
        mapping(uint256 => uint256) storage _userPointEpoch,
        mapping(uint256 => IVotingEscrow.UserPoint[1000000000]) storage _userPointHistory,
        uint256 _tokenId,
        uint256 _t
```

In Aerodrome, the balanceOfNFTAt function is defined inside a library known as BalanceLogicLibrary. This allows us to use storage mappings as arguments in the library, which can then be passed to the balance function.

We have 2 such mappings: `userPointEpoch` and `userPointHistory`. Respectively these are used to track the length of the array and to store our user points.

> Why not use `UserPoint[]`? This is likely a carryover from the original [Vyper Implementation](https://github.com/curvefi/curve-dao-contracts/blob/master/contracts/VotingEscrow.vy#L95). Vyper introduced the DynArray type in 0.3.2, after the Curve contracts were written.

We also need the veNFT ID (`_tokenId`) and the timestamp `_t` to query the balance at.

### Binary search to find the nearest epoch

Our first line fetches the "epoch" of the UserPoint, where epochs are incrementally written with a strictly increasing timestamp. This results in an array sorted by timestamp.

When we pass a timestamp, we want to find the point **before or at that timestamp**. We achieve this via the `getPastUserPointIndex` function. This returns the index in the UserPoint array of the point that we want.

Note that:

- User Epochs are written at 1
- If userEpoch = 0, that means we have no data for that tokenId

This function does 2 things:

**Step 1: check for early return cases**

Before we commit to the binary search, we check some simple cases to save gas, specifically:

```solidity
        /// get the current epoch (effectively the array length)
        uint256 _userEpoch = _userPointEpoch[_tokenId];

        /// if no data, return an index of zero
        if (_userEpoch == 0) return 0;

        /// if the most recent value is in the past, skip binary search and just use the latest value
        if (_userPointHistory[_tokenId][_userEpoch].ts <= _timestamp) return (_userEpoch);

        /// user epochs for actual data start at 1. Hence if the ts of that user point is in the future, the user had no voting power at that point.
        /// this would be the case if one queried before a user's first lock with that tokenId
        if (_userPointHistory[_tokenId][1].ts > _timestamp) return 0;

```

**Step 2: Perform binary search**

Explaining binary search in depth is beyond the scope of this article, but suffice to say that, for an arbitrary timestamp `t`, Binary search allows us to query a sorted array in O(log N) time complexity.

Thus, for users with frequent updates to their lock, we have an efficient way to retrieve the index of the UserPoint we need.

### Retrieving our voting power

Now we have the nearest epoch at or before an arbitray timestamp, we can compute the voting power.

```solidity
        /// epochs with data will begin at 1, so if no data at that point, return 0 for voting power
        if (_epoch == 0) return 0;

        /// otherwise used the fetched id to grab the point from storage
        IVotingEscrow.UserPoint memory lastPoint = _userPointHistory[_tokenId][_epoch];

        /// Aerodrome allows for veNFTs that set a "permanent" lock that does not decay.
        /// we do not cover this case in this post but it's here for completeness
        if (lastPoint.permanent != 0) {
            return lastPoint.permanent;

        /// this is our balanceOfNFT calcuation from earlier. We take the original bias loaded from the point
        /// then compute how much voting power has been lost due to time decay since that point was written
        /// In other words this is:
        /// (starting voting power at lastPoint.ts) - (rate of change per second) * (seconds since lastPoint.ts)
        } else {
            lastPoint.bias -= lastPoint.slope * (_t - lastPoint.ts).toInt128();
            /// in the event that someone holds a lock for a long time their voting power might go negative - we dont allow this so bound voting power at 0
            if (lastPoint.bias < 0) {
                lastPoint.bias = 0;
            }
            return lastPoint.bias.toUint256();
        }
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

There is one final piece missing though: don't we need to **remove** a user's voting power once they've hit zero?

Specifically, in the final 2 years after User A's stake unlocks, we expect to remove their voting power from the total voting power curve. Unfortunately, this doesn't appear to be stored anywhere in `slope` or `bias`.

This is entirely correct, and constitutes one of the more tricky parts of building out a global supply - how do we "schedule" changes to the slope _in the future_.

### Scheduled curve changes and `dslope`

What we need to do, is have a way to log, at the point of deposit:

- When the user's lock expires
- How should we adjust the global curve when said lock expires

Fortunately for us, we know all the above information - we store the `endDate` inside the `LockedBalance`, and we know the slope of the user's decay curve from `UserPoint`.

What this means is that we can store a list of `slopeChanges`: times in the future when we need to _decrease_ the global `slope`, by removing the user's `slope`. Put another way, slope changes represent when the global voting power curve should get **shallower**, because a particular tokenId has no voting power left, and so is no longer decreasing.

That's the high level overview anyway, at this point we can start reviewing the full checkpointing implementation to see how it is achieved in practice.

## The full checkpointing function

[The full checkpoint function inside Aerodrome](https://github.com/mtfuji25/aerodome-contracts/blob/77015d18d333ef553546e4570c19b89043311cc6/contracts/VotingEscrow.sol#L591) is a large, 150+ line function. While it may look intimidating, we have all the building blocks we need to understand it.

We can break the function up into the following parts:

1. Initialize Variables
1. Computing the user point
1. Backfilling total supply
1. Adding the user point to the global point
1. Adjusting dslopes
1. Writing the user point

### Arguments

Let's first take a look at the arguments to the function:

```solidity
function _checkpoint(uint256 _tokenId, LockedBalance memory _oldLocked, LockedBalance memory _newLocked)
```

Checkpointing takes the veNFT ID (\_tokenId), and a pair of LockedBalance structs representing the previous lock data and the updated lock data.

Our job is to update the UserPoint and GlobalPoint's associated with this \_tokenId based on the difference between \_oldLocked and \_newLocked.

Our first step draws heavily upon the logic we've seen up to now. Our aim here is to compute the bias and slope to store on the UserPoint. Let's first look at the full code:

```solidity
        UserPoint memory uOld;
        UserPoint memory uNew;
        int128 oldDslope = 0;
        int128 newDslope = 0;
        uint256 _epoch = epoch;

        if (_tokenId != 0) {
            uNew.permanent = _newLocked.isPermanent ? _newLocked.amount.toUint256() : 0;
            // Calculate slopes and biases
            // Kept at zero when they have to
            if (_oldLocked.end > block.timestamp && _oldLocked.amount > 0) {
                uOld.slope = _oldLocked.amount / iMAXTIME;
                uOld.bias = uOld.slope * (_oldLocked.end - block.timestamp).toInt128();
            }
            if (_newLocked.end > block.timestamp && _newLocked.amount > 0) {
                uNew.slope = _newLocked.amount / iMAXTIME;
                uNew.bias = uNew.slope * (_newLocked.end - block.timestamp).toInt128();
            }

            // Read values of scheduled changes in the slope
            // _oldLocked.end can be in the past and in the future
            // _newLocked.end can ONLY by in the FUTURE unless everything expired: than zeros
            oldDslope = slopeChanges[_oldLocked.end];
            if (_newLocked.end != 0) {
                if (_newLocked.end == _oldLocked.end) {
                    newDslope = oldDslope;
                } else {
                    newDslope = slopeChanges[_newLocked.end];
                }
            }
        }
```

### Initializing Variables

First, we initialize our variables.

```solidity
        UserPoint memory uOld;
        UserPoint memory uNew;
        int128 oldDslope = 0;
        int128 newDslope = 0;
        uint256 _epoch = epoch;
```

Our UserPoints and slopeChanges (called dslopes) are initialized with default values. We will come back to epoch but we need to first talk about slope changes...

Every time we create or alter a lock, we store a value `dslope` (change in slope) that will be scheduled at a point in time. We store this in a mapping:

```solidity

// timestamp => dslope
mapping(uint256 => int128) public slopeChanges;
```

> `dslope` follows conventions from calculus where you might see $`\frac{dSlope}{dt}`$ aka: the rate of change of the slope over time. It might be helpful to think of `dslope` as a analgous to a second derivative - `slope` is already a measure of how fast voting power decays, and so you can think of `dslope` as something like $`\frac{d^2VotingPower}{dt^2}`$. What's interesting here is that, by scheduling slope changes into discrete intervals, the Global curve avoids having an actual second derivative above zero, which makes things a lot easier for us.

So in our code above, we initialize the slope changes to zero before beginning our first code block.

### Computing voting power

```solidity
        if (_tokenId != 0) {
            uNew.permanent = _newLocked.isPermanent ? _newLocked.amount.toUint256() : 0;
            // Calculate slopes and biases
            // Kept at zero when they have to
            if (_oldLocked.end > block.timestamp && _oldLocked.amount > 0) {
                uOld.slope = _oldLocked.amount / iMAXTIME;
                uOld.bias = uOld.slope * (_oldLocked.end - block.timestamp).toInt128();
            }
            if (_newLocked.end > block.timestamp && _newLocked.amount > 0) {
                uNew.slope = _newLocked.amount / iMAXTIME;
                uNew.bias = uNew.slope * (_newLocked.end - block.timestamp).toInt128();
            }
```

Assuming a tokenId is passed (0 being a manual checkpoint), then we need to update the new and the old user point.

The code itself is duplicated for new and old but is gated by a condition:

```solidity
if (_oldLocked.end > block.timestamp && _oldLocked.amount > 0) {
```

Which simply checks that the lock is not expired nor empty. It's consequently entirely possible to pass a zero value lock to the checkpoint function, perhaps during an exit, in which case there is no bias nor slope to compute.

The body of the function should be obvious if you've been following along up until now. We compute the slope and bias based on the elapsed time since the last lock, and the amount in the lock.

Next we fetch our slope changes. These changes are batched together for all users at the locked.end - we can see this elsewhere in the codebase in the `createLock` function:

```solidity
    function _createLock(uint256 _value, uint256 _lockDuration, address _to) internal returns (uint256) {
        uint256 unlockTime = ((block.timestamp + _lockDuration) / WEEK) * WEEK; // Locktime is rounded down to weeks
```

Because all locks are created with end dates rounded to a weekly interval, all `slopeChanges` will be batched on weekly intervals. We will see the importance of this later on.

At any rate we fetch the old slope changes (oldDslope) and check if the lock end dates have changed, if not, we simply set the newDslope to the old, else we load the newDslope variable from the new end date, as seen below:

```solidity
        oldDslope = slopeChanges[_oldLocked.end];
            if (_newLocked.end != 0) {
                if (_newLocked.end == _oldLocked.end) {
                    newDslope = oldDslope;
                } else {
                    newDslope = slopeChanges[_newLocked.end];
                }
            }
```

### Backfilling Total Supply

Having computed, but not yet written, our UserPoint, we move on to GlobalPoints. GlobalPoints reuse many of the same concepts as we see in UserPoints, where array-like behaviour is achieved via an epoch variable to store the index of the GlobalPoints, and the points themselves are stored in a mapping and orderd by timestamp to allow for efficient binary searching:

```solidity
mapping(uint256 => GlobalPoint) internal _pointHistory; // epoch -> unsigned global point

/// ...

uint256 public epoch;
```

We initialize our GlobalPoint into memory by checking if we have an epoch > 0, we then enter this for loop:

```solidity
        // Go over weeks to fill history and calculate what the current point is
        {
            uint256 t_i = (lastCheckpoint / WEEK) * WEEK;
            for (uint256 i = 0; i < 255; ++i) {
                // Hopefully it won't happen that this won't get used in 5 years!
                // If it does, users will be able to withdraw but vote weight will be broken
                t_i += WEEK; // Initial value of t_i is always larger than the ts of the last point
                int128 d_slope = 0;
                if (t_i > block.timestamp) {
                    t_i = block.timestamp;
                } else {
                    d_slope = slopeChanges[t_i];
                }
                lastPoint.bias -= lastPoint.slope * (t_i - lastCheckpoint).toInt128();
                lastPoint.slope += d_slope;
                if (lastPoint.bias < 0) {
                    // This can happen
                    lastPoint.bias = 0;
                }
                if (lastPoint.slope < 0) {
                    // This cannot happen - just in case
                    lastPoint.slope = 0;
                }
                lastCheckpoint = t_i;
                lastPoint.ts = t_i;
                lastPoint.blk = initialLastPoint.blk + (blockSlope * (t_i - initialLastPoint.ts)) / MULTIPLIER;
                _epoch += 1;
                if (t_i == block.timestamp) {
                    lastPoint.blk = block.number;
                    break;
                } else {
                    _pointHistory[_epoch] = lastPoint;
                }
            }
        }
```

This loop is _critical_ to understanding how we track total supply in a dynamic system on chain.

First, we set an initial value for a timestamp, t_i, that will be used to iterate over periods between our last checkpoint and now:

```solidity
uint256 t_i = (lastCheckpoint / WEEK) * WEEK;
```

In the event that the last checkpoint was written mid week, integer math is used to round t_i to the start of a given week.

We then enter a for loop of up to 255 iterations. t_i is incremegted by a week each loop, including at the start - this crucially prevents us writing out of order GlobalPoints in the event that the most recent point was ahead of the floored initial t_i value.

```solidity
  for (uint256 i = 0; i < 255; ++i) {
    // Hopefully it won't happen that this won't get used in 5 years!
    // If it does, users will be able to withdraw but vote weight will be broken
    t_i += WEEK; // Initial value of t_i is always larger than the ts of the last point
```

Next we bound t_i to the present moment - ensuring even if we add a week we dont 'overshoot' and write to the future.

We fetch the current set of slope changes that are at the value of t_i, remembering that, as these are batched by week, d_slope will only have a value if we are exactly on a week boundary.

There are therefore 2 crucial states we can be in when fetching t_i:

1. t_i can be exactly on a week boundary
2. t_i can be in between a week boundary

As a new GlobalPoint is written with every new deposit, then with a frequently used contract such as Aerodrome's VotingEscrow, most of the time, t_i will be a single write at block.timestamp. Indeed, every time `_checkpoint` is called, at least one write (the final write) will be where t_i = block.timestamp.

The power of this function however, is that if between the last point being written and now, we crossed a weekly interval, t_i will be able to fetch a value of d_slope from slopeChanges.

```solidity
    int128 d_slope = 0;
    if (t_i > block.timestamp) {
        t_i = block.timestamp;
    } else {
        d_slope = slopeChanges[t_i];
    }
```

Next we compute the values of the lastPoint / the GlobalPoint. This is similar to the user point but note how we INCREASE the slope for the NEXT iteration using the slope changes. Again, what we're saying here is that at crucial t_i values, a number of locks will have expired, and cease to have voting power that can be reduced. Hence, make the slope shallower.

```solidity
    lastPoint.bias -= lastPoint.slope * (t_i - lastCheckpoint).toInt128();
    lastPoint.slope += d_slope;
    if (lastPoint.bias < 0) {
        // This can happen
        lastPoint.bias = 0;
    }
    if (lastPoint.slope < 0) {
        // This cannot happen - just in case
        lastPoint.slope = 0;
    }
    lastCheckpoint = t_i;
    lastPoint.ts = t_i;

```

Finally, we conclude our loop, we incremenent our epoch/the global point index in memory and write a new point if we are not at the present yet, else we need to write some more data before we can finish.

```solidity
    _epoch += 1;
    if (t_i == block.timestamp) {
        break;
    } else {
        _pointHistory[_epoch] = lastPoint;
    }
```

### Updating Slope Changes

If you've followed this far, most of the rest of the function should be fairly self explanatory. I do want to highlight one section though and that is the adjustment of slope changes.

```solidity
    // Schedule the slope changes (slope is going down)
    // We subtract new_user_slope from [_newLocked.end]
    // and add old_user_slope to [_oldLocked.end]
    if (_oldLocked.end > block.timestamp) {
        // oldDslope was <something> - uOld.slope, so we cancel that
        oldDslope += uOld.slope;
        if (_newLocked.end == _oldLocked.end) {
            oldDslope -= uNew.slope; // It was a new deposit, not extension
        }
        slopeChanges[_oldLocked.end] = oldDslope;
    }

    if (_newLocked.end > block.timestamp) {
        // update slope if new lock is greater than old lock and is not permanent or if old lock is permanent
        if ((_newLocked.end > _oldLocked.end)) {
            newDslope -= uNew.slope; // old slope disappeared at this point
            slopeChanges[_newLocked.end] = newDslope;
        }
        // else: we recorded it already in oldDslope
    }
```

In the event that a user changes her lock, we need to make adjustments to the slopeChanges to account for this.

We primarily care about 3 things regarding slope changes:

1. Has the old lock already ended?
2. Has the new lock already ended?
3. Has the lock date changed between the old and new lock?

In the first case, if the old lock has already ended, there is nothing to adjust on slopeChanges, for the OLD lock (the slope change has already occured).

Else, we run the following code:

```solidity
    if (_oldLocked.end > block.timestamp) {
        // oldDslope was <something> - uOld.slope, so we cancel that
        oldDslope += uOld.slope;
        if (_newLocked.end == _oldLocked.end) {
            oldDslope -= uNew.slope; // It was a new deposit, not extension
        }
        slopeChanges[_oldLocked.end] = oldDslope;
    }
```

As you can see from the comment, the oldDslope needs the old lock slope removed from it, and replaced by the new lock slope. In the event that the lock dates have not changed (the user is increasing her deposit but not changing the exit), this means we can overwrite the oldDslope value at oldLocked.end - else we need to write the changes to a new location in slopeChanges.

In this case we run the following code:

```solidity
    if (_newLocked.end > block.timestamp) {
        // update slope if new lock is greater than old lock and is not permanent or if old lock is permanent
        if ((_newLocked.end > _oldLocked.end)) {
            newDslope -= uNew.slope; // old slope disappeared at this point
            slopeChanges[_newLocked.end] = newDslope;
        }
        // else: we recorded it already in oldDslope
    }
```

Similar to the above if the lock has expired, no need to write to slope changes. If the new lock has the same end date, we already wrote changes to the oldDslope above. Else we update the newDslope value at the new end date.

## Conclusion

While that is not a completely exhaustive walkthrough of the whole Aerodrome codebase, I hope that covers one of the more conceptually tricky parts, and gives some understanding as to the intuition and mechanisms at play.

If you're interested in discussing more about veTokenomics, feel free to message me. Alternatively, Aragon specialises in deploying customizable, turnkey solutions for projects looking to leverage advanced tokenomic models like the above.
