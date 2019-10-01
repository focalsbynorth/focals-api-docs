---
layout: default
title: Encryption
---
# Focals Developer API End to End Encryption

The Focals Developer API uses a flexible encryption scheme that allows sensitive fields to be safely encrypted while non-sensitive fields remain plaintext for validation purposes.

The North Ability Client Library can perform both encryption and decryption for you.

Note in all examples in this document, __JWE__ refers to a JWE encrypted object in accordance with this RFC: https://tools.ietf.org/html/rfc7516

# Encryption

The encryption function takes these inputs and produces the following outputs:

Input:
* payload - a JSON object with sensitive fields to be encrypted
* fields - A list of JSON paths of the fields in payload that need to be encrypted

Output:
A JSON object with the following top level fields:
* version - semantic version of the encryption wrapper, set to 2.0.0 at this time
* plain - (optional, see note below) the original JSON but with each encrypted field replaced with a JSON Reference to the encrypted section
* encrypted - a JWE with JOSE headers.  When decrypted this will provide the plain data that the JSON References in `plain` will resolve against

Object keys are never encrypted. If keys are sensitive then they need to be put in a subobject, then that entire subobject needs to be encrypted.

Encryption steps:
1. Create output object with `version` field set to '2.0.0'
1. Escape any existing `$ref` instances in the source document with a double dollar sign, i.e. `$$ref`
1. For each JSON path in the `fields` list:
    1. Copy the value to the encrypted section
    1. Replace the value in the source document with a local JSON reference to the value in the `encrypted` field of the output object.
1. Copy source document with all sensitive values replaced with `$ref$` into the `plain` field on the output object
1. Encrypt the entire `encrypted` section using the recipient public keys

Example
Before encryption:

[
    true,
    "salt",
    "pepper",
    "paprika",
    {
        "foo": "bar",
        "moo": [
            "a", "b", "c"
        ]
    },
    { "$ref": "#/0" }
]
After encrypting /1, /2, /3, and /4/foo (encrypted section is written in plain for demonstration)

{
    "version": "2.0.0"
    "plain": [
        true,
        { "$ref": "#/encrypted/0" },
        { "$ref": "#/encrypted/1" },
        { "$ref": "#/encrypted/2" },
        {
            "foo": { "$ref": "#/encrypted/3" },
            "moo": [
                "a", "b", "c"
            ]
        },
        { "$$ref": "#/0" }
    ]
    "encrypted": __JWE__ [
        "salt",
        "pepper",
        "paprika",
        "bar"
    ]
}

Note that the $ref from the original data was preserved by escaping it with a double dollar sign.


# Decryption

Decryption occurs by reversing the encryption steps.
The decryption function takes these inputs and produces the following outputs:

Inputs:
* encrypted v2 abilities packet
* private key

Outputs:
* original source document in plain-text

Decryption steps:
1. Decrypt the entire `encrypted` JWE object using the private key
1. Replace all local JSON references in the `plain` object with their values, which are resolved from the `encrypted` object.
1. Escape any `$$ref` instances in the source document with `$ref`, restoring their original encoding.
1. Return the `plain` object, which is now identical to the original source object.

## Special Cases

If plain is omitted then the entire packet is in the value of encrypted. That means this:

{
    "version": "2.0.0",
    "encrypted": __JWE__ {
        "foo": "bar"
        "actionId": "reply"
    }
}
is equivalent to this:

{
    "version": "2.0.0",
    "plain": { "$ref": "#/encrypted/0" }
    "encrypted": __JWE__ [
        {
            "packetId": "XYZ"
            "actionId": "reply"
        }
    ]
}

## Getting the device public keys

Before sending an encrypted message it is necessary to retrieve the public keys for the recipient.
The Focals Developer API provides a route to retrieve the device public keys for a given user.
These public keys can be used as inputs into the North Ability Client Library encryption function.

The easiest way to get these keys is to use the North Ability Node.js Client Library `getPublicKeys` function.

If the library is not available in your desired language, instead directly query the North cloud route:
`GET https://cloud.bynorth.com/v1/apiintegration/device-keys` with the following query string parameters:
* userId
* integrationId
* apiKey
* apiSecret
