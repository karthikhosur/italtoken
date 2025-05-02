# Creating a Stellar Custom Asset for ITAL Token

To create the ITAL token as a Stellar custom asset for your project, you'll need to follow these steps:

## 1. Setup Stellar Accounts

First, you need to establish the issuing and distribution accounts:

```javascript
const StellarSdk = require('stellar-sdk');

// Generate issuer account (in production, generate securely and store offline)
const issuerKeypair = StellarSdk.Keypair.random();
console.log('Issuer Public Key:', issuerKeypair.publicKey());
console.log('Issuer Secret Key:', issuerKeypair.secret()); // Store securely in production

// Generate distribution account
const distributionKeypair = StellarSdk.Keypair.random();
console.log('Distribution Public Key:', distributionKeypair.publicKey());
console.log('Distribution Secret Key:', distributionKeypair.secret()); // Store securely
```

## 2. Fund the Accounts

For testnet, you can use the Friendbot. For mainnet, you'll need actual XLM:

```javascript
const server = new StellarSdk.Server('https://horizon-testnet.stellar.org');

// Fund issuer account on testnet
async function fundIssuer() {
  try {
    const response = await fetch(
      `https://friendbot.stellar.org?addr=${issuerKeypair.publicKey()}`
    );
    const responseJSON = await response.json();
    console.log("Issuer account funded:", responseJSON);
  } catch (error) {
    console.error("Error funding issuer account:", error);
  }
}

// Fund distribution account on testnet
async function fundDistribution() {
  try {
    const response = await fetch(
      `https://friendbot.stellar.org?addr=${distributionKeypair.publicKey()}`
    );
    const responseJSON = await response.json();
    console.log("Distribution account funded:", responseJSON);
  } catch (error) {
    console.error("Error funding distribution account:", error);
  }
}

await fundIssuer();
await fundDistribution();
```

## 3. Configure the Issuing Account

Set up the issuing account with the proper flags for your asset:

```javascript
async function configureIssuer() {
  try {
    const issuerAccount = await server.loadAccount(issuerKeypair.publicKey());
    
    const transaction = new StellarSdk.TransactionBuilder(issuerAccount, {
      fee: StellarSdk.BASE_FEE,
      networkPassphrase: StellarSdk.Networks.TESTNET
    })
      // Set home domain for the asset
      .addOperation(StellarSdk.Operation.setOptions({
        homeDomain: 'cvmine.com' // Your platform's domain
      }))
      // Configure asset flags
      .addOperation(StellarSdk.Operation.setOptions({
        setFlags: StellarSdk.AuthRevocableFlag, // Enable revocation if needed
        clearFlags: StellarSdk.AuthRequiredFlag // Not requiring authorization
      }))
      .setTimeout(30)
      .build();
      
    transaction.sign(issuerKeypair);
    const result = await server.submitTransaction(transaction);
    console.log('Issuer account configured:', result.hash);
    return result;
  } catch (error) {
    console.error('Error configuring issuer account:', error);
    throw error;
  }
}

await configureIssuer();
```

## 4. Create the Trustline Between Distribution and Issuer

The distribution account needs to trust the ITAL asset:

```javascript
async function createDistributionTrustline() {
  try {
    // Define the ITAL asset
    const italAsset = new StellarSdk.Asset('ITAL', issuerKeypair.publicKey());
    
    // Load the distribution account
    const distributionAccount = await server.loadAccount(distributionKeypair.publicKey());
    
    // Create a trustline transaction
    const transaction = new StellarSdk.TransactionBuilder(distributionAccount, {
      fee: StellarSdk.BASE_FEE,
      networkPassphrase: StellarSdk.Networks.TESTNET
    })
      .addOperation(StellarSdk.Operation.changeTrust({
        asset: italAsset,
        limit: '6000000000000' // 6 trillion (your max supply)
      }))
      .setTimeout(30)
      .build();
      
    transaction.sign(distributionKeypair);
    const result = await server.submitTransaction(transaction);
    console.log('Distribution trustline created:', result.hash);
    return result;
  } catch (error) {
    console.error('Error creating distribution trustline:', error);
    throw error;
  }
}

