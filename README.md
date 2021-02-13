# You Need A Transaction Manager (YNATM)

Thank you [kendricktan](https://github.com/kendricktan) for writing this
library.

[![circleci](https://badgen.net/circleci/github/ethereum-optimism/ynatm)](https://app.circleci.com/pipelines/github/ethereum-optimism/ynatm)
[![npm](https://badgen.net/npm/v/ynatm)](https://www.npmjs.com/package/@eth-optimism/ynatm)

**(For Ethereum)**

With the recent spike in gas prices, you can't just send a 1 GWEI gas price for your Ethereum tx and hope that it will get mined.

This small module helps you guarantee that your transaction gets mined within a reasonable time frame, by bumping up the gas price (up till a threshold) until your transaction gets mined.

## Examples

### Quickstart

```bash
npm install ynatm
```

```javascript
const ynatm = require("ynatm");

const nonce = provider.getTransactionCount(SENDER_ADDRESS);

const txOptions = {
  from: SENDER_ADDRESS,
  to: RECIPIENT_ADDRESS,
  nonce
}

const tx = await ynatm.send({
  sendTransactionFunction: (gasPrice) =>
    wallet.sendTransaction({ ...txOptions, gasPrice }),
  minGasPrice: ynatm.toGwei(1),
  maxGasPrice: ynatm.toGwei(20),
  gasPriceScalingFunction: ynatm.LINEAR(5), // Scales by 5 GWEI in gasPrice between each try
  delay: 15000, // Waits 15 second between each try
});
```

### Contract Interaction

Since `ynatm` is framework agnostic, you can also use it for contract interaction like so:

```javascript
const ynatm = require("ynatm");

const nonce = provider.getTransactionCount(SENDER_ADDRESS);
const options = {
  from: SENDER_ADDRESS,
  nonce,
}

const ethersSendContractFunction = (gasPrice) => {
  const tx = MyContract.functionName(params, { ...options, gasPrice });
  const txRecp = await tx.wait(1); // wait for 1 confirmations
  return txRecp;
};

const web3SendContractFunction = (gasPrice) => {
  // Web3 by default waits for the receipt
  return MyContract.methods.functionName(params).send({ ...options, gasPrice });
};

const tx = await ynatm.send({
  sendTransactionFunction: ethersSendContractFunction, // or web3SendContractFunction
  minGasPrice: ynatm.toGwei(1),
  maxGasPrice: ynatm.toGwei(20),
  gasPriceScalingFunction: ynatm.LINEAR(5), // Scales by 5 GWEI in gasPrice between each try
  delay: 15000, // Waits 15 second between each try
});
```

### Custom `gasPriceScalingFunction`

You can define your own `gasPriceScalingFunction`, which takes in a destructured object containing the following keys:
- `x`: X'th number of try
- `y`: Current gasPrice
- `c`: Constant, `minGasPrice`

```javascript
const customGasScalingFunction = ({ x, y, c }) => {
  return ...
}
```

### Immediate Error Handling with `rejectImmediatelyOnCondition`

The expected behavior when the transaction manager hits an error is to:

1. Check if the error meets the condition specified in `rejectImmediatelyOnCondition` (Defaults to checking for reverts)
   - If the condition is met, all future transactions are cancelled the the promise is rejected
2. Checks to see if all the transactions have failed
   - If all transactions have failed, reject the last error
3. Keep trying

That means that if you're queued up 5 invalid transactions, all 5 of them will need to fail before you can thrown an error.

If you'd like to speed up the process and immediately throw an error when the first invalid transaction is thrown matches a certain criteria, you can do so by overriding the `rejectImmediatelyOnCondition` like so:

```javascript
const ynatm = require("ynatm");

const rejectOnTheseMessages = (err) => {
  const errMsg = err.toString().toLowerCase();

  const conditions = ["revert", "gas", "nonce", "invalid"];

  for (const i of conditions) {
    if (errMsg.includes(i)) {
      return true;
    }
  }

  return false;
};

const nonce = await provider.getTransactionCount(SENDER_ADDRESS);

const tx = {
  from: SENDER_ADDRESS,
  to: RECIPIENT_ADDRESS,
  nonce,
  data: '0x'
}

await ynatm.send({
  sendTransactionFunction: (gasPrice) => wallet.sendTransaction({ ...tx, gasPrice }),
  minGasPrice: ynatm.toGwei(1),
  maxGasPrice: ynatm.toGwei(20),
  gasPriceScalingFunction: ynatm.LINEAR(5),
  delay: 15000,
  rejectImmediatelyOnCondition: rejectOnTheseMessages,
});
```

## Testing

```bash
# Terminal 1
yes '' | geth --dev --dev.period 15 --http --http.addr '0.0.0.0' --http.port 8545 --http.api 'eth,net,web3,account,admin,personal' --unlock '0' --allow-insecure-unlock

# Terminal 2
yarn test
```

If you don't have `geth` installed locally, you can also use `docker`

```bash
# Terminal 1
docker run -p 127.0.0.1:8545:8545/tcp --entrypoint /bin/sh ethereum/client-go:v1.9.14 -c "yes '' | geth --dev --dev.period 15 --http --http.addr '0.0.0.0' --http.port 8545 --http.api 'eth,net,web3,account,admin,personal' --unlock '0' --allow-insecure-unlock"

# Terminal 2
yarn test
```
