# Computing Total Supply for Higher Order Polynomials

> This section is more advanced & speculative - so consider it a WIP but I thought it was interesting.

Polynomial decay functions with orders higher than 1 (aka: quadratic and beyond) are challenging in the EVM to compute in closed form. This is because we run into the same problem as with the linear curve: evaulating at time t in the naive way would require recomputing voting power for all users.

That said, you can in theory use an extremely similar approach as Curve/Aerodrome use in the linear case to compute higher order functions (although this is experimental and I would advise being careful with the implementation).

We noted above that polynomial decay functions with orders higher than 1 (aka: quadratic and beyond) are challenging in the EVM to compute in closed form. This is because we run into the same problem as with the linear curve: evaulating at time t in the naive way would require recomputing voting power for all users.

Fortunately, you can use an extremely similar approach as Curve/Aerodrome use in the linear case to compute higher order functions (although this is experimental and I would advise being careful with the implementation).

> I'm not 100% sure if this generalises to all slopes, but we can show a worked example for a simple curve in $`x^2`$ - I'd argue that there may be value in a cubic curve, but beyond that you most likely introduce needless complexity with minimal benefit, at least in the voting escrow case.

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

## Increasing Voting power with Nonlinear curves

Increasing voting power has a nice property that _extending_ a lock and _increasing amount_ make little sense. This means it's most realistic to offer new locks only, with a merge function at `duration`. If we wish to have a nonlinear curve without such functionality, then we can guarantee that the veNFT's currently locked amount and start date won't change, ergo `votingPower(t)` can just use the global `LockedBalance`.

This means we can simply evaluate the curve at `t` and get the user's voting power. The problem comes with total supply.

### Do you need total supply?

See the [section on quadratic curves](#Advanced: Computing Total Supply for Higher Order Polynomials) for this point.

### Abandoning Total Supply

If we don't need total supply, and we are increasing in voting power _we don't need to checkpoint_. We simply evaluate the user's voting power at t according to the function.

The main changes are then in auxilliary functions in our escrow code.

#### Migrations

What if we wanted to migrate to a more complex system in the future - perhaps adding total supply for a nonlinear increasing curve?

In this case, we would need to keep a record of scheduled changes to the coefficients of global supply. However it would also mean that, if we don't store this data, state cannot be read.

I suspect the main challenge here is designing a migration plan such that no votes are pending post-migration that use totalSupply. Access of state before the migration time should revert as this is unbounded.
