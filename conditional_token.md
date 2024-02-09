# Conditional Token

```rust

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct ConditionalToken {
    pub token_id: String,
    pub market_id: String,
    pub outcome: String,
    pub bet_amount: Uint128,
    pub odds: f64,
    pub bet_timestamp: u64,
    pub user_address: String,
    pub potential_payout: Uint128,
    pub is_redeemed: bool,
}


// Example function to issue a conditional token upon betting
fn place_bet(deps: DepsMut, info: MessageInfo, market_id: String, outcome: String, bet_amount: Uint128, odds: f64) -> StdResult<Response> {
    // Generate a unique token_id, for simplicity using a UUID or similar method is assumed
    let token_id = "unique_token_id".to_string();

    let market = MARKETS.load(deps.storage, market_id.clone())?;
    if market.market_status != "Open" {
        return Err(StdError::generic_err("Market is not open for betting"));
    }

    // Here, calculate the potential payout based on bet_amount and odds
    let potential_payout = bet_amount.multiply_ratio(Uint128::from(odds as u128), Uint128::one());

    let conditional_token = ConditionalToken {
        token_id: token_id.clone(),
        market_id,
        outcome,
        bet_amount,
        odds,
        bet_timestamp: env.block.time.seconds(),
        user_address: info.sender.to_string(),
        potential_payout,
        is_redeemed: false,
    };

    TOKENS.save(deps.storage, token_id, &conditional_token)?;
    Ok(Response::new().add_attribute("method", "place_bet"))
}
```

## 1. Market Settlement
Before any redemption can occur, the market must be settled. This involves determining the actual outcome of the event and marking the corresponding conditional tokens as eligible for redemption.

Determine the Outcome: Once the event concludes, the true outcome is determined through reliable data sources, possibly using oracles for decentralized verification.
Update Market Status: The market status is updated to "Settled," and the winning outcome is recorded.

## 2. Token Validation
When a token holder wishes to redeem their token, the system must validate that the token represents a bet on the winning outcome and that the market is settled.

Check Token Outcome: Verify that the conditional token's outcome matches the winning outcome of its associated market.
Check Market Status: Ensure the market is indeed settled. Tokens from unsettled markets cannot be redeemed.

## 3. Calculate Redemption Value
The value of each conditional token upon redemption depends on the initial bet amount, the odds, and the total payout pool available for the winning outcome. 

### Formula for Redemption Value Based on Odds

The payout for a winning bet primarily depends on the odds at which the bet was matched, regardless of the overall pool.

Redemption Value = Bet Amount × Odds

This formula calculates the total return to the user if their bet wins. To find the net profit (or the actual "winnings"), you would subtract the original stake from the redemption value:

Net Winnings =(Bet Amount×Odds)−Bet Amount

Or simplified:

Net Winnings=Bet Amount×(Odds−1)

### Including Platform Commission

If the platform takes a commission on winnings, you apply the commission rate to the net winnings, not the total redemption value. The formula to calculate the take-home payout after commission would be:

Take-Home Payout =Bet Amount×Odds−(Net Winnings×Commission Rate)

### Example Scenario

Assume:

-   A user places a bet of $100 on "Team A wins".
-   The bet gets matched at odds of 5.0.
-   The platform's commission rate on winnings is 5%.

Calculating the total redemption value:

Redemption Value =100×5 =$500

Calculating net winnings:

Net Winnings =100×(5−1) =$400

Applying the commission to net winnings (5% of $400):

Commission =400×0.05 =$20

Calculating the take-home payout:

Take-Home Payout =500−20 =$480

So, the user's take-home payout, after winning the bet and subtracting the platform's commission, would be $480.

## 4. Redeem and Burn Tokens
Once the redemption value is calculated, the platform processes the redemption by transferring the corresponding value to the user and then burning the token to prevent double spending.

Transfer Value: The calculated payout is transferred to the user's wallet. This might involve converting the payout pool's assets into the desired cryptocurrency or token format.
Burn Token: To finalize the redemption, the conditional token is burned, removing it from circulation and ensuring that it cannot be redeemed again.
Example Redemption Logic Implementation
Here's a simplified pseudo-code example of how redemption logic might be implemented in a smart contract:

```rust

fn redeem_conditional_token(deps: DepsMut, env: Env, info: MessageInfo, token_id: String) -> StdResult<Response> {
    let token = TOKENS.load(deps.storage, &token_id)?;
    let market = MARKETS.load(deps.storage, &token.market_id)?;

    // Ensure the market is settled and the token matches the winning outcome
    if market.market_status != "Settled" || market.winning_outcome != token.outcome {
        return Err(StdError::generic_err("Token cannot be redeemed"));
    }

    // Calculate the user's payout based on the token's potential payout
    let payout = token.potential_payout;

    // Transfer payout to user
    // (Implementation details for transferring funds would go here)

    // Burn the token
    TOKENS.remove(deps.storage, &token_id);

    Ok(Response::new().add_attribute("method", "redeem_conditional_token").add_attribute("payout", payout.to_string()))
}
```


## Voluntary Token Burning Process
## Token Ownership Verification:

The user initiates a request to burn their conditional token.
The smart contract first verifies that the token exists and that the requester is indeed the owner of the token.
Check Market Settlement:

The contract checks the associated market's status to ensure it is settled. This ensures that the user isn't trying to burn a token that might still have value (in case the market hasn't been settled yet or there was a mistake).
Validate Token Outcome:

The contract also validates that the token represents a losing bet. While this isn't strictly necessary for a voluntary burn (since users might want to burn tokens for any reason), it ensures clarity in the process and helps prevent accidental burns of potentially valuable tokens.

## Burning the Token:

Upon validation, the smart contract proceeds to burn the token, effectively removing it from circulation. This is achieved by decrementing the token's total supply in the contract and removing the token's association with the user's wallet.
Optionally, the contract can emit an event logging the burn action for transparency and record-keeping.
Smart Contract Example
Here's a simplified pseudo-code example to illustrate voluntary token burning for a losing bet:

```rust

fn burn_conditional_token(deps: DepsMut, env: Env, info: MessageInfo, token_id: String) -> StdResult<Response> {
    let token = TOKENS.load(deps.storage, &token_id)?;

    // Ensure the requester owns the token and the market is settled
    if info.sender != token.user_address {
        return Err(StdError::generic_err("Only the token owner can burn it"));
    }

    let market = MARKETS.load(deps.storage, &token.market_id)?;
    if market.market_status != "Settled" {
        return Err(StdError::generic_err("Market not settled yet"));
    }

    // Optional: Check if the token represents a losing outcome
    if market.winning_outcome == token.outcome {
        return Err(StdError::generic_err("This token represents a winning bet"));
    }

    // Proceed to burn the token
    TOKENS.remove(deps.storage, &token_id);

    Ok(Response::new()
        .add_attribute("action", "burn_conditional_token")
        .add_attribute("token_id", token_id)
        .add_attribute("status", "burned"))
}