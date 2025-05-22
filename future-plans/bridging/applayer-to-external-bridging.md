---
description: How AppLayer would bridge with other blockchains
---

# AppLayer-to-External Bridging (Ethereum, Solana, etc.)

Bridging between AppLayer and other conventional blockchains brings multiple edge cases. For example, it's not possible to natively push data into these chains without paying transaction fees, and those external networks are limited on both processing power and how much signature verification can be done.

Knowing this, at least for now, the bridging implementation for them would be handled by Sentinels and owned by AppLayer Labs and its most trusted partners to ensure operational safety. The contract would check signatures of Validators and Sentinels but only the Sentinels could write into the contracts on these chains.

This method of bridging would follow lock/release mechanisms, unless a token was fully integrated with mint/burn bridging.

