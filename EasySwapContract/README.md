# EasySwap
## Overview
This is an on-chain ERC721 orderbook split into four modules:
- `EasySwapOrderBook` is the facade/exchange contract. Users call it to create, cancel, edit, match, batch-match orders.
- `EasySwapVault` is the escrow contract. **Ask/offer** orders escrow NFTs; **Bid** orders escrow ETH. Only the configured orderbook can withdraw/deposit/edit escrow balances.
- `OrderStorage` is the order index. Orders are stored by `OrderKey`, and indexed by `collection->side->price`. And orders at the same price are a linked queue. Prices are kept in a red-black tree. Bid side chooses the highest price first; Ask/Offer side chooses the lowest price first.
- `ProtocolManager` stores protocol fee share in basis points.
```mermaid
flowchart TD
    User[User / Wallet]

    User --> |calls|OB[EasySwapOrderBook]

    subgraph Exchange[Exchange Contract]
        OB
        Storage[OrderStorage]
        Validator[OrderValidator]
        Protocol[ProtocolManager]
    end

    OB --> |inherits|Storage
    OB --> |inherits|Validator
    OB --> |inherits|Protocol
    OB --> |uses|Vault[EasySwapVault]

    subgraph Libraries[Libraries]
        LibOrder[LibOrder]
        RBTree[RedBlackTreeLibrary]
        LibPayInfo[LibPayInfo]
        TransferSafe[LibTransferSafeUpgradeable]
    end

    OB --> LibOrder
    OB --> RBTree
    OB --> LibPayInfo
    OB --> TransferSafe

    Storage --> LibOrder
    Storage --> RBTree
    Validator --> LibOrder
    Protocol --> LibPayInfo
    Vault --> LibOrder
    Vault --> TransferSafe

    Vault --> ERC721[ERC721 NFT Contracts]
```
## Confusing Terminology

| Action | Stock Market Term | NFT Marketplace Term |
| :--- | :--- | :--- |
| **Wanting to Sell** | Ask / Offer | **List** (Listing Price) |
| **Wanting to Buy** | Bid | **Offer** / Bid |

## Order Types
- List order: Seller locks one ERC721 in the vault and waits for a buyer.
- Offer order: Buyer locks ETH in the vault and waits for a seller. A buyer pre-creates/deposits a Bid order when they are not doing “buy now,” but making an offer.
    - `FixedPriceForItem`: order targets a specific tokenId.
    - `FixedPriceForCollection`: bid can match any token in the collection.

## Data Structure
- OrderKey:
```
orderKey = hash(side, saleKind, maker, nft, price, expiry, salt)
```
- Storage shape:
```
orders[orderKey] = DBOrder(order, next)

priceTrees[collection][side]
  Ask/Offer side: ascending prices, best = first/lowest
  Bid side: ascending tree but best = last/highest

orderQueues[collection][side][price]:
  head -> orderKey -> orderKey -> ... -> tail
```
## Order Matching priority
```
collection
  -> side
    -> best price
      -> FIFO orders at that price
```

## Timeline
- Deploy
```mermaid
sequenceDiagram
    actor Deployer
    participant VaultProxy as EasySwapVault Proxy
    participant VaultImpl as EasySwapVault Implementation
    participant OBProxy as EasySwapOrderBook Proxy
    participant OBImpl as EasySwapOrderBook Implementation

    Deployer->>VaultProxy: deployProxy(EasySwapVault)
    VaultProxy->>VaultImpl: initialize()
    VaultImpl->>VaultImpl: __Ownable_init(deployer)

    Deployer->>OBProxy: deployProxy(EasySwapOrderBook, protocolShare, vault, name, version)
    OBProxy->>OBImpl: initialize(protocolShare, vault, name, version)
    OBImpl->>OBImpl: init Context / Ownable / ReentrancyGuard / Pausable
    OBImpl->>OBImpl: __OrderStorage_init()
    OBImpl->>OBImpl: __ProtocolManager_init(protocolShare)
    OBImpl->>OBImpl: __OrderValidator_init(name, version)
    OBImpl->>OBImpl: setVault(vault)

    Deployer->>VaultProxy: setOrderBook(orderBook)
    VaultProxy->>VaultImpl: orderBook = EasySwapOrderBook address

    Note over VaultProxy,OBProxy: Deployment complete
    Note over VaultProxy: Vault only accepts asset commands from OrderBook
    Note over OBProxy: OrderBook knows which Vault to use
```
- Create Listing
```mermaid
sequenceDiagram
    actor Seller
    participant NFT as ERC721 Collection
    participant OB as EasySwapOrderBook
    participant Vault as EasySwapVault
    participant Storage as OrderStorage

    Seller->>NFT: setApprovalForAll(Vault, true)
    Seller->>OB: makeOrders([List order])
    OB->>OB: validate maker, price, salt, expiry
    OB->>Vault: depositNFT(orderKey, seller, collection, tokenId)
    Vault->>NFT: safeTransferFrom(seller, Vault, tokenId)
    OB->>Storage: _addOrder(order)
    OB-->>Seller: emit LogMake(orderKey)
```
- Create Offer/Bid
```mermaid
sequenceDiagram
    actor Buyer
    participant OB as EasySwapOrderBook
    participant Vault as EasySwapVault
    participant Storage as OrderStorage

    Buyer->>OB: makeOrders([Bid order]) + ETH
    OB->>OB: calculate price * amount
    OB->>OB: validate maker, price, salt, expiry
    OB->>Vault: depositETH(orderKey, amount)
    Vault->>Vault: ETHBalance[orderKey] += msg.value
    OB->>Storage: _addOrder(order)

    alt buyer sent too much ETH
        OB-->>Buyer: refund extra ETH
    end

    OB-->>Buyer: emit LogMake(orderKey)
```

