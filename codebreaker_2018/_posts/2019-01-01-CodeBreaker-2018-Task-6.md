---
layout: default
permalink: /CodeBreaker-2018-Task-6/
title: NSA Codebreaker 2018, Task 6
---

Task 6 requires a way to decrypt the victim's encryption key without paying the ransom. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/vid_not_checked.png)

Below are some of the reasons this is possible:<br>
- Escrow contract does not check to see if a victim ID has already been registered<br>
- Ransom amount is set by the Ransom contract<br>
- Ransom amount is not validated by Escrow<br>

# Exploit Contract by Registering with ransomAmount = 0 #

One method to achieve the desired outcome is to register a Ransom contract with same victim ID and ransom amount equal to 0. To make it easier, the other arguments, except for the authToken, can be hard coded. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/contract_changes_1.png)

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/contract_changes_2.png)

The deployment of the contract and payment of the ransom happens the same as it did for the victim, except the Ransom contract has been modified with a ransom amount equal to 0. Below are the steps:<br>

1. Register modified Contract<br>
2. Call payRansom<br>
3. Call getDecryptionKey<br>

The authToken can be generated using the python module [pyotp](https://github.com/pyauth/pyotp). 
{% highlight python %}
import pyotp
totp = pyotp.TOTP('7P57CDMIUWYV5BMHT3C7HIGJOYRLDVZT')
totp.now()
{% endhighlight %}

# Exploit Contract by Causing Revert #

There is another option to get the decryption key without paying. This method still requires you to deploy a modified contract, and send the complete ransom amount. When the decryptCallback function is called the Escrow contract will pass control to the Ransom contract by calling the fulfillContract function. If this function causes a revert, then the ransom will never leave the escrow and go into the attacker's control. The transaction that caused the failure will still be [included in the blockchain](https://ethereum.stackexchange.com/questions/39817/are-failed-transactions-included-in-the-blockchain), so that the caller still has to pay for the gas to process. The failed block can be [retrievable just like any other block](https://ethereum.stackexchange.com/questions/56908/where-can-i-find-failed-transactions); here is an example of a [failed block](https://etherscan.io/tx/0xd1c64ba497a831db014fcd1a41a12f4a08b9f5ee18ed6171f8fef06cc3e5f817) provided. 

Note for the below example, I set the ransom amount to 0 for testing since I didn't have the 100 ether to do it with the ransom amount set to 100 ether. 

I added "require(fulfilled == true);", which will always be false and cause a revert. I deployed the contract and called the payRansom function; the transaction was recorded in block 1894033. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/fulfilled_revert.png)

This is block 1894033 and contains the transaction hashes for included transactions, only 1 for this block. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/payRansom_block.png)

This is the first transaction for block 1894033, which corresponds to the payRansom function. The function arguments are found under input. We can check the next few block to look for the  transaction for the oracle calling decryptCallback. If there were more blocks and transactions, then we could use python to iterate through each transaction in a block and filter on the from and to addresses. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/payRansom_trans.png)

The decrypted key is recorded in the transaction's input. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/decryptCallback_trans.png)

The transaction receipt can be used to determine if a function succeeded (1) or failed (0). The transaction for the oracle calling decryptCallback failed. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/trans_status.png)

Additionally, as suspected there is no event emitted from the decryptCallback function, since the transaction failed. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/transaction_events.png)

The decrypted encryption key will forever be included in the blockchain, and Ether paid can be retrieved using requestRefund. 

# Exploit Contract Through Race Condition #
If the authToken can't be determined in order to deploy a modified contract, the decrypted encryption key can still be retrieved by the below methods to exploit a race condition.

## Exploit by Calling requestRefund ##

This method requires that the full ransom amount be paid, but afterwards any Ether paid can be retrieved. This exploit is possible since the decryptCallback function does not check if the escrowMap[id] is greater than the ransomAmount (you may see a trend here, many of the attacks I mention in task 7 are because of this). This attack is known as [Front Running (AKA Transaction-Ordering Dependence)](https://consensys.github.io/smart-contract-best-practices/known_attacks/#front-running-aka-transaction-ordering-dependence), which is when the transactions with a contract have to happen in a certain order. 

In the Escrow contract, paying the ransom and the ransom being moved to the attacker's control happen in two different transactions. This opens the possibility for someone to exploit a race condition, and submit a third transaction for calling the function requestRefund in between the previous transactions. [Solidity: Transaction-Ordering Attacks](https://medium.com/coinmonks/solidity-transaction-ordering-attacks-1193a014884e) has a detailed example of this attack. In short the malicious party, us, would submit the transaction for requestRefund with a higher gas price than the transaction with decryptCallback, ensuring that miners will prioritize our transaction over others when including transactions in blocks to mine. 

Below are the steps for this:<br>
1. Create Event filter to monitor for DecryptEvent from Escrow contract<br>
2. Prepare two transactions, one calling payRansom, the other calling requestRefund with a higher gas price<br>
3. Send the payRansom transaction and wait for Event to be emitted<br>
4. When the DecryptEvent is emitted, send the requestRefund transaction<br>
5. Wait for requestRefund to be mined<br>
6. If the requestRefund transaction was mined before the transaction with decryptCallback, then the escrowMap[id] would have underflowed, and the requestRefund can be called to retrieve any Ether paid in an unsuccessful attempt. <br>

If the Escrow contract has enough Ether to cover the ransom amount, then the above attack can be changed to protect the attacker from having Ether tied up on any unsuccessful attempts. Instead of sending an encrypted file that will be successful, the attacker can send one that will cause the decryption to fail; the rest of the attack is the same. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/Timing_Attack.png)

On an successful attack the attacker will unflow escrowMap[id], receive Ether sent and Ether from contract. On an unsuccessful attack the attacker will still receive the Ether they sent, since the decryption failed. After a successful attack the attacker can send the encrypted file, receive the decryption key, and then use the requestRefund to retrieve Ether sent. 

The above attack can also be used for task 7. 

## Exploit by Calling payRansom Twice ##

There is another method that takes advantage of the communication to the off chain oracle and ordering of transactions. I have not tested this attack, but it works the same as the above attack for causing a revert, except the failure is caused in decryptCallback instead of fulfillContract. This method is to call the payRansom function twice, one with a file to cause the decryption to fail, and the second with one to cause it to succeed. This requires the attacker to have enough Ether to pay the ransom twice, and runs the risk of losing half the Ether spent. 

On success the attacker can retrieve the Ether spent using requestRefund and get the decryption key from the failed transaction. This is because the first callback will be for the file that failed decryption; the Ether spent will be returned to the caller and encFileMap[id] will be cleared. On the second call the function decryptCallback will check to ensure encFileMap[id] is not empty, since it was cleared in the first callback this transaction will fail. As mentioned earlier, failed transactions are still recorded in the blockchain and their arguments can be read, revealing the decrypted encryption key. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_6/Timing_Attack_payRansom.png)


