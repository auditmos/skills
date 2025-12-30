* [ ] All signatures include and verify nonces
* [ ] Nonces properly incremented after signature consumption
* [ ] chain_id is included in EIP-712 domain separator
* [ ] All relevant parameters are included in signed messages
* [ ] Signatures have expiration timestamps (deadline)
* [ ] Deadline validation implemented (block.timestamp <= deadline)
* [ ] ecrecover() return values are checked for address(0)
* [ ] Using OpenZeppelin's ECDSA library to prevent malleability
* [ ] Signature verification happens before state changes (CEI pattern)
* [ ] No signature reuse possible after revocation/state change
