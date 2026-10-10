# STUDENT-QUESTIONS.md — Discussion questions (submit with your repo)

Answer directly under each question. 150–300 words each — **reasoning over length**.

---

## A. Permission design

**A1.** The vault holds `MINTER_ROLE`, so it can `burn` any user's balance. Explain why that is a risk, then write out how you would change `Vault` and `SimpleStablecoin` to remove it.

> Your answer:
The vault is given MINTER_ROLE so that a deposit can mint sUSD and a redemption can burn it. The risk sits in SimpleStablecoin.burn. That function only checks that the caller holds the role. It does not check that the burned balance belongs to the caller. 

Redemption only needs to burn the coins of the person who is asking for collateral back. The present role is wider than that job. Today's redeem happens to burn msg.sender, but the permission already exists. A later bug in the vault, or any path that makes the vault call burn with an address chosen by an attacker, can wipe an honest balance. The holder then has no sUSD left with which to redeem, while the collateral can still sit in the vault. The link between a person and that person's deposit is broken.

Solutions: burn should stop accepting an arbitrary account, and the vault should burn only sUSD it already holds. In Vault.redeem, the user first approves the vault to spend the user's sUSD. The vault pulls that amount with transferFrom, then burns its own received balance. If the user has not approved, redemption fails. The vault never names another user's address inside burn. Minting can remain limited to the vault, and it should happen only after the collateral transfer has succeeded. The vault can still issue coins and retire coins in its own custody, but it can no longer delete someone else's balance.

<br><br><br>

**A2.** In this contract `DEFAULT_ADMIN_ROLE`, `MINTER_ROLE` and `PAUSER_ROLE` all go to the same address. How would you split them in production, and who holds each?

> Your answer:
The constructor gives DEFAULT_ADMIN_ROLE, MINTER_ROLE, and PAUSER_ROLE to one address. The power to change roles, the power to issue the coin, and the power to freeze the system therefore share one key. If that key leaks, is bribed, or signs the wrong transaction, all three fail together.

So in production, we need split the three roles across different holders:

DEFAULT_ADMIN_ROLE would sit in a cold multisig, for example three of five, behind a timelock. It would only grant and revoke roles. It would not mint day to day, and it would not be the account that presses pause. A grant would not take effect immediately, so others could see a bad transaction.

MINTER_ROLE would not be held by any person's wallet. Only the vault contract would hold it. A person would have no direct way to call mint. New coins would appear only after collateral had entered the vault.

PAUSER_ROLE would go to a different emergency multisig, with different members, for example two of three from a security group. Those people could pause transfers and minting when an attack is visible, but they could not mint and they could not hand the mint role to someone else. Unpausing would also pass through a timelock, so redemption could not be left shut indefinitely.

<br><br><br>

---

## B. Pausing and redemption

**B1.** `_update` is the single entry point for every balance change, so `pause()` freezes transfers, minting and redemption together. If you wanted "pause transfers but **allow redemption**", how would you change it? Give the approach — full code not required.

> Your answer:
_update is the shared door for every balance change, and whenNotPaused sits on that door. Transfers, mints, and burns all stop together. Redemption calls burn, and burn also enters _update, so a pause closes the path back to the collateral.

If the aim is to pause transfers while still allowing redemption, the same check should not cover every balance change. The two addresses in _update already show which operation this is. When neither from nor to is the zero address, the call is an ordinary transfer and should revert while paused. When from is the zero address, the call is a mint and should also revert, so nobody can expand the supply during the emergency. When to is the zero address, the call is a burn. Redemption is that burn, so it should continue during the pause.

Secondary-market transfers then stop, and new minting stops, while a person who already holds sUSD can still call redeem and receive mUSDC from the vault. The pause limits circulation and further issuance. It does not cancel the exit promise of the stablecoin.

One implementation detail matters. If redemption first uses transferFrom and only then burns the vault's own balance, that transfer is an ordinary transfer and the pause would reject it. A redemption that must work during a pause therefore needs a burn path that does not rely on an ordinary transfer. The user might burn the coins directly, or redemption might have its own entrance that the pause does not block. The design choice is to treat the three kinds of balance change separately, rather than binding all of them to one switch.

<br><br><br>

**B2.** In 2008, when a money-market fund "broke the buck", redemptions were frozen for days. In 2023 USDC depegged to $0.87 after a reserve bank failed, but redemptions were **not** shut. Compare the two responses — what does closing the redemption channel, or leaving it open, do to a stablecoin?

