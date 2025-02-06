# Unified Bridge & lxly.js Quick Start Guide

This guide will help you get started with the **Unified Bridge** and the **lxly.js** library. It covers the transaction states during bridging, describes the utility file for client initialization, demonstrates how to bridge and claim assets, and explains how to track your transactions using the Transaction API.

---

## 1. Transaction States Overview

When bridging assets, a transaction goes through three key states:

- **BRIDGED:**  
  The transaction has been initiated on the source chain via the `bridgeAsset` API.

- **READY_TO_CLAIM:**  
  The asset is now available on the destination chain and is waiting to be claimed.

- **CLAIMED:**  
  The asset has been successfully claimed on the destination chain using the `claimAsset` API.

---

## 2. Utility File: `utils_lxly.js`

The `utils_lxly.js` file sets up your connection with the Lxly bridge. It includes the `getLxLyClient` function, which:

- **Initializes the LxLy client:**  
  Configures network details and logging.

- **Sets up Providers:**  
  Uses `HDWalletProvider` (or a similar provider) for each network with the required RPC endpoints, bridge contract addresses, and account details.

This file ensures that all network configurations are consistent and simplifies interacting with the bridge contracts.

### Example: `utils_lxly.js`
```javascript
const getLxLyClient = async (network = 'testnet') => {
  const lxLyClient = new LxLyClient();
  return await lxLyClient.init({
    log: true,
    network: network,
    providers: {
      0: {
        provider: new HDWalletProvider([config.user1.privateKey], config.configuration[0].rpc),
        configuration: {
          bridgeAddress: config.configuration[0].bridgeAddress,
          bridgeExtensionAddress: config.configuration[0].bridgeExtensionAddress,
          wrapperAddress: config.configuration[0].wrapperAddress,
          isEIP1559Supported: true
        },
        defaultConfig: {
          from: config.user1.address
        }
      },
      1: {
        provider: new HDWalletProvider([config.user1.privateKey], config.configuration[1].rpc),
        configuration: {
          bridgeAddress: config.configuration[1].bridgeAddress,
          bridgeExtensionAddress: config.configuration[1].bridgeExtensionAddress,
          isEIP1559Supported: false
        },
        defaultConfig: {
          from: config.user1.address
        }
      },
    }
  });
}
```
## 3. Bridging Assets (`bridge_asset.js`)

This file demonstrates how to initiate a cross-chain asset transfer.

- **sourceNetworkId:**  
  The network ID for the source chain (e.g., `0` for Sepolia).

- **destinationNetworkId:**  
  The network ID for the destination chain (e.g., `1` for Cardona).

- **bridgeAsset API:**  
  Invoked as `bridgeAsset(amount, userAddress, destinationNetwork)`, where `amount` is specified in the smallest unit (like wei for Ether).

### Example: `bridge_asset.js`
```javascript
const { getLxLyClient, tokens, configuration, from, to } = require('./utils/utils_lxly');

const execute = async () => {
    // Initialize the lxly client
    const client = await getLxLyClient();

    // Define source network ID (e.g., Sepolia)
    const sourceNetworkId = 0;
    // Get the API for the Ether token on the source network
    const token = client.erc20(tokens[sourceNetworkId].ether, sourceNetworkId);

    // Define destination network ID (e.g., Cardona)
    const destinationNetworkId = 1;
    // Bridge a specific amount of Ether (in wei)
    const result = await token.bridgeAsset("10000000000000000", to, destinationNetworkId);

    // Log the transaction hash and receipt
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
  .finally(() => {
    process.exit(0);
  });
```
 
## 4. Claiming Bridged Assets (`claim_asset.js`)

After the asset has been bridged, you must claim it on the destination chain.

- **sourceNetworkId:**  
  The network ID for the source chain where the asset was originally bridged.

- **destinationNetworkId:**  
  The network ID for the destination chain where the asset is now available.

- **claimAsset API:**  
  Invoked as `claimAsset(bridgeTransactionHash, sourceNetworkId, options)`, where `bridgeTransactionHash` is the hash returned from the `bridgeAsset` call.

### Example: `claim_asset.js`
```javascript
const execute = async () => {
    // The transaction hash from the bridgeAsset call on the source chain
    const bridgeTransactionHash = "0x1fc6858b20c75189a9fa8f3ae60c2a255cc3c41a058781f33daa57fc0f80b81a";
		
    // Initialize the lxly client
    const client = await getLxLyClient();
    
    // Define source network ID (e.g., Sepolia)
    const sourceNetworkId = 0;
    // Define destination network ID (e.g., Cardona)
    const destinationNetworkId = 1;
    // Get the API for the Ether token on the destination network
    const token = client.erc20(tokens[destinationNetworkId].ether, destinationNetworkId);
    
    // Claim the bridged asset using the claimAsset API
    const result = await token.claimAsset(bridgeTransactionHash, sourceNetworkId, { returnTransaction: false });
    console.log("result", result);
    
    // Log the transaction hash and receipt
    const txHash = await result.getTransactionHash();
    console.log("txHash", txHash);
    
    const receipt = await result.getReceipt();
    console.log("receipt", receipt);
}

execute()
  .then(() => {})
  .catch(err => {
    console.error("err", err);
  });
```

## Transaction API

The Transaction API provides detailed information on a bridge transaction associated with a user’s wallet. It includes real-time status updates, the token bridged, the amount transferred, and the source and destination chains. This is especially useful for building user interfaces that display transaction statuses.

### API Endpoints

- **Testnet:**  
  `https://api-gateway.polygon.technology/api/v3/transactions/testnet?userAddress={userAddress}`

- **Mainnet:**  
  `https://api-gateway.polygon.technology/api/v3/transactions/mainnet?userAddress={userAddress}`

> **Note:**  
> Replace `{userAddress}` with the wallet address associated with the cross-chain transaction.  
> **Attach your API Key in the header!**

### Example cURL Command

```bash
curl --location 'https://api-gateway.polygon.technology/api/v3/transactions/mainnet?userAddress={userAddress}' \
--header 'x-api-key: <your-api-key-here>'
```

## Final Notes

- **Transaction Flow:**
  - Start with a **BRIDGED** state when you call `bridgeAsset`.
  - The asset becomes **READY_TO_CLAIM** once available on the destination chain.
  - It transitions to **CLAIMED** after you use `claimAsset`.

- **Consistent Configuration:**  
  The utility file (`utils_lxly.js`) ensures that all network configurations and provider details remain consistent.

- **API Tracking:**  
  Use the Transaction API endpoints to monitor the status and details of your cross-chain transfers.
