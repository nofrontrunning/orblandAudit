# Orb Tipping Jar audit notes

## Critical
None.

## High

### H1 Tips can be stolen

The tipping jar distributes tips to the invoker of the associated suggestion. As the tips grow in size, it could be profitable for anyone to purchase the Orb and invoke the suggestion if the tip invoker portion size is close to the Orb's current price, then `relinquish` with auction to sell the Orb. A random account with no interest in the Orb could simply monitor the tip amount and front run the invoke function with purchasing the Orb + invoke suggestion + claim tips to steal all tips from the current keeper. 

This could further escalate into a honey pot situation and cause a chaotic MEV war speculating on the Orb's short-term keepership for profit leading to frequent change-hands of the Orb and unnecessary auctions instead of using the Orb in the designed way, which could lead to a frustrating experience for the parties involved. 

However it is important to acknowledge a random account purchasing the Orb is a perfectly allowed scenario thus one could argue the tips are not considered stolen at all. Whether this is a feature or a bug is up to the Orb team to decide, it is relatively hard to predict how the market will react to it. 

Also under the current design, the keeper could be incentivized to invoke the suggestion prematurely due to fear of accumulating too much tips and got stolen, which might go against the design intention of the tipping jar.

Recommendations - Carefully evaluating the incentives and design intention, if the decision is against speculation, consider introducing restrictions on the distribution of tips if necessary.

### H2 Keeper might not be able to invoke a tipped suggestion

The tipping jar records suggestions in an orb agnostic mapping `suggestedInvocations` which maps invocation hash to it's preimage clear text. When a suggestion is added through the function `suggestInvocation(orb, invocationClearText)`, it checks the given text length is no longer than the targeting Orb's `cleartextMaximumLength`. 

However the tipping function allows tipping any suggestions in `suggestedInvocations` to any Orb instead of the targeting Orb when the suggestion is added. This design is flexible and allows the same question to be tipped and invoked by different Orbs, but it does not check if the tip targeting Orb is able to invoke this particular suggestion since each Orb's `cleartextMaximumLength` is different and can be set freely by the Orb owner. This issue could cause certain tipped suggestions being un-invokable by the targeting Orb at all.

Recommendations - consider adding a check during tipping to ensure the suggestion size fits in the Orb's limit.

### H3 Tipper might be able to withdraw tips after the suggestion is invoked before tips are claimed

