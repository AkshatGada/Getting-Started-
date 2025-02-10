# Agglayer Bridge & Call Quick Start Guide

This guide demonstrates how to transfer an asset and call a contract from Agglayer-connected chains with a single click. In this tutorial, you'll learn to:

- Use the `bridgeAndCall` API to execute a cross-chain contract call.
- Claim the bridged transaction using the `claimMessage` API.

For this example, we assume:
- You have deployed a simple `Counter.sol` contract on the Silicon Sepolia testnet.
- You are bridging from Cardona (source chain, network ID: 1) to Silicon Sepolia (destination chain, network ID: 0).
- Your configuration files (`config.js` and `utils_lxly.js`) are set up with the appropriate network settings, RPC endpoints, and account details.

─────────────────────────────  
## Step 1: Prerequisites

Ensure you have the following:
- **Node.js** (v14.0.0 or later) and **npm** (v6.0.0 or later)
- A Hardhat wallet (or similar) configured for testing
- Sufficient test ETH on both Cardona and Silicon Sepolia networks
- The `Counter.sol` contract deployed on Silicon Sepolia
- Properly configured files (`config.js` and `utils_lxly.js`) with all necessary network settings and account details

─────────────────────────────  
## Step 2: Configure Your Environment

Before running any scripts, verify your configuration files are correctly set up:
- **config.js:** Contains network settings, contract addresses, and account details.
- **utils_lxly.js:** Initializes the lxly.js client with providers for both Cardona (source) and Silicon Sepolia (destination).

Refer to the Agglayer Unified Bridge repository for sample configurations if needed.

─────────────────────────────  
## Step 3: Bridge and Call the Contract

In this step, run the `bridge_and_call.js` script to initiate a cross-chain transaction that calls the `increment` function on your deployed `Counter` contract. No token is transferred (amount is set to `0x0`), but the contract function is executed on the destination chain.

**What Happens:**
- **Token:** Set as Ether (address: `0x0000000000000000000000000000000000000000`)
- **Amount:** `0x0` (no token transfer)
- **Source Network:** Cardona (network ID: 1)
- **Destination Network:** Silicon Sepolia (network ID: 0)
- **Call Address:** Address of the deployed `Counter` contract on Silicon Sepolia
- **Fallback Address:** User's address (to receive funds if the transaction fails)
- **Call Data:** Encoded ABI for the `increment("0x4")` function call
- **Force Update:** Global exit root update enabled

**Code Walkthrough: `bridge_and_call.js`**
```javascript
const { getLxLyClient, tokens, configuration, from } = require('./utils/utils_lxly');
const { Counter } = require("../../ABIs/Counter");

const execute = async () => {
    const client = await getLxLyClient();

    // Set token as Ether.
    const token = "0x0000000000000000000000000000000000000000";
    // No token is being bridged.
    const amount = "0x0";
    // Bridging from Cardona.
    const sourceNetwork = 1;
    // Sending to Silicon Sepolia.
    const destinationNetwork = 0;
    // Address of the deployed Counter contract on Silicon Sepolia.
    const callAddress = "0x43854F7B2a37fA13182BBEA76E50FC8e3D298CF1";
    // Fallback address in case the transaction fails.
    const fallbackAddress = from;
    // Force update global exit root.
    const forceUpdateGlobalExitRoot = true;
    // Get the contract instance for the Counter contract.
    const callContract = client.contract(Counter, callAddress, destinationNetwork);
    // Prepare call data for the "increment" function.
    const callData = await callContract.encodeAbi("increment", "0x4");  
    
    let result;
    // Execute the bridgeAndCall function.
    if (client.client.network === "testnet") {
        console.log("testnet");
        result = await client.bridgeExtensions[sourceNetwork].bridgeAndCall(
            token,
            amount,
            destinationNetwork,
            callAddress,
            fallbackAddress,
            callData,
            forceUpdateGlobalExitRoot,
            permitData = "0x0"  // Optional parameter.
        );
    } else {
        console.log("mainnet");
        result = await client.bridgeExtensions[sourceNetwork].bridgeAndCall(
            token,
            amount,
            destinationNetwork,
            callAddress,
            fallbackAddress,
            callData,
            forceUpdateGlobalExitRoot,
        );
    }

    console.log("result", result);
    const txHash = await result.getTransactionHash();
    console.log("txHash", txHash);
    const receipt = await result.getReceipt();
    console.log("receipt", receipt); 
}

execute()
  .then(() => {})
  .catch(err => {
    console.error("err", err);
  })
  .finally(_ => {
    process.exit(0);
  });
```

