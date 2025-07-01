# Connecting to Shardeum: A Developer's Guide to JSON-RPC

Welcome, developers! This tutorial will guide you through connecting to the Shardeum network using JSON-RPC, performing essential operations, and handling potential issues. We'll leverage the powerful `ethers.js` library for wallet interactions and direct `fetch` calls for raw JSON-RPC requests, using a practical, step-by-step example.

**Why this guide?** While official documentation provides valuable information, sometimes a hands-on, clearly explained example makes all the difference. This tutorial aims to be that difference, breaking down the process into easily digestible parts.

## 1. Introduction to Shardeum and JSON-RPC

Shardeum is an EVM-compatible blockchain designed for scalability. Like other Ethereum-based networks, it exposes its functionality via a JSON-RPC (Remote Procedure Call) interface. This allows applications to interact with the blockchain, query data, send transactions, and much more.

JSON-RPC is a stateless, lightweight, remote procedure call (RPC) protocol. It uses JSON as its data format. When you interact with a blockchain node, you're essentially sending JSON-RPC requests over HTTP.

## 2. Prerequisites

Before we dive into the code, ensure you have the following installed:

*   **Node.js:** A JavaScript runtime environment. You can download it from [nodejs.org](https://nodejs.org/). We recommend an LTS (Long Term Support) version.
*   **npm (Node Package Manager):** This comes bundled with Node.js.
*   **MetaMask (or similar browser wallet):** Our example code is designed for a browser environment where a wallet like MetaMask injects `window.ethereum`. This is essential for user authentication and transaction signing.
*   **Basic JavaScript knowledge:** Familiarity with `async/await`, classes, and ES6 modules will be helpful.

### Installing Dependencies

Our tutorial will primarily use the `ethers.js` library, which simplifies interacting with Ethereum-compatible blockchains.

First, create a new project directory and initialize npm:

```bash
mkdir shardeum-rpc-tutorial
cd shardeum-rpc-tutorial
npm init -y
```

Next, install `ethers.js`:

```bash
npm install ethers
```

## 3. The `ShardeumAPI` Class: Your Gateway to Shardeum

We'll be working with a `ShardeumAPI` class that encapsulates all our interactions with the Shardeum network. This class demonstrates two primary ways to interact with Shardeum's JSON-RPC:

1.  **Using `ethers.js` `BrowserProvider` and `Signer`:** This is the high-level, recommended way for wallet interactions (e.g., connecting a wallet, sending transactions). `ethers.js` handles the underlying JSON-RPC calls for you.
2.  **Direct `fetch` calls:** For simple read-only queries (e.g., getting balance or gas price), you can make raw JSON-RPC POST requests using the `fetch` API. This demonstrates the raw protocol.

Here's the full `ShardeumAPI` class we'll be breaking down:

```javascript
import { ethers } from 'ethers';

class ShardeumAPI {
  constructor() {
    // Shardeum Liberty 2.X Testnet JSON-RPC URL
    // Replace with mainnet URL ('https://mainnet.shardeum.org') when needed
    this.rpcUrl = 'https://api-testnet.shardeum.org'; 
    this.provider = null; // Ethers.js provider (for interacting with the blockchain)
    this.signer = null;   // Ethers.js signer (for signing transactions with the user's wallet)
    this.txHistory = {};  // Simple in-memory transaction history
  }

  /**
   * Initializes the connection to the user's wallet (e.g., MetaMask).
   * This is the "authentication" step.
   * @returns {Promise<boolean>} True if initialized successfully, throws error otherwise.
   */
  async initialize() {
    try {
      if (typeof window.ethereum !== 'undefined') {
        // Use BrowserProvider for injected web3 wallets like MetaMask
        this.provider = new ethers.BrowserProvider(window.ethereum);
        
        // Request access to the user's accounts. This will prompt MetaMask.
        await this.provider.send("eth_requestAccounts", []);
        
        // Get the signer, which represents the connected user's account
        this.signer = await this.provider.getSigner();
        console.log("Wallet connected successfully!");
        return true;
      } else {
        throw new Error('Wallet not found. Please install MetaMask or another wallet extension.');
      }
    } catch (error) {
      console.error("Failed to connect wallet:", error);
      throw new Error(`Wallet connection failed: ${error.message}`);
    }
  }

  /**
   * Gets the currently connected wallet address.
   * @returns {Promise<string>} The connected wallet address.
   */
  async getCurrentAddress() {
    if (!this.signer) throw new Error('Wallet not connected. Call initialize() first.');
    return await this.signer.getAddress();
  }

  /**
   * Fetches the balance of a given address using a raw JSON-RPC call.
   * @param {string} address The address to query.
   * @returns {Promise<string>} The balance in SHM (formatted from Wei).
   */
  async getBalanceOf(address) {
    try {
      const res = await fetch(this.rpcUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          method: 'eth_getBalance',
          params: [address, 'latest'], // 'latest' refers to the latest block
          id: 1, // JSON-RPC request ID
          jsonrpc: '2.0' // JSON-RPC version
        })
      });
      const data = await res.json();

      // Check for RPC errors in the response
      if (data.error) {
        throw new Error(`RPC Error (${data.error.code}): ${data.error.message}`);
      }
      
      // formatEther converts Wei (smallest unit) to Ether/SHM
      return ethers.formatEther(data.result);
    } catch (error) {
      console.error(`Error getting balance for ${address}:`, error);
      throw new Error(`Failed to get balance: ${error.message}`);
    }
  }

  /**
   * Fetches the balance of the currently connected wallet.
   * @returns {Promise<string>} The balance in SHM.
   */
  async getMyBalance() {
    const myAddress = await this.getCurrentAddress();
    return await this.getBalanceOf(myAddress);
  }

  /**
   * Sends SHM from the connected wallet to a specified address.
   * This uses the Ethers.js signer for transaction creation and signing.
   * @param {string} toAddress The recipient address.
   * @param {number} amount The amount of SHM to send (as a number).
   * @returns {Promise<string>} The transaction hash.
   */
  async sendSHM(toAddress, amount) {
    if (!this.signer) throw new Error('Wallet not connected. Call initialize() first.');
    
    try {
      // Convert SHM amount (e.g., 0.1) to Wei (smallest unit)
      const value = ethers.parseEther(amount.toString());
      
      // Get current gas price via raw JSON-RPC
      const gasPrice = await this.getGasPrice(); 

      // Create and send the transaction using the signer
      // MetaMask will prompt the user to confirm this transaction
      const tx = await this.signer.sendTransaction({
        to: toAddress,
        value, // Amount in Wei
        gasPrice // Gas price in Wei (bigint)
      });

      console.log("Transaction sent! Hash:", tx.hash);
      
      // Store transaction history (simple in-memory example)
      const sender = await this.getCurrentAddress();
      if (!this.txHistory[sender]) this.txHistory[sender] = [];
      this.txHistory[sender].push({
        to: toAddress,
        amount,
        hash: tx.hash,
        timestamp: Date.now()
      });

      return tx.hash;
    } catch (error) {
      console.error("Error sending SHM:", error);
      // Ethers.js errors often have a 'code' property for specific issues
      if (error.code === 'ACTION_REJECTED') {
        throw new Error("Transaction rejected by user.");
      }
      throw new Error(`Failed to send SHM: ${error.message}`);
    }
  }

  /**
   * Fetches the current gas price using a raw JSON-RPC call.
   * @returns {Promise<string>} The gas price in Wei (hex string).
   */
  async getGasPrice() {
    try {
      const res = await fetch(this.rpcUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          method: 'eth_gasPrice',
          id: 1,
          jsonrpc: '2.0'
        })
      });
      const data = await res.json();
      
      if (data.error) {
        throw new Error(`RPC Error (${data.error.code}): ${data.error.message}`);
      }
      
      return data.result; // Returns hex string of gas price in Wei
    } catch (error) {
      console.error("Error getting gas price:", error);
      throw new Error(`Failed to get gas price: ${error.message}`);
    }
  }

  /**
   * Fetches details of a transaction by its hash using a raw JSON-RPC call.
   * @param {string} txHash The transaction hash.
   * @returns {Promise<object>} The transaction details object.
   */
  async getTransactionByHash(txHash) {
    try {
      const res = await fetch(this.rpcUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          method: 'eth_getTransactionByHash',
          params: [txHash],
          id: 1,
          jsonrpc: '2.0'
        })
      });
      const data = await res.json();
      
      if (data.error) {
        throw new Error(`RPC Error (${data.error.code}): ${data.error.message}`);
      }

      return data.result;
    } catch (error) {
      console.error(`Error getting transaction by hash ${txHash}:`, error);
      throw new Error(`Failed to get transaction: ${error.message}`);
    }
  }

  /**
   * A wrapper around sendSHM for a specific use case (e.g., donations).
   * Demonstrates how to build on top of core functionalities.
   */
  async donate(toDonationAddress, amount) {
    console.log(`Attempting to donate ${amount} SHM to ${toDonationAddress}...`);
    return await this.sendSHM(toDonationAddress, amount);
  }

  /**
   * Helper to format Wei (from blockchain) into SHM (human-readable).
   * @param {BigNumberish} wei The amount in Wei.
   * @returns {string} The amount in SHM.
   */
  formatSHM(wei) {
    return ethers.formatEther(wei);
  }

  /**
   * Helper to parse SHM (human-readable) into Wei (for transactions).
   * @param {number | string} shm The amount in SHM.
   * @returns {bigint} The amount in Wei.
   */
  parseSHM(shm) {
    return ethers.parseEther(shm.toString());
  }
}

// Export a singleton instance of the API
export default new ShardeumAPI();
```

## 4. Step-by-Step Explanation of `ShardeumAPI`

Let's break down each part of the `ShardeumAPI` class.

### 4.1. `constructor()`

```javascript
class ShardeumAPI {
  constructor() {
    this.rpcUrl = 'https://api-testnet.shardeum.org'; // Shardeum Liberty 2.X Testnet
    this.provider = null;
    this.signer = null;
    this.txHistory = {};
  }
  // ... rest of the class
}
```

*   **`this.rpcUrl`**: This is the crucial endpoint for interacting with the Shardeum network. For the Shardeum Liberty 2.X Testnet, it's `https://api-testnet.shardeum.org`. If you were connecting to the mainnet (once launched), you would change this URL accordingly.
*   **`this.provider`**: An `ethers.js` `Provider` object. It's an abstraction that connects to the blockchain and allows you to query data (e.g., `eth_getBalance`, `eth_gasPrice`).
*   **`this.signer`**: An `ethers.js` `Signer` object. It represents an Ethereum account and can sign transactions and messages. In a browser DApp, this typically comes from the connected MetaMask wallet.
*   **`this.txHistory`**: A simple JavaScript object to store transaction details in memory for demonstration purposes.

### 4.2. Connecting to Shardeum and Authentication (`initialize()`)

This is the very first step your DApp should take to connect a user's wallet.

```javascript
  async initialize() {
    try {
      if (typeof window.ethereum !== 'undefined') {
        // 1. Create an Ethers.js BrowserProvider
        this.provider = new ethers.BrowserProvider(window.ethereum);
        
        // 2. Request user's accounts (authentication)
        await this.provider.send("eth_requestAccounts", []);
        
        // 3. Get the Signer object
        this.signer = await this.provider.getSigner();
        console.log("Wallet connected successfully!");
        return true;
      } else {
        throw new Error('Wallet not found. Please install MetaMask or another wallet extension.');
      }
    } catch (error) {
      console.error("Failed to connect wallet:", error);
      throw new Error(`Wallet connection failed: ${error.message}`);
    }
  }
```

**Explanation:**

1.  **`typeof window.ethereum !== 'undefined'`**: This checks if a web3-compatible wallet (like MetaMask) is injected into the browser window. MetaMask makes its API available via `window.ethereum`.
2.  **`new ethers.BrowserProvider(window.ethereum)`**: If `window.ethereum` exists, we use it to create an `ethers.js` `BrowserProvider`. This provider abstracts away the complexities of interacting with the injected wallet's API.
3.  **`this.provider.send("eth_requestAccounts", [])`**: This is the crucial authentication step. It's an underlying JSON-RPC method call that `ethers.js` wraps. When executed:
    *   MetaMask (or the connected wallet) will pop up, asking the user to grant your DApp permission to access their accounts.
    *   If the user approves, `eth_requestAccounts` returns an array of account addresses.
    *   If the user rejects, an error is thrown (which our `try...catch` block handles).
4.  **`this.signer = await this.provider.getSigner()`**: Once accounts are granted, we get a `Signer` object from the provider. This `signer` represents the *currently selected* account in the user's wallet and is capable of signing messages and sending transactions.

**Connection Flow:**

```mermaid
graph TD
    A[Start Application] --> B{Check for window.ethereum};
    B -- No --> C[Display "Install Wallet" Error];
    B -- Yes --> D[Create ethers.BrowserProvider(window.ethereum)];
    D --> E{Call provider.send("eth_requestAccounts", [])};
    E -- User Rejects --> F[Display "Wallet Connection Rejected" Error];
    E -- User Accepts --> G[Get Signer from Provider];
    G --> H[Wallet Connected Successfully];
    H --> I[Application Ready for Transactions];
```

### 4.3. Performing a Sample Request: `eth_getBalance`

The `eth_getBalance` method is a fundamental JSON-RPC call to query an account's balance. Our `getBalanceOf` method demonstrates how to make a raw `fetch` request for this.

```javascript
  async getBalanceOf(address) {
    try {
      const res = await fetch(this.rpcUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          method: 'eth_getBalance',
          params: [address, 'latest'], // Query balance at the latest block
          id: 1,
          jsonrpc: '2.0'
        })
      });
      const data = await res.json();

      if (data.error) {
        throw new Error(`RPC Error (${data.error.code}): ${data.error.message}`);
      }
      
      return ethers.formatEther(data.result); // Convert Wei to SHM
    } catch (error) {
      console.error(`Error getting balance for ${address}:`, error);
      throw new Error(`Failed to get balance: ${error.message}`);
    }
  }
```

**Explanation:**

1.  **`fetch(this.rpcUrl, { ... })`**: We use the standard `fetch` API to send an HTTP POST request to our Shardeum RPC endpoint.
2.  **`method: 'POST'`**: All JSON-RPC requests are typically POST requests.
3.  **`headers: { 'Content-Type': 'application/json' }`**: Specifies that the request body is JSON.
4.  **`body: JSON.stringify({...})`**: This is the JSON-RPC request payload.
    *   **`method: 'eth_getBalance'`**: The name of the JSON-RPC method we want to call.
    *   **`params: [address, 'latest']`**: An array of parameters for the method. `eth_getBalance` expects the account address and a block tag (`'latest'` for the current block).
    *   **`id: 1`**: A unique identifier for the request. The response will include the same ID.
    *   **`jsonrpc: '2.0'`**: The version of the JSON-RPC protocol.
5.  **`const data = await res.json()`**: Parses the JSON response from the RPC server.
6.  **`if (data.error)`**: **Error Handling!** We check if the response contains an `error` object. This is how RPC errors are returned.
7.  **`ethers.formatEther(data.result)`**: The `data.result` for `eth_getBalance` is the balance in **Wei** (the smallest unit of Ether/SHM) as a hexadecimal string. `ethers.js`'s `formatEther` utility converts this Wei amount into a human-readable decimal number representing SHM.

### 4.4. Sending Transactions: `sendSHM`

Sending a transaction, especially one that transfers value (like SHM), requires the user's wallet to sign it. This is where the `signer` object from `ethers.js` shines.

```javascript
  async sendSHM(toAddress, amount) {
    if (!this.signer) throw new Error('Wallet not connected. Call initialize() first.');
    
    try {
      const value = ethers.parseEther(amount.toString()); // Convert SHM to Wei
      const gasPrice = await this.getGasPrice(); // Get current gas price

      const tx = await this.signer.sendTransaction({
        to: toAddress,
        value, 
        gasPrice 
      });

      console.log("Transaction sent! Hash:", tx.hash);
      
      // ... (transaction history logging)
      return tx.hash;
    } catch (error) {
      console.error("Error sending SHM:", error);
      if (error.code === 'ACTION_REJECTED') {
        throw new Error("Transaction rejected by user.");
      }
      throw new Error(`Failed to send SHM: ${error.message}`);
    }
  }
```

**Explanation:**

1.  **`if (!this.signer)`**: Ensures the wallet is connected before attempting to send a transaction.
2.  **`const value = ethers.parseEther(amount.toString());`**: Converts the human-readable SHM amount (e.g., `0.1`) into its equivalent in Wei (a large `bigint`) which the blockchain understands.
3.  **`const gasPrice = await this.getGasPrice();`**: We call our `getGasPrice` helper (which makes a raw `eth_gasPrice` RPC call) to get the current network gas price. This is crucial for successful transactions.
4.  **`await this.signer.sendTransaction({...})`**: This is the core method for sending value transfers or interacting with smart contracts.
    *   **`to: toAddress`**: The recipient's address.
    *   **`value: value`**: The amount of SHM (in Wei) to send.
    *   **`gasPrice: gasPrice`**: The price per unit of gas, in Wei.
    *   When this line executes, MetaMask will typically pop up a confirmation dialog for the user to review and approve the transaction.
5.  **`const tx = await ...`**: If the user approves, `sendTransaction` will return a `TransactionResponse` object containing details like the transaction hash (`tx.hash`). This hash can be used to track the transaction's status on the blockchain explorer.

### 4.5. Other Helper Functions

*   **`getCurrentAddress()`**: A simple utility to get the address of the connected wallet using `this.signer.getAddress()`.
*   **`getMyBalance()`**: Combines `getCurrentAddress` and `getBalanceOf` to easily get the connected user's balance.
*   **`getGasPrice()`**: Similar to `getBalanceOf`, this performs a raw JSON-RPC call for `eth_gasPrice`. The result is typically a hexadecimal string representing the gas price in Wei.
*   **`getTransactionByHash(txHash)`**: Another raw JSON-RPC call for `eth_getTransactionByHash`, useful for fetching details of a previously sent transaction.
*   **`donate(toDonationAddress, amount)`**: A wrapper function showing how you might build higher-level abstractions on top of the core `sendSHM` functionality.
*   **`formatSHM(wei)` / `parseSHM(shm)`**: Essential `ethers.js` utilities for converting between Wei (blockchain native unit) and SHM (human-readable unit).

## 5. Handling Errors Properly

Robust error handling is crucial for any production-ready application. Our `ShardeumAPI` class incorporates `try...catch` blocks around all network and wallet interactions.

### 5.1. Common Error Scenarios and Handling

1.  **Wallet Not Found (`initialize()`):**
    *   **Error:** `Wallet not found. Please install MetaMask or another wallet extension.`
    *   **Reason:** `window.ethereum` is `undefined`.
    *   **Handling:** The `if (typeof window.ethereum !== 'undefined')` check catches this. Your UI should prompt the user to install a wallet.

2.  **Wallet Connection Rejected (`initialize()`):**
    *   **Error:** `Wallet connection failed: User rejected the request.` (or similar, `Error: User rejected the request.`)
    *   **Reason:** The user clicked "Reject" or closed the wallet's connection prompt.
    *   **Handling:** The `try...catch` around `await this.provider.send("eth_requestAccounts", [])` will catch this. You can display a message indicating the user rejected the connection.

3.  **Transaction Rejected by User (`sendSHM()`):**
    *   **Error:** `Transaction rejected by user.` (`ethers` often provides a `code: 'ACTION_REJECTED'` for this).
    *   **Reason:** The user declined to sign the transaction in their wallet.
    *   **Handling:** Specific `if (error.code === 'ACTION_REJECTED')` check allows you to give precise feedback to the user.

4.  **RPC Errors (e.g., `getBalanceOf`, `getGasPrice`):**
    *   **Error:** The RPC response itself contains an `error` object (e.g., `{"jsonrpc":"2.0","id":1,"error":{"code":-32000,"message":"invalid address"}}`).
    *   **Reason:** Invalid parameters, insufficient gas, node issues, etc.
    *   **Handling:** We explicitly check `if (data.error)` after parsing the JSON response. This allows you to extract the error `code` and `message` from the RPC server and display it to the user.

5.  **Network Errors (e.g., `fetch` failures):**
    *   **Error:** `TypeError: Failed to fetch`, network timeout, etc.
    *   **Reason:** RPC endpoint is down, user has no internet connection, firewall issues.
    *   **Handling:** The outer `try...catch` block around `fetch` will catch these.

By strategically placing `try...catch` blocks and inspecting error objects, you can provide meaningful feedback to your users and make your application more resilient.

## 6. Putting It All Together: A Simple Usage Example

To use your `ShardeumAPI` class, create an `index.js` (or similar) file in your project:

```javascript
// index.js
import shardeumAPI from './ShardeumAPI.js'; // Assuming ShardeumAPI.js is in the same directory

async function runExample() {
  console.log("--- Shardeum JSON-RPC Tutorial ---");

  try {
    // 1. Initialize the API (connect wallet)
    console.log("Attempting to connect wallet...");
    await shardeumAPI.initialize();
    console.log("Wallet connection successful!");

    // 2. Get current connected address
    const myAddress = await shardeumAPI.getCurrentAddress();
    console.log(`Connected Address: ${myAddress}`);

    // 3. Get my balance
    const myBalance = await shardeumAPI.getMyBalance();
    console.log(`My Balance: ${myBalance} SHM`);

    // 4. Get current gas price
    const gasPriceWei = await shardeumAPI.getGasPrice();
    console.log(`Current Gas Price: ${gasPriceWei} Wei`);
    console.log(`Current Gas Price (Gwei): ${shardeumAPI.formatSHM(gasPriceWei * 1e9)} Gwei`); // Convert to Gwei for better readability

    // --- Send SHM (requires user interaction in MetaMask) ---
    // IMPORTANT: Replace with a *testnet* address and small amount for testing!
    const recipientAddress = '0xYourTestRecipientAddressHere'; // <<< REPLACE THIS!
    const sendAmount = 0.001; // <<< Small amount for testing

    if (recipientAddress !== '0xYourTestRecipientAddressHere') {
      console.log(`Attempting to send ${sendAmount} SHM to ${recipientAddress}...`);
      const txHash = await shardeumAPI.sendSHM(recipientAddress, sendAmount);
      console.log(`Transaction submitted! Hash: ${txHash}`);
      console.log(`You can view the transaction here: https://explorer-liberty20.shardeum.org/transaction/${txHash}`);

      // 5. Get transaction details by hash
      console.log("Fetching transaction details...");
      // A small delay might be needed for the transaction to be propagated and indexed
      await new Promise(resolve => setTimeout(resolve, 5000)); // Wait 5 seconds
      const txDetails = await shardeumAPI.getTransactionByHash(txHash);
      console.log("Transaction Details:", txDetails);
    } else {
      console.warn("\nSkipping SHM transfer: Please replace '0xYourTestRecipientAddressHere' with a real testnet address to enable this functionality.");
      console.warn("You can get testnet SHM from the Shardeum Faucet: https://faucet.liberty20.shardeum.org/");
    }

  } catch (error) {
    console.error("\n--- An error occurred! ---");
    console.error(error.message);
    // Log full error for debugging
    // console.error(error); 
  } finally {
    console.log("\n--- Example finished. ---");
  }
}

