# Design Tech Specs for Bracket Game

## Overview

Bracket Game is an app for trading votes for your favorite sport teams within various sport-leagues, across the current
season and also future seasons. Each league (NFL, NBA, NCAAB, etc) has a unique contract in which every team from that
league is represented by one“collective” of trusted fans who control a unique multi-sig Safe wallet. Each collective
receives payouts from the Game based on the exit round of its team in the real-life tournament.

In Bracket Game, fans can buy "votes" for any collective – a non-transferable balance of units representing the support
of the team by its fans. The price of votes for each team is determined by its own distinct bonding curve.
Buying/selling votes increases/decreases the outstanding supply of that collective's votes. Redeeming burns the votes,
leaving the total outstanding supply intact.

Each buy/sell transaction is subject to fees, split in three categories:

1. Pool fees – a share of tx value, which is added to league-wide pool of winnings.
2. Collective fees – a share of tx value, which is sent directly to the collective wallet for whichever specific
   collective was traded.
3. Protocol fees – a share of tx value, which is sent to a separate wallet controlled by Nilli, the maker of
   Bracket.Game.

Each league’s real-life tournament happens regularly – a season/year/etc. – and has a set of rounds of games. (A Season
is how we call the time-frame between the end of the previous tournament and the end of the current tournament.)

At the end of each tournament we have a prize pool of winnings accumulated from the "pool fees" of all transactions
happened during the season.

The prize pool is distributed to collectives after a real-life tournament is completed, based on:

- the provided winnings breakdown for each round: Rd 64 - 0.25% | Rd 32 - 0.50% | Rd 16 - 1.50% | Rd 8 - 3.00% | Rd 4 -
  8.00% | Rd 2 - 14.00% | Champ - 30.00%
- the exit round of each team. This information comes from an Oracle.
  - (Note: the oracle model will come from a second contract that we are building now and will audit with later in the
    game)

Once Oracle provides the exit round of each team, and verifies that everything is correct, we trigger the winnings
distribution:

- each collective's multi-sig Safe wallet receives a share of the prize pool, based on the provided winnings breakdown
  and the exit round of the team they represent.
- once the prize pool is completely distributed, the season is considered distributed. The next season starts.
- fans can redeem their votes for a share of the collective's multi-sig balance, based on the fan's voting power: how
  many votes they have relative to the total supply of votes of a team.

## Terminology

### League

A sports league, such as NFL, NBA, etc. Each league is represented by its own instance of our bonding curve-based smart
contract, `BG_Beta.sol`. The contract shares the common prize pool between all teams within the league.

### Season

A period of time between the winnings distribution events of the previous tournament and the current tournament. Prize
pool, rounds number, winnings breakdown, collective fees can each be Season-specific, in case there are any changes with
the real-life tournament.

### Vote

A vote represents fan's support for a specific team. They may be used with Snapshot for fans to express how they would
like their treasury to be used. Votes are non-transferable, one can only buy, sell, and redeem them.

### Fan

A user of the app, who can buy, sell, and redeem votes for a team. A fan's voting power is determined by the number of
votes they hold relative to the total supply of votes for the team. Voting power indicates the fan's share of the
collective's treasury.

### Collective

Collective is the representation of a fanbase of a specific real-life team. For each collective we create its own unique
multi-sig Safe wallet controlled by a few signers: us – Nilli — and a group of manually defined trusted fans, called
“Trustees”.

All interactions to the curve are done using the collective's wallet address. Inside the contract, each collective has:

- votes supply amount
- votes burnt amount (redeemed votes)
- a fees balance
- a mapping of fans' balances

A collective must be initialized inside the contract: the first vote of a collective can be bought only by the
collective itself.

### Bonding Curve

A mathematical equation controls the price of a vote based on the existing supply. Each collective has its own curve:
buying votes of one collective does not affect the vote price of another collective.

`Vote price = Outstanding vote supply / 100,000`

### Prize Pool

The prize pool of winnings for a league, which is used to reward collectives and fans. It is accumulated from the pool
fees of all transactions during the season. At the end of the season, the prize pool is distributed to collectives
multi-sig wallets, from which fans can redeem their votes for a share of the winnings.

The distribution happens based on the team's exit round – provided by an Oracle – and the winnings breakdown for each
round – provided by us sometime before the end of each given season.

## The common flow

1. Nilli deploys a contract for a new league. We initialize the contract with the stablecoin accepted, fees structure,
   current season's parameters. We grant roles.
2. Each collective is initialized by buying its first vote from the collective's multi-sig wallet.
3. Fans can buy, sell votes of initialized collectives. The price of votes is determined by the bonding curve.
4. Each transaction results in a set of fees, where the pool fee is added to the prize pool of winnings and collective
   fee to the collective’s multisig.
5. With each team elimination, Oracle provides the exit round of each team.
6. Once the tournament is over, Oracle verifies the exit rounds.
7. Once verified, we trigger the winnings distribution for the current season. The 100% of the prize pool is distributed
   to collectives based on the provided winnings breakdown and the exit round of the team they represent.
8. The season is considered distributed. The next season starts.
9. All throughout the season, fans have the right to redeem their votes for a share of the collective's multi-sig
   balance, based on the fan's voting power.

## Roles

Three roles are needed for the secure operation of the contract:

