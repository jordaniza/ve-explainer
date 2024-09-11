# Generalised approaches to non-linear curves

In this section we look at the case where we want a curve where voting power is _increasing_ in t. We first look at the changes needed then ask the question: can this be generalised so we have a single formula for increasing and decreasing curves?

## Part 1: Increasing voting power

Increasing voting power refers to the scenario where a user's voting power gets larger, the longer the user is staked.

While in a decreasing system users are 'punished' with reduced voting power if they don't keep a maximised lock, in an increasing system, users are incentivised not to unlock, as the opportunity cost to do so is the time it would take to restore original voting power.

How might we model this?

Assuming linearity, there's a lot of overlap with the decreasing case.

- We still can use a bias, slope combination for point histories.
- We still can aggregate all user's slopes into a single global slope.

What changes do we have, then:

- The end date of the lock will typically change from a fixed end date, to some sort of exit queue mechanism.
- We need to keep track of a lock's initial start date
- Extending a lock has no meaning.
- Increasing the amount of a lock is counterintuitive.
- We may have a maximum multiplier of the initial deposit that is reached after t seconds from the start date.

We will use an example of a linear increase in voting power, again over 4 years, from 0% -> 100%. We will then explore going beyond 100% to say 200%.

## Code Changes

### Slope and Bias for the User

Looking at the checkpointing logic:

```solidity
// old logic
uNew.slope = _newLocked.amount / ProtocolTimeLibraryV2.iMAXTIME;
uNew.bias = uNew.slope * (_newLocked.end - block.timestamp).toInt128();
```

In the case of 0 -> 100%, the slope is actually unchanged (the bias calculation is what will change, to reflect an increase it voting power).
(MAXTIME represents the time over which the user goes from 0 -> 100% voting power, versus the other way round).

We will need to bound the bias to be <= amount, this can be done in a later code block.

In the case of the bias, it is calculated as (slope \* timeRemaining), which will be a positive number that decreases as `timeRemaining` -> 0.

One can invert this curve by storing a startDate in the lock, rewriting the bias as:

```solidity
uNew.bias = uNew.slope * (block.timestamp - _newLocked.start)
```

Which is now increasing. Note as well that it doesn't make sense for the start date of a lock to change unless we are merging locks, in which case we may wish to take the newest lock.

## Bounding to 100%

The above code will continue linearly increasing, ad infinitum. We note that, in the existing code, `newLocked` can never be passed as having already expired, so uNew.bias can never be < 0. To make sure that old locks never have negative biases, we need to check the `balanceOfNFTAt` function:

```solidity
function balanceOfNFTAt(uint256 _tokenId, uint256 _t) external view returns (uint256) {
    uint256 _epoch = getPastUserPointIndex(_tokenId, _t);
    // epoch 0 is an empty point
    if (_epoch == 0) return 0;
    UserPoint memory lastPoint = _userPointHistory[_tokenId][_epoch];
    if (lastPoint.permanent != 0) {
        return lastPoint.permanent;
    } else {
        lastPoint.bias -= lastPoint.slope * (_t - lastPoint.ts).toInt128();
        // here is where we bound the bias for the user
        if (lastPoint.bias < 0) {
            lastPoint.bias = 0;
        }
        return lastPoint.bias.toUint256();
    }
}
```

Note the final `else` block in the function: this is where `bias` is bounded to a minimum of zero.

In our increasing case, we need to ensure 2 things:

1. newLocked.createdAt should not result in a bias > amount (a system invariant)

   - It would be prudent to add this check when computing the new bias, just in case.

2. our `balanceOfNFTAt` function needs changing:

```solidity
// additive function is all we need here
// t is the timestamp to check which is always gte lastPoint.ts
// and it's the same here, check seconds between last ts and t
lastPoint.bias += lastPoint.slope * (_t - lastPoint.ts).toInt128();
// here is where we bound the bias for the user
if (lastPoint.bias  > MAX) {
    lastPoint.bias = MAX;
}
```

`MAX` here would need to be some some multiple of the user's amount, in this case 100%. We could do this in 2 ways:

1. Fetch the locked balance to ensure lastPoint.bias <= locked.amount
2. Do away with slopes and biases for increasing curves.

> Bounding to 200% or more would simply be a case of repeating the above but MAX = 2 \* amount, and any checks in the checkpoint function should also ensure this is adhered to.

