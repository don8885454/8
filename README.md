const StellarSDK = require("@stellar/stellar-sdk");

const server = new StellarSDK.Horizon.Server("https://api.testnet.minepi.com");
const NETWORK_PASSPHRASE = "Pi Testnet";

// prepare keypairs
const issuerKeypair = StellarSDK.Keypair.fromSecret("GC4MRSNQONZUNWVXZNTK6DGKM636PBCFZ4VNYI7MTYB5ZENFUMBF3WE4"); // use actual secret key here
const distributorKeypair = StellarSDK.Keypair.fromSecret("GB3QIVLHFFUQONTJD262BABTQUT23CATBPN3OVQ4UXCZ5ZOW7XI3QLJT"); // use actual secret key here

// define a token
// token code should be alphanumeric and up to 12 characters, case sensitive
const customToken = new StellarSDK.Asset("GabtToken", issuerKeypair.publicKey());

const distributorAccount = await server.loadAccount(distributorKeypair.publicKey());

// look up base fee
const response = await server.ledgers().order("desc").limit(1).call();
const latestBlock = response.records[0];
const baseFee = latestBlock.base_fee_in_stroops;

// prepare a transaction that establishes trustline
const trustlineTransaction = new StellarSDK.TransactionBuilder(distributorAccount, {
  fee: baseFee,
  networkPassphrase: NETWORK_PASSPHRASE,
  timebounds: await server.fetchTimebounds(90),
})
  .addOperation(StellarSDK.Operation.changeTrust({ asset: customToken, limit: undefined }))
  .build();

trustlineTransaction.sign(distributorKeypair);

// submit a tx
await server.submitTransaction(trustlineTransaction);
console.log("Trustline created successfully");

//====================================================================================
//
...
"_links": {
  "toml": {
    "href": ""
  }
}
...
//====================================================================================
//
now mint TestToken by sending from issuer account to distributor account

const issuerAccount = await server.loadAccount(issuerKeypair.publicKey());

const paymentTransaction = new StellarSDK.TransactionBuilder(issuerAccount, {
  fee: baseFee,
  networkPassphrase: NETWORK_PASSPHRASE,
  timebounds: await server.fetchTimebounds(90),
})
  .addOperation(
    StellarSDK.Operation.payment({
      destination: distributorKeypair.publicKey(),
      asset: customToken,
      amount: "100000", // amount to mint
    })
  )
  .build();

paymentTransaction.sign(issuerKeypair);

// submit a tx
await server.submitTransaction(paymentTransaction);
console.log("Token issued successfully");

// checking new balance of the distributor account
const updatedDistributorAccount = await server.loadAccount(distributorKeypair.publicKey());
updatedDistributorAccount.balances.forEach((balance) => {
  if (balance.asset_type === "native") {
    console.log(`Test-Pi Balance: ${balance.balance}`);
  } else {
    console.log(`${balance.asset_code} Balance: ${balance.balance}`);
  }
});
