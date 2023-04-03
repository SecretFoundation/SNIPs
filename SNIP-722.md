# SNIP-722 - Extension of [SNIP-721](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md) for Badges, POAPs, and Non-Transferable NFTs

This document describes several optional enhancements to the SNIP-721 specification.  Contracts that support the existing SNIP-721 standard are still considered SNIP-721 compliant.  Clients should be written so as to benefit from these SNIP-722 features when available, but provide fallbacks for when these features are not available.

The features specified in this document enable a SNIP-722 compliant contract to be used for badges and POAPs as well as non-transferable tokens.

* [Badges/POAPs](#badges)

	Queries
    * [ImplementsTokenSubtype](#ImplementsTokenSubtype)

* [Non-Transferable Tokens](#Non-Transferable-Tokens)

    Messages
    * [MintNft](#mintnft)
    * [BatchMintNft](#batchmintnft)

    Queries
    * [NftDossier](#nftdossier)
    * [IsTransferable](#istransferable)
    * [ImplementsNonTransferableTokens](#implementsnontransferabletokens)

## <a name="badges"></a>Badges/POAPs

The primary update for Badges/POAPs is the addition of a `token_subtype` field in the [Metadata](#metadata) [Extension](#extension) struct.  This field is intended to be used by applications in order to differentiate NFTs that are used as Badges/POAPs so that they can be displayed as such because they will be used for things like trophies, achievements, proof of attendence, etc...  An example of the definition of Metadata and Extension as used by Stashh is included below.


#### <a name="metadata"></a>Metadata
This is the metadata for a token that follows CW-721 metadata specification, which is based on ERC721 Metadata JSON Schema.  The [reference implementation](https://github.com/baedrik/snip721-reference-impl) will throw an error if both `token_uri` and `extension` are provided. 
```
{
	"token_uri": "optional_uri_pointing_to_off-chain_JSON_metadata",
	"extension": {
		"...": "..."
	}
}
```
| Name      | Type                                | Description                                                                          | Optional | Value If Omitted     |
|-----------|-------------------------------------|--------------------------------------------------------------------------------------|----------|----------------------|
| token_uri | string                              | Uri pointing to off-chain JSON metadata                                              | yes      | nothing              |
| extension | [Extension (see below)](#extension) | Data structure defining on-chain metadata                                            | yes      | nothing              |

The reference implementation will throw an error if both `token_uri` and `extension` are provided.

#### <a name="extension"></a>Extension
This is an on-chain metadata extension struct that conforms to the Stashh metadata standard (which in turn implements https://docs.opensea.io/docs/metadata-standards).  Urls should be prefixed with `http://`, `https://`, `ipfs://`, or `ar://`.
```
{
	"image": "optional_image_url",
	"image_data": "optional_raw_svg_image_data",
	"external_url": "optional_url_to_view_token_on_your_site",
	"description": "optional_token_description",
	"name": "optional_token_name",
	"attributes": [
		{
			"display_type": "optional_display_format_for_numerical_traits",
			"trait_type": "optional_name_of_the_trait",
			"value": "trait value",
			"max_value": "optional_max_value_for_numerical_traits"
		},
		{
			"...": "...",
		},
	],
	"background_color": "optional_six-character_hexadecimal_background_color_(without_pre-pended_`#`)",
	"animation_url": "optional_url_to_multimedia_file",
	"youtube_url": "optional_url_to_a_YouTube_video",
	"media": [
		{
			"file_type": "optional_file_type",
			"extension": "optional_file_extension",
			"authentication": {
				"key": "optional_decryption_key_or_password",
				"user": "optional_username_for_authentication"
			},
			"url": "url_pointing_to_the_multimedia_file"
		},
		{
			"...": "...",
		},
	],
	"protected_attributes": [ "list", "of_attributes", "whose_types", "are_public", "but_values", "are_private" ],
	"token_subtype": "badge"
}
```
| Name                 | Type                                         | Description                                                                          | Optional | Value If Omitted     |
|----------------------|----------------------------------------------|--------------------------------------------------------------------------------------|----------|----------------------|
| image                | string                                       | Url to the token's image                                                             | yes      | nothing              |
| image_data           | string                                       | Raw SVG image data that should only be used if there is no `image` field             | yes      | nothing              |
| external_url         | string                                       | Url to view the token on your site                                                   | yes      | nothing              |
| description          | string                                       | Text description of the token                                                        | yes      | nothing              |
| name                 | string                                       | Name of the token                                                                    | yes      | nothing              |
| attributes           | array of [Trait (see below)](#trait)         | Token's attributes                                                                   | yes      | nothing              |
| background_color     | string                                       | Background color represented as a six-character hexadecimal without a pre-pended #   | yes      | nothing              |
| animation_url        | string                                       | Url to a multimedia file                                                             | yes      | nothing              |
| youtube_url          | string                                       | Url to a YouTube video                                                               | yes      | nothing              |
| media                | array of [MediaFile (see below)](#mediafile) | List of multimedia files using Stashh specifications                                 | yes      | nothing              |
| protected_attributes | array of string                              | List of attributes whose types are public but whose values are private               | yes      | nothing              |
| token_subtype        | string                                       | token subtype used to signify what the NFT is used for, such as "badge"              | yes      | nothing              |

#### <a name="trait"></a>Trait
Trait describes a token attribute as defined in https://docs.opensea.io/docs/metadata-standards.
```
{
	"display_type": "optional_display_format_for_numerical_traits",
	"trait_type": "optional_name_of_the_trait",
	"value": "trait value",
	"max_value": "optional_max_value_for_numerical_traits"
}
```
| Name         | Type   | Description                                                                          | Optional | Value If Omitted     |
|--------------|--------|--------------------------------------------------------------------------------------|----------|----------------------|
| display_type | string | Display format for numerical traits                                                  | yes      | nothing              |
| trait_type   | string | Name of the trait                                                                    | yes      | nothing              |
| value        | string | Trait value                                                                          | no       |                      |
| max_value    | string | Maximum value for this numerical trait                                               | yes      | nothing              |

#### <a name="mediafile"></a>MediaFile
MediaFile is the data structure used by Stashh to reference off-chain multimedia files.  It allows for hosted files to be encrypted or authenticated with basic authentication, and for the decryption key or username/password to also be included in the on-chain private metadata.  Urls should be prefixed with `http://`, `https://`, `ipfs://`, or `ar://`.
```
{
	"file_type": "optional_file_type",
	"extension": "optional_file_extension",
	"authentication": {
		"key": "optional_decryption_key_or_password",
		"user": "optional_username_for_authentication"
	},
	"url": "url_pointing_to_the_multimedia_file"
}
```
| Name           | Type                                          | Description                                                                                 | Optional | Value If Omitted     |
|----------------|-----------------------------------------------|---------------------------------------------------------------------------------------------|----------|----------------------|
| file_type      | string                                        | File type.  Stashh currently uses: "image", "video", "audio", "text", "font", "application" | yes      | nothing              |
| extension      | string                                        | File extension                                                                              | yes      | nothing              |
| authentication | [Authentication (see below)](#authentication) | Credentials or decryption key for a protected file                                          | yes      | nothing              |
| url            | string                                        | Url to the multimedia file                                                                  | no       |                      |

#### <a name="authentication"></a>Authentication
Authentication is used to provide the decryption key or username/password for protected files.
```
{
	"key": "optional_decryption_key_or_password",
	"user": "optional_username_for_authentication"
}
```
| Name | Type   | Description                                                                          | Optional | Value If Omitted     |
|------|--------|--------------------------------------------------------------------------------------|----------|----------------------|
| key  | string | Decryption key or password                                                           | yes      | nothing              |
| user | string | Username for basic authentication                                                    | yes      | nothing              |

## Queries

### ImplementsTokenSubtype
ImplementsTokenSubtype indicates whether the contract implements the `token_subtype` Extension field.  Because legacy SNIP-721 contracts do not implement this query and do not implement token subtypes, any use of this query should always check for an error response, and if the response is an error, it can be considered that the contract does not implement subtypes.  Because message parsing ignores input fields that a contract does not expect, this query should be used before attempting a message that uses the `token_subtype` Extension field.  If the message is sent to a SNIP-721 contract that does not implement `token_subtype`, that field will just be ignored and the resulting NFT will still be created/updated, but without a `token_subtype`.

##### Request
```
{
	"implements_token_subtype": {}
}
```
##### Response
```
{
	"implements_token_subtype": {
		"is_enabled": true | false
	}
}
```
| Name        | Type | Description                                                             | Optional | 
|-------------|------|-------------------------------------------------------------------------|----------|
| is_enabled  | bool | True if the contract implements token subtypes                          | no       |

## Non-Transferable Tokens

Non-transferable tokens are NFTs that can never have an owner other than the address it was minted to.  In order to provide the owner of a non-transferable token a way to get rid of an unwanted NFT, non-transferable tokens must always have the ability to be burned (see [here](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#burning)).  Also, it should be noted that setting royalties for a non-transferable token has no purpose, because it can never be transferred as part of a sale.  Because the intent of [VerifyTransferApproval](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#verifytransferapproval) is to provide contracts a way to know before-hand whether an attempt to transfer tokens will fail, the [reference implementation](https://github.com/baedrik/snip721-reference-impl) has modified VerifyTransferApproval to consider any non-transferable token as unapproved for transfer.

## Messages

The following are examples of how the [reference implementation](https://github.com/baedrik/snip721-reference-impl) of non-transferable tokens provides the ability to specify whether an NFT should be transferable at the time it is minted.

### MintNft
MintNft mints a single token.  

##### Request
```
{
	"mint_nft": {
		"token_id": "optional_ID_of_new_token",
		"owner": "optional_address_the_new_token_will_be_minted_to",
		"public_metadata": {
			"token_uri": "optional_uri_pointing_to_off-chain_JSON_metadata",
			"extension": {
				"...": "..."
			}
		},
		"private_metadata": {
			"token_uri": "optional_uri_pointing_to_off-chain_JSON_metadata",
			"extension": {
				"...": "..."
			}
		},
		"serial_number": {
			"mint_run": 3,
			"serial_number": 67,
			"quantity_minted_this_run": 1000,
		},
		"royalty_info": {
			"decimal_places_in_rates": 4,
			"royalties": [
				{
					"recipient": "address_that_should_be_paid_this_royalty",
					"rate": 100,
				},
				{
					"...": "..."
				}
			],
		},
		"transferable": true | false,
		"memo": "optional_memo_for_the_mint_tx",
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name             | Type                                      | Description                                                                                   | Optional | Value If Omitted     |
|------------------|-------------------------------------------|-----------------------------------------------------------------------------------------------|----------|----------------------|
| token_id         | string                                    | Identifier for the token to be minted                                                         | yes      | minting order number |
| owner            | string (HumanAddr)                        | Address of the owner of the minted token                                                      | yes      | env.message.sender   |
| public_metadata  | [Metadata (see above)](#metadata)         | The metadata that is publicly viewable                                                        | yes      | nothing              |
| private_metadata | [Metadata (see above)](#metadata)         | The metadata that is viewable only by the token owner and addresses the owner has whitelisted | yes      | nothing              |
| serial_number    | [SerialNumber](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#serialnumber) | The SerialNumber for this token          | yes      | nothing              |
| royalty_info     | [RoyaltyInfo](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#royaltyinfo)   | RoyaltyInfo for this token               | yes      | default RoyaltyInfo  |
| transferable     | bool                                      | True if the minted token should be transferable                                               | yes      | true                 |
| memo             | string                                    | `memo` for the mint tx that is only viewable by addresses involved in the mint (minter, owner)| yes      | nothing              |
| padding          | string                                    | An ignored string that can be used to maintain constant message length                        | yes      | nothing              |

Setting royalties for a non-transferable token has no purpose, because it can never be transferred as part of a sale.

##### Response
```
{
	"mint_nft": {
		"token_id": "ID_of_minted_token",
	}
}
```
The ID of the minted token should also be returned in a LogAttribute with the key `minted`.


### BatchMintNft
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
					"token_uri": "optional_uri_pointing_to_off-chain_JSON_metadata",
					"extension": {
						"...": "..."
					}
				},
				"private_metadata": {
					"token_uri": "optional_uri_pointing_to_off-chain_JSON_metadata",
					"extension": {
						"...": "..."
					}
				},
				"serial_number": {
					"mint_run": 3,
					"serial_number": 67,
					"quantity_minted_this_run": 1000,
				},
				"royalty_info": {
					"decimal_places_in_rates": 4,
					"royalties": [
						{
							"recipient": "address_that_should_be_paid_this_royalty",
							"rate": 100,
						},
						{
							"...": "..."
						}
					],
				},
				"transferable": true | false,
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
| padding | string                                | An ignored string that can be used to maintain constant message length | yes      | nothing          |

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
The IDs of the minted tokens should also be returned in a LogAttribute with the key `minted`.

#### Mint
The Mint object defines the data necessary to mint one token.
```
{
	"token_id": "optional_ID_of_new_token",
	"owner": "optional_address_the_new_token_will_be_minted_to",
	"public_metadata": {
		"token_uri": "optional_uri_pointing_to_off-chain_JSON_metadata",
		"extension": {
			"...": "..."
		}
	},
	"private_metadata": {
		"token_uri": "optional_uri_pointing_to_off-chain_JSON_metadata",
		"extension": {
			"...": "..."
		}
	},
	"serial_number": {
		"mint_run": 3,
		"serial_number": 67,
		"quantity_minted_this_run": 1000,
	},
	"royalty_info": {
		"decimal_places_in_rates": 4,
		"royalties": [
			{
				"recipient": "address_that_should_be_paid_this_royalty",
				"rate": 100,
			},
			{
				"...": "..."
			}
		],
	},
	"transferable": true | false,
	"memo": "optional_memo_for_the_mint_tx"
}
```
| Name             | Type                                      | Description                                                                                    | Optional | Value If Omitted     |
|------------------|-------------------------------------------|------------------------------------------------------------------------------------------------|----------|----------------------|
| token_id         | string                                    | Identifier for the token to be minted                                                          | yes      | minting order number |
| owner            | string (HumanAddr)                        | Address of the owner of the minted token                                                       | yes      | env.message.sender   |
| public_metadata  | [Metadata (see above)](#metadata)         | The metadata that is publicly viewable                                                         | yes      | nothing              |
| private_metadata | [Metadata (see above)](#metadata)         | The metadata that is viewable only by the token owner and addresses the owner has whitelisted  | yes      | nothing              |
| serial_number    | [SerialNumber](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#serialnumber) | The SerialNumber for this token           | yes      | nothing              |
| royalty_info     | [RoyaltyInfo](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#royaltyinfo)   | RoyaltyInfo for this token                | yes      | default RoyaltyInfo  |
| transferable     | bool                                      | True if the minted token should be transferable                                                | yes      | true                 |
| memo             | string                                    | `memo` for the mint tx that is only viewable by addresses involved in the mint (minter, owner) | yes      | nothing              |

Setting royalties for a non-transferable token has no purpose, because it can never be transferred as part of a sale.

## Queries

### NftDossier
SNIP-722 adds a `transferable` field to the [NftDossier response of SNIP-721](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#nftdossier).  NftDossier returns all the information about a token that the viewer is permitted to view.  If no [viewer](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#viewerinfo) is provided, NftDossier will only display the information that has been made public.  The response may include the owner, the public metadata, the private metadata, the reason the private metadata is not viewable, the royalty information, the mint run information, whether the token is transferable, whether ownership is public, whether the private metadata is public, and (if the querier is the owner,) the approvals for this token as well as the inventory-wide approvals for the owner.  The implementation may choose to hide royalty recipient addresses.  See [here](https://github.com/baedrik/snip721-reference-impl/blob/master/README.md#nftdossier) for a description of how the reference implementation determines who is permitted to view royalty recipient addresses.

##### Request
```
{
	"nft_dossier": {
		"token_id": "ID_of_the_token_being_queried",
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
		"include_expired": true | false
	}
}
```
| Name            | Type                                                                                       | Description                                                           | Optional | Value If Omitted |
|-----------------|--------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                                                                     | ID of the token being queried                                         | no       |                  |
| viewer          | [ViewerInfo](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#viewerinfo) | The address and viewing key performing this query                     | yes      | nothing          |
| include_expired | bool                                                                                       | True if expired approvals should be included in the response          | yes      | false            |

##### Response
```
{
	"nft_dossier": {
		"owner": "address_of_the_token_owner",
		"public_metadata": {
			"token_uri": "optional_uri_pointing_to_off-chain_JSON_metadata",
			"extension": {
				"...": "..."
			}
		},
		"private_metadata": {
			"token_uri": "optional_uri_pointing_to_off-chain_JSON_metadata",
			"extension": {
				"...": "..."
			}
		},
		"display_private_metadata_error": "optional_error_describing_why_private_metadata_is_not_viewable_if_applicable",
		"royalty_info": {
			"decimal_places_in_rates": 4,
			"royalties": [
				{
					"recipient": "optional_address_that_should_be_paid_this_royalty",
					"rate": 100,
				},
				{
					"...": "..."
				}
			],
		},
		"mint_run_info": {
			"collection_creator": "optional_address_that_instantiated_this_contract",
			"token_creator": "optional_address_that_minted_this_token",
			"time_of_minting": 999999,
			"mint_run": 3,
			"serial_number": 67,
			"quantity_minted_this_run": 1000,
		},
		"transferable": true | false,
		"owner_is_public": true | false,
		"public_ownership_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
		"private_metadata_is_public": true | false,
		"private_metadata_is_public_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
		"token_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"view_private_metadata_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"transfer_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
			},
			{
				"...": "..."
			}
		],
		"inventory_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"view_private_metadata_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"transfer_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
			},
			{
				"...": "..."
			}
		]
	}
}
```
| Name                                  | Type                                                  | Description                                                                                               | Optional |
|---------------------------------------|-------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|----------|
| owner                                 | string (HumanAddr)                                    | Address of the token's owner                                                                              | yes      |
| public_metadata                       | [Metadata (see above)](#metadata)                     | The token's public metadata                                                                               | yes      |
| private_metadata                      | [Metadata (see above)](#metadata)                     | The token's private metadata                                                                              | yes      |
| display_private_metadata_error        | string                                                | If the private metadata is not displayed, the corresponding error message                                 | yes      |
| royalty_info                          | [RoyaltyInfo](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#royaltyinfo) | The token's RoyaltyInfo                                            | yes      |
| mint_run_info                         | [MintRunInfo](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#mintruninfo) | The token's MintRunInfo                                            | yes      |
| transferable                          | bool                                                  | True if this token is transferable                                                                        | no*      |
| owner_is_public                       | bool                                                  | True if ownership is public for this token                                                                | no       |
| public_ownership_expiration           | [Expiration](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#expiration)   | When public ownership expires for this token.  Can be a blockheight, time, or never    | yes      |
| private_metadata_is_public            | bool                                                  | True if private metadata is public for this token                                                         | no       |
| private_metadata_is_public_expiration | [Expiration](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#expiration)   | When public display of private metadata expires.  Can be a blockheight, time, or never | yes      |
| token_approvals                       | array of [Snip721Approval](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#snip721approval)| List of approvals for this token                      | yes      |
| inventory_approvals                   | array of [Snip721Approval](https://github.com/SecretFoundation/SNIPs/blob/master/SNIP-721.md#snip721approval)| List of inventory-wide approvals for the token's owner| yes      |

The `transferable` field is mandatory for SNIP-722 compliant contracts, but because SNIP-722 is an optional extension to SNIP-721, any NftDossier response that does not include the field can be considered to come from a contract that only implements transferable tokens (considered equivalent to `transferable` = true)

### IsTransferable
IsTransferable indicates whether the token is transferable.  This query is not authenticated.

##### Request
```
{
	"is_transferable": {
		"token_id": "ID_of_the_token_whose_transferability_is_being_queried"
	}
}
```
| Name        | Type   | Description                                                                              | Optional | Value If Omitted |
|-------------|--------|------------------------------------------------------------------------------------------|----------|------------------|
| token_id    | string | The ID of the token whose transferability is being queried                               | no       |                  |

##### Response
```
{
	"is_transferable": {
		"token_is_transferable": true | false
	}
}
```
| Name                   | Type | Description                                                             | Optional | 
|------------------------|------|-------------------------------------------------------------------------|----------|
| token_is_transferable  | bool | True if the token is transferable                                       | no       |

### ImplementsNonTransferableTokens
ImplementsNonTransferableTokens indicates whether the contract implements non-transferable tokens.  Because legacy SNIP-721 contracts do not implement this query and do not implement non-transferable tokens, any use of this query should always check for an error response, and if the response is an error, it can be considered that the contract does not implement non-transferable tokens.  Because message parsing ignores input fields that a contract does not expect, this query should be used before attempting to mint a non-transferable token.  If the message is sent to a SNIP-721 contract that does not implement non-transferable tokens, the `transferable` field will just be ignored and the resulting NFT will still be created, but will always be transferable.

##### Request
```
{
	"implements_non_transferable_tokens": {}
}
```
##### Response
```
{
	"implements_non_transferable_tokens": {
		"is_enabled": true | false
	}
}
```
| Name        | Type | Description                                                             | Optional | 
|-------------|------|-------------------------------------------------------------------------|----------|
| is_enabled  | bool | True if the contract implements non-transferable tokens                 | no       |
