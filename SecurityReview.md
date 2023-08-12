# Orb security review
Commit fff3771b8788f0c60a65c73f779326a0305d3622

### issue
1. if the owner `relinquishWithAuction()` after `listWithPrice()`, the auctionBeneficiary will be set to `owner` instead of address(0). This will trigger unnecessary royalty split when finalize auction. Also as we discussed, the keeper could avoid splitting auction income with `beneficiary` by setting `royaltyNumerator = 0`. => consider restrict this function if it is not meant for creator or make it usable for both creator and keeper. 

### Notes
1. maybe future iterations - some storage vars can be tightly packed to save gas e.g. uint32 for date, time, uint64/128 for prices, uint16 for factors, percentage denominators etc.
2. no need to initialize `auctionBeneficiary` as address(0) in L145
3. function `setAuctionParameters()` is missing docstring for newly added `newKeeperMinimumDuration`
4. function `finalizeAuction()` has repeating code with `purchase()` when it calculates royalty,  Consider refactoring. Similarly in functions `relinquish()` and `relinquishWithAuction()`, these two could be combined with a bool flag to indicate auction.
5. consider using keeper or beneficiary as the auctionBeneficiary address depending on who started the auction, avoid introducing a new add(0) concept to represent beneficiary.
6. function `relinquish()` makes little sense after adding function `relinquishWithAuction()` which does everything `relinquish()` does immediately with potential income through auction in the future. 
