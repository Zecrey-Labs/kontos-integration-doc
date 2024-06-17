# Guide to Connect to Kontos Wallet

## Introduction

This guide provides essential information and instructions for DApp developers on how to integrate their applications with the Kontos wallet using WalletConnect. While the integration largely follows the standard WalletConnect 2.0 protocol, there are specific considerations and unique logic required for working with the Kontos wallet. This guide highlights these key differences and provides the necessary steps to ensure a smooth and secure integration process.

## Prerequisites

Before you begin, ensure you have the following:

- **Integration with WalletConnect protocol**: This is essential for connecting to the Kontos wallet. Make sure you have WalletConnect protocol properly integrated into your DApp. You can find the integration guide for WalletConnect [here](https://docs.walletconnect.com/).
- **DApp should support contract wallets and correctly handle accounts based on the chain and address combinations provided by the namespace (\*important)**: Unlike EOA wallets, contract wallets might have different addresses across different chains, and your DApp needs to filter the correct account based on the current chain. You can find something helpful [here](https://docs.walletconnect.com/api/sign/smart-contract-wallet-usage).

## Supported Session Requests

Kontos wallet supports the following session requests:

- `PERSONAL_SIGN`: "personal_sign"
- `WALLET_ADD_ETHEREUM_CHAIN`: "wallet_addEthereumChain"
- `WALLET_SWITCH_ETHEREUM_CHAIN`: "wallet_switchEthereumChain"
- `ETH_SIGN`: "eth_sign"
- `ETH_SIGN_TRANSACTION`: "eth_signTransaction"
- `ETH_SIGN_TYPED_DATA`: "eth_signTypedData"
- `ETH_SIGN_TYPED_DATA_V3`: "eth_signTypedData_v3"
- `ETH_SIGN_TYPED_DATA_V4`: "eth_signTypedData_v4"
- `ETH_SEND_TRANSACTION`: "eth_sendTransaction"

### Note on Chain Management

If you have implemented chain and address mapping management as per the prerequisites, you do not need to handle `wallet_addEthereumChain` and `wallet_switchEthereumChain`. Kontos Wallet is chain-agnostic during usage, meaning these session requests are unnecessary for the wallet's operation.

## Recommended Kontos Wallet Invocation Process

To provide a seamless user experience, it's recommended to include a dedicated Kontos connection button (with an icon) in your wallet selection panel. When the user clicks this button, the connection process will be initiated by splitting the URI obtained from WalletConnect into four parameters and appending them to the Kontos wallet URL as query parameters. And it's recommended to close the popup once it succeeds. Below is the code example with comments explaining each step:

```javascript
/**
 * Formats the WalletConnect URI into a query string compatible with Kontos Wallet.
 *
 * @param {string} link - The WalletConnect URI to be formatted.
 * @returns {string} - The formatted query string to append to the Kontos wallet URL.
 * @throws {Error} - Throws an error if the URI format is invalid.
 */
function formatWCLink(link) {
    // Check if the link starts with 'wc:'
    if (!link.startsWith("wc:")) {
        throw new Error("Invalid link format");
    }

    // Split the 'wc:' prefix and the parameters part
    const [wcPart, queryPart] = link.split("?");
    if (!queryPart) {
        throw new Error("Invalid link format");
    }

    // Construct the new query string
    const formattedQueryString = `?wc=${wcPart.slice(3)}&${queryPart}`;

    return formattedQueryString;
}

/**
 * Initiates the connection process with Kontos Wallet.
 */
async function connect() {
    ...
    // Connect to WalletConnect and obtain the URI
    const { uri, ... } = await client.connect({
        pairingTopic: pairing?.topic,
        requiredNamespaces,
        optionalNamespaces,
    });

    // If the URI is obtained, format it and open the Kontos Wallet in a popup window
    if (uri) {
        const queryLink = formatWCLink(uri);
        const url = "https://wallet.kontos.io/home" + queryLink;
        const windowName = "popupKontosWallet";
        const windowFeatures = "width=375,height=667";
        window.kontosPopup = window.open(url, windowName, windowFeatures);
        return;
    }
    ...
}

/**
 * After the session successfully connects, close the popup and continue.
 */
async function onSessionConnected() {
    ...
    // Close the popup window after the request is processed
    if (window.kontosPopup) {
        window.kontosPopup.close();
    }
    ...
}
```

The same goes for requests:

```javascript
/**
 * Sends a session request and opens Kontos Wallet in a popup window.
 */
async function sendSessionRequest(request) {
    ...
    // Open the Kontos Wallet popup
    const url = "https://wallet.kontos.io/home";
    const windowName = "popupKontosWallet";
    const windowFeatures = "width=375,height=667";
    window.kontosPopup = window.open(url, windowName, windowFeatures);

    // Send the session request
    const result = await client.request(request);

    // Handle the result of the session request
    console.log("Request result:", result);

    // Close the popup window after the request is processed
    if (window.kontosPopup) {
        window.kontosPopup.close();
    }

    return result;
}

```

### Note on Closing Popup

To enhance user experience, ensure that the popup window is closed after the connection or session request is successfully processed. By doing so, users can seamlessly return to your DApp after performing necessary actions in the Kontos Wallet.

### Special Note on Transaction Handling with Kontos Wallet

Unlike typical EOA (Externally Owned Accounts) wallets, Kontos Wallet handles user transactions through a broker role on the Kontos chain for cross-chain processing. This means that the transaction is not executed directly by the user but is managed by the broker. During this process, there will be a waiting period, and the transaction hash will not be immediately available as a return value.

Currently, Kontos returns an erroneous transaction hash. Therefore, it is recommended that your DApp should avoid analyzing and processing the returned transaction hash to display it to the user. This is an ideal practice to prevent confusion, although it is not strictly required.

### Important Note on Signature Verification

It is important to note that signature verification for contract wallets differs from EOA (Externally Owned Accounts) addresses. Currently, Kontos supports signature verification via the ERC-1271 standard. This means that verification is done by calling the `isValidSignature` method on the contract wallet (AA address). For more details on how this standard works, please refer to [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271).
