# MVP Specification

[ ] Bidder can submit a bid for asset in ETH
[ ] Bidder can receive token upon accepted bid

[ ] Owner can prove ownership of asset
[ ] Owner can accept/reject bid for asset


# User Journey

## Bidder

### Bid Journey
1. Bidder identifies an asset they want to purchase
2. Bidder copies Asset URI
3. Bidder navigates to ACL
4. Bidder submits Asset URI and bid in ETH to ACL

### Accept Journey
1. Owner is notified of bid by third-party
2. Owner reviews and accepts bid
3. Owner proves ownership 
4. Owner mints Asset token and transfers it to winning bidder

# Core Concepts

1. Asset
2. Ownership
3. Notification
4. Market

## Asset

- Assets are digital property rights and their relationship to a resource or a set of resources.

- Assets can be created by anyone *using the ACL protocol*.

- Assets are described by their ID, URI, and metadata.

[TODO] Decide how metadata will be managed, open or controlled?

## Ownership

- Ownership is a mapping from an asset to an address.

- Only one owner is permitted.

- An asset is mapped to an address once the address proves it is able to exercise the digital right it's attempting to claim as its property.

- Prospective owners can generate a unique hash or file using their signature from the ownership service.

- The Ownership Oracle service verifies proof-of-ownership by searching the listed URI for the Ownership hash or file signed by the prospective owner's address. 

[TODO] Describe how ownership is proven for a set of example assets
[TODO] Research proof-of-ownership oracles 

## Notification

- The protocol incentivises buyers to seek out and bid on valuable assets ahead of those asset owners entering the market.

- Therefore, notification is the process by which owners are notified of bids on their assets.

[TODO] Decide if the protocol incentivises notification and, if so, describe how 

## Market

- Market is a collection of bids stored against an asset as a continuous auction.

- Any user can submit a bid for any asset on the web directly to the network.

- Users send the bid amount to the contract with their bid and a valid asset object.

- Bids can only be accepted or rejected by an owner.

- Once a bid is accepted, the amount is transferred to the owner, the token is minted and transferred to the bidder, and the owner is updated to the bidder.

[TODO] Investigate auction designs where bids are collected for tokens before they are minted

## Examples

### Websites
- the right to edit the website
- the right to control access to the website
- the right to manage website domain

### Applications & Interfaces
- the right to edit the application
- the right to access application functions
- the right to access application storage
- the right to control access to application functions and/or storage

### Hosted Media
- the right to upload
- the right to edit the media
- the right to edit the media's metadata
- the right to edit the media's source file
- the right to write to the page where media is displayed
- the right to control playback of media
- the right to control access to the media

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
