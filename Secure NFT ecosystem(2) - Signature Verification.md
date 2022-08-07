# Secure NFT ecosystem(1) - Digital Signature

## 1. Introduction

[Digital signature](https://en.wikipedia.org/wiki/Digital_signature), based on the public-key cryptography, is the infrastructure of blockchain technology. A valid digital signature, where the prerequisites are satisfied, gives a recipient very high confidence that the message was created by a known sender (authenticity), and that the message was not altered in transit (integrity). 

Digital signature is widely used in smart contracts for authentication and Integrity, especially in whitelist mint and order-book NFT marketplaces cause it helps save transaction costs (sign off chain, verify on chain). However, the convenience also introduces some risks in the NFT marketplaces. Today, we'd like to talk about the usability and risks of digital signature in the NFT ecosystem.

## 2. Application of Digital Signature in NFT Markets

Considering digital signature can tell what the message mean, and from who the message is sent, digital signature is widely applied in the NFT ecosystem, not only in NFT token contracts e.g. for whitelist mint, but also in NFT marketplaces for order validation. The main advantage of this policy is that signing signature is an off-chain behavior, only verification is on chain, this could considerably save the transaction fees. The rest of this section will talk about the usability of digital signature.

### 2.1. Whitelist Mint

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
This contract is [NBA NFT contract](https://etherscan.io/address/0xdd5a649fc076886dfd4b9ad6acfc9b5eb882e83c#code), launched by the NBA official. The function `mint_approved()` implements whitelist mint: the NBA NFT owner signs a mint message (`info` variable) which includes the minter, and sends the message to the permitted minter, then the minter could invoke `approved_mint` with the signed viriable. The contract will then validate the signer of the message (`signer == recovered`) and process the mint procedure. Because `signer` is a variable referring to the NBA contract owner, only the NBA owner could sign the mint message. In another words, only the contract owner could determine who has whitelist mint permissions. Now, many NFT projects choose using digital signature to implement whitelist mint.

### 2.2. Order Verification

Order verification is another major application of digital signature in NFT ecosystem.
The NFT marketplaces play an important role in the NFT ecosystem as they provides the trading functionality for the NFTs. As each NFT token is non-fungible, the automated market maker (AMM) trading policy which is adopted in fungible tokens (i.e. ERC-20 tokens) is not applicable here in NFT markets anymore, thus, most NFT marketplaces, e.g. OpenSea, LooksRare, and X2Y2 take order-book model as their trading policy.

order-book trading policy is one of the most popular trading policies in traditional finance, e.g. stock markets and future markets. The policy is simple: You have a maker, aka a person who wants to sell an asset for a specific price, and you have a taker, aka a person who wants to buy it and agrees with seller's price, then the order matches. The process is same in the orderbook NFT marketplaces, the only difference is the process of order offering: the NFT marketplaces use digital signature for order verification. Figure 1 describes an example of the whole trading process of one of the orderbook marketplaces: OpenSea.

![OpenSea Trading Process](./image/orderbook.png)

**<center> Fig 1. OpenSea trading Process </center>**

The seller signs a seller order and stores it on OpenSea's server, the buyer could retrieve the signed sell order from OpenSea's server, and then invokes the buy order transaction with the signed sell order as parameters. The marketplace contract will validate the order to make sure the sell order is indeedly signed by the seller and complete the trade process.




## 3. Security Issues

There's no doubt that the digital signature could help save a lot of gas in the NFT ecosystem: Only 1 transaction, instead of 2, is needed to complete a whitelist mint or trade. However, such a covenient method also has some risks.

The [Horton Principle](https://en.wikipedia.org/wiki/Horton_Principle) is a maxim for cryptographic systems and can be expressed as "Authenticate what is being meant, not what is being said" or "mean what you sign and sign what you mean", it requires signing the action totally and precisely. If the signature is partially or non-Accurate, the result is disastrous.

Recalling the NFT contract in section 2.1, The goal of `info` variable is to validate that the transaction is signed by the mint site and that the user is whitelisted. The verify function `verify` does a standard signature verification, but it's missing one CRITICAL component: there's no check that `info.from == msg.sender`. The contract verifies the signer of the signature indeed, but it doesn't verifies the minter. In another word, the signature missed the CRITICAL minter information! This means that the same signature can be re-used by anyone else as many times as they want, only need one valid signature to loop this.  Fortunately, [a report](https://twitter.com/cygaar_dev/status/1516853784999198721?ref_src=twsrc%5Etfw%7Ctwcamp%5Etweetembed%7Ctwterm%5E1516853784999198721%7Ctwgr%5E2a3ea07544052948f3b9fabb5ff17a187d536939%7Ctwcon%5Es1_&ref_url=https%3A%2F%2Fabmedia.io%2F20220421-nba-the-association-nft-contract-exploit) disclosed the vulneribility sonn after the project was launched to prevent more loss.

Another security issue is about OpenSea, in the early of 2022, researchers disclose a potential vulnerability of OpenSea marketplace contract (version: wyvern 2.2), which implements the core functionality of NFTs trading. 

In Wyvern protocol, users author listings (sell offers) or offers (buy offers) off-chain, and signature of offers are verified on chain. Wyvern offers contains many parameters and the parameters are aggregated together into a single bytes to calculate the digest of the offer, then the contract will validate the signature of the digest of the offer. The parameters aggregation method is simply combining the parameters into a bytes string with the following bytes combining methods.
 
```solidity
index = ArrayUtils.unsafeWriteAddress(index, order.target);
index = ArrayUtils.unsafeWriteUint8(index, uint8(order.howToCall));
index = ArrayUtils.unsafeWriteBytes(index, order.calldata);
index = ArrayUtils.unsafeWriteBytes(index, order.replacementPattern);
index = ArrayUtils.unsafeWriteAddress(index, order.staticTarget);
index = ArrayUtils.unsafeWriteBytes(index, order.staticExtradata);
index = ArrayUtils.unsafeWriteAddress(index, order.paymentToken);
```

For example, if the parameters compose of 3 components: `(address, bytes[])`, and the parameters are `(0x9a534628b4062e123ce7ee2222ec20b86e16ca8f, "0xc098")`, the aggregated bytes would be `0x0000000000000000000000009a534628b4062e123ce7ee2222ec20b86e16ca8fc098`, just `address` + `bytes[]`. Seems to be easy and clear, right?

Now, consider a more complex example, the structure of parameters is `(address, bytes[], byte[])`.

parameter 1 is `(0x9a534628b4062e123ce7ee2222ec20b86e16ca8f, "0xab", "0xcdef")`.

parameter 2 is `(0x9a534628b4062e123ce7ee2222ec20b86e16ca8f, "0xabcd", "0xef")`.

The aggregated bytes are:

parameter 1: `0x0000000000000000000000009a534628b4062e123ce7ee2222ec20b86e16ca8fabcdef`

parameter 2: `0x0000000000000000000000009a534628b4062e123ce7ee2222ec20b86e16ca8fabcdef`

Wow! two different parameters have the same aggregated result, which means their digests are **SAME**, resulting that one signature could verify the 2 different parameters.

This is because there are many variable-length components in the parameters, and we can truncate part of the variables and attach the truncated parts to its previous or later components. Unfortunately, Wyvern contracts have many variable-length parameters as the below shows.

```solidity
    ......
    address target;
    /* HowToCall. */
    AuthenticatedProxy.HowToCall howToCall;
    /* Calldata. */
    bytes calldata;
    /* Calldata replacement pattern, or an empty byte array for no replacement. */
    bytes replacementPattern;
    /* Static call target, zero-address for no static call. */
    address staticTarget;
    /* Static call extra data. */
    bytes staticExtradata;
    ......
```

The impact of the vulneribility is that exploiters (if possible) could control the victim's account to execute some malicious behavior. The detailed analysis of the vulneribility is [here](https://nft.mirror.xyz/VdF3BYwuzXgLrJglw5xF6CHcQfAVbqeJVtueCr4BUzs)

Both of the two risks mentioned in this section all violate the Horton Principle, as the NFT contract does not sign the minter, and the Wyvern contract signs a structureless parameters so that the meaning of the action could be modified while the presentation (saying) of the parameters still remains.

## 4. Summary And Suggestions

1. Be careful about double-use problem when designing the signature verification system
2. Obey the Horton Principle, sign what you mean, not what you say. The signature should contains all-around and accurate information needed. For instance, to avoid the risk of OpenSea Wyvern 2.2, you should sign the **structured** parameters after **encoded** by abi.encode method in solidity instead of only bytes string combined of parameters because the former has the length and structure information of each parameters while the latter does not.