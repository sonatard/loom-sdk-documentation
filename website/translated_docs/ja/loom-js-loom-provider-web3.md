---
id: loom-js-loom-provider-web3
title: Loom.js + Web3.js
sidebar_label: Loom.js + Web3.js
---
# 概要

`loom-js`には`LoomProvider`が備わっている。これはイーサリアム開発者がLoom DAppチェーン内で実行されるスマートコントラクトの呼び出しやデプロイができると同時に、`Web3.js`をプロバイダとして接続することを可能にする。さらなる詳細は[EVMページ](evm)をチェックしよう。

NPMで`loom-js`をインストール

```shell
yarn add loom-js
# or if you prefer...
npm install loom-js
```

# コントラクトのインスタンス化

## SimpleContract

あるSolidityコントラクトが、すでにコンパ入りされLoom DAppチェーン上にデプロイされているとしよう。

    pragma solidity ^0.4.18;
    
    contract SimpleStore {
      function set(uint _value) public {
        value = _value;
      }
    
      function get() public constant returns (uint) {
        return value;
      }
    
      uint value;
    }
    

堅さのコンパイラでコンパイルされたバイナリを使って、次のステップではLoom Dappチェーン用の`genesis.json` を作成しよう。 (コンパイルされたバイナリに`location`を設定するのを忘れないように)

```Javascript
{
    "contracts": [
        {
            "vm": "EVM",
            "format": "hex",
            "name": "SimpleStore",
            "location": "/path_to_simple_store/SimpleStore.bin"
        }
    ]
}

```

コントラクトをコンパイルしたら、次のABIインターフェースが生成される:

```js
const ABI = [{
  "constant": false,
  "inputs": [{
    "name": "_value",
    "type": "uint256"
  }],
  "name": "set",
  "outputs": [],
  "payable": false,
  "stateMutability": "nonpayable",
  "type": "function"
}, {
  "constant": true,
  "inputs": [],
  "name": "get",
  "outputs": [{
    "name": "",
    "type": "uint256"
  }],
  "payable": false,
  "stateMutability": "view",
  "type": "function"
}]
```

インスタンス化および`LoomProvider`での`Web3`の使用は、イーサリアムのノードを使用するのに似ているが、まず`loom-js`クライアントを正しく初期化する必要がある。

```js
import {
  NonceTxMiddleware, SignedTxMiddleware, Client,
  Address, LocalAddress, CryptoUtils, LoomProvider, EvmContract
} from '../loom.umd'

import Web3 from 'web3'

// この関数は初期化およびクライアントを返却をする
function getClient(privateKey, publicKey) {
  const client = new Client(
    'default',
    'ws://127.0.0.1:46657/websocket',
    'ws://127.0.0.1:9999/queryws',
  )

  client.txMiddleware = [
    new NonceTxMiddleware(publicKey, client),
    new SignedTxMiddleware(privateKey)
  ]

  return client
}

// キーの設定
const privateKey = CryptoUtils.generatePrivateKey()
const publicKey = CryptoUtils.publicKeyFromPrivateKey(privateKey)

// クライアントの準備
const client = getClient(privateKey, publicKey)
```

クライアントの準備ができたので、今度は`Web3`をインスタンス化しよう。`Web3`を適切にインスタンス化するために、`client`と共に`LoomProvider`を渡そう。

```js
const web3 = new Web3(new LoomProvider(client))
```

コントラクトをインスタンス化する準備ができた。

```js
// 公開鍵を基にアドレスを取得
const fromAddress = LocalAddress.fromPublicKey(publicKey).toString()

// コントラクトアドレスの取得 (アドレスは必要なく、genesis.json中での特定の名前だけで良い)
const loomContractAddress = await client.getContractAddressAsync('SimpleStore')

// Web3と互換性を持つようloom addressをhexaへ変換
const contractAddress = CryptoUtils.bytesToHexAddr(loomContractAddress.local.bytes)

// コントラクトのインスタンス化
const contract = new web3.eth.Contract(ABI, contractAddress, {from: fromAddress})
```

コントラクトはインスタンス化され、準備が整った。

# トランザクションと呼び出し

`Web3 Contract`のインスタンス化が終わったら、トランザクション(`send`) や呼び出し(`call`)のためのコントラクトメソッドを次のように使用できる:

```js
(async function () {
  // バリューを47に設定
  await contract.methods.set(47).send()

  // バリューの取得
  const result = await contract.methods.get().call()
  // 結果は47となる
})()
```

## まとめ

全て準備が整ったので、DAppチェーンが稼働していることを確認してから、次のコードを実行してみよう。`Value: hello!`とコンソールにプリントされるはずだ。

```js
import {
  NonceTxMiddleware, SignedTxMiddleware, Client,
  Address, LocalAddress, CryptoUtils, LoomProvider, EvmContract
} from '../loom.umd'

import Web3 from 'web3'

// この関数は初期化およびクライアントの返却をする
function getClient(privateKey, publicKey) {
  const client = new Client(
    'default',
    'ws://127.0.0.1:46657/websocket',
    'ws://127.0.0.1:9999/queryws',
  )

  client.txMiddleware = [
    new NonceTxMiddleware(publicKey, client),
    new SignedTxMiddleware(privateKey)
  ]

  return client
}

// キーの設定
const privateKey = CryptoUtils.generatePrivateKey()
const publicKey = CryptoUtils.publicKeyFromPrivateKey(privateKey)

// クライアントの準備
const client = getClient(privateKey, publicKey)

// web3の設定
const web3 = new Web3(new LoomProvider(client))

;(async () => {
  // コントラクトABIの設定
  const ABI = [{"constant":false,"inputs":[{"name":"_value","type":"uint256"}],"name":"set","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"}]

  // 秘密鍵を基にアドレスを取得
  const fromAddress = LocalAddress.fromPublicKey(publicKey).toString()

  // コントラクトアドレスの取得 (アドレスは必要なく、genesis.json中での特定の名前だけで良い)
  const loomContractAddress = await client.getContractAddressAsync('SimpleStore')

  // Web3と互換性を持つようloom addressをhexaへ変換
  const contractAddress = CryptoUtils.bytesToHexAddr(loomContractAddress.local.bytes)

  // コントラクトのインスタンス化
  const contract = new web3.eth.Contract(ABI, contractAddress, {from: fromAddress})

  // 47のバリューを設定
  await contract.methods.set(47).send()

  // バリューの取得
  const result = await contract.methods.get().call()
  // result should be 47
})()

```