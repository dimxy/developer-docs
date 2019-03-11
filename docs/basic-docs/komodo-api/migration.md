# Migration API

Migration API allows to move coin or token value between chains or even within the same chain.
The principle of migration assumes that some amount of coins or tokens is burned in the source chain and then exactly the same amount 
is created in the destination chain   
There are several ways of value migration in Komodo platform:
- MoMoM notarized migration
- migration with manual notarization 
- selfimport migration.

The migration process consists of making a burn transaction in the source chain and making an import transaction for the burned value 
which is created in the source chain but is sent to the destination chain. Komodo validation code checks that for the import transaction 
there exists a corresponding burn transaction and that it is not spent more than once.

The following migration RPC calls interact with the `komodod` software, and are made available through the `komodo-cli` software.

# MoMoM notarized migration  

MoMoM notarized migration API allows the migration of coin or token value based on Komodo's highly scalable notarization process when 
elected and trusted notary nodes store fingeprints (what is called MoM, merkle root of merkle roots) of assets blockchains' blocks 
in the main komodo chain and fingerprints of fingerprints (what is called MoMoM, 'merkle root of merkle roots of merkle roots') 
are delivered back into the assets chain (as back notarizations)
(more about the notarization process is here: https://komodoplatform.com/komodo-platforms-new-scalability-tech/).

The workflow of the MoMoM value migration is following:
- On the source chain user calls migrate_createburntransaction rpc and sends the burn transaction by sendrawtransaction call
- On the source chain the user runs migrate_createimportransaction and passes to it the burn transaction and 'payouts' object in hex format 
which he received from the previous call
- On the main Komodo (KMD) chain the used calls migrate_completeimpottransaction where he passes the import transaction in hex format
which he received from the previous call. On this stage the proof object for the burn transaction inside the import transaction 
is extended with MoMoM data. This allows to check the burn transaction on the destination chain by using standard Komodo notarization process
without the need to create proof objects additionally

## createrawtransaction

**createrawtransaction '[{ "txid": "id_string", "vout": number }, ... ]' '{ "address": amount, ... }'**

The `createrawtransaction` method creates a transaction, spending the given inputs and sending to the given addresses. The method returns a hex-encoded raw transaction.

::: tip
This is a raw transaction, and therefore the inputs are not signed and the transaction is not stored in the wallet nor transmitted to the network.
:::

### Arguments:

Structure|Type|Description
---------|----|-----------
"transactions"                               |(string, required)           |a json array of json objects
"txid"                                       |(string, required)           |the transaction id
"vout"                                       |(numeric, required)          |the output number
"addresses"                                  |(string, required)           |a json object with addresses as keys and amounts as values
"address"                                    |(numeric, required)          |the key is the address, the value is the COIN amount
