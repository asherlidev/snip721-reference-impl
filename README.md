# SNIP-721 Reference Implementation
This is a reference implementation of the [SNIP-721 specification](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md) (which is currently being updated).  It not only implements the base requirements of SNIP-721,  but it also includes additional functionality that may be helpful for many use cases.  As SNIP-721 is a superset of the [CW-721 specification](https://github.com/CosmWasm/cosmwasm-plus/blob/master/packages/cw721/README.md), this implementation is CW-721 compliant; however, because CW-721 does not support privacy, a number of the CW-721-compliant functions may not return all the information a CW-721 implementation would.  For example, the OwnerOf query will not display the approvals for a token unless the token owner has supplied his address and viewing key.  In order to strive for CW-721 compliance, a number of functions that require authentication use optional parameters that the CW-721 counterpart does not have.  If the optional authentication parameters are not supplied, the responses will only display information that the token owner has made public.

### Terms
- __Message__ - This is an on-chain interface. It is triggered by sending a transaction, and receiving an on-chain response which is read by the client. Messages are authenticated both by the blockchain, and by the secret enclave.
- __Query__ - This is an off-chain interface. Queries are done by returning data that a node has locally, and are not public. Query responses are returned immediately, and do not have to wait for blocks.
- __Cosmos Message Sender__ - The account that is found under the `sender` field in a standard Cosmos SDK message. This is also the signer of the message.

### Padding
Users may want to enforce constant length messages to avoid leaking data. To support this functionality, every message includes an optional `padding` field. This optional `padding` field is ignored during message processing.

### Requests
Requests should be sent as base64 encoded JSON. Future versions of Secret Network may add support for other formats as well, but at this time we recommend usage of JSON only. For this reason the parameter descriptions specify the JSON type which must be used. In addition, request parameters will include in parentheses a CosmWasm (or other) underlying type that this value must conform to. E.g. a recipient address is sent as a string, but must also be parsed to a bech32 address.

### Responses
Message responses will be JSON encoded in the `data` field of the Cosmos response, rather than in the `logs`, except in the case of MintNft and BatchMintNft messages, where the token ID(s) will be returned in both the `data` and `logs` fields.  This is because minting may frequently be done by a contract, and `data` fields of responses from callback messages do not get forwarded to the sender of the initial message.

# Instantiating The Token Contract
##### Request
```
{
    “name”: “name_of_the_token”,
    “symbol”: “token_symbol”,
	“admin”: “optional_admin_address”,
	“entropy”: “string_used_as_entropy_when_generating_random_viewing_keys”,
	“config”: {
		“public_token_supply”: true|false,
		“public_owner”: true|false,
		“enable_sealed_metadata”: true|false,
		“unwrapped_metadata_is_private”: true|false,
		“minter_may_update_metadata”: true|false,
		“owner_may_update_metadata”: true|false,
		“enable_burn”: true|false
	},
	“post_init_callback”: {
		“msg”: “base64_encoded_Binary_representing_the_msg_to_perform_after_initialization”,
		“contract_address”: “address_of_the_contract_being_called_after_initialization”,
		“code_hash”: “code_hash_of_the_contract_being_called_after_initialization”,
		“send”: [
			{
				“denom”: “denom_string_for_native_coin_being_sent_with_this_message”,
				“amount”: “amount_of_native_coin_being_sent”
			},
			{
				"...": "..."
			}
		]
	}
}
```
| Name               | Type                                                   | Description                                                         | Optional | Value If Omitted   |
|--------------------|--------------------------------------------------------|---------------------------------------------------------------------|----------|--------------------|
| name               | string                                                 | Name of the token contract                                          | no       |                    |
| symbol             | string                                                 | Token contract symbol                                               | no       |                    |
| admin              | string (HumanAddr)                                     | Address to be given admin authority                                 | yes      | env.message.sender |
| entropy            | string                                                 | String used as entropy when generating random viewing keys          | no       |                    |
| config             | [Config (see below)](#config)                          | Privacy configuration for the contract                              | yes      | defined below      |
| post_init_callback | [PostInitCallback (see below)](#postinitcallback)      | Information used to perform a callback message after initialization | yes      | nothing            |

### <a name="config"></a>Config
Config is the privacy configuration for the contract.
* public_token_supply - This config value indicates whether the token IDs and the number of tokens controlled by the contract are public.  If the token supply is private, only minters can view the token IDs and number of tokens controlled by the contract (default: False)
* public_owner - This config value indicates whether token ownership is public or private by default.  Regardless of this setting a user has the ability to change whether the ownership of their tokens is public or private (default: False)
* enable_sealed_metadata - This config value indicates whether sealed metadata should be enabled.  If sealed metadata is enabled, the private metadata of a newly minted token is not viewable by anyone, not even the owner, until the owner calls the Reveal function.  When Reveal is called, the sealed metadata is irreversibly unwrapped and moved to the public metadata (as default).  If unwrapped_metadata_is_private is set to true, the sealed metadata will remain as private metadata after unwrapping, but the owner will now be able to see it.  Anyone will be able to query the token to know whether it has been unwrapped.  This simulates buying/selling a wrapped card that no one knows which card it is until it is unwrapped. If sealed metadata is not enabled, all tokens are considered unwrapped when minted (default: False)
* unwrapped_metadata_is_private - This config value indicates if the Reveal function should keep the sealed metadata private after unwrapping.  This config value is ignored if sealed metadata is not enabled (default: False)
* minter_may_update_metadata - This config value indicates whether a minter is permitted to update a token's metadata (default: True)
* owner_may_update_metadata - This config value indicates whether the owner of a token is permitted to update a token's metadata (default: False)
* enable_burn - This config value indicates whether burn functionality is enabled (default: False)
```
{
	“public_token_supply”: true|false,
	“public_owner”: true|false,
	“enable_sealed_metadata”: true|false,
	“unwrapped_metadata_is_private”: true|false,
	“minter_may_update_metadata”: true|false,
	“owner_may_update_metadata”: true|false,
	“enable_burn”: true|false
}
```
| Name                          | Type | Optional | Value If Omitted |
|-------------------------------|------|----------|------------------|
| public_token_supply           | bool | yes      | false            |
| public_owner                  | bool | yes      | false            |
| enable_sealed_metadata        | bool | yes      | false            |
| unwrapped_metadata_is_private | bool | yes      | false            |
| minter_may_update_metadata    | bool | yes      | true             |
| owner_may_update_metadata     | bool | yes      | false            |
| enable_burn                   | bool | yes      | false            |

### <a name="postinitcallback"></a>Post Init Callback
Post)init_callback is the optional callback message to execute after the token contract has initialized.  This can be useful if another contract is instantiating this token contract and needs the token contract to inform the creating contract of the address it has been given
```
{
	“msg”: “base64_encoded_Binary_representing_the_msg_to_perform_after_initialization”,
	“contract_address”: “address_of_the_contract_being_called_after_initialization”,
	“code_hash”: “code_hash_of_the_contract_being_called_after_initialization”,
	“send”: [
		{
			“denom”: “denom_string_for_native_coin_being_sent_with_this_message”,
			“amount”: “amount_of_native_coin_being_sent”
		},
		{
			"...": "..."
		}
	]
}
```
| Name             | Type                                  | Description                                                                                           | Optional | Value If Omitted |
|------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------|----------|------------------|
| msg              | string (base64 encoded Binary)        | Base64 encoded Binary representation of the callback message to perform after contract initialization | no       |                  |
| contract_address | string (HumanAddr)                    | Address of the contract to call after initialization                                                  | no       |                  |
| code_hash        | string                                | Code hash of the contract to call after initialization                                                | no       |                  |
| send             | array of [Coin (see below)](#coin)    | List of native Coin amounts to send with the callback message                                         | no       |                  |

#### <a name="coin"></a>Coin
Coin is the payment to send with the post-init callback message.  Although `send` is not an optional field of the Post Init Callback, because it is an array, you can just use `[]` to not send any payment with the callback message
```
{
	“denom”: “denom_string_for_native_coin_being_sent_with_this_message”,
	“amount”: “amount_of_native_coin_being_sent”
}
```
| Name   | Type             | Description                                                               | Optional | Value If Omitted |
|--------|------------------|---------------------------------------------------------------------------|----------|------------------|
| denom  | string           | The denomination of the native Coin (uscrt for SCRT)                      | no       |                  |
| amount | string (Uint128) | The amount of the native Coin to send with the Post Init Callback message | no       |                  |

# Messages
## MintNft
MintNft mints a single token.

##### Request
```
{
	"mint_nft": {
		"token_id": "optional_ID_of_new_token",
		"owner": "optional_address_the_new_token_will_be_minted_to",
		"public_metadata": {
			"name": "optional_public_name",
			"description": "optional_public_text_description",
			"image": "optional_public_uri_pointing_to_an_image_or_additional_off-chain_metadata"
		},
		"private_metadata": {
			"name": "optional_private_name",
			"description": "optional_private_text_description",
			"image": "optional_private_uri_pointing_to_an_image_or_additional_off-chain_metadata"
		},
		"memo": "optional_memo_for_the_mint_tx",
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name             | Type                                   | Description                                                                                   | Optional | Value If Omitted     |
|------------------|----------------------------------------|-----------------------------------------------------------------------------------------------|----------|----------------------|
| token_id         | string                                 | Identifier for the token to be minted                                                         | yes      | minting order number |
| owner            | string (HumanAddr)                     | Address of the owner of the minted token                                                      | yes      | env.message.sender   |
| public_metadata  | [Metadata (see below)](#metadata)      | The metadata that is publicly viewable                                                        | yes      | nothing              |
| private_metadata | [Metadata (see below)](#metadata)      | The metadata that is viewable only by the token owner and addresses the owner has whitelisted | yes      | nothing              |
| memo             | string                                 | A memo for the mint tx that is only viewable by addresses involved in the mint (minter/owner) | yes      | nothing              |
| padding          | string                                 | An Ignored string that can be used to maintain constant message length                        | yes      | nothing              |

##### Response
```
{
	"mint_nft": {
		"token_id": "ID_of_minted_token",
	}
}
```
The ID of the minted token will also be returned in a LogAttribute with the key `minted`.

### <a name="metadata"></a>Metadata
Metadata for a token that follows CW-721 metadata specification, which is based on ERC721 Metadata JSON Schema.
```
{
	"name": "optional_name",
	"description": "optional_text_description",
	"image": "optional_uri_pointing_to_an_image_or_additional_off-chain_metadata"
}
```
| Name        | Type   | Description                                                           | Optional | Value If Omitted     |
|-------------|--------|-----------------------------------------------------------------------|----------|----------------------|
| name        | string | String that can be used to identify an asset's name                   | yes      | nothing              |
| description | string | String that can be used to describe an asset                          | yes      | nothing              |
| image       | string | String that can hold a link to additional off-chain metadata or image | yes      | nothing              |

## BatchMintNft
BatchMintNft mints a list of tokens.

##### Request
```
{
	"batch_mint_nft": {
		"mints": [
			{
				"token_id": "optional_ID_of_new_token",
				"owner": "optional_address_the_new_token_will_be_minted_to",
				"public_metadata": {
					"name": "optional_public_name",
					"description": "optional_public_text_description",
					"image": "optional_public_uri_pointing_to_an_image_or_additional_off-chain_metadata"
				},
				"private_metadata": {
					"name": "optional_private_name",
					"description": "optional_private_text_description",
					"image": "optional_private_uri_pointing_to_an_image_or_additional_off-chain_metadata"
				},
				"memo": "optional_memo_for_the_mint_tx"
			},
			{
				"...": "..."
			}
		],
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type                                  | Description                                                            | Optional | Value If Omitted |
|---------|---------------------------------------|------------------------------------------------------------------------|----------|------------------|
| mints   | array of [Mint (see below)](#mint)    | A list of all the mint operations to perform                           | no       |                  |
| padding | string                                | An Ignored string that can be used to maintain constant message length | yes      | nothing          |

##### Response
```
{
	"batch_mint_nft": {
		"token_ids": [
			"IDs", "of", "tokens", "that", "were", "minted", "..."
		]
	}
}
```
The IDs of the minted tokens will also be returned in a LogAttribute with the key `minted`.

### <a name="mint"></a>Mint
Mint defines the data necessary to perform one minting operation
```
{
	"token_id": "optional_ID_of_new_token",
	"owner": "optional_address_the_new_token_will_be_minted_to",
	"public_metadata": {
		"name": "optional_public_name",
		"description": "optional_public_text_description",
		"image": "optional_public_uri_pointing_to_an_image_or_additional_off-chain_metadata"
	},
	"private_metadata": {
		"name": "optional_private_name",
		"description": "optional_private_text_description",
		"image": "optional_private_uri_pointing_to_an_image_or_additional_off-chain_metadata"
	},
	"memo": "optional_memo_for_the_mint_tx"
}
```
| Name             | Type                                 | Description                                                                                        | Optional | Value If Omitted     |
|------------------|--------------------------------------|----------------------------------------------------------------------------------------------------|----------|----------------------|
| token_id         | string                               | Identifier for the token to be minted                                                              | yes      | minting order number |
| owner            | string (HumanAddr)                   | Address of the owner of the minted token                                                           | yes      | env.message.sender   |
| public_metadata  | [Metadata (see above)](#metadata)    | The metadata that is publicly viewable                                                             | yes      | nothing              |
| private_metadata | [Metadata (see above)](#metadata)    | The metadata that is viewable only by the token owner and addresses the owner has whitelisted      | yes      | nothing              |
| memo             | string                               | A memo for the mint tx that is only viewable by addresses involved in the mint (minter/owner)      | yes      | nothing              |

## SetPublicMetadata
SetPublicMetadata will set the public metadata to the input metadata if the message sender is either the token owner or an approved minter and they have been given this power by the configuration value chosen during instantiation

##### Request
```
{
	"set_public_metadata": {
		"token_id": "ID_of_token_whose_metadata_should_be_updated",
		"metadata": {
			"name": "optional_public_name",
			"description": "optional_public_text_description",
			"image": "optional_public_uri_pointing_to_an_image_or_additional_off-chain_metadata"
		},
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name     | Type                                 | Description                                                            | Optional | Value If Omitted |
|----------|--------------------------------------|------------------------------------------------------------------------|----------|------------------|
| token_id | string                               | ID of the token whose metadata should be updated                       | no       |                  |
| metadata | [Metadata (see above)](#metadata)    | The new public metadata for the token                                  | no       |                  |
| padding  | string                               | An Ignored string that can be used to maintain constant message length | yes      | nothing          |

##### Response
```
{
	"set_public_metadata": {
		"status": "success"
	}
}
```

## SetPrivateMetadata
SetPrivateMetadata will set the private metadata to the input metadata if the message sender is either the token owner or an approved minter and they have been given this power by the configuration value chosen during instantiation.  The private metadata of a sealed token may not be altered until after it has been unwrapped.

##### Request
```
{
	"set_private_metadata": {
		"token_id": "ID_of_token_whose_metadata_should_be_updated",
		"metadata": {
			"name": "optional_private_name",
			"description": "optional_private_text_description",
			"image": "optional_private_uri_pointing_to_an_image_or_additional_off-chain_metadata"
		},
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name     | Type                                 | Description                                                            | Optional | Value If Omitted |
|----------|--------------------------------------|------------------------------------------------------------------------|----------|------------------|
| token_id | string                               | ID of the token whose metadata should be updated                       | no       |                  |
| metadata | [Metadata (see above)](#metadata)    | The new private metadata for the token                                 | no       |                  |
| padding  | string                               | An Ignored string that can be used to maintain constant message length | yes      | nothing          |

##### Response
```
{
	"set_private_metadata": {
		"status": "success"
	}
}
```

## <a name="reveal"></a>Reveal
Reveal unwraps the sealed metadata, irreversibly marking the token as unwrapped.  If the `unwrapped_metadata_is_private` configuration value is true, the formerly sealed metadata will remain private, otherwise it will be made public.

##### Request
```
{
	"reveal": {
		"token_id": "ID_of_the_token_to_unwrap",
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name     | Type                 | Description                                                            | Optional | Value If Omitted |
|----------|----------------------|------------------------------------------------------------------------|----------|------------------|
| token_id | string               | ID of the token to unwrap                                              | no       |                  |
| padding  | string               | An Ignored string that can be used to maintain constant message length | yes      | nothing          |

##### Response
```
{
	"reveal": {
		"status": "success"
	}
}
```

## MakeOwnershipPrivate
MakeOwnershipPrivate is used when the token contract was instantiated with the `public_owner` configuration value set to true.  It allows an address to make all of its tokens have private ownership by default.  The owner may still use SetGlobalApproval or SetWhitelistedApproval to make ownership public as desired.

##### Request
```
{
	"make_ownership_private": {
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name     | Type                 | Description                                                            | Optional | Value If Omitted |
|----------|----------------------|------------------------------------------------------------------------|----------|------------------|
| padding  | string               | An Ignored string that can be used to maintain constant message length | yes      | nothing          |

##### Response
```
{
	"make_ownership_private": {
		"status": "success"
	}
}
```

## SetGlobalApproval
The owner of a token can use SetGlobalApproval to make ownership and/or private metadata viewable by everyone.  This can be set for a single token or for an owner's entire inventory of tokens by choosing the appropriate AccessLevel.  SetGlobalApproval can also be used to revoke any global approval previously granted.

##### Request
```
{
	"set_global_approval": {
		"token_id": "optional_ID_of_the_token_to_grant_or_revoke_approval_on",
		"view_owner": "approve_token"|"all"|"revoke_token"|"none",
		"view_private_metadata": "approve_token"|"all"|"revoke_token"|"none",
		"expires": "never"|{"at_height": 999999}|{"at_time":999999},
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                                       | Description                                                                                        | Optional | Value If Omitted |
|-----------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------|----------|------------------|
| token_id              | string                                     | If supplying either `approve_token` or `revoke_token` access, the token whose privacy is being set | yes      | nothing          |
| view_owner            | [AccessLevel (see below)](#accesslevel)    | Grant or revoke everyone's permission to view the ownership of a token/inventory                   | yes      | nothing          |
| view_private_metadata | [AccessLevel (see below)](#accesslevel)    | Grant or revoke everyone's permission to view the private metadata of a token/inventory            | yes      | nothing          |
| expires               | [Expiration (see below)](#expiration)      | Expiration of any approval granted in this message.  Can be a blockheight, time, or never          | yes      | "never"          |
| padding               | string                                     | An Ignored string that can be used to maintain constant message length                             | yes      | nothing          |

##### Response
```
{
	"set_global_approval": {
		"status": "success"
	}
}
```
### <a name="accesslevel"></a>AccessLevel
AccessLevel determines the type of access being granted or revoked to the specified address in a SetWhitelistedApproval message or to everyone in a SetGlobalApproval message.  The levels are mutually exclusive for any address (or for everyone if it is a global approval).  The levels are:
* `"approve_token"` - grant approval only on the token specified in the message
* `"revoke_token"` - revoke a previous approval on the specified token
* `"all"` - grant approval for all tokens in the message signer's inventory.  This approval will also apply to any tokens the signer acquires after granting `all` approval
* `"none"` - revoke any approval (both token and inventory-wide) previously granted to the specified address (or for everyone if using SetGlobalApproval)

If the message signer grants an address (or everyone in the case of SetGlobalApproval) `all` (inventory-wide) approval, it will remove any individual token approvals previously granted to that address (or granted to everyone in the case of SetGlobalApproval), and grant that address `all` (inventory-wide) approval.  If an address (or everyone in the case of SetGlobalApproval) already has `all` approval, and the message signer grants it `approve_token` approval, if the expiration of the new `approve_token` approval is the same as the expiration of the previous `all` approval, it will just leave the `all` approval in place.  If the expirations are different, it will grant `approve_token` approval with the specified expiration for the input token, and all other tokens will be changed to `approve_token` approvals with the expiration of the previous `all` approval, and the `all` (inventory-wide) approval will be removed.  If the message signer applies `revoke_token` access to an address that currently has inventory-wide approval, it will remove the inventory-wide approval, and create `approve_token` approvals for that address on every token in the signer's inventory EXCEPT the token specified with the `revoke_token` message.  In other words, it will only revoke the approval on that single token.

### <a name="expiration"></a>Expiration
Expiration is used to set an expiration for any approvals granted in the message.  Expiration can be set to a specified blockheight, a time in seconds since epoch 01/01/1970, or "never".  Values for blockheight and time are specified as a u64.  If no expiration is given, it will default to "never".

One should be aware that the current blockheight and time is not available to a query on Secret Network at this moment, but there are plans to make the BlockInfo available to queries in a future hardfork.  To get around this limitation, the contract saves the BlockInfo every time a message is executed, and uses the blockheight and time of the last message execution to check viewing permission expiration during a query.  Therefore it is possible that a whitelisted address may be able to view the owner or metadata of a token passed its permission expiration if no one executed any contract message since before the expiration.  However, because transferring/burning a token is executing a message, it does have the current blockheight and time available and can not occur if transfer permission has expired.

* `"never"` - the approval will never expire
* `{"at_time": 1700000000}` - the approval will expire 1700000000 seconds after 01/01/1970 (time value is u64)
* `{"at_height": 3000000}` - the approval will expire at block height 3000000 (height value is u64)

## SetWhitelistedApproval
The owner of a token can use SetWhitelistedApproval to grant an address permission to view ownership, view private metadata, and/or to transfer a single token or every token in the owner's inventory.  SetWhitelistedApproval can also be used to revoke any approval previously granted to the address.

##### Request
```
{
	"set_whitelisted_approval": {
		"address": "address_being_granted_or_revoked_approval",
		"token_id": "optional_ID_of_the_token_to_grant_or_revoke_approval_on",
		"view_owner": "approve_token"|"all"|"revoke_token"|"none",
		"view_private_metadata": "approve_token"|"all"|"revoke_token"|"none",
		"transfer": "approve_token"|"all"|"revoke_token"|"none",
		"expires": "never"|{"at_height": 999999}|{"at_time":999999},
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                                       | Description                                                                                        | Optional | Value If Omitted |
|-----------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------|----------|------------------|
| address               | string (HumanAddr)                         | Address to grant or revoke approval to/from                                                        | no       |                  |
| token_id              | string                                     | If supplying either `approve_token` or `revoke_token` access, the token whose privacy is being set | yes      | nothing          |
| view_owner            | [AccessLevel (see above)](#accesslevel)    | Grant or revoke the address' permission to view the ownership of a token/inventory                 | yes      | nothing          |
| view_private_metadata | [AccessLevel (see above)](#accesslevel)    | Grant or revoke the address' permission to view the private metadata of a token/inventory          | yes      | nothing          |
| transfer              | [AccessLevel (see above)](#accesslevel)    | Grant or revoke the address' permission to transfer a token/inventory                              | yes      | nothing          |
| expires               | [Expiration (see above)](#expiration)      | The expiration of any approval granted in this message.  Can be a blockheight, time, or never      | yes      | "never"          |
| padding               | string                                     | An Ignored string that can be used to maintain constant message length                             | yes      | nothing          |

##### Response
```
{
	"set_whitelisted_approval": {
		"status": "success"
	}
}
```

## Approve
Approve is used to grant an address permission to transfer a single token.  This can only be performed by the token's owner or, in compliance with CW-721, an address that has inventory-wide approval to transfer the owner's tokens.  Approve is provided to maintain compliance with CW-721, but the owner can use SetWhitelistedApproval to accomplish the same thing if specifying a `token_id` and `approve_token` AccessLevel for `transfer`.
##### Request
```
{
	"approve": {
		"spender": "address_being_granted_approval_to_transfer_the_specified_token",
		"token_id": "ID_of_the_token_that_can_now_be_transferred_by_the_spender",
		"expires": "never"|{"at_height": 999999}|{"at_time":999999},
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                                     | Description                                                                                          | Optional | Value If Omitted |
|-----------------------|------------------------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| spender               | string (HumanAddr)                       | Address being granted approval to transfer the token                                                 | no       |                  |
| token_id              | string                                   | ID of the token that the spender can now transfer                                                    | no       |                  |
| expires               | [Expiration (see above)](#expiration)    | The expiration of this token transfer approval.  Can be a blockheight, time, or never                | yes      | "never"          |
| padding               | string                                   | An Ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"approve": {
		"status": "success"
	}
}
```

## Revoke
Revoke is used to revoke from an address the permission to transfer this single token.  This can only be performed by the token's owner or, in compliance with CW-721, an address that has inventory-wide approval to transfer the owner's tokens (referred to as an operator later). However, one operator may not revoke transfer permission of even one single token away from another operator.  Revoke is provided to maintain compliance with CW-721, but the owner can use SetWhitelistedApproval to accomplish the same thing if specifying a `token_id` and `revoke_token` AccessLevel for `transfer`.
##### Request
```
{
	"revoke": {
		"spender": "address_being_revoked_approval_to_transfer_the_specified_token",
		"token_id": "ID_of_the_token_that_can_no_longer_be_transferred_by_the_spender",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                    | Description                                                                                          | Optional | Value If Omitted |
|-----------------------|-------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| spender               | string (HumanAddr)      | Address no longer permitted to transfer the token                                                    | no       |                  |
| token_id              | string                  | ID of the token that the spender can no longer transfer                                              | no       |                  |
| padding               | string                  | An Ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"revoke": {
		"status": "success"
	}
}
```

## ApproveAll
ApproveAll is used to grant an address permission to transfer all the tokens in the message sender's inventory.  This will include the ability to transfer any tokens the sender acquires after granting this inventory-wide approval.  This also gives the address the ability to grant another address the approval to transfer a single token.  ApproveAll is provided to maintain compliance with CW-721, but the message sender can use SetWhitelistedApproval to accomplish the same thing by using `all` AccessLevel for `transfer`.

##### Request
```
{
	"approve_all": {
		"operator": "address_being_granted_inventory-wide_approval_to_transfer_tokens",
		"expires": "never"|{"at_height": 999999}|{"at_time":999999},
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                                     | Description                                                                                          | Optional | Value If Omitted |
|-----------------------|------------------------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| operator              | string (HumanAddr)                       | Address being granted approval to transfer all of the message sender's tokens                        | no       |                  |
| expires               | [Expiration (see above)](#expiration)    | The expiration of this inventory-wide transfer approval.  Can be a blockheight, time, or never       | yes      | "never"          |
| padding               | string                                   | An Ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"approve_all": {
		"status": "success"
	}
}
```

## RevokeAll
RevokeAll is used to revoke an inventory-wide transfer approval granted to an address.  RevokeAll is provided to maintain compliance with CW-721, but the message sender can use SetWhitelistedApproval to accomplish the same thing by using `none` AccessLevel for `transfer`.

##### Request
```
{
	"revoke_all": {
		"operator": "address_being_revoked_a_previous_inventory-wide_approval_to_transfer_tokens",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                    | Description                                                                                          | Optional | Value If Omitted |
|-----------------------|-------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| operator              | string (HumanAddr)      | Address being revoked a previous approval to transfer all of the message sender's tokens             | no       |                  |
| padding               | string                  | An Ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"revoke_all": {
		"status": "success"
	}
}
```

## TransferNft
TransferNft is used to transfer ownership of the token to the recipient address.  This requires a valid `token_id` and the message sender must either be the owner or an address with valid transfer approval.  If the recipient address is the same as the current owner, no transfer will be done (transaction history will not include a transfer that does not change ownership).  If the token is transferred to a new owner, its single-token approvals will be cleared.

##### Request
```
{
	"transfer_nft": {
		"recipient": "address_receiving_the_token",
		"token_id": "ID_of_the_token_being_transferred",
		"memo": "optional_memo_for_the_transfer_tx",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type               | Description                                                                                                                       | Optional | Value If Omitted |
|-----------|--------------------|-----------------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| recipient | string (HumanAddr) | Address receiving the token                                                                                                       | no       |                  |
| token_id  | string             | Identifier of the token to be transferred                                                                                         | no       |                  |
| memo      | string             | Memo for the transfer transaction that is only viewable by addresses involved in the transfer (recipient, sender, previous owner) | yes      | nothing          |
| padding   | string             | An Ignored string that can be used to maintain constant message length                                                            | yes      | nothing          |

##### Response
```
{
	"transfer_nft": {
		"status": "success"
	}
}
```

## BatchTransferNft
BatchTransferNft is used to perform multiple token transfers.  The message sender may specify a list of tokens to transfer to one recipient address in each `Transfer` object, and any `memo` provided will be applied to every token transferred in that one `Transfer` object.  The message sender may provide multiple `Transfer` objects to perform transfers to multiple addresses, providing a different memo for each address if desired.  Each individual transfer of a token will show separately in transaction histories.  The message sender must have permission to transfer all the tokens listed (either by being the owner or being granted transfer approval) and every listed `token_id` must be valid.  A contract may use the VerifyTransferApproval query to verify that it has permission to transfer all the tokens.  If the message sender does not have permission to transfer any one of the listed tokens, the entire message will fail (no tokens will be transferred) and the error will provide the ID of the first token encountered in which the sender does not have the required permission.  If any token transfer involves a recipient address that is the same as its current owner, that transfer will not be done (transaction history will not include a transfer that does not change ownership), but all the other transfers will proceed.  Any token that is transferred to a new owner will have its single-token approvals cleared.

##### Request
```
{
	"batch_transfer_nft": {
		"transfers": [
			{
				"recipient": "address_receiving_the_tokens",
				"token_ids": [
					"list", "of", "token", "IDs", "to", "transfer"
				],
				"memo": "optional_memo_applied_to_the_transfer_tx_for_every_token_listed_in_this_Transfer_object"
			},
			{
				"...": "..."
			}
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type                                          | Description                                                                                          | Optional | Value If Omitted |
|-----------|-----------------------------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| transfers | array of [Transfer (see below)](#transfer)    | List of `Transfer` objects to process                                                                | no       |                  |
| padding   | string                                        | An Ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"batch_transfer_nft": {
		"status": "success"
	}
}
```

### <a name="transfer"></a>Transfer
The Transfer object provides a list of tokens to transfer to one recipient address, as well as an optional memo that would be included with every logged token transfer.
```
{
	"recipient": "address_receiving_the_tokens",
	"token_ids": [
		"list", "of", "token", "IDs", "to", "transfer", "..."
	],
	"memo": "optional_memo_applied_to_the_transfer_tx_for_every_token_listed_in_this_Transfer_object"
}
```
| Name      | Type               | Description                                                                                                                       | Optional | Value If Omitted |
|-----------|--------------------|-----------------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| recipient | string (HumanAddr) | Address receiving the listed tokens                                                                                               | no       |                  |
| token_ids | array of string    | List of token IDs to transfer to the recipient                                                                                    | no       |                  |
| memo      | string             | Memo for the transfer transactions that is only viewable by addresses involved in the transfer (recipient, sender, previous owner)| yes      | nothing          |

## <a name="send"></a>SendNft
SendNft is used to transfer ownership of the token to the `contract` address, and then call the recipient's BatchReceiveNft (or ReceiveNft, [see below](#receiver)) if the recipient contract has registered its receiver interface with the NFT contract.  If the recipient contract registered that it implements BatchReceiveNft, a BatchReceiveNft callback will be performed with only the single token ID in the `token_ids` array.  This requires a valid `token_id` and the message sender must either be the owner or an address with valid transfer approval.  If the recipient address is the same as the current owner, no transfer will be done (transaction history will not include a transfer that does not change ownership), but the BatchReceiveNft (or ReceiveNft) callback will be performed if registered.  If the token is transferred to a new owner, its single-token approvals will be cleared.  If the BatchReceiveNft (or ReceiveNft) callback fails, the entire transaction will be reverted (even the transfer will not take place).

##### Request
```
{
	"send_nft": {
		"contract": "address_receiving_the_token",
		"token_id": "ID_of_the_token_being_transferred",
		"msg": "optional_base64_encoded_Binary_message_sent_with_the_BatchReceiveNft_callback",
		"memo": "optional_memo_for_the_transfer_tx",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name     | Type                           | Description                                                                                                                | Optional | Value If Omitted |
|----------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| contract | string (HumanAddr)             | Address receiving the token                                                                                                | no       |                  |
| token_id | string                         | Identifier of the token to be transferred                                                                                  | no       |                  |
| msg      | string (base64 encoded Binary) | Msg included when calling the recipient contract's BatchReceiveNft (or ReceiveNft)                                         | yes      | nothing          |
| memo     | string                         | Memo for the transfer tx that is only viewable by addresses involved in the transfer (recipient, sender, previous owner)   | yes      | nothing          |
| padding  | string                         | An Ignored string that can be used to maintain constant message length                                                     | yes      | nothing          |

##### Response
```
{
	"send_nft": {
		"status": "success"
	}
}
```

## <a name="batchsend"></a>BatchSendNft
BatchSendNft is used to perform multiple token transfers, and then call the recipient contracts' BatchReceiveNft (or ReceiveNft [see below](#receiver)) if they have registered their receiver interface with the NFT contract.  The message sender may specify a list of tokens to send to one recipient address in each `Send` object, and any `memo` or `msg` provided will be applied to every token transferred in that one `Send` object.  If the list of transferred tokens belonged to multiple previous owners, a separate BatchReceiveNft callback will be performed for each of the previous owners.  If the contract only implements ReceiveNft, one ReceiveNft will be performed for every sent token.  Therefore it is highly recommended to implement BatchReceiveNft if there is the possibility of being sent multiple tokens at one time.  This will reduce gas costs by around 80k per token sent.  

The message sender may provide multiple `Send` objects to perform sends to multiple addresses, providing a different `memo` and `msg` for each address if desired.  Each individual transfer of a token will show separately in transaction histories.  The message sender must have permission to transfer all the tokens listed (either by being the owner or being granted transfer approval) and every token ID must be valid.  A contract may use the [VerifyTransferApproval](#verifyapproval) query to verify that it has permission to transfer all the tokens.  If the message sender does not have permission to transfer any one of the listed tokens, the entire message will fail (no tokens will be transferred) and the error will provide the ID of the first token encountered in which the sender does not have the required permission.  If any token transfer involves a recipient address that is the same as its current owner, that transfer will not be done (transaction history will not include a transfer that does not change ownership), but all the other transfers will proceed.  Any token that is transferred to a new owner will have its single-token approvals cleared.
If any BatchReceiveNft (or ReceiveNft) callback fails, the entire transaction will be reverted (even the transfers will not take place).

##### Request
```
{
	"batch_send_nft": {
		"sends": [
			{
				"contract": "address_receiving_the_tokens",
				"token_ids": [
					"list", "of", "token", "IDs", "to", "transfer", "..."
				],
				"msg": "optional_base64_encoded_Binary_message_sent_with_every_BatchReceiveNft_callback_made_for_this_one_Send_object",
				"memo": "optional_memo_applied_to_the_transfer_tx_for_every_token_listed_in_this_Send_object"
			},
			{
				"...": "..."
			}
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type                                  | Description                                                                                                        | Optional | Value If Omitted |
|-----------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| sends     | array of [Send (see below)](#send)    | List of `Send` objects to process                                                                                  | no       |                  |
| padding   | string                                | An Ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"batch_send_nft": {
		"status": "success"
	}
}
```

### <a name="send"></a>Send
The Send object provides a list of tokens to transfer to one recipient address, as well as an optional memo that would be included with every logged token transfer, and an optional msg that would be included with every BatchReceiveNft or ReceiveNft ([see below](#receiver)) callback made as a result of this Send object.
```
{
	"contract": "address_receiving_the_tokens",
	"token_ids": [
		"list", "of", "token", "IDs", "to", "transfer", "..."
	],
	"msg": "optional_base64_encoded_Binary_message_sent_with_every_BatchReceiveNft_callback_made_for_this_one_Send_object",
	"memo": "optional_memo_applied_to_the_transfer_tx_for_every_token_listed_in_this_Send_object"
}
```
| Name      | Type                           | Description                                                                                                               | Optional | Value If Omitted |
|-----------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| contract  | string (HumanAddr)             | Address receiving the listed tokens                                                                                       | no       |                  |
| token_ids | array of string                | List of token IDs to send to the recipient                                                                                | no       |                  |
| msg       | string (base64 encoded Binary) | Msg included with every BatchReceiveNft (or ReceiveNft) callback performed for this `Send` object                         | yes      | nothing          |
| memo      | string                         | Memo for the transfer txs that is only viewable by addresses involved in the transfer (recipient, sender, previous owner) | yes      | nothing          |

## BurnNft
BurnNft is used to burn a single token, providing an optional memo to include in the burn's transaction history if desired.  If the contract has not enabled burn functionality using the init configuration `enable_burn`, BurnNft will result in an error.  The token owner and anyone else with valid transfer approval may use BurnNft.

##### Request
```
{
	"burn_nft": {
		"token_id": "ID_of_the_token_to_burn",
		"memo": "optional_memo_for_the_burn_tx",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type               | Description                                                                                                                       | Optional | Value If Omitted |
|-----------|--------------------|-----------------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| token_id  | string             | Identifier of the token to burn                                                                                                   | no       |                  |
| memo      | string             | Memo for the burn tx that is only viewable by addresses involved in the burn (message sender and previous owner if different)     | yes      | nothing          |
| padding   | string             | An Ignored string that can be used to maintain constant message length                                                            | yes      | nothing          |

##### Response
```
{
	"burn_nft": {
		"status": "success"
	}
}
```

## BatchBurnNft
BatchBurnNft is used to burn multiple tokens.  The message sender may specify a list of tokens to burn in each `Burn` object, and any `memo` provided will be applied to every token burned in that one `Burn` object.  The message sender will usually list every token to be burned in one `Burn` object, but if a different memo is needed for different tokens being burned, multiple `Burn` objects may be listed. Each individual burning of a token will show separately in transaction histories.  The message sender must have permission to transfer/burn all the tokens listed (either by being the owner or being granted transfer approval).  A contract may use the VerifyTransferApproval query to verify that it has permission to transfer/burn all the tokens.  If the message sender does not have permission to transfer/burn any one of the listed tokens, the entire message will fail (no tokens will be burned) and the error will provide the ID of the first token encountered in which the sender does not have the required permission.

##### Request
```
{
	"batch_burn_nft": {
		"burns": [
			{
				"token_ids": [
					"list", "of", "token", "IDs", "to", "burn"
				],
				"memo": "optional_memo_applied_to_the_burn_tx_for_every_token_listed_in_this_Burn_object"
			},
			{
				"...": "..."
			}
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type                                  | Description                                                                                                        | Optional | Value If Omitted |
|-----------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| burns     | array of [Burn (see below)](#burn)    | List of `Burn` objects to process                                                                                  | no       |                  |
| padding   | string                                | An Ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"batch_burn_nft": {
		"status": "success"
	}
}
```

### <a name="burn"></a>Burn
The Burn object provides a list of tokens to burn, as well as an optional memo that would be included with every token burn transaction history.
```
{
	"token_ids": [
		"list", "of", "token", "IDs", "to", "burn", "..."
	],
	"memo": "optional_memo_applied_to_the_burn_tx_for_every_token_listed_in_this_Burn_object"
}
```
| Name      | Type            | Description                                                                                                                    | Optional | Value If Omitted |
|-----------|-----------------|--------------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| token_ids | array of string | List of token IDs to burn                                                                                                      | no       |                  |
| memo      | string          | Memo for the burn txs that is only viewable by addresses involved in the burn (message sender and previous owner if different) | yes      | nothing          |

## CreateViewingKey
CreateViewingKey is used to generate a random viewing key to be used to authenticate account-specific queries.  The `entropy` field is a client-supplied string used as part of the entropy supplied to the rng that creates the viewing key.

##### Request
```
{
	"create_viewing_key": {
		"entropy": "string_used_as_part_of_the_entropy_supplied_to_the_rng",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type   | Description                                                                                                        | Optional | Value If Omitted |
|-----------|--------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| entropy   | string | String used as part of the entropy supplied to the rng that generates the random viewing key                       | no       |                  |
| padding   | string | An Ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"viewing_key": {
		"key": "the_created_viewing_key"
	}
}
```

## SetViewingKey
SetViewingKey is used to set the viewing key to a predefined string.  It will replace any key that currently exists.  It would be best for users to call CreateViewingKey to ensure a strong key, but this function is provided so that contracts can also utilize viewing keys.

##### Request
```
{
	"set_viewing_key": {
		"key": "the_new_viewing_key",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type   | Description                                                                                                        | Optional | Value If Omitted |
|-----------|--------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| key       | string | The new viewing key for the message sender                                                                         | no       |                  |
| padding   | string | An Ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"viewing_key": {
		"key": "the_message_sender's_viewing_key"
	}
}
```

## AddMinters
AddMinters will add the provided addresses to the list of authorized minters.  This can only be called by the admin address.

##### Request
```
{
	"add_minters": {
		"minters": [
			"list", "of", "addresses", "to", "add", "to", "the", "list", "of", "minters", "..."
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type                        | Description                                                                                                        | Optional | Value If Omitted |
|---------|-----------------------------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| minters | array of string (HumanAddr) | The list of addresses to add to the list of authorized minters                                                     | no       |                  |
| padding | string                      | An Ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"add_minters": {
		"status": "success"
	}
}
```

## RemoveMinters
RemoveMinters will remove the provided addresses from the list of authorized minters.  This can only be called by the admin address.

##### Request
```
{
	"remove_minters": {
		"minters": [
			"list", "of", "addresses", "to", "remove", "from", "the", "list", "of", "minters", "..."
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type                        | Description                                                                                                        | Optional | Value If Omitted |
|---------|-----------------------------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| minters | array of string (HumanAddr) | The list of addresses to remove from the list of authorized minters                                                | no       |                  |
| padding | string                      | An Ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"remove_minters": {
		"status": "success"
	}
}
```

## SetMinters
SetMinters will precisely define the list of authorized minters.  This can only be called by the admin address.

##### Request
```
{
	"set_minters": {
		"minters": [
			"list", "of", "addresses", "that", "have", "minting", "authority", "..."
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type                        | Description                                                                                 | Optional | Value If Omitted |
|---------|-----------------------------|---------------------------------------------------------------------------------------------|----------|------------------|
| minters | array of string (HumanAddr) | The list of addresses to are allowed to mint                                                | no       |                  |
| padding | string                      | An Ignored string that can be used to maintain constant message length                      | yes      | nothing          |

##### Response
```
{
	"set_minters": {
		"status": "success"
	}
}
```

## SetContractStatus
SetContractStatus allows the contract admin to define which messages the contract will execute.  This can only be called by the admin address.

##### Request
```
{
	"set_contract_status": {
		"level": "normal"|"stop_transactions"|"stop_all",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type                                             | Description                                                                                 | Optional | Value If Omitted |
|---------|--------------------------------------------------|---------------------------------------------------------------------------------------------|----------|------------------|
| level   | [ContractStatus (see below)](#contractstatus)    | The level that defines which messages the contract will execute                             | no       |                  |
| padding | string                                           | An Ignored string that can be used to maintain constant message length                      | yes      | nothing          |

##### Response
```
{
	"set_contract_status": {
		"status": "success"
	}
}
```
### <a name="contractstatus"></a>ContractStatus
ContractStatus indicates which messages the contract will execute. The possible values are:
* `"normal"` - the contract will execute all messages
* `"stop_transactions"` - the contract will not allow any minting, burning, sending, or transferring of tokens
* `"stop_all"` - the contract will only execute a SetContractStatus message

## ChangeAdmin
ChangeAdmin will allow the current admin to transfer admin privileges to another address (which will be the only admin address).  This can only be called by the current admin address.

##### Request
```
{
	"change_admin": {
		"address": "address_of_the_new_contract_admin",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type               | Description                                                                                 | Optional | Value If Omitted |
|---------|--------------------|---------------------------------------------------------------------------------------------|----------|------------------|
| address | string (HumanAddr) | Address of the new contract admin                                                           | no       |                  |
| padding | string             | An Ignored string that can be used to maintain constant message length                      | yes      | nothing          |

##### Response
```
{
	"change_admin": {
		"status": "success"
	}
}
```

## <a name="registerreceive"></a>RegisterReceiveNft
A contract will use RegisterReceiveNft to notify the NFT contract that it implements ReceiveNft and possibly also BatchReceiveNft [(see below)](#receiver).  This enables the NFT contract to call the registered contract whenever it is Sent a token (or tokens).  ReceiveNft only informs the recipient contract that it has been sent a single token; while, BatchReceiveNft can be used to inform a contract that it was sent multiple tokens.  This is particularly useful if a contract needs to be sent a deck of "cards", as it will save around 80k in gas for every token that was sent.  If a contract implements BatchReceiveNft, the NFT contract will always call BatchReceiveNft even if there is only one token being sent, in which case the `token_ids` array will only have one element.

##### Request
```
{
	"register_receive_nft": {
		"code_hash": "code_hash_of_the_contract_implementing_a_receiver_interface",
		"also_implements_batch_receive_nft": true|false,
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                              | Type   | Description                                                                                                                | Optional | Value If Omitted |
|-----------------------------------|--------|----------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| code_hash                         | string | Code hash of the message sender, which is a contract that implements a receiver interface                                  | no       |                  |
| also_implements_batch_receive_nft | bool   | true if the message sender contract also implements BatchReceiveNft so it can be informed that it was sent a list of tokens| yes      | false            |
| padding                           | string | An Ignored string that can be used to maintain constant message length                                                     | yes      | nothing          |

##### Response
```
{
	"register_receive_nft": {
		"status": "success"
	}
}
```

# Queries
Queries are off-chain requests, that are not cryptographically validated; therefore, this contract utilizes viewing keys to authenticate address-specific queries.  It makes viewing key validation resource intensive in order to combat offline brute-force attempts to guess viewing keys.  Also, even if a user has not set a viewing key, it will perform the same resource intensive processing to prevent an attacker from knowing that a key has not been set and provides the same error response whether there is no viewing key or if the input key does not match.

Any query that inquires about a specific token will return an error if the input token ID does not exist.  If the token supply is public, the error will indicate that the token does not exist.  If the token supply is private, the query will return the same error response whether the token does not exist or the querier does not have permission to view the requested information.

<a name="queryblockinfo"></a>One should be aware that the current blockheight and time is not available to a query on Secret Network at this moment, but there are plans to make the BlockInfo available to queries in a future hardfork.  To get around this limitation, the contract saves the BlockInfo every time a message is executed, and uses the blockheight and time of the last message execution to check viewing permission expiration during a query.  Therefore it is possible that a whitelisted address may be able to view the owner or metadata of a token past its permission expiration if no one executed any contract message since before the expiration.  However, because transferring/burning a token is executing a message, it does have the current blockheight and time available and can not occur if transfer permission has expired.

## ContractInfo
ContractInfo returns the contract's name and symbol.  This query is not authenticated.

##### Request
```
{
	"contract_info": {}
}
```
##### Response
```
{
	"contract_info": {
		"name": "contract_name",
		"symbol": "contract_symbol"
	}
}
```

## ContractConfig
ContractConfig returns the configuration values that were selected when the contract was instantiated.  See [Config](#config) for an explanation of the configuration options.  This query is not authenticated.

##### Request
```
{
	"contract_config": {}
}
```
##### Response
```
{
	"contract_config": {
		“token_supply_is_public”: true|false,
		“owner_is_public”: true|false,
		“sealed_metadata_is_enabled”: true|false,
		“unwrapped_metadata_is_private”: true|false,
		“minter_may_update_metadata”: true|false,
		“owner_may_update_metadata”: true|false,
		“burn_is_enabled”: true|false
	}
}
```
| Name                          | Type | Description                                                                                | Optional | 
|-------------------------------|------|--------------------------------------------------------------------------------------------|----------|
| token_supply_is_public        | bool | True if token IDs and the number of tokens controlled by the contract are public           | no       |
| owner_is_public               | bool | True if newly minted coins have public ownership as default                                | no       | 
| sealed_metadata_is_enabled    | bool | True if newly minted coins have sealed metadata                                            | no       |
| unwrapped_metadata_is_private | bool | True if sealed metadata remains private after unwrapping                                   | no       |
| minter_may_update_metadata    | bool | True if authorized minters may alter a token's metadata                                    | no       | 
| owner_may_update_metadata     | bool | True if a token owner may alter its metadata                                               | no       | 
| burn_is_enabled               | bool | True if burn functionality is enabled                                                      | no       |

## Minters
Minters returns the list of addresses that are authorized to mint tokens.  This query is not authenticated.

##### Request
```
{
	"minters": {}
}
```
##### Response
```
{
	"minters": {
		“minters”: [
			"list", "of", "authorized", "minters", "..."
		]
	}
}
```
| Name    | Type                        | Description                              | Optional | 
|---------|-----------------------------|------------------------------------------|----------|
| minters | array of string (HumanAddr) | List of addresses with minting authority | no       |

## RegisteredCodeHash
RegisteredCodeHash will display the code hash of the specified contract if it has registered its receiver interface and will indicate whether the contract implements [BatchReceiveNft](#receiver).

##### Request
```
{
	"registered_code_hash": {
		"contract": "address_of_the_contract_whose_registration_is_being_queried"
	}
}
```
| Name     | Type               | Description                                                           | Optional | Value If Omitted |
|----------|--------------------|-----------------------------------------------------------------------|----------|------------------|
| contract | string (HumanAddr) | The address of the contract whose registration info is being queried  | no       |                  |

##### Response
```
{
	"registered_code_hash": {
		"code_hash": "code_hash_of_the_registered_contract",
		"also_implements_batch_receive_nft": true|false
	}
}
```
| Name                               | Type   | Description                                                     | Optional | 
|------------------------------------|--------|-----------------------------------------------------------------|----------|
| code_hash                          | string | The code hash of the registered contract                        | yes      |
| also_implements_batch_receive_nft  | bool   | True if the registered contract also implements BatchReceiveNft | no       |

## NumTokens
NumTokens returns the number of tokens controlled by the contract.  If the contract's token supply is private, only an authenticated minter's address will be allowed to perform this query.

##### Request
```
{
	"num_tokens": {
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		}
	}
}
```
| Name   | Type                                  | Description                                                         | Optional | Value If Omitted |
|--------|---------------------------------------|---------------------------------------------------------------------|----------|------------------|
| viewer | [ViewerInfo (see below)](#viewerinfo) | The address and viewing key performing this query                   | yes      | nothing          |

##### Response
```
{
	"num_tokens": {
		"count": 99999
	}
}
```
| Name    | Type         | Description                                  | Optional | 
|---------|--------------|----------------------------------------------|----------|
| count   | number (u32) | Number of tokens controlled by this contract | no       |

### <a name="viewerinfo"></a>ViewerInfo
The ViewerInfo object provides the address and viewing key of the querier.  It is optionally provided in queries where public responses and address-specific responses will differ.
```
{
	"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
	"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
}
```
| Name        | Type               | Description                                                                                                           | Optional | Value If Omitted |
|-------------|--------------------|-----------------------------------------------------------------------------------------------------------------------|----------|------------------|
| address     | string (HumanAddr) | Address performing the query                                                                                          | no       |                  |
| viewing_key | string             | The querying address' viewing key                                                                                     | no       |                  |

## AllTokens
AllTokens returns an optionally paginated list of all the token IDs controlled by the contract.  If the contract's token supply is private, only an authenticated minter's address will be allowed to perform this query.  When paginating, supply the last token ID received in a response as the `start_after` string of the next query to continue listing where the previous query stopped.

##### Request
```
{
	"all_tokens": {
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
		"start_after": "optional_string_where_results_will_only_be_token_IDs_that_come_later_in_lexicographical_order",
		"limit": 10
	}
}
```
| Name        | Type                                  | Description                                                                              | Optional | Value If Omitted |
|-------------|---------------------------------------|------------------------------------------------------------------------------------------|----------|------------------|
| viewer      | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                                        | yes      | nothing          |
| start_after | string                                | Results will only list token IDs that come after this string in lexicographical order    | yes      | nothing          |
| limit       | number (u32)                          | Number of token IDs to return                                                            | yes      | 300              |

##### Response
```
{
	"token_list": {
		"tokens": [
			"list", "of", "token", "IDs", "controlled", "by", "the", "contract", "..."
		]
	}
}
```
| Name    | Type            | Description                                                          | Optional | 
|---------|-----------------|----------------------------------------------------------------------|----------|
| tokens  | array of string | A list of token IDs controlled by this contract                      | no       |

## IsUnwrapped
IsUnwrapped indicates whether the token has been unwrapped.  If sealed metadata is not enabled, all tokens are considered to be unwrapped.

##### Request
```
{
	"is_unwrapped": {
		"token_id": "ID_of_the_token_whose_unwrapped_status_is_being_queried"
	}
}
```
| Name        | Type   | Description                                                                              | Optional | Value If Omitted |
|-------------|--------|------------------------------------------------------------------------------------------|----------|------------------|
| token_id    | string | The ID of the token whose unwrapped status is being queried                              | no       |                  |

##### Response
```
{
	"is_unwrapped": {
		"token_is_unwrapped": true|false
	}
}
```
| Name                | Type | Description                                                          | Optional | 
|---------------------|------|----------------------------------------------------------------------|----------|
| token_is_unwrapped  | bool | True if the token is unwrapped (or sealed metadata is not enabled)   | no       |

## <a name="ownerof"></a>OwnerOf
OwnerOf returns the owner of the specified token if the querier is the owner or has been granted permission to view it.  If the querier is the owner, OwnerOf will also display all the addresses that have been given transfer permission.  The transfer approval list is provided as part of CW-721 compliance; however, the token owner is advised to use [NftDossier (see below)](#nftdossier) for a more complete list that includes view_owner and view_private_metadata approvals (which CW-721 is not capable of screening).  If no [viewer](#viewerinfo) is provided, OwnerOf will only display the owner if ownership is public for this token.

##### Request
```
{
	"owner_of": {
		"token_id": "ID_of_the_token_being_queried",
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
		"include_expired": true|false
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |
| viewer          | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                     | yes      | nothing          |
| include_expired | bool                                  | True if expired transfer approvals should be included in the response | yes      | false            |

##### Response
```
{
	"owner_of": {
		"owner": "address_of_the_token_owner",
		"approvals": [
			{
				"spender": "address_with_transfer_approval",
				"expires": "never"|{"at_height": 999999}|{"at_time":999999}
			},
			{
				"...": "..."
			}
		]
	}
}
```
| Name      | Type                                                 | Description                                              | Optional | 
|-----------|------------------------------------------------------|----------------------------------------------------------|----------|
| owner     | string (HumanAddr)                                   | Address of the token's owner                             | no       |
| approvals | array of [Cw721Approval (see below)](#cw721approval) | List of approvals to transfer this token                 | no       |

### <a name="cw721approval"></a>Cw721Approval
The Cw721Approval object is used to display CW-721-style approvals which are limited to only permission to transfer, as CW-721 does not enable ownership or metadata privacy.
```
{
	"spender": "address_with_transfer_approval",
	"expires": "never"|{"at_height": 999999}|{"at_time":999999}
}
```
| Name    | Type                                  | Description                                                                     | Optional | 
|---------|---------------------------------------|---------------------------------------------------------------------------------|----------|
| spender | string (HumanAddr)                    | Address whitelisted to transfer a token                                         | no       |
| expires | [Expiration (see above)](#expiration) | The expiration of this transfer approval.  Can be a blockheight, time, or never | no       |

## <a name="nftinfo"></a>NftInfo
NftInfo returns the public metadata of a token.  It follows CW-721 specification, which is based on ERC-721 Metadata JSON Schema.

##### Request
```
{
	"nft_info": {
		"token_id": "ID_of_the_token_being_queried"
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |

##### Response
```
{
	"nft_info": {
		"name": "optional_name_of_the_token",
		"description": "optional_description",
		"image": "optional_uri_containing_an_image_or_additional_metadata"
	}
}
```
| Name        | Type   | Description                                              | Optional | 
|-------------|--------|----------------------------------------------------------|----------|
| name        | string | Name of the token                                        | yes      |
| description | string | Token description                                        | yes      |
| image       | string | Uri to an image or additional metadata                   | yes      |

## AllNftInfo
AllNftInfo displays the result of both [OwnerOf](#ownerof) and [NftInfo](#nftinfo) in a single query.  This is provided for CW-721 compliance, but for more complete information about a token, use [NftDossier](#nftdossier), which will include private metadata and view_owner and view_private_metadata approvals if the querier is permitted to view this information.

##### Request
```
{
	"all_nft_info": {
		"token_id": "ID_of_the_token_being_queried",
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
		"include_expired": true|false
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |
| viewer          | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                     | yes      | nothing          |
| include_expired | bool                                  | True if expired transfer approvals should be included in the response | yes      | false            |

##### Response
```
{
	"all_nft_info": {
		"access": {
			"owner": "address_of_the_token_owner",
			"approvals": [
				{
					"spender": "address_with_transfer_approval",
					"expires": "never"|{"at_height": 999999}|{"at_time":999999}
				},
				{
					"...": "..."
				}
			]
		},
		"info": {
			"name": "optional_name_of_the_token",
			"description": "optional_description",
			"image": "optional_uri_containing_an_image_or_additional_metadata"
		}
	}
}
```
| Name        | Type                                              | Description                                                                 | Optional | 
|-------------|---------------------------------------------------|-----------------------------------------------------------------------------|----------|
| access      | [Cw721OwnerOfResponse (see below)](#cw721ownerof) | The token's owner and its transfer approvals if permitted to view this info | no       |
| info        | [Metadata (see above)](#metadata)                 | The token's public metadata                                                 | yes      |

### <a name="cw721ownerof"></a>Cw721OwnerOfResponse
The Cw721OwnerOfResponse object is used to display a token's owner if the querier has view_owner permission, and the token's transfer approvals if the querier is the token's owner.
```
{
	"owner": "address_of_the_token_owner",
	"approvals": [
		{
			"spender": "address_with_transfer_approval",
			"expires": "never"|{"at_height": 999999}|{"at_time":999999}
		},
		{
			"...": "..."
		}
	]
}
```
| Name      | Type                                                 | Description                                              | Optional | 
|-----------|------------------------------------------------------|----------------------------------------------------------|----------|
| owner     | string (HumanAddr)                                   | Address of the token's owner                             | yes      |
| approvals | array of [Cw721Approval (see above)](#cw721approval) | List of approvals to transfer this token                 | no       |

## PrivateMetadata
PrivateMetadata returns the private metadata of a token if the querier is permitted to view it.  It follows CW-721 metadata specification, which is based on ERC-721 Metadata JSON Schema.  If the metadata is sealed, no one is permitted to view it until it has been unwrapped with [Reveal](#reveal).  If no [viewer](#viewerinfo) is provided, PrivateMetadata will only display the private metadata if the private metadata is public for this token.

##### Request
```
{
	"private_metadata": {
		"token_id": "ID_of_the_token_being_queried",
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |
| viewer          | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                     | yes      | nothing          |

##### Response
```
{
	"private_metadata": {
		"name": "optional_private_name_of_the_token",
		"description": "optional_private_description",
		"image": "optional_private_uri_containing_an_image_or_additional_metadata"
	}
}
```
| Name        | Type   | Description                                                      | Optional | 
|-------------|--------|------------------------------------------------------------------|----------|
| name        | string | Private name of the token                                        | yes      |
| description | string | Private token description                                        | yes      |
| image       | string | Private uri to an image or additional metadata                   | yes      |

## <a name="nftdossier"></a>NftDossier
NftDossier returns all the information about a token that the viewer is permitted to view.  If no `viewer` is provided, NftDossier will only display the information that has been made public.  The response may include the owner, the public metadata, the private metadata, the reason the private metadata is not viewable, whether ownership is public, whether the private metadata is public, and if the querier is the owner, the approvals for this token as well as the inventory-wide approvals for the owner.

##### Request
```
{
	"nft_dossier": {
		"token_id": "ID_of_the_token_being_queried",
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
		"include_expired": true|false
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |
| viewer          | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                     | yes      | nothing          |
| include_expired | bool                                  | True if expired approvals should be included in the response          | yes      | false            |

##### Response
```
{
	"nft_dossier": {
		"owner": "address_of_the_token_owner",
		"public_metadata": {
			"name": "optional_public_name_of_the_token",
			"description": "optional_public_description",
			"image": "optional_public_uri_containing_an_image_or_additional_metadata"
		},
		"private_metadata": {
			"name": "optional_private_name_of_the_token",
			"description": "optional_private_description",
			"image": "optional_private_uri_containing_an_image_or_additional_metadata"
		},
		"display_private_metadata_error": "optional_error_describing_why_private_metadata_is_not_viewable_if_applicable",
		"owner_is_public": true|false,
		"public_ownership_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
		"private_metadata_is_public": true|false,
		"private_metadata_is_public_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
		"token_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
				"view_private_metadata_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
				"transfer_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
			},
			{
				"...": "..."
			}
		],
		"inventory_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
				"view_private_metadata_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
				"transfer_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
			},
			{
				"...": "..."
			}
		]
	}
}
```
| Name                                  | Type                                                  | Description                                                                            | Optional | 
|---------------------------------------|-------------------------------------------------------|----------------------------------------------------------------------------------------|----------|
| owner                                 | string (HumanAddr)                                    | Address of the token's owner                                                           | yes      |
| public_metadata                       | [Metadata (see above)](#metadata)                     | The token's public metadata                                                            | yes      |
| private_metadata                      | [Metadata (see above)](#metadata)                     | The token's private metadata                                                           | yes      |
| display_private_metadata_error        | string                                                | If the private metadata is not displayed, the corresponding error message              | yes      |
| owner_is_public                       | bool                                                  | True if ownership is public for this token                                             | no       |
| public_ownership_expiration           | [Expiration (see above)](#expiration)                 | When public ownership expires for this token.  Can be a blockheight, time, or never    | yes      |
| private_metadata_is_public            | bool                                                  | True if private metadata is public for this token                                      | no       |
| private_metadata_is_public_expiration | [Expiration (see above)](#expiration)                 | When public display of private metadata expires.  Can be a blockheight, time, or never | yes      |
| token_approvals                       | array of [Snip721Approval (see below)](#snipapproval) | List of approvals for this token                                                       | yes      |
| inventory_approvals                   | array of [Snip721Approval (see below)](#snipapproval) | List of inventory-wide approvals for the token's owner                                 | yes      |

### <a name="snipapproval"></a> Snip721Approval
The Snip721Approval object is used to display all the approvals (and their expirations) that have been granted to a whitelisted address.  The expiration field will be null if the whitelisted address does not have that corresponding permission type.
```
{
	"address": "whitelisted_address",
	"view_owner_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
	"view_private_metadata_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
	"transfer_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
}
```
| Name                             | Type                                  | Description                                                                                 | Optional | 
|----------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------|----------|
| address                          | string (HumanAddr)                    | The whitelisted address                                                                     | no       |
| view_owner_expiration            | [Expiration (see above)](#expiration) | The expiration for view_owner permission.  Can be a blockheight, time, or never             | yes      |
| view_private_metadata_expiration | [Expiration (see above)](#expiration) | The expiration for view__private_metadata permission.  Can be a blockheight, time, or never | yes      |
| transfer_expiration              | [Expiration (see above)](#expiration) | The expiration for transfer permission.  Can be a blockheight, time, or never               | yes      |

## TokenApprovals
TokenApprovals returns whether the owner and private metadata of a token is public, and lists all the approvals specific to this token.  Only the token's owner may perform TokenApprovals.

##### Request
```
{
	"token_approvals": {
		"token_id": "ID_of_the_token_being_queried",
		"viewing_key": "the_token_owner's_viewing_key",
		"include_expired": true|false
	}
}
```
| Name            | Type   | Description                                                           | Optional | Value If Omitted |
|-----------------|--------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string | ID of the token being queried                                         | no       |                  |
| viewing_key     | string | The token owner's viewing key                                         | no       |                  |
| include_expired | bool   | True if expired approvals should be included in the response          | yes      | false            |

##### Response
```
{
	"token_approvals": {
		"owner_is_public": true|false,
		"public_ownership_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
		"private_metadata_is_public": true|false,
		"private_metadata_is_public_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
		"token_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
				"view_private_metadata_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
				"transfer_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
			},
			{
				"...": "..."
			}
		],
	}
}
```
| Name                                  | Type                                                  | Description                                                                            | Optional | 
|---------------------------------------|-------------------------------------------------------|----------------------------------------------------------------------------------------|----------|
| owner_is_public                       | bool                                                  | True if ownership is public for this token                                             | no       |
| public_ownership_expiration           | [Expiration (see above)](#expiration)                 | When public ownership expires for this token.  Can be a blockheight, time, or never    | yes      |
| private_metadata_is_public            | bool                                                  | True if private metadata is public for this token                                      | no       |
| private_metadata_is_public_expiration | [Expiration (see above)](#expiration)                 | When public display of private metadata expires.  Can be a blockheight, time, or never | yes      |
| token_approvals                       | array of [Snip721Approval (see above)](#snipapproval) | List of approvals for this token                                                       | no       |

## ApprovedForAll
ApprovedForAll displays all the addresses that have approval to transfer all of the specified owner's tokens.  This is provided to comply with CW-721 specification, but because approvals are private on  Secret Network, if the `owner`'s viewing key is not provided, no approvals will be displayed.  For a more complete list of inventory-wide approvals, the owner should use [InventoryApprovals](#inventoryapprovals) which also includes view_owner and view_private_metadata approvals.

##### Request
```
{
	"approved_for_all": {
		"owner": "address_whose_approvals_are_being_queried",
		"viewing_key": "owner's_viewing_key"
		"include_expired": true|false
	}
}
```
| Name            | Type               | Description                                                                              | Optional | Value If Omitted |
|-----------------|--------------------|------------------------------------------------------------------------------------------|----------|------------------|
| owner           | string (HumanAddr) | The address whose approvals are being queried                                            | no       |                  |
| viewing_key     | string             | The owner's viewing key                                                                  | yes      | nothing          |
| include_expired | bool               | True if expired approvals should be included in the response                             | yes      | false            |

##### Response
```
{
	"approved_for_all": {
		"operators": [
				{
					"spender": "address_with_transfer_approval",
					"expires": "never"|{"at_height": 999999}|{"at_time":999999}
				},
				{
					"...": "..."
				}
		]
	}
}
```
| Name      | Type                                                 | Description                                              | Optional | 
|-----------|------------------------------------------------------|----------------------------------------------------------|----------|
| operators | array of [Cw721Approval (see above)](#cw721approval) | List of approvals to transfer all of the owner's tokens  | no       |

## <a name="inventoryapprovals"></a> InventoryApprovals
InventoryApprovals returns whether all the address' tokens have public ownership and/or public display of private metadata, and lists all the inventory-wide approvals the address has granted.

##### Request
```
{
	"inventory_approvals": {
		"address": "address_whose_approvals_are_being_queried",
		"viewing_key": "the_viewing_key_associated_with_this_address",
		"include_expired": true|false
	}
}
```
| Name            | Type               | Description                                                           | Optional | Value If Omitted |
|-----------------|--------------------|-----------------------------------------------------------------------|----------|------------------|
| address         | string (HumanAddr) | The address whose inventory-wide approvals are being queried          | no       |                  |
| viewing_key     | string             | The viewing key associated with this address                          | no       |                  |
| include_expired | bool               | True if expired approvals should be included in the response          | yes      | false            |

##### Response
```
{
	"inventory_approvals": {
		"owner_is_public": true|false,
		"public_ownership_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
		"private_metadata_is_public": true|false,
		"private_metadata_is_public_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
		"inventory_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
				"view_private_metadata_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
				"transfer_expiration": "never"|{"at_height": 999999}|{"at_time":999999},
			},
			{
				"...": "..."
			}
		],
	}
}
```
| Name                                  | Type                                                  | Description                                                                            | Optional | 
|---------------------------------------|-------------------------------------------------------|----------------------------------------------------------------------------------------|----------|
| owner_is_public                       | bool                                                  | True if ownership is public for all of this address' tokens                            | no       |
| public_ownership_expiration           | [Expiration (see above)](#expiration)                 | When public ownership expires for all tokens.  Can be a blockheight, time, or never    | yes      |
| private_metadata_is_public            | bool                                                  | True if private metadata is public for all of this address' tokens                     | no       |
| private_metadata_is_public_expiration | [Expiration (see above)](#expiration)                 | When public display of private metadata expires.  Can be a blockheight, time, or never | yes      |
| inventory_approvals                   | array of [Snip721Approval (see above)](#snipapproval) | List of inventory-wide approvals for this address                                      | no       |

## Tokens
Tokens displays an optionally paginated list of all the token IDs that belong to the specified owner.  It will only display the owner's tokens on which the querier has view_owner permission.  If no viewing key is provided, it will only display the owner's tokens that have public ownership.  When paginating, supply the last token ID received in a response as the `start_after` string of the next query to continue listing where the previous query stopped.

##### Request
```
{
	"tokens": {
		"owner": "address_whose_inventory_is_being_queried",
		"viewer": "address_of_the_querier_if_different_from_owner",
		"viewing_key": "querier's_viewing_key"
		"start_after": "optional_string_where_results_will_only_be_token_IDs_that_come_later_in_lexicographical_order",
		"limit": 10
	}
}
```
| Name        | Type               | Description                                                                              | Optional | Value If Omitted |
|-------------|--------------------|------------------------------------------------------------------------------------------|----------|------------------|
| owner       | string (HumanAddr) | The address whose inventory is being queried                                             | no       |                  |
| viewer      | string (HumanAddr) | The querier's address if different from the owner                                        | yes      | nothing          |
| viewing_key | string             | The querier's viewing key                                                                | yes      | nothing          |
| start_after | string             | Results will only list token IDs that come after this string in lexicographical order    | yes      | nothing          |
| limit       | number (u32)       | Number of token IDs to return                                                            | yes      | 30               |

##### Response
```
{
	"token_list": {
		"tokens": [
			"list", "of", "the", "owner's", "tokens", "..."
		]
	}
}
```
| Name    | Type            | Description                                                          | Optional | 
|---------|-----------------|----------------------------------------------------------------------|----------|
| tokens  | array of string | A list of token IDs owned by the specified `owner`                   | no       |

## <a name="verifyapproval"></a> VerifyTransferApproval
VerifyTransferApproval will verify that the specified address has approval to transfer the entire provided list of tokens.  As explained [above](#queryblockinfo), queries may experience a delay in revealing expired permissions, so it is possible that a transfer attempt will still fail even after being verified by VerifyTransferApproval.  If the address does not have transfer approval on all the tokens, the response will indicate the first token encountered that can not be transferred by the address.

##### Request
```
{
	"verify_transfer_approval": {
		"tokens": [
			"list", "of", "tokens", "to", "check", "for", "transfer", "approval", "..."
		],
		"address": "address_to_use_for_approval_checking",
		"viewing_key": "address'_viewing_key"
	}
}
```
| Name        | Type               | Description                                                                              | Optional | Value If Omitted |
|-------------|--------------------|------------------------------------------------------------------------------------------|----------|------------------|
| tokens      | array of string    | List of tokens to check for the address' transfer approval                               | no       |                  |
| address     | string (HumanAddr) | Address being checked for transfer approval                                              | no       |                  |
| viewing_key | string             | The address' viewing key                                                                 | no       |                  |

##### Response
```
{
	"verify_transfer_approval": {
		"approved_for_all": true|false,
		"first_unapproved_token": "token_id"
	}
}
```
| Name                   | Type   | Description                                                                       | Optional | 
|------------------------|--------|-----------------------------------------------------------------------------------|----------|
| approved_for_all       | bool   | True if the `address` has transfer approval on all the `tokens`                   | no       |
| first_unapproved_token | string | The first token in the list that the `address` does not have approval to transfer | yes      |

## TransactionHistory
TransactionHistory displays an optionally paginated list of transactions (mint, burn, and transfer) in reverse chronological order that involve the specified address.

##### Request
```
{
	"transaction_history": {
		"address": "address_whose_tx_history_is_being_queried",
		"viewing_key": "address'_viewing_key"
		"page": "optional_page_to_display",
		"page_size": 10
	}
}
```
| Name        | Type               | Description                                                                                                       | Optional | Value If Omitted |
|-------------|--------------------|-------------------------------------------------------------------------------------------------------------------|----------|------------------|
| address     | string (HumanAddr) | The address whose transaction history is being queried                                                            | no       |                  |
| viewing_key | string             | The address' viewing key                                                                                          | no       |                  |
| page        | number (u32)       | The page number to display, where the first transaction shown skips the page * page_size most recent transactions | yes      | 0                |
| page_size   | number (u32)       | Number of transactions to return                                                                                  | yes      | 30               |

##### Response
```
{
	"transaction_history": {
		"txs": [
			{
				"tx_id": 9999,
				"blockheight": 999999,
				"token_id": "token_involved_in_the_tx",
				"action": {
					"transfer": {
						"from": "previous_owner_of_the_token",
						"sender": "address_that_sent_the_token_if_different_than_the_previous_owner",
						"recipient": "new_owner_of_the_token"
					}
				},
				"memo": "optional_memo_for_the_tx"
			},
			{
				"tx_id": 9998,
				"blockheight": 999998,
				"token_id": "token_involved_in_the_tx",
				"action": {
					"mint": {
						"minter": "address_that_minted_the_token",
						"recipient": "owner_of_the_newly_minted_token"
					}
				},
				"memo": "optional_memo_for_the_tx"
			},
			{
				"tx_id": 9997,
				"blockheight": 999997,
				"token_id": "token_involved_in_the_tx",
				"action": {
					"burn": {
						"owner": "previous_owner_of_the_token",
						"burner": "address_that_burned_the_token_if_different_than_the_previous_owner",
					}
				},
				"memo": "optional_memo_for_the_tx"
			},
			{
				"...": "..."
			}
		],
	}
}
```
| Name | Type                           | Description                                                                            | Optional | 
|------|--------------------------------|----------------------------------------------------------------------------------------|----------|
| txs  | array of [Tx (see below)](#tx) | list of transactions in reverse chronological order that involve the specified address | no       |

### <a name="tx"></a> Tx
The Tx object contains all the information pertaining to a mint, burn, or transfer transaction.
```
{
	"tx_id": 9999,
	"blockheight": 999999,
	"token_id": "token_involved_in_the_tx",
	"action": { TxAction::Transfer | TxAction::Mint | TxAction::Burn },
	"memo": "optional_memo_for_the_tx"
}
```
| Name        | Type                              | Description                                                                             | Optional | 
|-------------|-----------------------------------|-----------------------------------------------------------------------------------------|----------|
| tx_id       | number (u64)                      | The transaction identifier                                                              | no       |
| blockheight | number (u64)                      | The number of the block that contains the transaction                                   | no       |
| token_id    | string                            | The token involved in the transaction                                                   | no       |
| action      | [TxAction (see below)](#txaction) | The type of transaction and the information specific to that type                       | no       |
| memo        | string                            | Memo for the transaction that is only viewable by addresses involved in the transaction | yes      |

### <a name="txaction"></a> TxAction
The TxAction object defines the type of transaction and holds the information specific to that type.

* TxAction::Mint
```
{
	"minter": "address_that_minted_the_token",
	"recipient": "owner_of_the_newly_minted_token"
}

```
| Name      | Type               | Description                                                                    | Optional | 
|-----------|--------------------|--------------------------------------------------------------------------------|----------|
| minter    | string (HumanAddr) | The address that minted the token                                              | no       |
| recipient | string (HumanAddr) | The address of the newly minted token's owner                                  | no       |

* TxAction::Transfer
```
{
	"from": "previous_owner_of_the_token",
	"sender": "address_that_sent_the_token_if_different_than_the_previous_owner",
	"recipient": "new_owner_of_the_token"
}

```
| Name      | Type               | Description                                                                    | Optional | 
|-----------|--------------------|--------------------------------------------------------------------------------|----------|
| from      | string (HumanAddr) | The previous owner of the token                                                | no       |
| sender    | string (HumanAddr) | The address that sent the token if different than the previous owner           | yes      |
| recipient | string (HumanAddr) | The new owner of the token                                                     | no       |

* TxAction::Burn
```
{
	"owner": "previous_owner_of_the_token",
	"burner": "address_that_burned_the_token_if_different_than_the_previous_owner",
}

```
| Name      | Type               | Description                                                                    | Optional | 
|-----------|--------------------|--------------------------------------------------------------------------------|----------|
| owner     | string (HumanAddr) | The previous owner of the token                                                | no       |
| burner    | string (HumanAddr) | The address that burned the token if different than the previous owner         | yes      |

# <a name="receiver"></a>Receiver Interface
When the token contract executes [SendNft](#send) and [BatchSendNft](#batchsend) messages, it will perform a callback to the receiving contract's receiver interface if the recipient had registered its code hash using [RegisterReceiveNft](#registerreceive).  BatchReceiveNft is preferred over ReceiveNft, because ReceiveNft does not allow the recipient to know who sent the token, only its previous owner, and ReceiveNft can only process one token.  So it is inefficient when sending multiple tokens to the same contract (a deck of game cards for instance).  ReceiveNft primarily exists just to maintain CW-721 compliance, and if the receiving contract registered that it implements BatchReceiveNft, BatchReceiveNft will be called, even when there is only one token_id in the message.
<a name="cwsender"></a>
Also, it should be noted that the CW-721 `sender` field is inaccurately named, because it is used to hold the address the token came from, not the address that sent it (which is not always the same).  The name is reluctantly kept in ReceiveNft to maintain CW-721 compliance, but BatchReceiveNft uses `sender` to hold the sending address (which matches both its true role and its SNIP-20 Receive counterpart).  Any contract that is implementing both Receiver Interfaces must be sure that the ReceiveNft `sender` field is actually processed like a BatchReceiveNft `from` field.  Again, apologies for any confusion caused by propagating inaccuracies, but because InterNFT is planning on using CW-721 standards, compliance with CW-721 might be necessary.

## ReceiveNft
ReceiveNft may be a HandleMsg variant of any contract that wants to implement a receiver interface.  BatchReceiveNft, which is more informative and more efficient, is preferred over ReceiveNft.  
```
{
	"receive_nft": {
		"sender": "address_of_the_previous_owner_of_the_token",
		"token_id": "ID_of_the_sent_token",
		"msg": "optional_base64_encoded_Binary_message_used_to_control_receiving_logic"
	}
}
```
| Name     | Type                           | Description                                                                                            | Optional | Value If Omitted |
|----------|--------------------------------|--------------------------------------------------------------------------------------------------------|----------|------------------|
| sender   | string (HumanAddr)             | Address of the token's previous owner ([see above](#cwsender) about this inaccurate naming convention) | no       |                  |
| token_id | string                         | ID of the sent token                                                                                   | no       |                  |
| msg      | string (base64 encoded Binary) | Msg used to control receiving logic                                                                    | yes      | nothing          |

## BatchReceiveNft
BatchReceiveNft may be a HandleMsg variant of any contract that wants to implement a receiver interface.  BatchReceiveNft, which is more informative and more efficient, is preferred over ReceiveNft.
```
{
	"batch_receive_nft": {
		"sender": "address_that_sent_the_tokens",
		"from": "address_of_the_previous_owner_of_the_tokens",
		"token_ids": [
			"list", "of", "tokens", "sent", "..."
		],
		"msg": "optional_base64_encoded_Binary_message_used_to_control_receiving_logic"
	}
}
```
| Name      | Type                           | Description                                                                                                              | Optional | Value If Omitted |
|-----------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| sender    | string (HumanAddr)             | Address that sent the tokens (this field has no ReceiveNft equivalent, [see above](#cwsender))                           | no       |                  |
| from      | string (HumanAddr)             | Address of the tokens' previous owner (this field is equivalent to the ReceiveNft `sender` field, [see above](#cwsender))| no       |                  |
| token_ids | array of string                | List of the tokens sent                                                                                                  | no       |                  |
| msg       | string (base64 encoded Binary) | Msg used to control receiving logic                                                                                      | yes      | nothing          |