How to Run:
```bash
node bridge_and_call.js
```

## Step 4: Claim the Bridge and Call Transaction

After executing the `bridgeAndCall` operation, claim the bridged transaction on the destination chain using the `claim_bridge_and_call.js` script.

### What Happens:
- **Claim Asset:**  
  The asset is claimed using the `claimAsset` API.
- **Build Payload & Claim Message:**  
  The script builds the payload using `buildPayloadForClaim` and then calls the `claimMessage` API to process the associated call message.

### Code Walkthrough: `claim_bridge_and_call.js`
```javascript
// Example code snippet from claim_bridge_and_call.js

const { getLxLyClient, tokens, configuration, from, to } = require('./utils/utils_lxly');

const execute = async () => {
    const client = await getLxLyClient();
    // Bridge transaction hash from the source chain.
    const bridgeTransactionHash = "0x7f1f06f24b3be354776bd351c11dd64f77e3429fb70a75bac3437a29eb6bbe9a"; 
    // Source network: Cardona (network ID: 1)
    const sourceNetworkId = 1;
    // Destination network: Silicon Sepolia (network ID: 0)
    const destinationNetworkId = 0;
    
    // Get API instance for Ether token on the destination network.
    const token = client.erc20(tokens[destinationNetworkId].ether, destinationNetworkId);
    
    // Claim the asset using the claimAsset API.
    const resultToken = await token.claimAsset(bridgeTransactionHash, sourceNetworkId, { returnTransaction: false });
    console.log("resultToken", resultToken);
    const txHashToken = await resultToken.getTransactionHash();
    console.log("txHashToken", txHashToken);
    const receiptToken = await resultToken.getReceipt();
    console.log("receiptToken", receiptToken);
    
    // Build the payload for claim.
    const resultMessage = await client.bridgeUtil.buildPayloadForClaim(bridgeTransactionHash, sourceNetworkId, bridgeIndex = 1)
        .then((payload) => {
            console.log("payload", payload);
            return client.bridges[destinationNetworkId].claimMessage(
                payload.smtProof,
                payload.smtProofRollup,
                BigInt(payload.globalIndex),
                payload.mainnetExitRoot,
                payload.rollupExitRoot,
                payload.originNetwork,
                payload.originTokenAddress,
                payload.destinationNetwork,
                payload.destinationAddress,
                payload.amount,
                payload.metadata
            );
        });
    console.log("resultMessage", resultMessage);

    const txHashMessage = await resultMessage.getTransactionHash();
    console.log("txHashMessage", txHashMessage);
    const receiptMessage = await resultMessage.getReceipt();
    console.log("receipt", receiptMessage);
}

execute()
  .then(() => {})
  .catch(err => {
    console.error("err", err);
  })
  .finally(_ => {
    process.exit(0);
  });
```

## Step 5: Verify the Transaction Status

Use Postman or cURL to query the Bridge API and verify the status of your transactions. This step ensures that:

- The `bridgeAndCall` transaction executed successfully.
- The claim processes have updated the transaction status to **CLAIMED**.

For example, query the Testnet endpoint:

```plaintext
https://api-gateway.polygon.technology/api/v3/transactions/testnet?userAddress={yourWalletAddress}
```
Replace {yourWalletAddress} with your actual wallet address, and include your API key in the request header.

How to Run :
```bash
node claim_bridge_and_call.js
```