// Ensure this script runs only in a browser-like environment or polyfill 'fetch' and 'window.ethereum' for Node.js
// For a simple browser HTML, you'd include this script.
// For Node.js, you'd need globalThis.fetch and a mock for window.ethereum which is complex.
// This code is intended for a browser environment where window.ethereum is available.

// To run in a browser:
// 1. Create an HTML file (e.g., `index.html`):
/*
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Shardeum RPC Example</title>
</head>
<body>
    <h1>Shardeum RPC Example</h1>
    <p>Open your browser's developer console (F12) to see the output.</p>
    <button onclick="runExample()">Run Shardeum Example</button>
    <script type="module" src="./index.js"></script>
</body>
</html>
*/
// 2. Open index.html in a browser with MetaMask installed.
// 3. Click the "Run Shardeum Example" button.

// To run in a Node.js environment (for server-side interactions without a wallet):
// You would need a different setup for the provider/signer, typically using a private key
// or a custom RPC endpoint that doesn't rely on window.ethereum.
// The current `initialize()` method is specifically for browser-based DApps.
```

**How to Run This Example (Browser-based):**

1.  Save the `ShardeumAPI` class code as `ShardeumAPI.js`.
2.  Save the example usage code as `index.js`.
3.  Create an `index.html` file in the same directory (as shown in the comments above).
4.  Open `index.html` in your web browser.
5.  Open your browser's developer console (usually by pressing F12).
6.  Click the "Run Shardeum Example" button on the webpage.

You will see MetaMask pop up to request connection and then to confirm transactions. All the output will be in your browser's console.
