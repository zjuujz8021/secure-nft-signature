# Secure NFT ecosystem(1) - Digital Signature

## 1. Introduction

[Digital signature](https://en.wikipedia.org/wiki/Digital_signature), based on the public-key cryptography, is the infrastructure of blockchain technology. A valid digital signature, where the prerequisites are satisfied, gives a recipient very high confidence that the message was created by a known sender (authenticity), and that the message was not altered in transit (integrity). 

Digital signature is widely used in smart contracts for authentication and Integrity, especially in order-book NFT marketplaces cause it helps save transaction costs to match an order. However, the convenience also introduces some risks in the NFT marketplaces. Today, we'd like to talk about the risks of digital signature in the NFT ecosystem.

## 2. Digital Signature



## 2. Application of Digital Signature in NFT Markets

Digital signature is applied in the NFT ecosystem with mainly 2 aspects, the below sections will describe the technology detail of them.

### 2.1. Order Verification

The NFT marketplaces plays an important role in the NFT ecosystem as they provides the trading functionality for the NFTs.   As each NFT token is non-fungible, the automated market maker (AMM) trading policy which is widely used in fungible tokens (i.e. ERC-20 tokens) is not applicable anymore, thus, most NFT marketplaces, e.g. OpenSea, LooksRare, and X2Y2  adopts 

## 2. Security Issues

In the early of 2022, researchers disclose a potential vulnerability of OpenSea marketplace contract (version: wyvern 2.2), which implements the core functionality of NFTs trading. 