> Your answer:
In 2008, a US money-market fund fell below one dollar per share, and the firm then suspended redemptions for days. Holders could see that the quoted value was wrong, and they also could not take cash out. They waited inside the freeze. Closing the exit stops a sale for a while, because there is nowhere to sell and nowhere to redeem. The uncertainty spreads to other funds. People fear that their own redemption will be closed next, so they run on products that are still open. For a stablecoin, closing redemption turns “one coin for one dollar” from a rule the contract can perform into a promise that is temporarily void. Even if the collateral still covers the supply, holders cannot experience that equation. The secondary market clears at a lower price, because buyers know they may be frozen too.

In 2023, a bank holding Circle's reserves failed, and USDC traded around $0.87, but redemption stayed open. The market price said that people doubted the reserves. The open window said that a path back to one dollar still existed. People who could redeem bought the cheap coins and exchanged them for dollars, and that trade pulled the market price back toward one dollar. An open redemption does not prove that the reserves are sufficient. It reveals a shortfall: if the money is not there, the first people to redeem take what remains, and later people bear the loss. When the reserves are actually present, open redemption is the mechanism that holds the peg. When the reserves are gone, open redemption distributes the loss quickly. Closing redemption hides a liquidity problem and can also hide insolvency. Leaving it open preserves the right to exit, and it leaves insolvency nowhere to hide.

<br><br><br>

---

## C. Depeg analysis

**C1.** Under what conditions does this coin depeg? Distinguish at least two classes of cause, and say how each one shows up in the invariant `totalCollateral() >= totalSupply()`.

> Your answer:
This coin can leave one dollar for two different reasons: solvency, and liquidity. The two show opposite results in totalCollateral() >= totalSupply().

A solvency failure means that circulating sUSD is not matched by enough mUSDC. Ex3 is the direct case. After the attacker receives MINTER_ROLE and mints one million sUSD, no new collateral enters the vault, supply is far above collateral, and the inequality fails. Another route is moving collateral out without reducing supply. Whenever someone can raise totalSupply without paying, or reduce the vault balance without burning sUSD, the invariant fails. The meaning is concrete: some coins have nothing behind them, and further redemptions give the remaining collateral to whoever arrives first.

A liquidity failure means that the collateral is still there, but holders cannot reach it. pause() is this case. Transfers, mints, and redemptions all enter _update and revert. The amount of mUSDC in the vault and the supply of sUSD do not change, so the invariant still holds. The coin can still trade below one dollar, because nobody can exchange it for collateral, and arbitrageurs cannot pull the price back by redeeming. A blacklist, a halted chain, or a redemption function that has been switched off belongs to the same class.

Seeing the invariant hold is therefore not enough to say that the coin is still worth one dollar. If the invariant fails, the backing is short. If the invariant holds while redemption is blocked, the books are balanced and the right to exit is gone. In both situations, people stop accepting the coin at one dollar.

<br><br><br>

**C2.** Suppose an attacker bribes their way to `MINTER_ROLE`, mints 1,000,000 sUSD out of nothing and redeems it all. Describe the flow of funds, and name the step that could have stopped them.

> Your answer:
The attacker bribes the admin who holds DEFAULT_ADMIN_ROLE. The admin calls grantRole and gives MINTER_ROLE to the attacker. From that moment, the attacker can mint without going through the vault.

The attacker calls stable.mint and creates one million sUSD, which is 1000000e6 raw units. Supply rises by one million, and totalCollateral() does not change. If honest users had already deposited one million mUSDC, the books move from “one million of collateral against one million of supply” to “one million of collateral against two million of supply.”

The attacker then calls vault.redeem for one million. The vault checks that it holds enough mUSDC, burns the attacker's one million sUSD, and transfers one million mUSDC to the attacker. Supply falls back to one million, and the collateral falls to zero. Honest users still have the sUSD they received when they deposited, but the vault is empty, so their later redemption fails for insufficient collateral. The funds move from other people's collateral, through the vault, into the attacker's wallet. What the attacker spends is only a number that was minted from nothing and burned again at redemption.

The step that could have stopped this is the grant, not the final redemption. Redemption pays whoever holds the coin out of whatever collateral is in the vault. It cannot tell whether a coin was minted by a deposit or by a minting right handed out later. The minting role should remain only on the vault contract, and deposit should mint only after transferFrom has received the collateral. A grant by the admin should also pass through a separate multisig and a timelock. If no personal wallet can receive MINTER_ROLE, bribing one admin cannot complete this mint by itself.

<br><br><br>

---

## D. Toward RWA

**D1.** Right now the collateral is `MockUSDC` and `totalCollateral()` just reads an on-chain balance — simple and reliable. If the collateral were **US Treasuries**, could this invariant still be written that way? What new problems appear?

