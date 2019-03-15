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
in the main komodo chain and after that, fingerprints of fingerprints (what is called MoMoM, 'merkle root of merkle roots of merkle roots') are delivered back into the assets chain (as back notarizations)
(more about the notarization process is here: https://komodoplatform.com/komodo-platforms-new-scalability-tech/).

The workflow of the MoMoM value migration is following:
- On the source chain user calls migrate_createburntransaction rpc and sends the burn transaction by sendrawtransaction call
- On the source chain the user runs migrate_createimportransaction and passes to it the burn transaction and 'payouts' object in hex format 
which user received from the previous call
- On the main Komodo (KMD) chain the user calls migrate_completeimpottransaction where the user passes the import transaction in hex format which was received from the previous call. On this stage the proof object for the burn transaction inside the import transaction 
is extended with MoMoM data. This allows to check the burn transaction on the destination chain by using standard Komodo notarization process
without the need to create proof objects additionally

## migrate_createburntransaction

**migrate_createburntransaction destChain destAddress amount [tokenid]**

The `migrate_createburntransaction` method creates a transaction burning some amount of coins or tokens. The methods also creates payouts object used for creating an import transaction for the burned amount of value. This method should be called on the source chain.
The method returns a created burn transaction which should be send to the source chain with `sendrawtransaction` method.
After the burn transaction successfully mined you would need to wait for some time for back notarization with MoMoM fingerprints for the mined block with teh burn transaction would come to the chain.
The other returned value `payouts` should be passed to the next method `migrate_createimporttransaction`.

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
"hex"                                |(string)                     |a hex string of the created burn transaction
"payouts"                            |(string)                     |a hex string of the created payouts (to be passed into migrate_createimporttransaction rpc method later)

## migrate_createimporttransaction

**migrate_createimporttransaction burntx payouts**

The `migrate_createimporttransaction` method performs a initial step in creating an import transaction . This method should be called on the source chain.
The method returns a created import transaction in hex. This value should be passed into the migrate_completeimporttransaction method on the main KMD chain to be extended with MoMoM proof object

### Arguments:

Structure|Type|Description
---------|----|-----------
"burntx"                                |(string, required)         |burn transaction in hex created on the previous step
"payout"                                |(string, required)         |payouts object in hex created on the previous step and used for creating an import transaction

### Response:

a hex string of the created import transaction

## migrate_completeimporttransaction

**migrate_completeimporttransaction importtx**

The `migrate_createimporttransaction` method performs a initial step in creating an import transaction . This method should be called on the source chain.
The method returns a created import transaction in hex. This value should be passed into the migrate_completeimporttransaction method on the main KMD chain to be extended with MoMoM proof object

### Arguments:

Structure|Type|Description
---------|----|-----------
"burntx"                                |(string, required)         |burn transaction in hex created on the previous step
"payout"                                |(string, required)         |payouts object in hex created on the previous step and used for creating an import transaction

### Response:

a hex string of the created import transaction



