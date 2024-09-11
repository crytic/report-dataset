## Vesting contract owner is not updated when VestingFactory owner changes

**Description**

After ownership of the VestingFactory contract is transferred, the previous owner still controls the previously created Vesting contracts and can withdraw funds from them. It is unclear whether this behavior is intended.

When creating a Vesting contract, VestingFactory assigns its own current owner to the created Vesting contract’s owner:

Ownership of the VestingFactory contract can be transferred in a two-step process. If the VestingFactory owner changes, all new Vesting contracts will be owned by the new VestingFactory owner; however, the previous VestingFactory owner will continue to control the existing Vesting contracts. As a result, two different owners will control different sets of Vesting contracts. The previous owner can withdraw funds from Vesting contracts before their vesting schedules complete.

**Recommendations**

Short term, document the process so that users are informed of who will control a Vesting contract when the VestingFactory owner is transferred.

Long term, exercise caution when implementing code that reuses contract owners for other contracts; ensure that transactions for transferring ownership of all the contracts owned by a single address are executed at once.
