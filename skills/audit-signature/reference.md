# Signature-Related Vulnerabilities

1. **Missing Nonce Replay** - Signatures without nonces can be replayed after state changes (e.g., KYC revocation)
2. **Cross Chain Replay** - Signatures without chain_id can be replayed across different chains
3. **Missing Parameter** - Critical parameters not included in signatures can be manipulated by attackers
4. **No Expiration** - Signatures without deadlines grant "lifetime licenses" and can be used indefinitely
5. **Unchecked ecrecover() Return** - Not checking if ecrecover() returns address(0) allows invalid signatures to pass
6. **Signature Malleability** - Elliptic curve symmetry allows computing valid signatures without the private key
