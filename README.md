# Nostr NFC APDU Protocol (NNAP)

**Version:** 0.1
**Date:** 2025-07-20
**Status:** Draft
**License:** MIT
**Authors:** La Crypta, Agustín Kassis, et al.
**Domain of Application:** Lightning Network NFC Payments (Phone-to-Phone, Wallet-to-POS)
**Inspired by:** Boltcard, LNURL, NWC, Android HCE, ISO-7816

---

## Overview

This specification defines the open NNAP (Nostr NFC APDU Protocol) used to initiate and complete Lightning Network payments using NFC communication between two Android devices:

* One acts as a Wallet (emulating an NFC card using Android HCE)
* The other as a POS (reading the Wallet and initiating a Lightning payment via NWC)

The protocol relies on the exchange of APDU commands (ISO 7816-4) and Nostr Wallet Connect (NIP-47) events.

---

## Roles

| Role       | Device                     | Function                                                                       |
| ---------- | -------------------------- | ------------------------------------------------------------------------------ |
| **Wallet** | Android phone (HCE Tag)    | Emulates tag with NDEF message; receives invoice and sends signed NWC event.   |
| **POS**    | Android phone (NFC Reader) | Reads Wallet tag, sends invoice, publishes event to relay, and tracks payment. |

---

## Transport Layer

* **NFC Protocol:** ISO/IEC 14443-4 (ISO-DEP)
* **Tag Type:** Emulated Type 4A tag (via Android Host Card Emulation)

---

## Application Identifier (AID)

A unique AID must be registered by the Wallet App to avoid conflict with existing card apps.

```hex
AID: F005570202
```

Declared in `apduservice.xml` of the Android Wallet App.

---

## Initial Request URI (NDEF Payload)

Upon NFC tap, the Wallet responds with an NDEF message encoded as:

```
nostr+walletconnect+request://relay=<relayUrl>
```

The POS sends the Lightning invoice to the Wallet over APDU chunks.

* **relay**: URL of the Nostr relay the POS should connect to immediately.
* The POS connects to the relay and waits for an incoming signed NWC event.

---

## APDU Commands

### 1. POS → Wallet: Send Invoice

**Instruction Code:** `0x10`

| Field | Description               |
| ----- | ------------------------- |
| CLA   | `0x80`                    |
| INS   | `0x10`                    |
| P1/P2 | Chunk index (MSB/LSB)     |
| Lc    | Length of payload         |
| Data  | Chunk of `bolt11` invoice |

---

### 2. Wallet → POS: Send NWC Event

**Instruction Code:** `0x20`

| Field | Description                      |
| ----- | -------------------------------- |
| CLA   | `0x80`                           |
| INS   | `0x20`                           |
| P1/P2 | Chunk index (MSB/LSB)            |
| Lc    | Length of payload                |
| Data  | Chunk of signed Nostr event JSON |

The POS reassembles the full event and publishes it to the previously connected relay.

---

## Nostr Event Format (NIP-47)

Example `pay_invoice` NWC event:

```json
{
  "id": "event_id",
  "pubkey": "wallet_pubkey",
  "kind": 23194,
  "created_at": 1687123123,
  "tags": [["p", "wallet_service_pubkey"]],
  "content": "{\"method\":\"pay_invoice\",\"params\":{\"invoice\":\"lnbc1...\",\"comment\":\"POS via NFC\"}}",
  "sig": "signature"
}
```

---

## Response Status Words

| Code   | Meaning                       |
| ------ | ----------------------------- |
| `9000` | Success                       |
| `6A82` | File not found / app inactive |
| `6B00` | Wrong parameters              |
| `6700` | Wrong length                  |

---

## Chunking Strategy

* All large messages (invoice, event) are split into 200-byte chunks
* Chunks are sequentially indexed with `P1/P2`
* Wallet and POS both reassemble full payloads

---

## Offline Support

This protocol supports flexible offline payment flows depending on which device has internet access:

* If the **user has internet** but the **POS does not**, the Wallet App may directly pay the invoice and receive the preimage via NWC response. It can then send the preimage back to the POS via APDU to verify the payment.

* If the **user lacks internet**, the Wallet App wraps the invoice inside a signed NWC event and sends it to the POS over APDU. The POS, having a connected relay, publishes the event on behalf of the user and waits for confirmation using any method it supports (e.g., NWC, LND subscription, polling a webhook, etc.).

* POS cannot pay; it only submits events

* Wallet signs events with Nostr private key

* Relay is defined upfront in URI for immediate connection

* Future versions may include replay protection, encryption, or challenge-response

---

## Future Extensions

* Add optional query params: `amount`, `comment`, `nonce`
* Support NIP-44 encryption
* Support POS pubkey in URI for mutual auth
* iOS reader compatibility (POS only)

---

## References

* [NIP-47: Nostr Wallet Connect](https://github.com/nostr-protocol/nips/blob/master/47.md)
* [NDEF Format](https://nfc-forum.org/our-work/specification/specifications/)
* [Android HCE Documentation](https://developer.android.com/guide/topics/connectivity/nfc/hce)
* [BOLT #11: Payment Request](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)
