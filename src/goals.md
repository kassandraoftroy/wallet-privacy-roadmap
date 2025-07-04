# Goals

This document outlines the medium-term goals for the Wallet SDK and Kohaku Wallet UX from the perspective of privacy. We want to make sure we are designing towards a strong/ideal privacy UX.

The main goal for the SDK and example wallet implementation is to **introduce a new chain-interop focused and privacy-first standard for Ethereum self-custody wallets of general use**.

Here we tackle what the privacy part could look like in practice. By attempting to define the ideal privacy UX first, we ensure that short-term milestones align with the medium/long-term vision.

---

## 0. Root Identity and Key Management

### 0.1 Everything Derives From a Seed Phrase
The entire wallet is deterministically derived from a single seed phrase (BIP-32 compliant etc). From this seed, users generate **User Accounts** which are isolated identity/persona namespaces. But these identity/persona namespaces are multi-chain and multi-credential abstractions, not just a naive 1 keypair == 1 persona/identity.

A multi-chain **User Account** derives these things (on each chain):
- **Canonical Public Address credential** (`bob.eth@arb` resolves to bob's canonical public address on arbitrum which is controlled by this key)
- **Stealth Meta Address credential** (`bob.eth@arb` has a mapping on arbitrum's canonical ERC-6538 registry contract set to a Stealth Meta Address, let's call it `st:bob.eth@arb`, which is controlled by these keys)
- **Mixer credentials** (credentials for Railgun, Privacy Pools, and Tornado Cash. Each mixer protocol creds are derived from keys in the HD wallet tree in some canonized way. _NOTE: likely this will only matter for mainnet, as current thinking is that we push all mixer activity to mainnet to maximize anonymity set there?_)
- **Ephemeral Addresses credentials** (for one-time DeFi interactions or payment links to give to third parties, a standard for how to derive these keypairs as needed)

Each **User Account** is a scoped, chain-aware identity. User explicitly "initializes" different chains for the given account, which defines the domain of chains that the user willingly interacts across seamlessly.
### 0.2 ERCification: Wallets eventually interop with this shared standard

Ideally this is an interoperable shared standard of how to interpret and recover credentials from the seed phrase. To me this could be enshrined in one or a set of ERCs. The goal is that you can take your seed phrase to another wallet client and "voila" you find all your assets even if scattered across a number of addresses and also has assets inside a particular Mixer application. Also, if all software goes boom and user really wants to "go it alone" they can read all the ERCs to technically have the information necessary to reconstruct everything.

---

## 1. Write UX: Maximize Private Actions by Default

The goal is to make privacy something that happens mostly by default, without the user needing to restrict themselves from doing the things they normally want to do onchain. In rare cases that a user WANTS some activity or holdings to be linked to their identity publically, this is something that the user can seek out explicitly outside of the defaults. But in all other cases, the wallet would opt for unlinked and/or untraceable activity and holdings. Let's go into more detail.

### 1.1 Proactive Mixing of Fund Balances
- As many tokens as possible with large enough transfer activity should have accessible mixer pools (especially on mainnet).
- Any token with a positive balance in the user account with a mixer pool should (ideally) get mixed eventually.
- Wallets can auto-mix balances on behalf of user based on preferences (e.g. "always mix 50% of my ETH"). Users could also click "mix" next to balances of tokens on a case by case basis as another UX. But mixing a good portion of your assets should be highly encouraged by the standard wallet UX.
- Mixed funds become “pTokens” (e.g. pETH, pDAI) after a pending period
- SDK has abstract enough interfaces/patterns that the different mixer protocols are plug-and-play (Privacy Pools, Railgun, TC). We make sure all 3 protocols can support the main features we want (mix, reassign funds to a target receiver in the mixer - ok if involves a temporary hop out).

### 1.2 Privacy of Fund Transfers
- **Receiving**: others sending to `bob.eth@chain` will use stealth addresses protocol automatically when `bob.eth@chain` has a record set on ERC-6538 registry. So even if sender doesn't ahve a "private balance" receiver address is still delinked. Bob can also create chain-specific payment links for reception from 3rd parties, and these links will always be to a fresh ephemeral address on the bob.eth User Account. When bob receives the funds to the User Account (either to an ephemeral or stealth address), bob does not need to be concerned with the technicality that it's in a fresh address. Bob only sees aggregated balances (see 1.7).
- **Sending**: if private funds exist for sender (e.g. sending X ETH, sender has at least X pETH already), the wallet send will be made private by default. If user does not have sufficient private funds, the user will be alerted that the send is NOT private and that it can be made private (but involves delay period e.g. mixing more pETH). When sending from Mixed private funds to a new user, ideally the funds for the target receiver also end up back in the mixer pool (ok if involves a hop out but ideally this whole process is automatic receiver just notices received pETH)
- The goal is most direct transfers from wallets will have both sender and receiver privacy. If no sender privacy, we still fallback to just delinked receiver.
- Where and when possible receiver will _automatically_ get their received funds back inside the mixer. So every mixer solution should have this feature that allows you to either reassign the funds to recipient directly in the mixer or else have a peripheral integration with the mixer so the funds go out but then automatically back in for the target receiver.

### 1.3 Private DeFi / Dapp Interactions
- When doing a DeFi or Dapp interaction that requires transferring assets, if those assets are fully covered from User Account private balances (supply liquidity to ETH/DAI on uniswap but holding pETH and pDAI), then the interaction is done privately by default. This means the funds come out of the mixer protocol into an ephemeral address, and the ephemeral address interacts with the Dapp.
- In general addresses should be 1-time-use as much as possible (or two-time for e.g. deposit, and later, withdraw). Much like bitcoin wallets everytime you touch an address (that isn't the Canonical Public Address) you should sweep all the funds and never reuse it again. E.g. you have 23 DAI in your address and want to swap 10 for ETH, the ETH output should go to a new fresh address and the 13 DAI change should also get transferred to a new fresh address.
- Pay gas fees for relayed operations either with some portion of the operation itself or with pETH wherever possible (see 1.5)

### 1.4 Chain- and Address-Agnostic Authorization
- Users approve actions in an intuitive way similar or even better than wallets of today (authorize the intended action/interaction just once, then it happens). No need to sign each ECDSA signature address-by-address individually as a UX.
- Wallet orchestrates eoa upgrades (EIP-7702), bridging, routing, relaying under the hood. User can get all the relevant information for their operations to be legible, but the "lazy" UX would be really simple and easy and low-click-count.

### 1.5 Privacy-Aware Relayers and Paymasters
- Relayers/paymasters should be paid in a privacy-preserving way (e.g. with pETH) where possible.
- Ideally / eventually would be ideal if relayers/paymasters could merge user opertaions being paid out of a mixer into one aggregated SNARK so that onchain we can't even tell how the repayments for the user op in this bundle maps to different entities / groupings.
- Should be compatible with open standards like ERC-4337 rather than enshrining some custom relayer layer.

### 1.6 Privacy Legibility
Users need to understand what is private and what is not in their wallet and this is a non-trivial UX problem to solve as it can easily become subtle and complex. To keep it simple we propose an "all or nothing" privacy legibility strategy:
- **Mixed assets** (e.g. pETH, pDAI) are explicitly marked as “Private”
- **Unlinked balances** (via stealth or ephemeral addresses) are treated as “normal” — not labeled private explicitly. We call anything in a mixer or having come directly out of a mixer into a fresh address a "private" holding/position. But every other holding is "not necessarily private" even if funds are received to random fresh address. If the funds didn't come directly from a mixer or are not sitting in a mixer, it's not considered private funds.

### 1.7 Unified, Privacy-Aware Balances
User balances are presented in a unified interface:
- DAI across chains and stealth/ephemeral addresses is shown as a single DAI balance
- However private (mixed) assets are shown separately: e.g. `pDAI`
- Users don’t _need_ to track exact address or chain info of where funds are stored unless they want to

---

## 2. Read UX: Private Chain Access

### 2.1 Private Reads from the Chain
Wallets avoid leaking interest in specific addresses, contracts, or storage

- Short term: TEE + ORAM-powered RPCs
- Long term: PIR-based decentralized state queries or on-device light clients

### 2.2 Minimize Centralized APIs

This seems easy in theory (don't call third party apis) but in practice there's a lot to think about. How about getting things like asset prices and logos and little UX details that make the user's life easy ?


---

## 3. Propagation UX: Private Broadcasting of Transactions

### 3.1 Private Transaction Propagation
Transactions or intents should be broadcast without revealing user IP or geographic metadata

- L1-native propagation privacy is preferred (if/when available)
- Interim options exist in out-of-protocol tools like Waku, Nym, Dandelion++, HOPR / Gnosis VPN.

---

