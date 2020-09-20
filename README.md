# Secret Network Improvement Proposals (SNIPs)

These documents pertain to standards for building Secret Contracts on Secret Network.

## Contributing

1. Review SNIP-0
2. Fork the repository by clicking "Fork" in the top right.
3. Add your SNIP to your fork of the repository. There is a template SNIP here.
4. Submit a Pull Request to Secret Foundation's "SecretGovernance" repository.

Your first PR should be a first draft of the final SNIP. An editor will manually review the first PR for a new SNIP. If your SNIP requires images, the image files should be included in a subdirectory of the assets folder for that SNIP as follows: assets/your-snip-name. When linking to an image in the SNIP, use relative links such as ../assets/your-snip-name/image.png.

**Make sure the 'author' line of your SNIP contains your GitHub username.**

## Governance

Here is a [summary of the governance processes](https://blog.scrt.network/secret-network-governance) of Secret Network. We use an implementation of the [Cosmos-SDK governance module](https://docs.cosmos.network/master/modules/gov) for binding proposals rejected or approved through SCRT-weighted voting. Additionally, our community relies on various off-chain processes to coordinate the Secret Network community. We believe transparency and inclusivity help interconnected projects drive progress through collaboration. Ultimately, cooperation and trust are necessary for sustainability of our network and community.

Currently, Secret Foundation receives a percentage of SCRT inflation for various marketing and growth initiatives. Also, there is a community pool with funds managed collectively by supporters of the network.

### Types of On-Chain Governance Proposals

* Signaling
* Community Spend
* Parameter Change

### Stages of Governance Proposals

#### 1. Deposits

For a proposal to be considered for voting, a minimum deposit of 1000 SCRT must be deposited within 1 week from when the proposal was submitted. Any SCRT holder may contribute to this deposit to support proposals, meaning the party submitting the proposal doesn’t necessarily need to provide the deposit itself. The deposit is required as a kind of protection against spam. If the proposal does not reach the minimum deposit threshold, deposits are refunded. If the proposal is approved or if it is rejected WITHOUT a veto, deposits will automatically be refunded to their respective depositor. When a proposal is vetoed with a supermajority, deposits will be burned.

#### 2. Voting

When the minimum deposit for a particular proposal is reached, the 1-week voting period begins. During this period, SCRT holders are able to cast their vote on that proposal. As mentioned, there are four voting options: Yes, No, NoWithVeto, and Abstain. Only staked tokens can participate in governance. Voting power is measured in terms of stake. The amount of SCRT you stake determines your influence on the decision. Delegators inherit the vote of the validators they are delegated to unless they cast their own vote, which will overwrite validator decisions.

#### 3. Tallying

Whether a proposal is accepted depends on the result of the coin voting by SCRT holders. The following requirements need to be satisfied for a proposal to be considered accepted:

* **Quorum:** More than 33.4% of the total staked tokens at the end of the voting period need to have participated.
* **Threshold:** More than 50% (after excluding Abstain votes) voted in favor of the proposal.
* **No Veto:** Less than 33.4% (after excluding Abstain votes) vetoed the decision.

#### 4. Implementation

Accepted proposals have to be implemented as part of the software that is run by validators in the network. Both community-spend and parameter-change proposals are implemented automatically. If a proposal is just offering direction (“signaling”), developers can build and pass it to the validators in order to upgrade the network.

## Resources

If you're interested in learning more about Secret Network, you should check out [Enigma's repository](https://github.com/enigmampc/SecretNetwork) and our [documentation site](https://build.scrt.network). Specifically, there is a [page about how to participate in governance](https://build.scrt.network/protocol/governance.html), and you can review past proposals on the [Secret Explorer](https://explorer.cashmaney.com/proposals) and [Puzzle](https://puzzle.report/secret/chains/secret-2/governance).

The [Cosmos Hub GWG](https://github.com/gavinly/CosmosGWG) assembled this [Cosmos-SDK parameters wiki](https://github.com/gavinly/CosmosParametersWiki) and [best practices for community-spend proposals](https://github.com/gavinly/CosmosCommunitySpend).

Here is a [blog post](https://blog.scrt.network/secretwasm-decentralized-private-computation) summarizing our collaboration with [Confio](https://confio.tech) building in parallel with [CosmWasm](https://www.cosmwasm.com). You might also refer to their collection of packages, including the cw20 token standard:
https://github.com/CosmWasm/cosmwasm-plus

## Community 🕵️

🚨 🚨 🚨

[ATTENTION 𝕊ΞCRΞT AGΞNTS](https://blog.scrt.network/secret-committees-empowering-secret-agents)

~~ThIs MeSsAgE wIlL NOT sElF-dEsTrUcT~~

🤫 🤫 🤫

Your mission, should you choose to accept it, is to join our governance committee and help us coordinate all the projects in the Secret Network ecosystem to accomplish our mission together.

[Secret Chat](https://go.rocket.chat/invite?host=chat.scrt.network&path=invite%2FXqY6pa) | [Blog](https://blog.scrt.network) | [Twitter](https://twitter.com/SecretNetwork) | [Forum](https://forum.scrt.network) | [Wiki](https://learn.scrt.network)
