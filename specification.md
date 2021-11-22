# MVP Specification

[] Bidder can submit a bid for uri in ETH
[] Bidder can receive token upon accepted bid

[] Owner can prove ownership of uri
[] Owner can accept/reject bid for uri


# User Journey

## Bidder

### Bid Journey
1. Bidder identifies an asset they want to purchase
2. Bidder copies Asset URI
3. Bidder navigates to ACL
4. Bidder submits Asset URI and bid in ETH to ACL

### Accept Journey
1. Owner is notified of bid by third-party who receives tokens for relaying messages that result in successful sales
2. Owner reviews and accepts bid
3. Owner proves ownership 
4. Owner mints Asset token and transfers it to winning bidder

# Core Concepts

1. Asset
2. Market
3. Ownership

## Asset

Assets are digital property rights and their relationship to a resource or a set of resources.

Assets can be created by anyone *using the ACL protocol*.

Assets are described by their ID, URI, and metadata.

Examples include:
    - image files
        - the right to edit the source file
    - music files
        - the right to edit the source file
    - video files
        - the right to edit the source file
    - website folders
        - the right to edit the folders
    - application packages
        - the right to edit the application
        - the right to call the functions
        - the right to 
    - media hosting websites
        - the right to upload
        - the right to edit the media
        - the right to edit the media's metadata
        - the right to 


## Ownership

Ownership is a mapping from an asset to an address.

An asset is mapped to an address once the address proves it is able to exercise the digital right it's attempting to claim as its property.

The Ownership Oracle service verifies proof-of-ownership by searching the listed URI for data signed by the address in question

The ownership mapping can only be written to by the Ownership Oracle. 

## Market




# Design

## Globals

- struct Asset
    - string `assetURI` 
    - address owner

//maps tokenId to Asset
- mapping(uint => Asset) public assets;

//maps assetURI to mapping of address to ownershipHash
- mapping(string => mapping (address => string)) public ownership;

- struct Bid
    - uint amount
    - address bidder
    - bool live

//maps tokenId to mapping from bidder to bid 
- mapping(uint256 => mapping(address => Bid)) private _assetBidders;

//tracks ERC721 tokens minted
- Counters.Counter private _tokenIdTracker;


## Functions

### Market Functionality

- `submitBid(string assetURI, amount uint256, address bidder)`
    - send ETH to the contract
    - create new `Bid` 
        - store `amount`, `msg.sender`
        - set `live` to `TRUE`
    - store in `_assetBidders`
        - set `tokenId` equal to `_tokenIdTracker`
        - increment `_tokenIdTracker`
        - store new `Bid` as value
    - store in `_tokenURIs`
        - store `tokenId` as key
        - store `assetURI` as value

- `removeBid(uint8 tokenId, address bidder)`
    - find bid using `tokenId` in `_assetBidders`
    - refund bid `amount` to `bidder`
    - delete bid
        - set bid equal to `0`

- `acceptBid(uint8 tokenId, address bidder)`
    - call `isOwner()` with `msg.sender` and `tokenId`
    - if false:
        - query `assets` to return the `assetURI` 
        - call `_verifyOwnership()` with `assetURI`
        - call `isOwner()` with `msg.sender` and `tokenId`
        - if false, `err ("Only an owner can accept bids for this asset)`
    - else:
        - call `_mintAndTransfer()`
        - sets `Live` to `FALSE` on `_assetBidders` for this `tokenId` and `address`
        - new token is minted and transferred to bidder

- `rejectBid(uint8 tokenId)`
    - function calls `isOwner()` with `msg.sender` and `tokenId`
    - if false:
        - querys `assets` to return the `assetURI` 
        - function calls `_verifyOwnership()` with the `assetURI`
        - function calls `isOwner()` with `msg.sender` and `tokenId`
        - if false, `err ("Only an owner can reject bids for this asset)`
    - else: 
        - sets `Live` to `FALSE` on `_assetBidders` for this `tokenId` and `address`
        - refund any other bids

- `rejectAllBids(uint8 tokenId)`
    - function calls `isOwner()` with `msg.sender` and `tokenId`
    - if false:
        - function calls `_verifyOwnership()` with the `assetURI`
        - function calls `isOwner()` with `msg.sender` and `tokenId`
        - if false, `err ("Only an owner can reject bids for this asset)`
    - else: 
        - sets `Live` to `FALSE` on all `Bids` in `_assetBidders` for this `tokenId`
        - refunds all bids


### Asset Functionality

- `generateOwnershipHash(string assetURI, address owner)`
    - concatenate `assetURI` and `msg.sender`'s `address` then hash
    - sign hash with `msg.sender`'s private key
    - store hash in `ownership` as `ownershipHash` against the `assetURI` and `msg.sender`'s address 

- `isOwner(uint8 tokenId, address owner) public view returns (bool)`
    - search `ownership` with `tokenId`
    - compare returned `address` with `owner`
    - return bool


## Private Functions

- `_mintAndTransfer(uint8 tokenId)`
    - mints ERC721 token and sends it to the successful bidder

- `_verifyOwnership(string assetURI, address owner)`
    - `assetURI` is used to query `ownership` and return `ownershipHash`
    - if `ownershipHash` exists:
        - send transaction to Ownership Oracle with `assetURI` and `ownershipHash`
        - if response is `TRUE`:
            - retrieve `tokenId` from `_tokenURIs`
            - update `assets` with `owner`
        else:
            - return `"Not an Owner"`    
    else:
        - return `"No ownership hash found"`


## External Services

- Ownership Oracle
    - Ownership oracle makes a request to the domain in the submitted `assetURI`
    - The oracle downloads the returned file and attempts to match the ownership hash
    - If the ownership hash is matched, returns `TRUE`
    - If the ownership hash is not matched, returns `FALSE`