The tipping jar designs the invocation and withdraw tips process in 2 functions, although these are supposed to be called in one transaction through `OrbInvocationRegistry.invokeWith*AndCall` as stated in the comment in [line 153](https://github.com/orbland/orb/blob/6e1486f9cba34fb7d9339fbc2aad3eca4591cb3b/src/OrbInvocationTipJar.sol#L153), it is currently not enforced. Direct interactions with the contract might use separate calls for these steps, which opens the door for tippers to withdraw their tip after the suggestion is invoked and before the tips are claimed, or by front running the second part `claimTipsForInvocation` which could also be used as a griefing attack on the keeper to lower the `totalClaimableTips` below keeper specified `minimumTipTotal`, force the transaction to fail.

This attack could be used as a bait with a large tip to encourage invocation of a targeted suggestion, later withdraw the large tip by front running the claiming process to get the suggestion evoked without actual tipping it causing the keeper to loose the invocation profit.

Recommendations - enforcing the invoking and claiming tips in one function inside `OrbInvocationRegistry` in future upgrade.


## Medium

### M1 Keepers can by pass platform fees with an unofficial Tip Jar

Tipping jar is designed in a modular style which makes the Orb agnostic to the tipping jar. Although there are obvious benefits, this design makes it easy to deploy third party tipping jars that's fully compatible with the Orb system. Since current tipping jar charges a platform fee on all tips, keepers might be incentivized to deploy / use a third party tipping jar in order to avoid platform fees. 

Recommendations - Assume front-end support is a major mitigation for this issue, it is a rather weak solution. However this might not be a significant enough concern to change the design structure of the Orb system. Consider carefully weighting the pros and cons of the design pattern, adding the tipping jar into the Orb system in next upgrade if the concern is significant.


### M2 Orb team can update `platformFee` to steal all tips

The `platformFee` state variable in the tipping jar contract specifies the portion of tips keepers pay to the Orb platform. This value should be either pre-agreed on by both parties and remain immutable or if it is designed to be updatable later, an opt-out mechanism should be provided to the keepers.  

Current tipping jar design sets this variable in the initializer and does not offer any ways to update it. However, the tipping jar owner can easily update this variable to `_FEE_DENOMINATOR` to steal all tips with an contract upgrade. 

Since the tipping jar is upgradable, there is no mitigation of privilege concerns. However certain practices could be implemented to improve the design, e.g.   

 1. setting this variable as a const if there are no plans to update it and extend the contract for future upgrades.
 2. associate this variable with each Orb at deploy time, this allows the Orb team to adjust `platformFee` but still have it locked in for each Orb.
 3. offer functions for keepers to opt-out if they do not agree with new `platformFee`.

### M3 Owner could stop a tipped suggestion being invoked 

The tipping jar design exposes Orb invocation questions way before the actual invocation happens, this might have an incentive impact on the owner of the Orb since they are not directly benefiting from the tips but rather has to answer the hard questions crowd-sourced through the tipping mechanism. In the case of the owner wants to avoid a certain suggestion with leading tips, they could potentially 

1. prepare early and wait for an opportunity when the orb is in the owner's control to reduce the `cleartextMaximumLength` to be smaller than the suggestion's length.

2. directly purchase the Orb under a different address and relinquish before reducing `cleartextMaximumLength`, this could be done anonymously

On top of this, some suggestions might be accidentally invalidated whenever the owner changes `cleartextMaximumLength` even without a malicious intent. 

Recommendations - The tipping jar should guarantee the invocation of a tipped suggestion, otherwise it should not allow this suggestion to be tipped. This is a little tricky under current design given the flexibility of the text length. Potential solution might be 

1. fix the max text length for all Orbs instead of allowing owners to adjust it 
2. modify the Orb to allow invocation from tip jar to by pass the length restriction. 

Both solutions have clear flaws and might not be suitable for the system, consider carefully evaluating the situation and make adjustment accordingly.


### M4 Duplicated invocations allowed by the Orb might not be suggestable into the tipping jar

The Orb allows duplicated invocations through `OrbInvocationRegistry` but the tipping jar does not allow this, `suggestInvocation` function will revert if the invocation already exists. This could cause confusing and mis-aligned behavior in the system.

Recommendations - unifies the behavior of the system, either allows duplicate invocations in both contracts or none. 

## Low

### L1 Lack of input checks on platformFee and platformAddress

The `initialize()` function in the tipping jar sets the official `platformFee` and `platformAddress` variables which can not be easily updated. 

However there are no validation checks for these variables, consider checking the `platformFee` is no larger than `_FEE_DENOMINATOR` and `platformAddress` is not zero.

### L2 Unable to tip an edge case invocation

The tipping jar allows suggesting invocations with empty clear text (hash value is not empty) if the `msg.value` is 0 through the function `suggestInvocation()`. Furthermore, the `OrbInvocationRegistry` also allows invocations with empty clear text.

However, there is no way to tip this invocation in the tipping jar due to [empty value check in line 139](https://github.com/orbland/orb/blob/6e1486f9cba34fb7d9339fbc2aad3eca4591cb3b/src/OrbInvocationTipJar.sol#L135)

```
    if (bytes(suggestedInvocations[invocationHash]).length == 0) {
        revert InvocationNotFound();
    }
```

Consider making it clear if empty clear text is allowed and enforcing checks accordingly.

### L3 Keeper could stop user tipping by front running it with `setMinimumTip`

The tipping jar allows the Orb keeper to set a minimum tip amount. However this function could be used by the keeper to to stop a tip by frontrunning the tipping transaction with `setMinimumTip` and increase the minimum tip to be higher than the tipping transaction.

Recommendations - the keeper's incentive to attack in this way is rather low and front running is generally hard to mitigate. Potential solution could be hard code the minimum tip into the contract, which has it's own limitations. Consider carefully weighting the pros and cons of different designs vs incentives before deciding if a fix is needed.


### L4 Lack of input checks for loop

The function `withdrawTips()` takes in 2 param arrays `address[] memory orbs, bytes32[] memory invocationHashes` and loop through them. However it does not check if these two arrays are of the same length.

Recommendations - check these two arrays has equal length before looping through them to avoid mis-matching errors.


## Note

### N1 Tipping jar can be used by un-official orbs, causing pollution to event logs

There are no immediate security concerns, however this design could pollute event logs and encourage usage of unofficial orb systems. 

Consider carefully weighting the pros and cons of this design, and checking the given orb is official in the tipping jar if the final decision is to restrict access to only official orbs.

### N2 Anyone can claim tips for past invocations not registered in the tipping jar, causing pollution to the `TipsClaim` event logs

The tipping jar allows claiming tips from past evocations which are not previously suggested into the tipping jar. All of the claiming value will be 0, and the transaction will succeed. This could lead to pollution of the `TipsClaim` event logs and unnecessary confusion. 

Consider disallowing claiming tips from evocations that's not previously suggested into the tipping jar.