> Your answer:
The collateral today is MockUSDC, an ERC-20 on this chain. totalCollateral() reads balanceOf for the vault, and that reading is the balance the vault actually holds. The transfer and the balance share one chain and one transaction, so “how much collateral is in the vault” and “how much sUSD is outside” can sit in one equation, and anyone can recheck it.

That writing cannot survive if the collateral becomes US Treasuries. Treasuries are recorded at a custodian bank and a registry, with an identifier, a maturity, and an ownership record. balanceOf on this chain cannot see those records. A number placed in the contract would only be a face value that someone reported, not the Treasury bill itself.

New problems follow. First, the equation would no longer maintain itself through transfers. A custodian, an accountant, or an oracle would update it on a schedule. The update can be late, false, double-counted, or outdated because the bills were moved after the check. Second, the market price of a Treasury bill is not its face value. Maturity and interest rates move it away from one dollar, so a contract that stores only face value will overstate or understate the collateral. Third, redemption cannot finish inside one transaction. Selling a bill takes settlement time, so the vault can be owed the bill and still have no dollars ready to pay. Fourth, the legal owner is the custody account or the issuing company, not the address that holds sUSD. An on-chain equation can show that the contract trusts a report. It cannot show that the holder owns those bills.

The invariant for Treasury collateral could at most say that the last verified custody balance covers the supply, and it would have to name the report time, the verifier, and the price convention. It would no longer be an on-chain balance that anyone can read directly.

<br><br><br>

**D2.** If the collateral were **a building**, how would you put it inside this vault? Which off-chain roles or legal structures would you have to introduce?

> Your answer:
A building cannot be transferred into the vault the way mUSDC can. A chain address cannot own land. The land registry records the title, and the legal holder is a person or a company. To bring a building into this vault, the building first has to become a right that can be recorded off chain, and the vault then holds a token that stands for that right.

I would set up a special-purpose company to buy and hold the building. Shares in that company, or a claim on its income, would be issued as tokens. The vault would hold those tokens, not the building. If users still deposit dollar-like collateral, the issuance rule can stay as it is. If what users receive is a share of the building, “one token equals one dollar” no longer fits, and the contract has to store a ratio between shares and an appraised value.

Several off-chain roles are required. The registry and a lawyer confirm that the title is clear and that no mortgage was left undisclosed. Directors or a trustee of the special-purpose company are the legal owners, and the agreements have to stop them from selling the building privately. An independent valuer publishes a value on a schedule, because a building's price does not update itself on every token transfer. A property manager handles rent and repairs. An insurer covers losses such as fire. An auditor checks that the company still owns the building and sends that result to the chain. If only qualified investors may hold the token, identity checks and transfer limits are also required.

These roles add trust. The contract can still keep the token ledger correct, but it cannot by itself prove that the building is still there, that the appraisal is honest, or that the company has not mortgaged the building to a bank. Redemption also rarely settles on the spot. Selling the building, or writing a holder's name onto the title, passes through a contract, taxes, and the registry. The invariant inside the vault can constrain token shares and the latest appraisal. The existence of the building remains in the legal structure above.

<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex4)

Turn the red tests green in `test/exercises/01_LoopTasks.t.sol` to cover the scenarios below, and write your test function names here:

| Scenario | Your test function name |
|---|---|
| Minting by a non-minter reverts | | test_Ex4_Mint_RevertsForNonMinter
| Transfers revert while paused | | test_Ex4_Pause_BlocksTransfers
| **Redemption** reverts while paused | | test_Ex4_Pause_BlocksRedeem
| An attacker cannot burn someone else's balance | | test_Ex4_AttackerCannotBurnOthersBalance
| ...but the vault holding `MINTER_ROLE` can | | test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance

That last pair is meant to be read together: the guard is written correctly, but the key was handed to the vault. Keep it in mind when you answer A1.

Now write one more scenario you consider **most likely to be attacked**, and say why you picked it:

> Your answer:
The scenario I consider most likely to be attacked is the admin key, the path Ex3 already walks. The contract places the admin, the minter, and the pauser in one address, and grantRole takes effect as soon as it is on chain. An attacker does not need an overflow or a break in the signature scheme. The attacker needs that key, or needs to bribe the person who holds it, and can then grant MINTER_ROLE to the attacker's own address and mint without depositing mUSDC. The collateral is already sitting in the vault, and redemption pays it to whoever holds sUSD. Compared with the vault's overly wide burn right, this path pays more directly. Using the burn right requires some way to call as the vault. Granting the role requires only a key that already exists. The tests show that minting without the role is rejected, so an ordinary user cannot take this path. The weak point is the admin who issues the role, not the ordinary mint function.