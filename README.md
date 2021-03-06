# Apple Receipt

[![Gem Version](https://badge.fury.io/rb/apple_receipt.svg)](https://badge.fury.io/rb/apple_receipt)
[![Build Status](https://travis-ci.org/koenrh/apple_receipt.svg?branch=master)](https://travis-ci.org/koenrh/apple_receipt)

This gem allows you to read and verify Apple receipts. It was originally built
to locally (server-side) verify the validity of receipts that are embedded in
[Status Update Notifications](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Chapters/Subscriptions.html#//apple_ref/doc/uid/TP40008267-CH7-SW13).
These receipts have a different format than the [documented](https://developer.apple.com/library/content/releasenotes/General/ValidateAppStoreReceipt/Chapters/ValidateLocally.html#//apple_ref/doc/uid/TP40010573-CH1-SW2)
App Store receipts you might be familiar with, which are [PKCS #7](https://tools.ietf.org/html/rfc2315)
containers with a payload (receipt data) encoded using [ASN.1](https://www.itu.int/itu-t/recommendations/rec.aspx?rec=X.690).

:warning: Note that this only covers the receipt data (signed data). You should
not rely on (local) verification for data that is in the notification object, but
not in the receipt (e.g. `notification_type`).

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'apple_receipt'
```

And then execute:

    bundle

Or install it yourself as:

    gem install apple_receipt

## Usage

```ruby
require 'apple_receipt'

# Check receipt's validity (certificate chain, and signature)
receipt_raw = File.read('./receipt.txt')
receipt = AppleReceipt::Receipt.new(receipt_raw)
receipt.valid?
# => true

# Read receipt's data (data in example shortened for brevity)
receipt.purchase_info
# => {
#   "quantity"=>"1",
#   "expires_date_formatted"=>"2018-01-23 17:03:44 Etc/GMT",
#   "is_in_intro_offer_period"=>"false",
#   "is_trial_period"=>"false",
#   "item_id"=>"1190360447",
#   "app_item_id"=>"947936149",
#   "transaction_id"=>"160000408504141",
#   "web_order_line_item_id"=>"160000011000001",
#   "bid"=>"com.foo.bar",
#   "product_id"=>"com.foo.bar.monthly",
#   "purchase_date"=>"2017-12-23 17:03:44 Etc/GMT",
#   "original_purchase_date"=>"2017_12_23 17:03:53 Etc/GMT"
# }
```

## Receipts

A receipt is encoded as base64, and is formatted as a [NeXTSTEP](https://en.wikipedia.org/wiki/Property_list#NeXTSTEP)
dictionary:

```
{
  "signature" = "[base64-encoded signature]";
  "purchase-info" = "[base64-encoded purchase data]";
  "pod" = "[integer]";
  "signing-status" = "0";
}
```

### Signature

The `signature` entry contains base64-encoded binary data, which has the following
layout:

- **1 byte** - Receipt version (e.g. version 3).
- **128 bytes** (version 2) or **256 bytes** (version 3) - Signature.
- **4 bytes** - Length (in number of bytes) of the certificate.
- **N bytes** - DER-encoded certificate.

The version 2 and 3 receipt certificates are signed, respectively, by:

- **Apple iTunes Store Certification Authority** (version 2)
  - Serial: 26 (`0x1a`)
  - Subject: `C=US, O=Apple Inc., OU=Apple Certification Authority, CN=Apple iTunes Store Certification Authority`
- **Apple Worldwide Developer Relations Certification Authority** (version 3)
  - Serial: 134752589830791184 (`0x1debcc4396da010`)
  - Subject: `C=US, O=Apple Inc., OU=Apple Worldwide Developer Relations, CN=Apple Worldwide Developer Relations Certification Authority`

Both certificates chain up to:

- **Apple Root CA**
  - Serial: 2 (`0x2`)
  - Subject: `C=US, O=Apple Inc., OU=Apple Certification Authority, CN=Apple Root CA`

### Purchase info

The `purchase-info` entry contains a base64-encoded NeXTSTEP dictionary that contains
the actual receipt data (purchase info).

## Validation

First, the signing certificate is parsed from the signature binary data. The
validation of the receipt works as follows.

1. Verify that the signing certificate is valid, i.e. it is not expired, and
   it chains up to either of the bundled Apple root certificates.
2. Verify that the signature over the signed data (version number and receipt
   data) is signed by the private key that correspond to the public key that is
   in the signing certificate.

## Contributing

Bug reports and pull requests are welcome on [GitHub](https://github.com/koenrh/apple_receipt).
This project is intended to be a safe, welcoming space for collaboration, and
contributors are expected to adhere to the [Contributor Covenant](https://www.contributor-covenant.org)
code of conduct.

## License

The gem is available as open source under the terms of the [ISC License](https://opensource.org/licenses/ISC).
