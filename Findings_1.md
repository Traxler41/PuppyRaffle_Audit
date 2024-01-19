# [M-1] The function `PuppyRaffle::enterRaffle()` is prone to Denial of Service Attack

**Description:**
The dual loop in the function `PuppyRaffle::enterRaffle()` to check for duplicates is susceptible to attackers to create a `Denial of Service Attack`.The `PuppyRaffle::players[]` array might become so large sometimes that it might be potentially impossible for new players to join in because with the increasing number of players in the raffle, the gas fees will keep on increasing due to the complex loop structure. 

```javascript
@>Denial Of Service
    for (uint256 i = 0; i < players.length - 1; i++) {
             for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

**Impact:**
1. Due to increasing gas fees in the later stages of entry, there will be rush to enter the raffle just when it begins, because the gas fees will be less and the players will be discouraged to enter at the later stages because the increasing gas fees will be sometimes impossible to pay. Thus causing denial of service.
2. A attacker might fill up the `PuppyRaffle::players[]` array with his own wallet addresses right at the beginning so that other players are discouraged to enter the raffle due to the increasing gas fees. Then he can win the raffle always, beating the purpose of a fair lottery. 

**Proof of Concept:**

If we have 2 sets 100 players enter, the gas costs will be as such:
- First 100 players: ~6252048 gas
- Next 100 players: ~18067726 gas

It is as 3x as more expensive for the second 100 players.

<details>
<summary>PoC</summary>
Place the following test into `PuppyRaffle.t.sol`.

```javascript
function test_denialOfService() public {

        vm.txGasPrice(1);

        uint256 playersnum = 100;
        address[] memory players = new address[](playersnum);
        for(uint256 i=0;i<playersnum;i++){
            players[i] = address(i);
        }
        //How much gas it costs??
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasLeft = gasleft();
        uint256 gasUsed1 = (gasStart-gasLeft) * tx.gasprice;


        //Next 100 players
        for(uint256 i=0;i<playersnum;i++){
            players[i] = address(i+100);
        }
        uint256 gasStart2 = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasLeft2 = gasleft();
        uint256 gasUsed2 = gasStart2-gasLeft2;

        console.log("Gas used earlier: ",gasUsed1);
        console.log("Gas used later :",gasUsed2);

        assert(gasUsed2 > gasUsed1);
    }
```
</details>


**Recommended Mitigation:**

There are a few recommendations:-

1. Consider allowing dupilicates. Consider allowing duplicates because the users can create multiple wallet addresses anyway. Only the same wallet address is prevented from entering multiple times. Not the user.
2. Consider using a mapping to check for duplicates. This would allow constant time lookup of whether a user has already entered.

```diff
+    mapping(address=>uint256) public addressToRaffleId;
+    uint256 public raffleID = 0;
     .
     .
     .
     function enterRaffle(address[] memory newPlayers) public
payable {
     require(msg.value = entranceFee * newPlayers.length,
"PuppyRaffle: Must send enough to enter raffle");
     for(uint256 i=0; i<newPlayers.length;i++){
        players.push(newPlayers[i]);
+       addressToRaffleId[newPlayers[i]] = raffleId;
     }

-    //Check for duplicates
+    //Check for duplicates for only new players
+    for (uint256 i=0; i<newPlayers.length;i++){
+       require(addressToRaffleId[newPlayers[i]] != 
raffleId, "PuppyRaffle: Duplicate Player");
+
-    for(uint256 i=0; i<players.length;i++){
-        for(uint256 j=i+1; j<players.length;j++){
-            require(players[i] != players[j],"PuppyRaflle: Duplicate Player");
-        }
-    }

+  }
    emit RaffleEnter(newPlayers);

}

```
