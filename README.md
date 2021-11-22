# ACL: A Price Discovery Protocol for Digital Property Rights

# Description

- The Web, like land, was initially developed under centralised top-down control
- This led to bad things: serfdom, fiefdoms, etc.
- Then we invented the idea of property rights which privatised property and decentralised it, aligning the incentives of each individual with the incentives of the collective by allowing everyone to capture the majority of the profit from the returns on their own labour, which was then taxed. This was soon spread to ideas with the advent of intellectual property.
- Now we have the web and, with it, a whole load of digital property.
- The web, once we decentralise digital property rights (NFTs) will experience an explosion in growth and development in much the same way
- Currently, we have a supply-side constraint. 
    - We're still tinkering at the edge of digital property rights: Art, Collectibles, tokenising physical property
- Some of the most interesting digital property rights that have been created include tweets, memes, and blog posts because they are digital properties that have existed for years but have been provisioned to the public under the auspices of centralised companies, typically in return for purchasing a subscription.
- Today, this is still driven by suppliers deciding to tokenise something they believe may be valuable. Speculative interest follows where true price discovery is both embryonic and opaque, creating significant information asymmetry. 
- [TODO] Add a point about how most of the speculative capital has been allocated to a small amount of total supply, 
- ACL subverts this model by allowing any user to place a bid on any digital property on the web using just a URI. 
    - This could link to a file, a server, a web page, an endpoint
    - What's important to understand here is that digital property ownership, can have value independently of legal ownership
    - Comingling legal property rights and digital property rights, in my opinion, is to misunderstand the nature of blockchain as an innovation. The two can be connected but are distinct. 
    
    // - For example, Jack has equity in Twitter, he has control over decision-making via his voting shares, but his tweet is legally the property of Twitter. Therefore, selling an NFT of his tweet to @sinaEstavi for $2.9m is not value attributed to the legal ownership of the tweet, but an entirely distinct set of property rights. [TODO] find another example besides Twitter who don't actually own the intellectual property in a tweet.

- ACL creates a market for every asset on the web, driven by demand rather than being constrained by available supply.
- This increases the amount of information in the market for digital property, improving the efficiency of price discovery and bringing more digital property to the market
- The existing model of digital property ownership and provision is also worse for creators
    [TODO] Explain why
- With ACL, the **whole** web is open for business.


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