## Steeper lines and starting from a minimum

We may want a situation where a user goes from a starting value that is > 0, or increases faster. Graphically, we can see each of these as adjustments to the curve, where the initial point is a change in the y-intercept, and increases beyond 100% over the same time period is a change in the slope.

A steeper line requires we change the way we think about slope. It's currently calculated as follows:

```solidity
// inside checkpoints
uNew.slope = _newLocked.amount / ProtocolTimeLibraryV2.iMAXTIME;
```

We could hack this by changing the denominator:

- Steeper slopes - divide the `iMAXTIME` by some value
- Shallower slopes - multiple the `iMAXTIME` by some value

For example, we want users to go from 0 -> 200% voting power over 4 years. We set `iMAXTIME` to 4 years, then write:

```solidity
// inside checkpoints
uNew.slope = _newLocked.amount / (ProtocolTimeLibraryV2.iMAXTIME / 2);
```

We then need to adjust our bounding function to reflect that the new maximum is `2 * amount`

You can certainly do this, but it's not especially intuitive what's going on. Let's think if we can generalise the formula a bit

## Part 2: A Formally definition of our linear curve

We need a formula for our curve that actually encompasses the correct variables. Consider a standard line as $`y = mx + c`$

In our case $`votingPower_{i,t} = slope_i \times time^* + startingPower_i`$

\*$`time`$ in this case could be time remaining or time since lock

In our case, we need a generalised formula for `slope`, this is a function of:

- amount locked
- `duration` over which the slope should be defined
- `multiplier` over the duration

### Generalized Formula Derivation

Let’s define the parameters:

1. **Initial Voting Power (`V_{initial}`)**: The starting voting power at $`t = 0`$. This could be 0% of the defined amount, 100%, or any other initial state.
2. **Final Voting Power (`V_{final}`)**: The voting power desired at the end of the duration. This could be 100% of the defined amount, 0%, or any other final state.
3. **Duration (`duration`)**: The time period over which the change occurs.
4. **Slope (`slope`)**: The rate of change of voting power per unit time.

The general form of the linear equation is:

$`votingPower(t) = slope \times t + c`$

Where:

- $`slope`$ is the rate of change.
- $`c`$ is the y-intercept, representing the initial voting power at $`t = 0`$.

### Applying Initial and Final Conditions

1. **Initial Condition**: At $`t = 0`$, voting power is $`V_{initial}`$.

   $`votingPower(0) = c = V_{initial}`$

   So, the equation becomes:

   $`votingPower(t) = slope \times t + V_{initial}`$

2. **Final Condition**: At $`t = duration`$, voting power should be $`V_{final}`$.

   $`votingPower(duration) = slope \times duration + V_{initial} = V_{final}`$

   Rearranging to solve for $`slope`$:

   $`slope \times duration = V_{final} - V_{initial}`$

   $`slope = \frac{V_{final} - V_{initial}}{duration}`$

### Generalized Voting Power Formula

Substituting $`slope`$ back into the linear equation, we get:

$`votingPower(t) = \left(\frac{V_{final} - V_{initial}}{duration}\right) \times t + V_{initial}`$

You can see some examples:

**100% -> 0% over 4 years**

- duration = 4 years
- V_initial = amount
- V_final = 0
- Initial value = amount
- bound at 0% or 4 years

- $`slope = \frac{0 - amount}{4 years}`$
- $`slope = -\frac{amount}{4 years}`$

**0% -> 100% over 2 years**

- duration = 2 years
- V_initial = 0
- V_final = amount
- initial value = 0
- bound at 2y OR 2x amount

- $`slope = \frac{amount - 0}{4 years}`$
- $`slope = \frac{amount}{2 years}`$

\*\*100% -> 600% over 6 weeks

- duration = 6 weeks
- V_initial = amount
- V_final = 6 \* amount
- initial value = amount
- bound at 6 weeks OR 6x amount

- $`slope = \frac{5}{6 weeks}`$

> Note that the above slope is 5 / (6 weeks), NOT 6, this is because we are increasing voting power by 5x

## Implementing in code:

We can define a generalised `getSlope` function

