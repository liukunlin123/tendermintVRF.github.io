#### **Algorand的vrf.go和proposal.go**

**vrf.go: 比较重要的几个函数**

```go
// VrfKeygenFromSeed deterministically generates a VRF keypair from 32 bytes of (secret) entropy.
func VrfKeygenFromSeed(seed [32]byte) (pub VrfPubkey, priv VrfPrivkey) {
	C.crypto_vrf_keypair_from_seed((*C.uchar)(&pub[0]), (*C.uchar)(&priv[0]), (*C.uchar)(&seed[0]))
	return pub, priv
}

// VrfKeygen generates a random VRF keypair.
func VrfKeygen() (pub VrfPubkey, priv VrfPrivkey) {
	C.crypto_vrf_keypair((*C.uchar)(&pub[0]), (*C.uchar)(&priv[0]))
	return pub, priv
}

// Prove constructs a VRF Proof for a given Hashable.
// ok will be false if the private key is malformed.
func (sk VrfPrivkey) Prove(message Hashable) (proof VrfProof, ok bool) {
	return sk.proveBytes(hashRep(message))
}

// Hash converts a VRF proof to a VRF output without verifying the proof.
// TODO: Consider removing so that we don't accidentally hash an unverified proof
func (proof VrfProof) Hash() (hash VrfOutput, ok bool) {
	ret := C.crypto_vrf_proof_to_hash((*C.uchar)(&hash[0]), (*C.uchar)(&proof[0]))
	return hash, ret == 0
}

// Verify checks a VRF proof of a given Hashable. If the proof is valid the pseudorandom VrfOutput will be returned.
// For a given public key and message, there are potentially multiple valid proofs.
// However, given a public key and message, all valid proofs will yield the same output.
// Moreover, the output is indistinguishable from random to anyone without the proof or the secret key.
func (pk VrfPubkey) Verify(p VrfProof, message Hashable) (bool, VrfOutput) {
	return pk.verifyBytes(p, hashRep(message))
}
```

#### **Ontology**

**utils.go和vrf.go定义了与vrf相关的函数**

**(node_utils.go和service.go)**

