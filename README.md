 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/IERC721Royalty.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/utils/Time.sol";

contract NFTMarketplace is ReentrancyGuard, Time {
    using Counters for Counters.Counter;
    Counters.Counter private _itemIds;
    Counters.Counter private _itemsSold;

    address payable owner;
    uint256 listingPrice = 0.025 ether;

    constructor() {
        owner = payable(msg.sender);
    }

    struct MarketItem {
        uint itemId;
        address nftContract;
        uint256 tokenId;
        address payable seller;
        address payable owner;
        uint256 price;
        bool sold;
        uint256 listedAt;
    }

    mapping(uint256 => MarketItem) private idToMarketItem;

    event MarketItemCreated (
        uint indexed itemId,
        address indexed nftContract,
        uint256 indexed tokenId,
        address seller,
        address owner,
        uint256 price,
        bool sold,
        uint256 listedAt
    );

    event ListingCanceled(
        uint256 indexed itemId,
        address indexed nftContract,
        uint256 indexed tokenId,
        address seller
    );

    event ListingPriceUpdated(
        uint256 indexed itemId,
        address indexed nftContract,
        uint256 indexed tokenId,
        address seller,
        uint256 newPrice
    );

    event RoyaltyPercentageSet(
        uint256 indexed itemId,
        address indexed nftContract,
        uint256 indexed tokenId,
        address creator,
        uint256 royaltyPercentage
    );

    function getListingPrice() public view returns (uint256) {
        return listingPrice;
    }

    function createMarketItem(
        address nftContract,
        uint256 tokenId,
        uint256 price,
        uint256 royaltyPercentage
    ) public payable nonReentrant {
        require(price > 0, "Price must be at least 1 wei");
        require(msg.value == listingPrice, "Price must be equal to listing price");

        _itemIds.increment();
        uint256 itemId = _itemIds.current();

        idToMarketItem[itemId] =  MarketItem(
            itemId,
            nftContract,
            tokenId,
            payable(msg.sender),
            payable(address(0)),
            price,
            false,
            block.timestamp
        );

        IERC721(nftContract).transferFrom(msg.sender, address(this), tokenId);

        if (royaltyPercentage > 0) {
            IERC721Royalty(nftContract).setTokenRoyalty(tokenId, msg.sender, royaltyPercentage);
            emit RoyaltyPercentageSet(itemId, nftContract, tokenId, msg.sender, royaltyPercentage);
        }

        emit MarketItemCreated(
            itemId,
            nftContract,
            tokenId,
            msg.sender,
            address(0),
            price,
            false,
            block.timestamp
        );
    }

    function updateListingPrice(uint256 itemId, uint256 newPrice) public nonReentrant {
        require(idToMarketItem[itemId].seller == msg.sender, "Only the seller can update the listing price");
        require(block.timestamp - idToMarketItem[itemId].listedAt <= 1 days, "Listing can only be updated within 24 hours of being listed");

        idToMarketItem[itemId].price = newPrice;
        emit ListingPriceUpdated(itemId, idToMarketItem[itemId].nftContract, idToMarketItem[itemId].tokenId, msg.sender, newPrice);
    }

    function cancelListing(uint256 itemId) public nonReentrant {
        require(idToMarketItem[itemId].seller == msg.sender, "Only the seller can cancel the listing");
        require(block.timestamp - idToMarketItem[itemId].listedAt <= 1 days, "Listing can only be canceled within 24 hours of being listed");

        IERC721(idToMarketItem[itemId].nftContract).transferFrom(address(this), msg.sender, idToMarketItem[itemId].tokenId);
        delete idToMarketItem[itemId];
        emit ListingCanceled(itemId, idToMarketItem[itemId].nftContract, idToMarketItem[itemId].tokenId, msg.sender);
    }

    function createMarketSale(
        address nftContract,
        uint256 itemId
    ) public payable nonReentrant {
        uint price = idToMarketItem[itemId].price;
        uint tokenId = idToMarketItem[itemId].tokenId;
        require(msg.value == price, "Please submit the asking price in order to complete the purchase");

        address creator = IERC721Royalty(nftContract).royaltyOwner(tokenId);
        uint256 royaltyAmount = (msg.value * IERC721Royalty(nftContract).tokenRoyalty(tokenId)) / 10000;
        payable(creator).transfer(royaltyAmount);
        idToMarketItem[itemId].seller.transfer(msg.value - royaltyAmount);
        IERC721(nftContract).transferFrom(address(this), msg.sender, tokenId);
        idToMarketItem[itemId].owner = payable(msg.sender);
        idToMarketItem[itemId].sold = true;
        _itemsSold.increment();
        payable(owner).transfer(listingPrice);
    }

    function fetchMarketItems(
        bool onlyBuyItNow,
        bool onlyAuction,
        bool sortByPriceAsc,
        bool sortByPriceDesc,
        bool sortByNewest,
        bool sortByRarity
    ) public view returns (MarketItem[] memory) {
        uint itemCount = _itemIds.current();
        uint unsoldItemCount = _itemIds.current() - _itemsSold.current();
        uint currentIndex = 0;

        MarketItem[] memory items = new MarketItem[](unsoldItemCount);
        for (uint i = 0; i < itemCount; i++) {
            if (idToMarketItem[i + 1].owner == address(0)) {
                uint currentId = i + 1;
                MarketItem storage currentItem = idToMarketItem[currentId];

                if (onlyBuyItNow && currentItem.price > 0) continue;
                if (onlyAuction && currentItem.price == 0) continue;

                items[currentIndex] = currentItem;
                currentIndex += 1;
            }
        }

        if (sortByPriceAsc) {
            sortItemsByPriceAsc(items);
        } else if (sortByPriceDesc) {
            sortItemsByPriceDesc(items);
        } else if (sortByNewest) {
            sortItemsByNewest(items);
        } else if (sortByRarity) {
            sortItemsByRarity(items);
        }

        return items;
    }

    function fetchMyNFTs() public view returns (MarketItem[] memory) {
        uint totalItemCount = _itemIds.current();
        uint itemCount = 0;
        uint currentIndex = 0;

        for (uint i = 0; i < totalItemCount; i++) {
            if (idToMarketItem[i + 1].owner == msg.sender) {
                itemCount += 1;
            }
        }

        MarketItem[] memory items = new MarketItem[](itemCount);
        for (uint i = 0; i < totalItemCount; i++) {
            if (idToMarketItem[i + 1].owner == msg.sender) {
                uint currentId = i + 1;
                MarketItem storage currentItem = idToMarketItem[currentId];
                items[currentIndex] = currentItem;
                currentIndex += 1;
            }
        }
        return items;
    }

    function fetchItemsCreated() public view returns (MarketItem[] memory) {
        uint totalItemCount = _itemIds.current();
        uint itemCount = 0;
        uint currentIndex = 0;

        for (uint i = 0; i < totalItemCount; i++) {
            if (idToMarketItem[i + 1].seller == msg.sender) {
                itemCount += 1;
            }
        }

        MarketItem[] memory items = new MarketItem[](itemCount);
        for (uint i = 0; i < totalItemCount; i++) {
            if (idToMarketItem[i + 1].seller == msg.sender) {
                uint currentId = i + 1;
                MarketItem storage currentItem = idToMarketItem[currentId];
                items[currentIndex] = currentItem;
                currentIndex += 1;
            }
        }
        return items;
    }

    function sortItemsByPriceAsc(MarketItem[] memory items) private pure {
        // Implement sorting algorithm to sort items by price in ascending order
    }

    function sortItemsByPriceDesc(MarketItem[] memory items) private pure {
        // Implement sorting algorithm to sort items by price in descending order
    }

    function sortItemsByNewest(MarketItem[] memory items) private pure {
        // Implement sorting algorithm to sort items by newest listed
    }

    function sortItemsByRarity(MarketItem[] memory items) private pure {
        // Implement sorting algorithm to sort items by rarity
    }
}
```
