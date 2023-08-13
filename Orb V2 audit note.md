# Orb V2 audit note

### 1. Removing the cooldown reset after an auction may introduce new attack vectors.

This update eliminated the cooldown reset upon the conclusion of auctions if the auction beneficiary is not the orb beneficiary. This redesigned approach ensures that the Orb does not get invoked without the appropriate cooldown interval. While this alteration was made intentionally, it could change the incentive design of the Orb system and might introduce new issues and attack vectors:

A. This strategy might inadvertently cause a new keeper to incur taxes without the chance to invoke the Orb, especially if the Orb is acquired before a cooldown concludes. Moreover, if the Orb's pricing plus the taxes paid by the keeper is lower compared to its last auctioned value, the keeper might experience a net loss, resulting in an unfavorable experience for the keeper.

B. The Orb Owner could exploit new keepers for tax revenue by acquiring the Orb just before the cooldown to avoid the Orb being invoked. This approach might turn lucrative (and hence no longer merely a griefing tactic) if the current price of the Orb is lower than the original owner auction price plus the applicable harberger tax during this period. Note this attack does not respect the profitability of the most recent keeper. Even if the keeper decides to set a new Orb price at twice the value of the last auctioned amount, it could still be less than the original owner's auction price, making it profitable for the owner to engage in such tactics. To maintain anonymity, the owner might utilize a fresh address for this purpose.

C. Depending on the Orb's pricing, the Orb Owner might be incentivized to encourage as many auctions as feasible, especially if the profits from these auctions surpass the tax revenues garnered over an equivalent duration. For instance, the owner might use a new address to purchase the Orb if the price is low and then relinquish() or foreclose() to force a new auction. The fact that the Orb Owner is not obligated to answer additional questions with each auction could further spur this trend. As a result, the Orb Owner may proactively engage in actions that lead to owner or keeper auctions, as well as associated attack vectors like the one mentioned in point B.

Recommended fix : The crux of this issue stems from the misalignment between harberger tax payments and Orb invocations. It's challenging to entirely rectify this under the existing system framework. The initial design, which allowed for cooldown resets, might also be susceptible to analogous attack vectors. It's advisable to thoroughly weigh the advantages and drawbacks of various alternatives and recalibrate the system as deemed necessary.



### 2. Unpinned solidity compiler

The contract uses unpinned solidity compiler ^0.8.20. 

In order to avoid unexpected compiler behavior, consider pinning the solidity compiler version to 0.8.20.