await createDistributionTrustline();
```

## 5. Issue Initial ITAL Tokens to Distribution Account

Now issue a portion of your tokens to the distribution account:

```javascript
async function issueInitialTokens() {
  try {
    // Define the ITAL asset
    const italAsset = new StellarSdk.Asset('ITAL', issuerKeypair.publicKey());
    
    // Load the issuer account
    const issuerAccount = await server.loadAccount(issuerKeypair.publicKey());
    
    // Create payment transaction
    const transaction = new StellarSdk.TransactionBuilder(issuerAccount, {
      fee: StellarSdk.BASE_FEE,
      networkPassphrase: StellarSdk.Networks.TESTNET
    })
      .addOperation(StellarSdk.Operation.payment({
        destination: distributionKeypair.publicKey(),
        asset: italAsset,
        amount: '200000000000' // 200 billion (your initial supply)
      }))
      .setTimeout(30)
      .build();
      
    transaction.sign(issuerKeypair);
    const result = await server.submitTransaction(transaction);
    console.log('Initial tokens issued:', result.hash);
    return result;
  } catch (error) {
    console.error('Error issuing initial tokens:', error);
    throw error;
  }
}

await issueInitialTokens();
```

## 6. Lock the Issuing Account (Optional)

If you want to make your token supply truly fixed, you can lock the issuing account:

```javascript
async function lockIssuerAccount() {
  try {
    // Load the issuer account
    const issuerAccount = await server.loadAccount(issuerKeypair.publicKey());
    
    // Create a transaction to lock the account
    const transaction = new StellarSdk.TransactionBuilder(issuerAccount, {
      fee: StellarSdk.BASE_FEE,
      networkPassphrase: StellarSdk.Networks.TESTNET
    })
      .addOperation(StellarSdk.Operation.setOptions({
        masterWeight: 0, // Remove ability to sign transactions
        lowThreshold: 1,
        medThreshold: 1,
        highThreshold: 1
        // No signers added, making the account effectively locked
      }))
      .setTimeout(30)
      .build();
      
    transaction.sign(issuerKeypair);
    const result = await server.submitTransaction(transaction);
    console.log('Issuer account locked:', result.hash);
    return result;
  } catch (error) {
    console.error('Error locking issuer account:', error);
    throw error;
  }
}

// Carefully consider before running this - it's irreversible!
// await lockIssuerAccount();
```

## 7. Create a Multi-Signature Setup for the Distribution Account

For production security, implement multi-signature requirements:

```javascript
async function setupMultiSig() {
  try {
    // Generate additional signing keys for security
    const signer1 = StellarSdk.Keypair.random();
    const signer2 = StellarSdk.Keypair.random();
    const signer3 = StellarSdk.Keypair.random();
    const signer4 = StellarSdk.Keypair.random();
    
    console.log('Signer 1:', signer1.publicKey(), signer1.secret());
    console.log('Signer 2:', signer2.publicKey(), signer2.secret());
    console.log('Signer 3:', signer3.publicKey(), signer3.secret());
    console.log('Signer 4:', signer4.publicKey(), signer4.secret());
    
    // Load the distribution account
    const distributionAccount = await server.loadAccount(distributionKeypair.publicKey());
    
    // Setup multi-signature requirements (3 of 5)
    const transaction = new StellarSdk.TransactionBuilder(distributionAccount, {
      fee: StellarSdk.BASE_FEE,
      networkPassphrase: StellarSdk.Networks.TESTNET
    })
      .addOperation(StellarSdk.Operation.setOptions({
        masterWeight: 1, // The distribution account's weight
        lowThreshold: 3,  // Require 3 signatures for low threshold operations
        medThreshold: 3,  // Require 3 signatures for medium threshold operations
        highThreshold: 3, // Require 3 signatures for high threshold operations
        signer: {
          ed25519PublicKey: signer1.publicKey(),
          weight: 1
        }
      }))
      .addOperation(StellarSdk.Operation.setOptions({
        signer: {
          ed25519PublicKey: signer2.publicKey(),
          weight: 1
        }
      }))
      .addOperation(StellarSdk.Operation.setOptions({
        signer: {
          ed25519PublicKey: signer3.publicKey(),
          weight: 1
        }
      }))
      .addOperation(StellarSdk.Operation.setOptions({
        signer: {
          ed25519PublicKey: signer4.publicKey(),
          weight: 1
        }
      }))
      .setTimeout(30)
      .build();
      
    transaction.sign(distributionKeypair);
    const result = await server.submitTransaction(transaction);
    console.log('Multi-signature setup complete:', result.hash);
    return result;
  } catch (error) {
    console.error('Error setting up multi-signature:', error);
    throw error;
  }
}

// await setupMultiSig();
```

## 8. Publish Asset Information via stellar.toml

Create a `stellar.toml` file and host it at `https://cvmine.com/.well-known/stellar.toml`:

