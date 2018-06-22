## Notes on installing loom with truffle and web3

### Steps to getting it set up locally

1. Read [web3, loomprovider & truffle](https://loomx.io/developers/docs/en/web3js-loom-provider-truffle.html) blog.
2. installed loom, by following [this guide to installing on macOS](https://loomx.io/developers/docs/en/prereqs.html). Then confirmed that the _go_ contract in that example worked, which meant I had to do _$GOPATH_ things. But I was able to interact with that contract, which proved to me that I had my loom installed correctly and that you CAN interact w contracts on it. 
3. Next up, how to interact with SOLIDITY contracts on the loom chain.  Followed the tutorial in #1, had a few false starts and a node upgrade. 

### My node / npm versions
```
~/Code/loom-test-simple-store 🐈  $ node -v
v8.11.3
~/Code/loom-test-simple-store 🐈  $ npm -v
5.6.0
```

## Issues I ran into: 
The scripts in the tutorial in #1 didn't work as written, so I ended up putting together a messy node script made up of a combination of:
* [the web3 tutorial scripts](https://loomx.io/developers/docs/en/web3js-loom-provider-truffle.html); 
* the [NodeJS and Browser Quickstart Guide Script](https://loomx.io/developers/docs/en/loom-js-quickstart.html)
* the [source code for loom-truffle-provider](https://github.com/loomnetwork/loom-truffle-provider/blob/master/src/index.ts). 

## Errors I found while following the tutorials
### LoomProvider
For example, initializing `LoomProvider` as detailed in the tutorial gave threw an `LoomProvider is not a constructor` error when I used it.  However, When I instead included `LoomProvider` explicitly via the `loom-js` require, the code initializing it, that was provided in the tutorial (`const web3 = new Web3(new LoomProvider(client, privateKey))`) worked.   

### Private Key generation
I also had trouble with the generating a public key when I used the privateKey as generated via the loom tutorial: `privateKey = readFileSync('./private_key', 'utf-8')`. I got the following error: 
```
/Users/sarah/Code/loom-test-simple-store/node_modules/tweetnacl/nacl-fast.js:2151
      throw new TypeError('unexpected type, use Uint8Array');
      ^

TypeError: unexpected type, use Uint8Array
    at checkArrayTypes (/Users/sarah/Code/loom-test-simple-store/node_modules/tweetnacl/nacl-fast.js:2151:13)
    at Function.nacl.sign.keyPair.fromSecretKey (/Users/sarah/Code/loom-test-simple-store/node_modules/tweetnacl/nacl-fast.js:2304:3)
    at Object.publicKeyFromPrivateKey (/Users/sarah/Code/loom-test-simple-store/node_modules/loom-js/dist/crypto-utils.js:48:49)
    at Object.<anonymous> (/Users/sarah/Code/loom-test-simple-store/index.js:34:31)
    at Module._compile (module.js:652:30)
    at Object.Module._extensions..js (module.js:663:10)
    at Module.load (module.js:565:32)
    at tryModuleLoad (module.js:505:12)
    at Function.Module._load (module.js:497:3)
    at Function.Module.runMain (module.js:693:10)
```

## Working Script
In short, a very hacked-together proof that I can interact with a smart contract (written in solidity) on a locally running loom blockchain.  I'd prefer to do this in web3 but node was quicker b/c those were my examples.  This code will not win any beauty awards but here is my index.js file
```
const { readFileSync } = require('fs')
const {
  NonceTxMiddleware, SignedTxMiddleware, Client,
  Contract, Address, LocalAddress, CryptoUtils, LoomProvider
} = require('loom-js')


const TruffleLoomProvider = require('loom-truffle-provider')
const chainId    = 'default'
const writeUrl   = 'http://127.0.0.1:46658/rpc'
const readUrl    = 'http://127.0.0.1:46658/query'
const privateKey = CryptoUtils.generatePrivateKey()
const loomTruffleProvider = new TruffleLoomProvider(chainId, writeUrl, readUrl, privateKey)
const publicKey = CryptoUtils.publicKeyFromPrivateKey(privateKey)
const Web3 = require('web3')

// Create the client
const client = new Client(
  'default',
  'ws://127.0.0.1:46657/websocket',
  'ws://127.0.0.1:9999/queryws',
)

// The address for the caller of the function
const from = LocalAddress.fromPublicKey(publicKey).toString()
console.log("from", from);

// Instantiate web3 client using LoomProvider
const web3 = new Web3(new LoomProvider(client, privateKey))
console.log("web3 declared");

const ABI = [{"anonymous":false,"inputs":[{"indexed":false,"name":"_value","type":"uint256"}],"name":"NewValueSet","type":"event"},{"constant":false,"inputs":[{"name":"_value","type":"uint256"}],"name":"set","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"}]

// contract address on my local loom network
// I got this from copy-pasting the contract address after running a deploy with loom
const contractAddress = '0x1a31b9b9d281d49001fe7f3f638000a739afc9c3'; 

const contract = new web3.eth.Contract(ABI, contractAddress, {from})

// just set some value and read it back to prove you can write and read. 190 is completely arbitrary
contract.methods.set(190).send().
    then(tx => {
        console.log("Tx complete", tx);
        contract.methods.get().call()
            .then(res => {
                console.log("res ", res)
            })
    })
```

### Working Truffle Config
Also here is my working truffle.js script
```
const { readFileSync } = require('fs')
const TruffleLoomProvider = require('loom-truffle-provider')

const chainId    = 'default'
const writeUrl   = 'http://127.0.0.1:46658/rpc'
const readUrl    = 'http://127.0.0.1:46658/query'
const privateKey = readFileSync('./private_key', 'utf-8')

const loomTruffleProvider = new TruffleLoomProvider(chainId, writeUrl, readUrl, privateKey)

module.exports = {
    loomTruffleProvider: loomTruffleProvider,
    TruffleLoomProvider:TruffleLoomProvider,
    chainId: chainId,
    writeUrl:writeUrl,
    readUrl:readUrl,
    privateKey: privateKey,

    networks: {
    loom_dapp_chain: {
      provider: loomTruffleProvider,
      network_id: '*'
    }
  }
}
```