- Buyer Accepts Listing
```mermaid
sequenceDiagram
    actor Buyer
    participant OB as EasySwapOrderBook
    participant Vault as EasySwapVault
    participant Storage as OrderStorage
    participant Seller
    participant NFT as ERC721 Collection

    Buyer->>OB: matchOrder(sellOrder, buyOrder) + ETH
    OB->>OB: validate sellOrder and buyOrder
    OB->>OB: fillPrice = sellOrder.price
    OB->>OB: filledAmount[sellOrderKey] = sellOrder.nft.amount
    OB-->>OB: protocolFee = fillPrice * protocolShare / 10000
    OB-->>Seller: transfer ETH minus protocol fee
    OB->>Vault: withdrawNFT(sellOrderKey, Buyer, collection, tokenId)
    Vault->>NFT: safeTransferFrom(Vault, Buyer, tokenId)

    alt buyer sent too much ETH
        OB-->>Buyer: refund extra ETH
    end

    OB-->>Buyer: emit LogMatch(...)
```

- Seller Accepts Bid
```mermaid
sequenceDiagram
    actor Seller
    participant OB as EasySwapOrderBook
    participant Vault as EasySwapVault
    participant Storage as OrderStorage
    participant Buyer
    participant NFT as ERC721 Collection

    Seller->>OB: matchOrder(sellOrder, buyOrder)
    OB->>OB: validate bid exists and is open
    OB->>OB: fillPrice = buyOrder.price
    OB->>Vault: withdrawETH(buyOrderKey, fillPrice, OB)
    Vault-->>OB: ETH
    OB-->>Seller: transfer ETH minus protocol fee
    OB->>OB: filledAmount[buyOrderKey] += 1

    alt sellOrder already listed / escrowed
        OB->>Storage: _removeOrder(sellOrder)
        OB->>Vault: withdrawNFT(sellOrderKey, Buyer, collection, tokenId)
        Vault->>NFT: safeTransferFrom(Vault, Buyer, tokenId)
    else fresh sellOrder from seller
        OB->>Vault: transferERC721(Seller, Buyer, asset)
        Vault->>NFT: safeTransferFrom(Seller, Buyer, tokenId)
    end

    OB-->>Seller: emit LogMatch(...)
```

- Cancel Order
```mermaid
sequenceDiagram
    actor Maker
    participant OB as EasySwapOrderBook
    participant Vault as EasySwapVault
    participant Storage as OrderStorage
    participant NFT as ERC721 Collection

    Maker->>OB: cancelOrders([orderKey])
    OB->>Storage: load orders[orderKey]
    OB->>OB: require maker == msg.sender
    OB->>OB: require not fully filled
    OB->>Storage: _removeOrder(order)

    alt List order
        OB->>Vault: withdrawNFT(orderKey, Maker, collection, tokenId)
        Vault->>NFT: safeTransferFrom(Vault, Maker, tokenId)
    else Bid order
        OB->>Vault: withdrawETH(orderKey, remainingETH, Maker)
        Vault-->>Maker: refund ETH
    end

    OB->>OB: filledAmount[orderKey] = CANCELLED
    OB-->>Maker: emit LogCancel(orderKey)
```

- Edit Order
```mermaid
sequenceDiagram
    actor Maker
    participant OB as EasySwapOrderBook
    participant Vault as EasySwapVault
    participant Storage as OrderStorage

    Maker->>OB: editOrders([{oldOrderKey, newOrder}])
    OB->>Storage: load old order
    OB->>OB: validate same side, maker, saleKind, NFT
    OB->>OB: validate new salt, expiry, unused hash
    OB->>Storage: _removeOrder(oldOrder)
    OB->>OB: filledAmount[oldOrderKey] = CANCELLED
    OB->>Storage: _addOrder(newOrder)

    alt List order
        OB->>Vault: editNFT(oldOrderKey, newOrderKey)
        Vault->>Vault: move NFTBalance old -> new
    else Bid order price/amount changed
        OB->>Vault: editETH(oldOrderKey, newOrderKey, oldRemaining, newRemaining)
        alt newRemaining > oldRemaining
            Maker-->>Vault: extra ETH
        else newRemaining < oldRemaining
            Vault-->>Maker: refund difference
        end
    end

    OB-->>Maker: emit LogCancel(oldOrderKey)
    OB-->>Maker: emit LogMake(newOrderKey)
```

- Orderbook Query Flow
```mermaid
flowchart TD
    A[Seller/Buyer asks getBestOrder / getOrders] --> B[Choose collection + side]
    B --> C{Side?}
    C -->|Bid| D[Start at highest price in red-black tree]
    C -->|List| E[Start at lowest price in red-black tree]
    D --> F[Read order queue at price]
    E --> F
    F --> G[Scan FIFO linked list]
    G --> H{Expired?}
    H -->|Yes| I[Skip order]
    H -->|No| J{Bid filter matches saleKind/tokenId?}
    J -->|No| I
    J -->|Yes| K[Return order]
    I --> L{More orders at same price?}
    L -->|Yes| G
    L -->|No| M[Move to next best price]
    M --> F
```
## To Do
- The escrow-first is clean and reliable, especially for a learning/orderbook-style contract. But production NFT marketplaces usually prefer signed off-chain orders plus approval-based settlement, because locking every NFT/ETH is heavy for users. A hybrid design is often the sweet spot.