```toml
# ITAL Token Stellar Asset Information
# https://cvmine.com/.well-known/stellar.toml

NETWORK_PASSPHRASE="Public Global Stellar Network ; September 2015"

# Organization Information
[DOCUMENTATION]
ORG_NAME="Talentmicro Innovations"
ORG_URL="https://talentmicro.com"
ORG_DESCRIPTION="Professional development incentive system for job seekers"
ORG_LOGO="https://talentmicro.com/logo.png"
ORG_PHYSICAL_ADDRESS="Your address"
ORG_OFFICIAL_EMAIL="contact@cvmine.com"

# ITAL Token Information
[[CURRENCIES]]
code="ITAL"
issuer="ISSUER_PUBLIC_KEY_HERE"
display_decimals=5
name="ITAL Token"
desc="A peer-to-peer incentive system for professional development"
conditions="The total supply is limited to 6 trillion ITAL tokens. Tokens are earned through verifiable activities on the CVmine platform."
image="https://cvmine.com/ital-logo.png"
fixed_number=600000000000
```

## 9. Create a Service for User Wallet Operations

Now integrate this into your Node.js app with a comprehensive service:

```javascript
// src/services/stellar.service.js
const StellarSdk = require('stellar-sdk');
const config = require('../config/stellar');
const mysql = require('../config/database');
const crypto = require('crypto');

// Configure Stellar SDK
const server = new StellarSdk.Server(config.horizonUrl);
const networkPassphrase = config.isTestnet 
  ? StellarSdk.Networks.TESTNET 
  : StellarSdk.Networks.PUBLIC;

class StellarService {
  constructor() {
    this.assetCode = 'ITAL';
    this.assetIssuer = config.issuerPublicKey;
    this.italAsset = new StellarSdk.Asset(this.assetCode, this.assetIssuer);
    
    // Distribution account should be loaded with a proper key management system
    // This is simplified for development
    this.distributionKeypair = StellarSdk.Keypair.fromSecret(config.distributionSecret);
  }
  
  // Create user's Stellar account and establish trustline
  async createUserStellarAccount(userId) {
    // Generate a key pair for the user
    const userKeypair = StellarSdk.Keypair.random();
    
    try {
      // First create the account (requires funding with XLM)
      const distributionAccount = await server.loadAccount(this.distributionKeypair.publicKey());
      
      let transaction = new StellarSdk.TransactionBuilder(distributionAccount, {
        fee: StellarSdk.BASE_FEE,
        networkPassphrase
      })
        .addOperation(StellarSdk.Operation.createAccount({
          destination: userKeypair.publicKey(),
          startingBalance: '1.5' // 1.5 XLM covers minimum balance + fees
        }))
        .setTimeout(30)
        .build();
      
      transaction.sign(this.distributionKeypair);
      await server.submitTransaction(transaction);
      
      // Now establish the trustline for ITAL tokens
      const userAccount = await server.loadAccount(userKeypair.publicKey());
      
      transaction = new StellarSdk.TransactionBuilder(userAccount, {
        fee: StellarSdk.BASE_FEE,
        networkPassphrase
      })
        .addOperation(StellarSdk.Operation.changeTrust({
          asset: this.italAsset,
          limit: '1000000000' // 1 billion ITAL limit
        }))
        .setTimeout(30)
        .build();
      
      transaction.sign(userKeypair);
      await server.submitTransaction(transaction);
      
      // Save the user's Stellar information in the database
      // In production, handle key storage securely (possibly client-side only)
      const encryptedSecret = this.encryptSecret(userKeypair.secret(), userId);
      
      await mysql.query(
        `UPDATE users SET 
         stellar_address = ?, 
         encrypted_stellar_secret = ?,
         has_stellar_account = TRUE 
         WHERE id = ?`,
        [userKeypair.publicKey(), encryptedSecret, userId]
      );
      
      return {
        publicKey: userKeypair.publicKey(),
        secret: userKeypair.secret() // In production, consider not returning this
      };
    } catch (error) {
      console.error('Error creating Stellar account:', error);
      throw new Error('Failed to create Stellar account: ' + error.message);
    }
  }
  
  // Encrypt user's secret key (for development - use HSM in production)
  encryptSecret(secret, userId) {
    const cipher = crypto.createCipher('aes-256-cbc', process.env.ENCRYPTION_KEY + userId);
    let encrypted = cipher.update(secret, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return encrypted;
  }
  
  // Decrypt user's secret key (for development - use HSM in production)
  decryptSecret(encryptedSecret, userId) {
    const decipher = crypto.createDecipher('aes-256-cbc', process.env.ENCRYPTION_KEY + userId);
    let decrypted = decipher.update(encryptedSecret, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
  }
  
  // Send ITAL tokens from distribution account to a user
  async sendTokensToUser(destinationPublicKey, amount) {
    try {
      const distributionAccount = await server.loadAccount(this.distributionKeypair.publicKey());
      
      const transaction = new StellarSdk.TransactionBuilder(distributionAccount, {
        fee: StellarSdk.BASE_FEE,
        networkPassphrase
      })
        .addOperation(StellarSdk.Operation.payment({
          destination: destinationPublicKey,
          asset: this.italAsset,
          amount: amount.toFixed(7) // 7 decimal places precision
        }))
        .setTimeout(30)
        .build();
      
      transaction.sign(this.distributionKeypair);
      const result = await server.submitTransaction(transaction);
      return result;
    } catch (error) {
      console.error('Error sending ITAL tokens:', error);
      throw new Error('Failed to send tokens: ' + error.message);
    }
  }
  
  // Commit a Merkle root to the blockchain for verification
  async commitMerkleRoot(merkleRoot) {
    try {
      const distributionAccount = await server.loadAccount(this.distributionKeypair.publicKey());
      
      // Use a data entry to store the Merkle root
      const transaction = new StellarSdk.TransactionBuilder(distributionAccount, {
        fee: StellarSdk.BASE_FEE,
        networkPassphrase,
        memo: StellarSdk.Memo.text('CVmine Merkle Root')
      })
        .addOperation(StellarSdk.Operation.manageData({
          name: 'cvmine_merkle_' + Date.now(), // Add timestamp to make unique
          value: merkleRoot
        }))
        .setTimeout(30)
        .build();
      
      transaction.sign(this.distributionKeypair);
      const result = await server.submitTransaction(transaction);
      return result;
    } catch (error) {
      console.error('Error committing Merkle root:', error);
      throw new Error('Failed to commit Merkle root: ' + error.message);
    }
  }
  
  // Get a user's on-chain ITAL balance
  async getUserStellarBalance(stellarAddress) {
    try {
      const account = await server.loadAccount(stellarAddress);
      
      // Find the ITAL balance in the account's balances
      const italBalance = account.balances.find(
        balance => 
          balance.asset_type !== 'native' && 
          balance.asset_code === this.assetCode && 
          balance.asset_issuer === this.assetIssuer
      );
      
      return italBalance ? parseFloat(italBalance.balance) : 0;
    } catch (error) {
      console.error('Error getting Stellar balance:', error);
      return 0; // Account might not exist or have a trustline
    }
  }
  
  // Transfer tokens between users (requires user's secret key)
  async transferBetweenUsers(senderUserId, recipientStellarAddress, amount) {
    try {
      // Get sender's Stellar information
      const [userResult] = await mysql.query(
        'SELECT stellar_address, encrypted_stellar_secret FROM users WHERE id = ? AND has_stellar_account = TRUE',
        [senderUserId]
      );
      
      if (!userResult.length) {
        throw new Error('Sender does not have a Stellar account');
      }
      
      // Decrypt the sender's secret key
      const senderSecret = this.decryptSecret(
        userResult[0].encrypted_stellar_secret, 
        senderUserId
      );
      
      const senderKeypair = StellarSdk.Keypair.fromSecret(senderSecret);
      const senderAccount = await server.loadAccount(senderKeypair.publicKey());
      
      // Create and submit the transaction
      const transaction = new StellarSdk.TransactionBuilder(senderAccount, {
        fee: StellarSdk.BASE_FEE,
        networkPassphrase
      })
        .addOperation(StellarSdk.Operation.payment({
          destination: recipientStellarAddress,
          asset: this.italAsset,
          amount: amount.toFixed(7)
        }))
        .setTimeout(30)
        .build();
      
      transaction.sign(senderKeypair);
      const result = await server.submitTransaction(transaction);
      return result;
    } catch (error) {
      console.error('Error transferring between users:', error);
      throw new Error('Failed to transfer tokens: ' + error.message);
    }
  }
}

module.exports = new StellarService();
```

