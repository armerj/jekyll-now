---
layout: default
permalink: /CodeBreaker-2018-Contract-Improvements/
title: NSA Codebreaker 2018, Escrow Contract Improvements
---

The Escrow contract has many problems that causes it to be vulnerable to exploitation; allowing the recovery of Ether and retrieval of the encryption key. Below are some simple fixes that would remove these vulnerabilities from the contract. The two biggest fixes are to prevent the decrypted encryption key leaking if the decryptCallback function fails, and to check the ransom amount is still paid at each step of the process and not just the beginning. 

# To Prevent Key Leakage #

To prevent leaking the key when the decryptCallback function fails, a second callback can be created. The decryptCallback should be changed to only move the ransom amount to the attackers control and on success emit an event to release the encryption key. When the oracle sees the event to release the encryption key the oracle will callback to the Escrow contract with the decrypted encryption key. 

![_config.yml]({{ site.baseurl }}/images/Codebreaker_2018/contract_improvements/Release_Key.png)

# To Prevent Re-registering #

The Escrow contract should check to see if the victim ID is already used, and if so fail.

# Keep Control of Execution #

When control is passed to the Escrow contract it shouldn't call a function from another untrustworthy contract, such as Ransom contract, but instead run until completion; protecting against re-rentrancy. 

## Fulfillment ##

The is fulfilled function should be removed from the Ransom contract, this can be implemented in the Escrow contract as either an additional variable in Victim struct or by checking if decryptKey has been released. 

## Paying the Ransom ##

The pay ransom function should be moved to the Ransom contract. Once called with enough Ether, the Ransom would call into the Escrow contract with everything required (encrypted file, Ether, victim ID, and encrypted encryption key), and the Escrow can validate everything and emit event without the Ransom contract changing the contract's state. 

# Validating Ransom Amount #

By letting the ransom amount be set by the Ransom contract, the threat actor is able to be flexible with his ransom amount. To keep this behavior the ransom amount should be validated in the Escrow or passed in when the Register contract validates the Ransom contract. This ensures the amount is in his control the whole time, and prevents the ransom amount from being manipulated. 

# To Prevent Underflow #

To prevent an underflow from happening the DecryptCallback function needs to ensure that escrowMap[id] is greater or equal to ransomAmount. If it is not the function should fail and not subtract the ransomAmount and causing an underflow. 