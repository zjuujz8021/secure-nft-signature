# Secure NFT ecosystem(1) - Digital Signature

## 1. Introduction

[Digital signature](https://en.wikipedia.org/wiki/Digital_signature), based on the public-key cryptography, is the infrastructure of blockchain technology. A valid digital signature, where the prerequisites are satisfied, gives a recipient very high confidence that the message was created by a known sender (authenticity), and that the message was not altered in transit (integrity). 

Digital signature is widely used in smart contracts for authentication and Integrity, especially in whitelist mint and order-book NFT marketplaces cause it helps save transaction costs (sign off chain, verify on chain). However, the convenience also introduces some risks in the NFT marketplaces. Today, we'd like to talk about the usability and risks of digital signature in the NFT ecosystem.

## 2. Application of Digital Signature in NFT Markets

Considering digital signature can tell what the message mean, and from who the message is sent, digital signature is widely applied in the NFT ecosystem, not only in NFT token contracts e.g. for whitelist mint, but also in NFT marketplaces for order validation. The main advantage of this policy is that signing signature is an off-chain behavior, only verification is on chain, this could considerably save the transaction fees. The rest of this section will talk about the usability of digital signature.

### 2.2 Whitelist Mint

"NFT minting" is the act of publishing an NFT on the blockchain, it is an important step which means the creation of an NFT. As most NFT projects would like to disseminate their products, they prefer to motivate users by whitelist mint (also called presale, alowlist mint, etc.), people wo win the spots could mint tokens with a lower price (even free), signature is used here to distinguish the whitelist minters and public (ordinary) minters. The below is an example of implementation of whitelist mint using digital signature.

``` solidity
    function mint_approved(
        vData memory info,
        uint256 number_of_items_requested,
        uint16 _batchNumber
    ) external {
        ...
        require(verify(info), "Unauthorised access secret");
        ...
    }
    function verify(vData memory info) public view returns (bool) {
        require(info.from != address(0), "INVALID_SIGNER");
        bytes memory cat =
            abi.encode(
                info.from,
                info.start,
                info.end,
                info.eth_price,
                info.dust_price,
                info.max_mint,
                info.mint_free
            );
        bytes32 hash = keccak256(cat);
        require(info.signature.length == 65, "Invalid signature length");
        bytes32 sigR;
        bytes32 sigS;
        uint8 sigV;
        bytes memory signature = info.signature;

        assembly {
            sigR := mload(add(signature, 0x20))
            sigS := mload(add(signature, 0x40))
            sigV := byte(0, mload(add(signature, 0x60)))
        }

        bytes32 data =
            keccak256(
                abi.encodePacked("\x19Ethereum Signed Message:\n32", hash)
            );
        address recovered = ecrecover(data, sigV, sigR, sigS);
        return signer == recovered;
    }
```
This contract is [NBA NFT contract](https://etherscan.io/address/0xdd5a649fc076886dfd4b9ad6acfc9b5eb882e83c#code), launched by the NBA official. The function `mint_approved()` implements whitelist mint: the NBA NFT owner signs a mint message (`info` variable) which includes the minter, and sends the message to the permitted minter, then the minters could invoke `approved_mint` with the signed viriable. The contract will then validate the signer and process the mint procedure. Because `signer` is a variable referring to the NBA contract owner, only the NBA owner could sign the mint message. In another words, only the contract owner could determine who has whitelist mint permissions. Now, many NFT projects choose using digital signature to implement whitelist mint.

### 2.1. Order Verification

Order verification is another major application of digital signature in NFT ecosystem.
The NFT marketplaces play an important role in the NFT ecosystem as they provides the trading functionality for the NFTs. As each NFT token is non-fungible, the automated market maker (AMM) trading policy which is adopted in fungible tokens (i.e. ERC-20 tokens) is not applicable here in NFT markets anymore, thus, most NFT marketplaces, e.g. OpenSea, LooksRare, and X2Y2 take order-book model as their trading policy.

order-book trading policy is one of the most popular trading policies in traditional finance, e.g. stock markets and future markets. The policy is simple: You have a maker, aka a person who wants to sell an asset for a specific price, and you have a taker, aka a person who wants to buy it and agrees with seller's price, then the order matches. The process is same in the orderbook NFT marketplaces, the only difference is the process of order offering: the NFT marketplaces use digital signature for order verification. Figure 1 describes an example of the whole trading process of one of the orderbook marketplaces: OpenSea.

![OpenSea Trading Process](./image/orderbook.png)

The seller signs a seller order and stores it on OpenSea's server, the buyer could retrieve the signed sell order from OpenSea's server, and then invokes the buy order transaction with the signed sell order as parameters. The marketplace contract will validate the order to make sure the sell order is indeedly signed by the seller and complete the trade process.

**<center> Fig 1. OpenSea trading Process </center>**


## 3. Security Issues

There's no doubt that the digital signature could help save a lot of gas in the NFT ecosystem: Only 1 transaction, instead of 2, is needed to complete a whitelist mint or trade. However, such a covenient method also has some risks.

In the early of 2022, researchers disclose a potential vulnerability of OpenSea marketplace contract (version: wyvern 2.2), which implements the core functionality of NFTs trading. 


## 4. Summary And Suggestions