## 10. Complete Implementation

To fully implement the ITAL token in your project:

1. **Configure Stellar Settings**: Create a configuration file with your issuer and distribution keys:

```javascript
// src/config/stellar.js
module.exports = {
  isTestnet: process.env.STELLAR_NETWORK === 'testnet',
  horizonUrl: process.env.STELLAR_NETWORK === 'testnet' 
    ? 'https://horizon-testnet.stellar.org' 
    : 'https://horizon.stellar.org',
  issuerPublicKey: process.env.STELLAR_ISSUER_PUBLIC_KEY,
  issuerSecret: process.env.STELLAR_ISSUER_SECRET, // Keep secure
  distributionPublicKey: process.env.STELLAR_DISTRIBUTION_PUBLIC_KEY,
  distributionSecret: process.env.STELLAR_DISTRIBUTION_SECRET, // Keep secure
};
```

2. **Update Environment Variables**: Add the necessary variables to your `.env` file:

```
# Stellar Configuration
STELLAR_NETWORK=testnet
STELLAR_ISSUER_PUBLIC_KEY=YOUR_ISSUER_PUBLIC_KEY
STELLAR_ISSUER_SECRET=YOUR_ISSUER_SECRET
STELLAR_DISTRIBUTION_PUBLIC_KEY=YOUR_DISTRIBUTION_PUBLIC_KEY
STELLAR_DISTRIBUTION_SECRET=YOUR_DISTRIBUTION_SECRET
ENCRYPTION_KEY=YOUR_STRONG_ENCRYPTION_KEY
```

