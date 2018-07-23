---
title: "以太坊Keystore基本原理"
date: 2018-07-23T17:53:18+08:00
lastmod: 2018-07-23T17:53:18+08:00
draft: false
keywords: []
description: ""
tags: ["以太坊"]
categories: []
author: ""

comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false

contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

hideHeaderAndFooter: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---
## 交易签名
非对称加密中会涉及到私钥和公钥的概念。我们这里称私钥为PrivKey，公钥为PubKey，需要验证的消息是message，签名为sign（由PrivKey+message得到）

A向B发送消息message，B收到message，但是B怎么确认消息是A发出，并且没有经过篡改呢？利用签名就可以达到这个目的。假设A发生消息时，同时发送公钥和签名，也就是message+PubKey+sign。B接收到message+PubKey+sign，发现由message+PubKey推导的签名结果和收到的sign一致，由此确认message未经篡改。那如何知道是否是A发送呢？答案是在PubKey本身包含A的地址信息。

更进一步，已知算法，私钥可以直接推导出公钥，公钥可以推导出账户地址，即PrivKey==\>PubKey==\>Address。

当以太坊广播一笔交易时，大致就使用了上面的原理。所以保存好私钥PrivKey至关重要。

## Keystore
初次使用以太坊钱包会发现，每次交易都会要求输入密码passphrase。这是由于以太坊使用了Keystore机制，本地只保存了Keystore，并未保存私钥PrivKey。当发起交易时，需要根据KeyStore+passphrase实时计算PrivKey来发起交易。这样的话，即使Keystore不小心被盗，我们的账户也能保证一定的安全。

借助于[keythereum](https://github.com/ethereumjs/keythereum)库，可以帮助我们进一步理解Keystore和PrivKey的关系

```js
var keythereum = require("keythereum");
var secp256k1=require("secp256k1");
var datadir = "/Users/bing/test/data"; //此处填入你的datadir
var address = "0x008aeeda4d805471df9b2a5b0f38a0c3bcba786b"; //此处填入你的账户地址 
var keyObject = keythereum.importFromFile(address, datadir); 
var password = "xxx"; //密码passphrase
var privateKey = keythereum.recover(password, keyObject); //获得私钥
console.log(privateKey.toString('hex'));
//var publicKey = secp256k1.publicKeyCreate(privateKey, false).slice(1);
var publicKey1 = secp256k1.publicKeyCreate(privateKey, false); //获得公钥
console.log(publicKey1.toString('hex'))
```

```json
// keystore文件大概长这样
{
  address: "008aeeda4d805471df9b2a5b0f38a0c3bcba786b",
  Crypto: {
    cipher: "aes-128-ctr",
    ciphertext: "5318b4d5bcd28de64ee5559e671353e16f075ecae9f99c7a79a38af5f869aa46",
    cipherparams: {
      iv: "6087dab2f9fdbbfaddc31a909735c1e6"
    },
    mac: "517ead924a9d0dc3124507e3393d175ce3ff7c1e96529c6c555ce9e51205e9b2",
    kdf: "pbkdf2",
    kdfparams: {
      c: 262144,
      dklen: 32,
      prf: "hmac-sha256",
      salt: "ae3cd4e7013836a3df6bd7241b12db061dbe2c6785853cce422d148a624ce0bd"
    }
  },
  id: "e13b209c-3b2f-4327-bab0-3bef2e51630d",
  version: 3
}
```


---
> 持续学习中，理解可能有误，敬请谅解

> 版权所有，转载请注明来源
