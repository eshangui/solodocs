# Send your coins

## `send` command


In this section, we will transfer ETH with **WOOKONG Solo**.

1. ETH transaction data is [RLP](https://github.com/ethereum/wiki/wiki/RLP) encoded, we do not need to add a dependency to [rlp package](https://github.com/ethereumjs/rlp) in our project explicitly as it is already in the dependency tree of [web3.js](https://github.com/ethereum/web3.js), just add the following code to the top of `index.js` is enough.
   
    ```js
    const { encode } = require('rlp');
    ```

2. Add the following function at the end of `index.js`.
    ```js
    async function processSend(command) {
        const ready = await isDeviceReady();
        if (!ready) {
            return;
        }
        // we need address to get nonce
        const derivePath = JSON.parse(command[1]);
        const addressResult = await getAddress(derivePath, 0);
        if (addressResult.code !== core.constants.rets.SUCCESS) {
            console.log(`get address failed: ${utils.parseReturnCode(addressResult.code)}`);
            return;
        }
        try {
            // get nonce
            const nonce = web3.utils.toHex(await web3.eth.getTransactionCount(addressResult.result.address));
            // get gas price
            const gasPrice = await web3.eth.getGasPrice()
            const gasPriceHex = web3.utils.toHex(gasPrice);
            // gas limit for ETH transaction is fixed to 21000 in Ethereum Yellow Book
            const gasLimit = 21000;
            const gasLimitHex = web3.utils.toHex(gasLimit);
            const to = command[2];
            const value = web3.utils.toHex(await web3.utils.toWei(command[3]));
            // raw data to sign
            const dataToSign = [nonce, gasPriceHex, gasLimitHex, to, value, '','0x01', '0x', '0x'];
            // raw data to sign, rlp encoded
            const rawToSign = encode(dataToSign).toString('hex');
            // call WOOKONG Solo for signature
            const signResult = await core.signEthereum(derivePath, rawToSign, true);
            if (signResult.code !== core.constants.rets.SUCCESS) {
                console.log('ETH sign failed:', utils.parseReturnCode(signResult.code));
                return;
            }
            let v = '';
            if (signResult.result.sign.v === '0x00') {
                v = '0x25';
            } else {
                v = '0x26';
            }
            // signed raw data
            const dataSigned = [nonce, gasPriceHex, gasLimitHex, to, value, '', v, signResult.result.sign.r, signResult.result.sign.s];
            // signed raw data, rlp encoded
            const rawSigned = `0x${encode(dataSigned).toString('hex')}`;
            // broadcast transaction to blockchain
            const txid = const txid = await new Promise((resolve, reject) => { 
                web3.eth.sendSignedTransaction(rawSigned, (err, hash) => {
                    if (err) {
                        reject(err);
                    } else {
                        resolve(hash);
                    }
                });
            });
            console.log(`transaction succeeded.`);
            console.log(`from: ${addressResult.result.address}`);
            console.log(`to: ${command[2]}`);
            console.log(`value: ${command[3]} Ether`);
            console.log(`gas price: ${gasPrice} Wei`);
            console.log(`gas limit: ${gasLimit}`);
            console.log(`you can see your transaction detail here: https://etherscan.io/tx/${txid}`);
        } catch (error) {
            console.log(`error: ${error.message}`,);
            return;
        }
    }
    ```
3. Now connect your **WOOKONG Solo** to your system, make sure it is well connected and unlocked, run `node index.js`, then run `send -p[0, 2147483692, 2147483708, 2147483648, 0, 0] -p0x7F825230F5F2A26523999c98e0E3f7E2697085A9 -p0.00001` under `WST>` prompt, you should change the ETH address in the command to your own address, wait for a few seconds then you will see the transaction information is shown on device screen, use '▲' and '▼' buttons to see the details, press 'OK' to sign this transaction when you make sure all information shown on screen is exactly what you input just now, then you may see output as shown below:

    ```shell
    WST> send [0,2147483692,2147483708,2147483648,0,0] 0x7F825230F5F2A26523999c98e0E3f7E2697085A9 0.00001
    transaction succeeded.
    from: 0x7A2e95248198B97C6E46829F3A977250121aa25d
    to: 0x7F825230F5F2A26523999c98e0E3f7E2697085A9
    value: 0.00001 Ether
    gas price: 3000000000 Wei
    gas limit: 21000
    txid is: 0xf87849866f9ceffd2607179b096f1f6d7b41ceb2664272302cd359383afe1e78
    ```