```solidity
// we may wish to use scaling factors or even fixed point libraries to increase precision but we don't need here
function getSlope(uint256 amount, uint256 multiplier, uint256 duration) public pure returns (int128 slope) {
    int128 v_inital = amount.toInt128();
    int128 v_final = (amount * multiplier).toInt128();
    return (v_final - v_initial) / duration.toInt128();
}

function getBiasUnbound(uint256 amount, uint256 timestamp, int128 slope) public pure returns (uint256) {
    // y = mt + c
    int128 bias = slope * timestamp.toInt128() + amount.toInt128();
    // never return negative values
    return bias < 0 ? 0 : uint256(bias);
}

// The above assumes a boundary - this is applicable for most use cases and it is trivial to set
// a sentinel value of max(uint256) if you want an unbounded increase
function getBias(uint256 amount, uint256 multiplier, uint256 duration, uint256 timestamp, uint256 boundary) public pure returns (uint256) {
    int128 slope = getSlope(amount, multiplier, duration);
    // unchanging voting power
    if (slope == 0) return amount * multiplier;
    uint256 bias = getBiasUnbound(amount, timestamp, slope);
    // bound increasing voting power to max @ boundary
    if (slope > 0 && bias > boundary) return boundary;
    // bound decreasing voting power to min @ boundary
    if (slope < 0 && bias < boundary) return boundary;
    return bias;
}
```

We can use these functions in our checkpoint logic: calulating slopes and biases on our `UserPoint` for `uNew` and `uOld` as needed.

- We can define constants or getters for our `multiplier`, `amount`, `duration` and `boundary` inside our curve
- The above functions can be public view functions for easy testing.
- When we call u{state}.slope, we call `getSlope`
- When we call u{state}.bias, we call `getBias`
  - We should regression test to see if there are valid examples where we need an unbound bias.
- Last point bias and slope calcs can also use these functions. We no longer require slope to be > 0 as we are allowing slope to be +/-
  - Globally, we use unbound bias conditions - a per-user bias does not apply to the global voting power.
- Permanent lock balance needs its own treatment. It does not make sense in an increasing slope.
- We can do the same adjustments to the user adjustments of last point and bias, but we do not need the boundary condition on slope as it can be negative.
- See below for dslope changes.

## Max values and dslope

The above covers the user side of things, but what about dslope?

Realistically speaking, we could do away with dslope in the increasing case with unbounded voting power increases. Technically voting power caps at `uint256.max` but we wont run into that, unless we have an extreme choice for multiplier or token decimals (in the linear case - for h/o polynomials, this is a bit more complicated).

Most implementations will likely bound a max-voting power, to stop the case where incumbent voters completely dominate the DAO with no hope for new participants to catch up. In this case, we need an equivalent of `dslope` to indicate where the global curve should get shallower, as a user's voting power levels out.

Fortunately for us, this change is the same in both the negative and positive slope cases. The change in slope (dslope) is always decreasing - we don't allow for scheduled increases in voting power in the linear case.

### Do you need slopes and biases?

We saw above that, in order to calculate a bounded maximum voting power multiple, one needs to fetch the locked amount. It's also true that in an increasing curve scenario, it doesn't make sense to:

1. Allow locks to increase amount: this would violate the principles of linearly increasing voting power.
2. Allow locks to change duration: as there is no consequence of an end date, there is no need to increase duration.

Therefore, it may make sense for us to simply create 1 NFT per lock and not allow modifications.

In this case, `balanceOfNFTAt` could be more easily written with simply the lock created date + the elapsed time.

Our UserPoints could then be safely deprecated (although we will still need `dslopes` as will be explained below). This mostly means we can save a few storage writes by writing to userPointHistory and userPointEpoch. We still need to write to global points.

Note that we may, wish to allow `Merging` -> combining multiple veNFTs with the most recent lock taking precedence. The use case here would be a user with multiple locks reaching the max duration, wishing to consolidate positions.

# Consequences for escrow logic

This section explores some of the wider changes that would need to be made to contracts to accomodate increasing voting power.

A few things jump to mind as 'not relevant' in the increasing case:

1. Permanent locks - if a curve is increasing in voting power, there is no benefit to auto-renewal.
2. Increasing amount - this cannot be done without resetting the lock
3. Increasing duration - this cannot be done as there is no duration
4. `endDate` in the LockedBalance needs to be replaced with `createdDate` in order to track properly
5. `dslope` changes happen not at unlock time, but instead after `duration`
6. Creating a lock should not allow a `maxDuration`
7. Withdrawals can be done immediately or subject to a withdrawal queue
