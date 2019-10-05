title: Ethereum以太坊签名验证算法Java实现
tags: []
categories: []
author: Formyown
date: 2019-02-16 15:41:00
---

> eth_sign
The sign method calculates an Ethereum specific signature with: sign(keccak256("\x19Ethereum Signed Message:\n" + len(message) + message))).

By adding a prefix to the message makes the calculated signature recognisable as an Ethereum specific signature. This prevents misuse where a malicious DApp can sign arbitrary data (e.g. transaction) and use the signature to impersonate the victim.
Note the address to sign with must be unlocked.
Parameters
account, message
DATA, 20 Bytes - address.
DATA, N Bytes - message to sign.
Returns
DATA: Signature

https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_sign