# UMAD-05: Payreq Request

The first request in the UMA protocol is the payreq request, which aligns with the same request in LNURL-PAY
([LUD-06](https://github.com/lnurl/luds/blob/luds/06.md)), but with a few added query parameters for compliance,
currency conversion, and authentication. This request tells the receiving VASP to generate an invoice for a specified
amount on behalf of the receiving user.

**NOTE:** This needs to be a POST request instead of a GET (like in LNURL) because with travel rule info, the
payerData field might be too long to fit in a URL.

```http
POST vasp2.com/<callback>
```

`<callback>` is the URL returned in the `callback` field of the LNURLP response.

The body of the request is a JSON object with the following fields:

```raw
{
  "payerData": {
    // See LUD-18 for details.
    <json payerdata>
  },
  // An int64 - This is the amount in the smallest unit of the specified receiving currency (eg. cents for USD).
  "amount": number,
  // The currency code of the receiving currency (eg. "USD"). This must be one of the currencies returned in the
  // LNURLP response.
  "currency": string,
  // The UMA protocol version that will be used for this transaction. See [UMAD-08](/umad-08-versioning.md).
  "umaVersion": "1.0",
  "payeeData":  {
    "name": { "mandatory": boolean },
    "identifier": { "mandatory": boolean },
    "countryCode": { "mandatory": boolean },
    ... All fields optional and more fields may be negotiated. See [LUD-22](https://github.com/lnurl/luds/pull/252)
  },
}
```

## Payee Data

The `payeeData` field is optional and is used to request additional information about the receiving user. The `mandatory`
field in each subfield indicates whether the receiving VASP is required to provide that information to proceed with the transaction.
See [LUD-22](https://github.com/lnurl/luds/pull/252) for more details. Note that the receiving VASP may choose to avoid sending
any payee identity information for privacy reasons, which may cause the payment to fail if the sending VASP requires it.
For that reason, the sender SHOULD NOT require any payee identity information to be sent by the receiver unless it is
absolutely necessary.

### Common Payee Data Fields

The following is a non-exhaustive list of common payee data fields that *may* be requested by the sender:

- `name`: The full name of the receiving user.
- `identifier`: The canonical receiving UMA address of the receiver.
- `countryCode`: The ISO 3166-1 alpha-2 country code of the receiving user.
- `email`: The email address of the receiving user.
- `accountNumber`: The account number of the receiving user at the receiving VASP.

Note that this struct is extensible, so any field can be added as long as it is agreed upon by both VASPs.

## Payer Data

The `payerData` field is a JSON object that contains information about the payer as described in
[LUD-18](https://github.com/lnurl/luds/blob/luds/18.md). There's also an UMA-specific payerdata field - `compliance`.
Its structure is:

```raw
{
  "compliance": {
    // The public key of the node that sender will use to send the payment. This can be used in place of utxos
    // if your compliance provider supports it.
    "nodePubKey": string|undefined,
    // A list of the expected UTXOs over which sender side may send transaction (sender's channels).
    // This is only needed if your compliance provider does not support checks via node public key.
    "utxos": string[]
    // The travel rule info for the transaction encrypted with the receiving VASP's encryptionPublicKey.
    "encryptedTravelRuleInfo": string|undefined,
    // The message format or protocol for the encryptedTravelRuleInfo. It is an optional string of the format
    // <protocol>@version (e.g. IVMS@101.2023). If this field is empty, assume raw freeform JSON.
    "travelRuleFormat": string|undefined,
    // An enum indicating whether the payer is a KYC'd customer of the sendingVASP.
    "kycStatus": KycStatus (enum),
    // The sending VASP's signature over sha256_hash(<sender UMA> (eg. $alice@vasp1.com) + signatureNonce + signatureTimestamp),
    "signature": string,
    "signatureNonce": string,
    "signatureTimestamp": number,
    // A url which the receiving VASP should call on transaction completion to notify the sending VASP of
    // the utxos used to complete the transaction. See [UMAD-07](/umad-07-post-tx-hooks.md).
    "utxoCallback": string|undefined,
  }
}
```

The `nodePubKey` and `utxos` fields can be used by the receiving VASP to pre-screen the transaction with a compliance
provider. The information included in the `encryptedTravelRuleInfo` field is dependent on regulations enforced on the
Sending VASP. It can be in any format, but if the `travelRuleFormat` field is included, the receiving VASP can use it
to parse the `encryptedTravelRuleInfo` field. For example, [IVMS](https://www.intervasp.org/#IVMS-1012023) is a
standardized format for Travel Rule information, which is supported by many VASPs.