3. **Create Initialization Script**: For initial setup and deployment:

```javascript
// scripts/initialize-stellar-token.js
require('dotenv').config();
const StellarService = require('../src/services/stellar.service');
const stellarConfig = require('../src/config/stellar');

// Generate all account components, establish trustlines, and issue tokens
async function initializeStellarToken() {
  try {
    console.log('Initializing ITAL token on Stellar...');
    // Implementation depends on whether you're creating new accounts or using existing ones
    console.log('ITAL token initialized successfully!');
  } catch (error) {
    console.error('Error initializing ITAL token:', error);
  }
}

initializeStellarToken();
```

4. **Integrate with Your Token Service**: Connect the Stellar service with your existing token service:

```javascript
// src/services/token.service.js (extension for blockchain sync)

// Add this method to your TokenService class
async syncUserToBlockchain(userId) {
  const connection = await mysql.getConnection();
  try {
    await connection.beginTransaction();
    
    // Get user information
    const [userResult] = await connection.query(
      'SELECT has_stellar_account, stellar_address FROM users WHERE id = ?',
      [userId]
    );
    
    if (!userResult.length) {
      throw new Error('User not found');
    }
    
    // Check if user already has a Stellar account
    if (!userResult[0].has_stellar_account) {
      // Create Stellar account for user
      await stellarService.createUserStellarAccount(userId);
    }
    
    // Get user's current off-chain balance
    const [balanceResult] = await connection.query(
      'SELECT current_balance FROM token_balances WHERE user_id = ?',
      [userId]
    );
    
    if (!balanceResult.length) {
      throw new Error('User has no token balance');
    }
    
    const offChainBalance = parseFloat(balanceResult[0].current_balance);
    
    // Get user's current on-chain balance
    const onChainBalance = await stellarService.getUserStellarBalance(
      userResult[0].stellar_address
    );
    
    // Calculate the difference to sync
    const syncAmount = offChainBalance - onChainBalance;
    
    if (syncAmount <= 0) {
      await connection.commit();
      return { status: 'NO_SYNC_NEEDED', offChainBalance, onChainBalance };
    }
    
    // Send the difference to the user's Stellar account
    const result = await stellarService.sendTokensToUser(
      userResult[0].stellar_address, 
      syncAmount
    );
    
    // Create a synchronization transaction record
    const transactionId = uuid.v4();
    await connection.query(
      `INSERT INTO transactions 
       (id, user_id, transaction_type, token_amount, blockchain_sync_status, blockchain_reference_id, status) 
       VALUES (?, ?, 'BLOCKCHAIN_SYNC', ?, 'SYNCED', ?, 'COMPLETED')`,
      [transactionId, userId, syncAmount, result.hash]
    );
    
    await connection.commit();
    return { 
      status: 'SYNCED', 
      transactionId,
      stellarTransactionHash: result.hash,
      syncAmount,
      newOnChainBalance: offChainBalance
    };
  } catch (error) {
    await connection.rollback();
    console.error('Error syncing to blockchain:', error);
    throw error;
  } finally {
    connection.release();
  }
}
```

This comprehensive implementation gives you all the components needed to create and manage the ITAL token on the Stellar network, with proper integration into your existing MySQL/Node.js architecture.
