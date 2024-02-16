### 1\. Unified Market Contract Logic

Market Creation:

-   Input: Market type (sports event or prediction question), array of possible outcomes, event timestamp.
-   Process: Generate a unique `market_id`, create a `Market` struct, and store it.
-   Conditional Token Issuance: For each outcome, mint conditional tokens associated with the market.
-   Example: If a market for a football match is created with outcomes \["win", "lose", "draw"\], conditional tokens for each outcome are minted.

Bet Placement:

-   Input: User, market\_id, bet type (back or lay), selected outcome, odds, stake.
-   Process: Check if the market exists and is open. Create a `Bet` struct, storing the bet details including a unique `bet_id`. Attempt immediate matching for market orders or add to the order book for limit orders.
-   Math: For a lay bet, liability = stake \* (odds - 1).

### 2\. Order Matching Contract Logic

Order Matching:

-   Input: Newly placed order or existing orders in the order book.
-   Process: For each new order, search the order book for compatible opposite orders (lay for back and vice versa) based on odds and timestamps. Prioritize orders with better odds and earlier placement.
-   Partial and Full Matches: If a full match is found, transfer stakes and update orders as matched. For partial matches, split the order, updating the unmatched portion accordingly.
-   Example: A back bet with odds of 2.0 and stake of $100 matches with a lay bet offering odds of 1.9 and liability of $90.

### 3\. Settlement Contract Logic

Market Settlement:

-   Input: `market_id`, actual outcome.
-   Process: Verify the market and outcome. Update the market's status to settled with the result. Iterate over bets within the market to determine winners and calculate payouts.
-   Payout Calculation: For winning back bets, payout = stake \* odds. For winning lay bets, payout = stake.
-   Deduct Commission: Calculate commission on net winnings using a predefined rate, e.g., 5%.

### 4\. Token Management and Trading Contract Logic

Token Minting and Burning:

-   Minting: For each new market outcome, mint a specific number of conditional tokens.
-   Burning: Upon settlement, winners redeem conditional tokens for payouts, where tokens are burned.
-   Example: If "Team A win" tokens are redeemed after Team A's victory, those tokens are burned in exchange for the payout.

Trade Initiation and Execution:

-   Trade Initiation: Users can list conditional tokens for sale or express interest in buying specific tokens at a set price.
-   Trade Execution: When a buy and sell order for the same token match in price, execute the trade by transferring tokens and funds between the parties.
-   Price Discovery: Utilize an automated market maker (AMM) model or order book to facilitate price discovery based on supply and demand.

### Mathematical Formulas

-   Liability Calculation for Lay Bets: The liability of a lay bet (the amount the layer risks losing) is indeed calculated as the stake multiplied by (odds - 1), since the layer needs to pay out this amount if the outcome they bet against occurs. Liability=Stake×(Odds−1)
-   Payout for Back Bets: The payout for a winning back bet includes the stake returned plus winnings calculated as stake multiplied by (odds - 1). However, the formula provided calculates the total return (stake + winnings). Payout=Stake×Odds
-   Payout for Lay Bets: For a winning lay bet, the layer keeps the backer's stake. This formula assumes the layer's liability has already been accounted for in their balance at the time the bet was matched. Payout=Stake + liability
-   Commission on Winnings: Typically, the commission is only charged on the net profit from a market, not the total return. If "Net Winnings" refers to profit (total payout minus the initial stake), then the formula is applied correctly. Otherwise, for clarity, especially in a back bet scenario, the formula should explicitly deduct the stake to calculate commission only on the winnings: Commission=(Payout−Stake)×CommissionRate OR Commission=NetWinnings×CommissionRate
  
### Commission Handling Example

-   Assume a user places a back bet with a $100 stake at odds of 3.0. The total payout on a win would be $300 (including the stake).
-   The net winnings (profit) would be $200 ($300 payout - $100 stake).
-   If the commission rate is 5%, the commission would be $10 ($200 \* 0.05).
-   The user's final take-home would be $290 ($300 payout - $10 commission).

### Example Scenario

1.  Market Creation: A market for a football match is created with outcomes \["win", "lose", "draw"\], each associated with conditional tokens.
2.  Bet Placement: A user places a back bet on "win" with $100 at odds of 2.0. Another user places a lay bet against "win" with $100 at odds of 2.0, accepting a liability of $100.
3.  Order Matching: The system automatically matches these orders based on their compatibility.
4.  Event Conclusion and Settlement: The match ends in a "win". The market is settled, and the back bet user wins $200 (including their initial stake), after deducting a 5% commission ($10), netting $190.
5.  Token Redemption: The winning user redeems their conditional tokens for the payout, and the tokens are burned.
   

### Example Scenario 2
Lets take example of a horse match. On each market there are runners, e.g. horses in a horse race. On runners the Back/Lay bets are placed. Back bet means that the runner will win, Lay bet that it will lose. Fig. 1 illustrates the classical view of a runner in the market. In the middle column at green is the price scale. The price can also be referred as odd. Bets are placed on a given price, which represents the chances of the runner win.
E.g. the price 2.0 is a 50% of chances (1/2 = 0.5), price 1.01 is a 99% of chances (1/1.01 = 0.99), 1000 is a 0.1% of chances (1/1000 = 0.001). 
On the left side, next to the prices column, in pink is the amount column formed by the Lay bets. On the right side, the column in blue, next to the prices column, is the amount column formed by the Back bets. 
The first and fifth column represent our own unmatched bets. In this case (Fig. 1) we have the following bets placed (but not matched yet):  - Lay of 10.00€ at 4.30 (Lay 10@4.3); - Back of 10.00€ at 4.70 (Back 10@4.7); and - Back of 5.00€ at 4.60 (Back 5@4.6).


![Alt text](<Screenshot 2024-02-14 at 11.15.45 AM.png>)

For example, in the market state of Fig. 1 if a Back bet 15@4.4 is placed (on the blue side) it will match 8.00@4.5 and 2.00@4.4 , and will leave the remaining 5.00@4.4 unmatched (waiting for someone to buy with a Lay bet). The traded volume information will be updated. This is how the prices move in the market. Since this bet was matched at two different prices, the global matched price of this bet can be calculated using equation 1.

![Alt text](<Screenshot 2024-02-14 at 11.22.17 AM.png>)

If a Back is placed above the best offer in the market (4.5 in Fig. 1), for example 15@4.9, it will stay in the market unmatched and, so for, waiting to be matched. The same happens to a Lay bet if it is placed at a lower price than the best offer (counter bet waiting to be matched). Only unmatched or partial unmatched bets can be canceled. 
