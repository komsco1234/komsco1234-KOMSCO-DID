KOMSCO Trusted Platform DID Method Specification
=================
18th April 2020

# Table of Contents
1. [Abstract](#abstract)
2. [DID Method Name](#name)
3. [Method Specific Identifier](#identifier)
    1. [Example](#example1)
4. [DID Document](#document)
    1. [Example](#example2)
5. [CRUD Operation Definitions](#crud)
    1. [Create (Register)](#create)
    2. [Read (Resolve)](#read)
    3. [Update](#update)
    4. [Delete](#delete)
6. [Security Considerations](#security)
7. [Privacy Considerations](#privacy)
8. [References](#references)
 
# Abstract <a name="abstract"></a>
KOMSCO, as a government-designated national ID card/passports manufacturer and issuer in the Republic of Korea, has always strived hard to respond to the world's trends and to achieve protection against forgery and alteration for a long time.
KOMSCO Trusted platform is the blockchain based identity system. KOMSCO Trusted Platform provides services in the public sector such as safe payment and ID authentication as well as information protection and management services. 
KOMSCO Decentralized Identifiers is a distributed identifier designed to provide a way for a community connected to the KOMSCO Trusted platform to uniquely identify an individual, organization, or IoT device. The role of a KOMSCO DID is to provide a service that supports id-authentication and personal information verification. 

The KOMSCO DID method specification conforms to the requirements specified in the Decentralized Identifiers (DIDs) v1.0 [**[1]**](https://w3c.github.io/did-core/), currently published by the W3C Credentials Community Group. 
For more information about DIDs and DID method specifications, please see the DID Spec and the DID Primer.
 
# DID Method Name <a name="name"></a>

The namestring that shall identify this DID method is: `komsco`

A DID that uses this method MUST begin with the following prefix: `did:komsco`. Per the DID specification, this string MUST be in lowercase. The remainder of the DID, after the prefix, is specified below.

# Method Specific Identifier <a name="identifier"></a>

The method specific identifier is composed of a Hex-encoded KOMSCO Identifier Number (KIN) (without a `0x` prefix).
```
komsco-did = "did:komsco:" + <first 16-bytes of public key w/ b58 encoding>
```
## Example <a name="example1"></a>

Example `komsco` DIDs:
```
did:komsco:123456789abcdefghi
```
# DID Document <a name="document"></a>

## Example <a name="example2"></a>
```
{
	"@context": "https://w3id.org/did/v1",
	"id": "did:komsco:123456789abcdefghi",
	
	"publicKey": [{
		"id": "did:komsco:123456789abcdefghi",
		"type": "Secp256k1VerificationKey2020",
		"owner": "did:komsco:223456789abcdefghi",
		"publicKeyHash": "e3FA89810623759d53361a297305c391c8280e66"
	       }],

	"authentication": [
		"id": "did:komsco:323456789abcdefghi",
	],
	"proof": [
		"id": "did:komsco:323456789abcdefghi",
	],
	"service": [{
		"type": "Schema",
		"serviceEndpoint": "https://komsco-did.herokuapp.com/"
	       }
	]
}
```

@context - (required) This is a JSON-LD keyword that points to the semantic schema of the document. This must point to https://w3id.org/did/v1, at least until a subsequent version of the context is created.
id - (required) This is another JSON-LD keyword that signifies the value as a URL representing the surrounding object. Note that DIDs are URLs. We are using the lower 16 bytes of the initial public key as the identifier, since this will provide randomness tied to the document.
publicKey - An array of PublicKey objects that represent the public keys that can validate this identity. Proofs on Verified Credentials and Proof Responses will reference the id of the key used to perform the signing.
authentication - Another array of PublicKey objects that represents identities associated with this DID Document. PublicKeys defined exclusively in the authentication section will only be used for authenticating the association to this DID Document for update purposes. This is useful for allowing a 2nd party, such as Workday, to help in key recovery.
service - An array of ServiceEndpoint objects that aid in service discovery. Service endpoints are optional; although in Workday, we plan to always include ID Hubs.
proof - An array of Proof objects containing cryptographic signatures over the contents of the rest of the DID Document, together metadata about the signature. The smart contract verifies the signature before storing the DID Document on the ledger.

# CRUD Operation Definitions <a name="crud"></a>

## Create <a name="create"></a>

In order to create a `komsco` DID, a KOMSCO Identity Manager (KIM) smart contract, and KOMSCO Service Manager (KSM) smart contract must be deployed on KOMSCO blockchain.
KIM is compliant with the * standard, and KSM is a default service key resolver.

`komsco` DID creation is done by submitting a transaction to the KIM smart contract invoking the following method:
```
function createIdentity(address recoveryAddress, address[] memory providers, address[] memory resolvers) public returns (uint KIN);
```
This will generate the corresponding id-string (KIN) and assign control to the caller address. Identities are denominated  by KINs, which are unique but otherwise uninformative.

## Read <a name="read"></a>

To construct a valid DID document from an `komsco` DID, the following steps are performed:

1. Invoke the `function getIdentity(uint KIN)` function to KIM for getting `MANAGEMENT` key.
1. For each returned key address, look up the associated key.
1. For each `MANAGEMENT` public key hash:
	1. Add a `publicKey` element of type `Secp256k1VerificationKey2020` and `KomscoManagementKey` to the DID Document.
    1. Add an `authentication` element of type `Secp256k1VerificationKey2020`, referencing this `publicKey`.
1. Invoke the `function getKeys(uint KIN)` function to KSM for getting  for `SERVICE` key.
1. For each `SERVICE` public key hash:
	1. Add a `publicKey` element of type `Secp256k1VerificationKey2020` and `KomscoServiceKey` to the DID Document.
	1. Add an `service` element of type  `Secp256k1VerificationKey2020`, referencing this `publicKey`.

Note: Service endpoints and other elements of a DID Document may be supported in future versions of this specification.

## Update <a name="update"></a>

The DID Document may be updated by invoking the relevant KSM smart contract functions as follows:
```
function addKey(address _key, uint KIN, bytes name)
function removeKey(address _key, uint KIN)
```
## Delete <a name="delete"></a>

Revoking the DID can be supported by executing a `destructIdentity` operation that is part of the KIM smart contract. This will remove the KIM and KSM's storage and code from the state, effectively marking the DID as revoked.
```
function destructIdentity(uint KIN)
```

# Security Considerations <a name="security"></a>

When a user creates and registers its own `komsco` DID in the KOMSCO blockchain, he (or she) can selectively register either recovery key or provider key. 
- Recovery key is a KOMSCO Trusted platform  address (either an external account or smart contract) that can be used to recover lost Identities when you accidentally lose your private key. The recovery key must be set to a different value than the management key. It is safest to physically and logically allow a trusted third party, separate from the user, to archive the recovery key.
- The provider key is the KOMSCO Trusted platform  address (external account or smart contract) that is authorized to be used on behalf of the management key. Since the provider key is allowed to operate in place of the management key, the provider key must be registered only when the user needs it, not at the time of distribution. Also, do not forget to delete the provider key when the proxy operation is complete.

# Privacy Considerations <a name="privacy"></a>

- The KOMSCO Trusted platform will have a claim/achievement structure in the next development phase. Claims are verifiable credentials, often consisting of a hash of personal information. GDPR protects pseudonymized data because of the "linkability" of an unreadable hash. Therefore, claims must be stored in a blockchain in such a way that personal information can not be inferred from hash-processed claims.

# References <a name="references"></a>
----------

 **[1]** https://w3c-ccg.github.io/did-spec/

 **[2]** https://github.com/WebOfTrustInfo/rebooting-the-web-of-trust-fall2017/blob/master/topics-and-advance-readings/did-primer.md

 **[3]** https://www.iso.org/iso-8601-date-and-time-format.html