- `CLAIMER_ROLE` – can transfer votes from one user to another. This role is used to safely transfer votes from an
  account
  Nilli controls to user wallets, for example when Trustees are gifted shares of their service or new fans claim their "
  first vote" sign-up gift: `transferVotes`.
- `MANAGER_ROLE` – can trigger the winnings distribution for the current season. This role is used to manage seasons,
  collective names, and to safely distribute winnings to collectives: `setSeason`, `setCollectivesFanbases`,
  `distributeSeasonWinnings`.
- `ORACLE_ROLE` – can provide the exit round of each team. This role is used to safely verify the exit rounds, allowing
  only the Oracle to call the verification functions: `receiveVerifiedCollectiveExitRound` and
  `receiveVerifiedTotalWinnings`.

## Buy Votes

`function buyVotes(address collective, uint256 amount, uint256 maxValue)`

Two cases:

- initializing a collective by buying the first vote. Only the collective's multi-sig wallet can buy the first vote.
- regular buying of votes. Any fan can buy votes of a collective.

Buying votes increases the supply of votes of a collective `collectives[collective].supply`, which in turn affects the
price of a vote. The price is calculated based on the bonding curve formula, the public price getter is `getBuyPrice(
collective, amount)`. Burnt via redeeming votes are not affecting the curve price.

After buying:

- the balance of the fan is increased.
- the supply of votes of the collective is increased.
- the fees are taken from the tx value and distributed to the pool, collective, and protocol wallets.
- the `Trade` event is emitted, which contains all the required information about the transaction, new states, fees:
  required for our app.

The slippage mechanism is implemented by having the `maxValue` parameter: the maximum amount of funds the user is
willing
to spend on the votes, including the fees. If by the time the transaction is mined, the price of votes has changed too
much, the tx will revert. The slippage is turned off when the provided parameter value is 0.

## Sell Votes

`function sellVotes(address collective, uint256 amount, uint256 minValue)`

Two cases:

- collective selling its votes. Collective CANNOT sell its last vote.
- regular selling of votes. Any fan can sell votes of a collective.

Selling votes decreases the supply of votes of a collective `collectives[collective].supply`, affecting the curve. The
public price getter is `getSellPrice(collective, amount)`.

Burnt via redeeming votes cannot be sold, only the `activeSupply = supply - burnt` can be traded.

After selling:

- the balance of the fan is decreased.
- the supply of votes of the collective is decreased.
- the fees are taken from the tx value and distributed to the pool, collective, and protocol wallets.
- the `Trade` event is emitted, which contains all the required information about the transaction, new states, fees:
  required for our app.

The slippage mechanism is implemented by having the `minValue` parameter: the minimum amount of funds the user is
willing
to receive from the votes, excluding the fees. If by the time the transaction is mined, the price of votes has changed
too much, the tx will revert. The slippage is turned off when the provided parameter value is 0.

## Redeem Votes

`function redeemVotes(address collective, uint256 amount)`

A fan can choose to redeem votes instead selling, they are always free to do both. Why a fan would redeem:

- they don’t want to be a part of the collective anymore.
- they wish to affect a cost to the collective’s treasury by their leaving.
- the redeem value would be higher than the sell value minus fees.

Redeeming votes burns the votes, increasing the `collectives[collective].burnt` amount. The total supply of votes is not
affected.

The public price getter is `getRedeemValue(collective, amount)`. The redeem value is calculated based on the votes share
of the active supply and the collective’s multi-sig balance:

```
activeSupply = collectiveSupply - collectiveBurntSupply
voteValue = collectiveTreasuryBalance / activeSupply
redeemValue = votesToRedeem * amount
```

Burnt votes affect:

- the amount of trade-able collective votes: burnt votes cannot be sold. A corollary is that burnt votes raise the
  absolute minimum price of a collective.
- the redeem price: burnt votes are ignored when calculating the redeem value.
- the fan’s voting power: the fan’s voting power is calculated based on the active supply. The more votes burnt, the
  higher the voting power of the fan with the same amount of votes.

## Transfer Votes

`function transferVotes(address collective, uint256 amount, address receiver) onlyRole(CLAIMER_ROLE)`

We as creators of Bracket.Game want to have an ability to rewards users with votes in various cases, like:

- sign-up gift
- referral program
- trustee status

The purpose of the `transferVotes` function is to safely transfer votes from an account controlled by us – Nilli – to an
arbitrary user. The account will buy some amounts of shares using the regular buying process. Later, the account’s
private key will be exposed to our back-end app, so the app can trigger the transfer.

## Winnings Distribution

`function distributeSeasonWinnings(address[] calldata collectives, uint256[] calldata winnings) onlyRole(MANAGER_ROLE)`

The total pool is accumulated during the season. By the end of the season, the total pool must be fully distributed
to collectives and reset back to 0 for the next season.
After the distribution, the season's prize pool value will be stored for historical
purposes: `seasons[seasonIdx].prizePool`.

Before the distribution the season must be verified by Oracle and not-yet-distributed: `season.isVerified == true`
and `season.isDistributed == false`.

The distribution is executed in batches, sending the funds to each collective's multi-sig wallet and marking them as
distributed in the current season. Once the prize pool is completely
distributed: `season.distributedPool >= season.prizePool`, the season is considered distributed by
setting `season.isDistributed` and its end block `season.endBlock`.

Distribution will send the share of the prize pool to each collective based on the provided winnings breakdown and
team's exit round.
