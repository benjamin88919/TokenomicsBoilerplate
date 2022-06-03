# Tokenomics Boilerplate
Tokenomics Boilerplate | Governance Tokens, Uniswap Buy-Backs and Staking in Solidity

In DeFi a projects success is greatly tied to the effectiveness of the tokenomics. Governance tokens can be distributed to the users to incentivise growth and fees can be used to buy these back on exchange. In this Solidity tutorial we are going to deploy a governance token, a Uniswap v3 liquidity pool, a buy back mechanism and a staking pool. The aim is to provide you with a tokenomics boilerplate which can act as a starting point for your next DeFi project.


## Disclaimer
It has not been audited by a external security company and should be thoroughly checked before being used on anything other than a testnet.

## Setup Instructions
Edit the .env-example.txt file and save it as .env 
Add a private key

Build using the following commands:

```shell
git clone [this repo]
cd TokenomicsBoilerplate
npm install
```

Because we are using external contracts from Uniswap v3 we need to fork whatever network we want to deploy on. Here I am going to deploy a local version of the Polygon Mumbai Testnet

```shell
npx hardhat node --fork https://polygon-mumbai.g.alchemy.com/v2/ALCHEMYAPIKEYHERE
```

From there we can test and deploy

```shell
npx hardhat test
```

Note you'll need some testnet funds in your wallet to deploy the contract. You can get these from here: https://faucet.polygon.technology/

```shell
npx hardhat run --network mumbai .\scripts\deploy.js
```

## How The Tokenomics Work

There are a number of moving parts in this boilerplate. Let’s look at how they work together to form a competitive ecosystem for a DeFi product.

So first we have the main contract which needs to do something useful and generate revenues. This could be a exchange, lending protocol or any other DeFi product that provides a useful service.

A governance token is issued by the main contract to users to incentivise use and provide voting rights on the governance of the protocol. The distribution could be in the form of an airdrop or ongoing liquidity incentives to promote growth.

A Uniswap v3 liquidity pool is set up during migration so that the token can be traded against the networks base asset i.e. ETH for Ethereum. This allows investors to buy the token on exchange and also the contract has the ability to use fees to buy back and burn the governance token.

A staking pool encourages holders of the governance token to stake their assets to receive rewards paid out in the governance token itself. This will provide a simple way for investors to generate a yield on their holdings.

## The ERC20 Governance Token

We are using the ERC20-Votes library as well to provide a mechanism for voting and delegating votes.

A constructor function mints a initial one million tokens during migration. These will be split 50/50 between the Uniswap liquidity pool and the staking contract.

constructor() ERC20("GovToken", "GOVTKN") ERC20Permit("GovToken") {
  _mint(msg.sender, 1000000 ether);
}
Finally we have some ownerOnly functions for minting and burning tokens. The owner will be the main contract enabling it to mint tokens as they are distributed and burn those bought back on exchange.

function mint(address _to, uint256 _amount) public onlyOwner {
  _mint(_to, _amount);
}


## Creating A Liquidity Pool In Hardhat

The liquidity pool is set up using Uniswap v3. Note most clones on other networks use v2 so the function names and parameters are slightly different.

We are interacting with Uniswap’s NonfungiblePositionManager to setup a liquidity pool for WETH-GOVTOKEN where wEth can be whatever the wrapped native token for the network is. I deployed this on Polygon so used MATIC. Then GovToken is our governance token address we created above.

Uniswap keeps a list of contract addresses and wrapped native tokens here: https://docs.uniswap.org/protocol/reference/deployments

## The Contract Owner & True Decentralization

As part of OpenZeppelin’s Ownable library you have a function to update the contract owner. We want to transfer ownership of the governance token to the main contract. This will allow the contract to mint and burn tokens in line with our code.

await govToken.transferOwnership(mainContract.address);
The key thing here is that we are removing the developer wallet used to deploy the contracts as a single point of centralized failure for the protocol.

I recently read through the excellent resource on Smart Contract Security by Consensys. However I strongly disagree with the general philosophy of having pausable, upgradable smart contracts. The general idea is that smart contract bugs will inevitably be found and you need a way to mitigate that risk.

I would argue that this moves away from the ideals of what decentralized finance should be where all parties have equal rights and the protocol is managed by code not people. Where possible I think developers should be striving to create truly decentralized products and survivorship bias will determine which protocols survive from a security perspective.

If we accept a need to own a contract, manage the data and be able to pause execution why are we using Ethereum in the first place?

## Distribution Of The Governance Token

Back to Solidity and we want to give out some governance tokens to users of the protocol to incentivise growth and decentralize the governance.

function distributeRewards(uint256 _commission) internal {
  uint256 govTokenSupply = IERC20Token(govToken).totalSupply();
  if (govTokenSupply < govTokenMaxSupply - 1 ether) {
    uint256 remaininggovTokens = govTokenMaxSupply - govTokenSupply;
    uint256 diminishingSupplyFactor =  remaininggovTokens * 100 / govTokenMaxSupply;
    uint256 govTokenDistro = _commission * diminishingSupplyFactor ;
    require(govTokenDistro >= 0, "govTokenDistro below zero");
    IERC20Token(govToken).mint( msg.sender, govTokenDistro);
  }
}
First we get the total supply and check if it’s below the maximum supply.

Then we use a simple algorithm to calculate a diminishing supply factor so that the tokens are distributed fast to start with and then slow down as we get closer to the max supply.

This does a number of things it increases the longevity of the incentives and it also allows room for price appreciation as the governance token becomes harder to obtain.

## Buy Back Mechanism Using Fees

From within the main contract we can trade MATIC or ETH for GovToken and then use the burn function to destroy those governance tokens we just purchased.

function buyBackAndBurn(uint256 _amountIn) internal {
  uint24 poolFee = 3000;
  ISwapRouter.ExactInputSingleParams memory params =
    ISwapRouter.ExactInputSingleParams({
      tokenIn: wEth,
      tokenOut: govToken,
      fee: poolFee,
      recipient: address(this),
      deadline: block.timestamp,
      amountIn: _amountIn,
      amountOutMinimum: 0,
      sqrtPriceLimitX96: 0
    });
  uint256 amountOut = ISwapRouter(uniRouter).exactInputSingle{value: _amountIn}(params);
  IERC20Token(govToken).burn(amountOut);
}
This decreases the supply and increases the price on exchange.

## Testing & Modelling A Distribution Schedule

We have a dynamic model here where governance tokens are being minted if the supply is below maxSupply and a varying rate. Then tokens are also being purchased back and burnt reducing the supply.

This potentially could see the maxSupply reached and then a few blocks later become active again as tokens are burnt.

The kind of distribution schedule we are looking to create is something like this.

