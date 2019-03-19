# Migration API

Migration API allows to move coin or token value between chains.
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

Requerement: the source and destination chains should have the equal CCid parameter and it should be greater than 100 (which means that chains are fungible).

# MoMoM notarized migration  

MoMoM notarized migration API allows the migration of coin or token value based on Komodo's highly scalable notarization process when 
elected and trusted notary nodes store fingeprints (what is called MoM, merkle root of merkle roots) of assets blockchains' blocks 
in the main komodo chain and after that, fingerprints of fingerprints (what is called MoMoM, 'merkle root of merkle roots of merkle roots') are delivered back into the assets chain (as back notarizations)
(more about the notarization process is here: https://komodoplatform.com/komodo-platforms-new-scalability-tech/).

The workflow of the MoMoM value migration is following:
- On the source chain user calls migrate_createburntransaction rpc and sends the burn transaction by sendrawtransaction call
- On the source chain the user runs migrate_createimportransaction and passes to it the burn transaction and 'payouts' object in hex format 
which user received from the previous call
- On the main Komodo (KMD) chain the user calls migrate_completeimpottransaction where the user passes the import transaction in hex format which was received from the previous call. On this stage the proof object for the burn transaction inside the import transaction 
is extended with MoMoM data. This allows to check the burn transaction on the destination chain by using standard Komodo notarization process without the need to create proof objects additionally.

## migrate_createburntransaction

**migrate_createburntransaction destChain destAddress amount [tokenid]**

The `migrate_createburntransaction` method creates a transaction burning some amount of coins or tokens. The methods also creates payouts object used for creating an import transaction for the burned amount of value. This method should be called on the source chain.
The method returns a created burn transaction which should be send to the source chain with `sendrawtransaction` method.
After the burn transaction successfully mined it might be needed to wait for some time for the back notarization with MoMoM fingerprints of the mined block with the burn transaction to reach the source chain.
The hex value of the burn transaction along with the other returned value `payouts` should be passed to the next method `migrate_createimporttransaction`.

### Arguments:

Structure|Type|Description
---------|----|-----------
"destChain"                                  |(string, required)           |the destination chain name
"destAddress"                                |(string, required)           |address on the destination chain where coins are to be sent or pubkey if tokens are to be sent
"amount"                                     |(numeric, required)          |the amount in coins or tokens that will be burned on the source chain and created on the destination chain. If it is tokens the amount should be set only to 1 (as only migration of non-fungible  tokens are supported at this time)
"tokenid"                                    |(string, optional)           |token id in hex, if set, it is tokens are to be migrated

### Response:

Structure|Type|Description
---------|----|-----------
"payouts"                                  |(string)                     |a hex string of the created payouts (to be passed into migrate_createimporttransaction rpc method later)
"BurnTxHex"                                |(string)                     |a hex string of the created burn transaction

## An alternate method of creating a customized burn transaction
If it is needed to create a customized burn transaction there is an additional rpc method `migrate_converttoexport` which converts passed transaction to a burn transaction. It adds proof data to the passed transaction and extracts the transaction vouts and calculates and burns their amount by sending it to an OP_RETURN vout which is added to the transaction. 
It is responsibility of the caller to fund and sign the returned burn transaction with rpc methos `fundrawtransaction` and `signrawtransaction`.
The signed burn transaction should be sent to the destination chain by the `sendrawtansaction` method.
Note: this method supports only coins (tokens are not supported).

## migrate_converttoexport

**migrate_converttoexport burntx**

The `migrate_converttoexport` method adds OP_RETURN object to the passed transaction.
The other returned value `payouts` should be passed to the next method `migrate_createimporttransaction`.

### Arguments:

Structure|Type|Description
---------|----|-----------
"burntx"                                  |(string, required)           |the burn transaction in hex
"destChain"                                |(string, required)           |the destination chain name

### Response:

Structure|Type|Description
---------|----|-----------
"payouts"                                  |(string)                     |a hex string of the created payouts (to be passed into migrate_createimporttransaction rpc method later)
"exportTx"                                 |(string)                     |a hex string of the returned burn transaction

## Burn transaction size consideration
Because the burn transaction is stored in import transaction OP_RETURN vout which size is limited by 10,001 bytes it is recommended to limit the burn transaction size to the 30% of the OP_RETURN object.

## migrate_createimporttransaction

**migrate_createimporttransaction burntx payouts**

The `migrate_createimporttransaction` method performs a initial step in creating an import transaction. This method should be called on the source chain.
The method returns a created import transaction in hex. This string should be passed into the migrate_completeimporttransaction method on the main KMD chain to be extended with MoMoM proof object.

### Arguments:

Structure|Type|Description
---------|----|-----------
"burntx"                                 |(string, required)         |burn transaction in hex created on the previous step
"payouts"                                |(string, required)         |payouts object in hex created on the previous step and used for creating an import transaction
"notarytxid-1"                             |(string, optional)         |notary approval transaction id 1, passed if MoMoM backp notarization solution is used
...
"notarytxid-N"                             |(string, optional)         |notary approval transaction id N, passed if MoMoM backp notarization solution is used

### Response:
Structure|Type|Description
---------|----|-----------
"ImportTxHex"                           |(string)         |a hex string of the created import transaction

Or errors may be returned. In case of errors it might be necessary to wait for some time before the back notarizations objects are sent in the destination chain.

## migrate_completeimporttransaction

**migrate_completeimporttransaction importtx**

The `migrate_completeimporttransaction` method performs the finalizing step in creating an import transaction. This method should be called on the KMD chain.
The method returns the import transaction in hex updated with MoMoM proof object which would confirm that the burn transaction exists in the source chain. 
This value of finalized import transaction may be passed to `sendrawtransaction` rpc on the destination chain.
In case of errors which may be returned while sending the import transaction it might be necessary to wait for some time before the notarizations objects are stored in the destination chain.

### Arguments:

Structure|Type|Description
---------|----|-----------
"importtx"                              |(string, required)         |burn transaction in hex created on the previous step
"offset"                                |(string, optional)         |offset of the current kmd blockchain height to search for a MoMoM

### Response:
Structure|Type|Description
---------|----|-----------
"ImportTxHex"                           |(string)         |import transaction in hex extended with MoMoM proof of the burn tx

Or errors may be returned. In case of errors it might be necessary to wait for some time before the notarizations objects are stored in the KMD chain.



# Notarization backup solution
There is an alternate solution for notarizing burn transaction by the notary operators in case of MoMoM notarization fails or slow.
For this the notary operators pick burn transactions sent to a special publishing resource, check them and return ids of transactions with burn transaction proof objects which are created in destination chains.
The worflow:
- A user creates a burn transaction with the above stated `migrate_createburntransaction` rpc method and publishes its hexademical representation to a publishing resource which is monitored by the notary operators (currently the discord channel ...) 
- The notary operators pick the burn transaction and check its structure and existence in the source chain with the rpc method `migrate_checkburntransactionsource`. If the burn transaction is successfully validated, the notary operators create approval transactions in the destination chain and publish their transaction ids back into the publishing resource.
- The user collects the transaction ids and calls `migrate_createimporttransaction` method, passing into it the collected notary approval transaction ids. Currently it is enough to have at least 5 successful notary approval transactions for an import transaction to be considered as valid in the destination chain.

## migrate_checkburntransactionsource

**migrate_checkburntransactionsource burntx**

The `migrate_checkburntransactionsource` method allows to a notary operator to check the burn transaction structure and verify its presence in the source chain.

### Arguments:

Structure|Type|Description
---------|----|-----------
"burntx"                                   |(string, required)           |the burn transaction in hex

### Response:

Structure|Type|Description
---------|----|-----------
"sourceSymbol"                             |(string)                     |source chain name
"targetSymbol"                             |(string)                     |target chain name
"targetCCid"                               |(number)                     |target chain CCid 
"tokenid"                                  |(string, optional)           |token id if it is token to migrate
"TxOutProof"                               |(string)                     |proof of burn transaction existence in the source chain  

## migrate_createnotaryapprovaltransaction

**migrate_createnotaryapprovaltransaction burntxid txoutproof**

A notary operator uses the `migrate_createnotaryapprovaltransaction` method to create an approval transaction in the destination chain with the proof of the burn transaction existence in the source chain.
Returned notary transaction should be sent to the destination chain with `sendrawtransaction` method

### Arguments:

Structure|Type|Description
---------|----|-----------
"burntxid"                                   |(string, required)           |the burn transaction id
"txoutproof"                                   |(string, required)           |the burn transaction id


### Response:

Structure|Type|Description
---------|----|-----------
"NotaryTxHex"                             |(string)                     |notary approval transaction in hex



