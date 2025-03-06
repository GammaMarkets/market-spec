Marketplace Protocol
---------------------------

`draft`

A protocol specification for decentralized marketplaces on Nostr that provides an interoperable, full-featured e-commerce framework.

## Table of Contents
1. Protocol Requirements
2. Core Protocol Components
3. Events and Kinds
4. Order Communication Flow and Payment Processing
5. Product Reviews
6. Implementation Guidelines

## 1. Protocol Requirements

The protocol defines both required core components and optional features to support diverse marketplace needs.

### Required Components
Implementations MUST support the following core features:

- Product listing events (Kind: 30402)
- Product collection events (Kind: 30405) for product-to-collection references
- Merchant's preferences
- Order communication and processing via [NIP-17](17.md) encrypted messages

### Optional Components
These features MAY be implemented based on specific marketplace needs:

- Extended product metadata
- Shipping options (Kind: 30406)
- Product collections (Kind: 30405) 
- Drafts following [NIP-37](37.md)
- Product reviews (Kind: 31555)
- Service assisted order and payment processing

#### Watch-only clients
Watch-only clients are applications that allow users to display products without implementing full e-commerce capabilities. These clients don't need to support all required components - product rendering alone can be sufficient. However, ideally, they should also handle logic for looking up collections, reviews, and shipping options. Support for order communication using [NIP-17](17.md) is optional.

## 2. Core Protocol Components

### Core Flows
1. Merchant Preferences
   - Application preferences via [NIP-89](89.md)
   - Payment method preferences via kind `0` tags

2. Order Processing
   - Encrypted buyer-seller communication
   - Status updates and confirmations
   
3. Shipping
   - Option definition and pricing
   - Geographic restrictions
   
4. Payment
   - Multiple payment methods
   - Verification and receipts

Standard e-commerce flow:
1. Product discovery
2. Cart addition
3. Merchant preference verification
4. Shipping calculation
5. Payment processing
6. Order confirmation
7. Encrypted message follow-up

### Merchant Preferences
Merchants MAY specify preferences for how they want users to interact with them, including which applications to use and payment methods to accept. These preferences ensure a consistent experience and streamline operations. Merchants indicate their preferences through two mechanisms:

1. Application Preferences ([NIP-89](89.md)):
- The recommended application MUST publish a kind `31990` event
- The merchant MUST publish a kind `31989` event recommending that application

2. Payment Preferences:
- Set via `payment_preference` tag in the merchant's kind `0` event
- Valid values: `manual | ecash | lud16` 
- Defaults to `manual` if not specified

Applications implementing this NIP MUST handle preferences as follows:

1. When `payment_preference` is `manual`:
- If merchant recommends an app: MUST direct users to that app
- If no app recommendation: Use traditional interactive flow (buyer places order and waits for merchant's payment request)

2. When `payment_preference` is `ecash` or `lud16`:
- If merchant recommends an app: SHOULD direct users there first, but they MAY also offer to continue if compatible with the payment preference
- If no recommendations: Use specified payment method directly

3. When no preferences are set:
- Use traditional interactive flow
   - Buyer sends order
   - Wait for merchant's payment request

Buyers can verify merchant preferences by:
- Checking kind `31990` events for recommended applications
- Checking kind `0` events for payment preferences

This verification helps buyers follow merchant-approved paths and avoid potential scams or poor experiences.

## 3. Events and Kinds

### Product Listing (Kind: 30402)

Products are the core element in a marketplace. Each product listing MUST contain basic metadata and MAY contain additional details. Their configuration is the source of truth, overriding other possible configurations of other market elements such as collections, no configuration is cascaded to products, they MUST explicitly reference an attribute to inherit it.

**Content**: Product description, markdown is allowed
**Required tags**:
- `d`: Unique product identifier for referencing the listing
- `title`: Product name/title for display
- `price`: Price information array `[<amount>, <currency>, <optional frequency>]`
  - amount: Decimal number (e.g., "10.99")
  - currency: ISO 4217 code (e.g., "USD", "EUR")
  - frequency: Optional subscription interval using ISO 8601 duration units (e.g. 'D' for daily, 'W' for weekly, 'Y' for yearly).

**Optional tags**:
- Product Details:
  - `type`: Product classification `[<type>, <format>]`
    - type: "simple", "variable", or "variation"
    - format: "digital" or "physical"
    - Default/if not present: type: "simple", format: "digital"
  - `visibility`: Display status ("hidden", "on-sale", "pre-order"). Default/if not present: "on-sale"
  - `stock`: Available quantity as integer
  - `summary`: Short product description
  - `spec`: Product specifications `[<key>, <value>]`, can appear multiple times

- Media:
  - `image`: Product images `[<url>, <dimensions>, <sorting-order>]`, MAY appear multiple times
    - url: Direct image URL
    - dimensions: Optional, in pixels, "<width>x<height>" format, if not present the place in the array should be respected by using an empty string `""`
    - sorting order: Optional integer for order sorting. Values are sorted from lowest to highest, independent of starting value (not restricted to start with 0 or 1)

- Physical Properties:
  - `weight`: Product weight `[<value>, <unit>]` using ISO 80000-1
  - `dim`: Dimensions `[<l>x<w>x<h>, <unit>]` using ISO 80000-1

- Location:
  - `location`: Human-readable location string or collection coordinates
  - `g`: Geohash for precise location lookup or collection coordinates

- Organization:
  - `t`: Product categories/tags, MAY appear multiple times
  - `a`: Product reference "30402:<pubkey>:<d-tag>", MUST appear only once to reference parent products in a variable/variation configuration
  - `a`: Collection reference "30405:<pubkey>:<d-tag>", MAY appear multiple times
  - `shipping_option`: Shipping options, MAY appear multiple times
    - Format: "30406:<pubkey>:<d-tag>" for direct options
    - Format: "30405:<pubkey>:<d-tag>" for collection shipping
    - `extra-cost`: Optional third element in the array, to add extra cost (in the product's currency) for the shipping method. In case of reference a collection the extra cost should be applied to all shipping options from the collection.

```jsonc
{
  "kind": 30402,
  "created_at": <unix timestamp>,
  "content": "<product description in markdown>",
  "tags": [
    // Required tags
    ["d", "<product identifier>"],
    ["title", "<product title>"],
    ["price", "<amount>", "<currency>", "<optional frequency>"],

    // Product details
    ["type", "<simple|variable|variation>", "<digital|physical>"],  // Defaults: simple, digital
    ["visibility", "<hidden|on-sale|pre-order>"],  // Default: on-sale
    ["stock", "<integer>"],  // Available quantity
    ["summary", "<short description>"],
    
    // Media and specs
    ["image", "<url>", "<dimensions>", "<sorting-order>"],
    ["spec", "<key>", "<value>"],  // Product specifications (e.g., "screen-size", "21 inch"). MAY appear multiple times
    
    // Physical properties (for shipping)
    ["weight", "<value>", "<unit>"],  // ISO 80000-1 units (g, kg, etc)
    ["dim", "<l>x<w>x<h>", "<unit>"], // ISO 80000-1 units (mm, cm, m)
    
    // Location
    ["location", "<address string>"],
    ["g", "<geohash>"],
    
    // Classifications
    ["t", "<category>"],
    
    // References
    ["shipping_option", "<30406|30405>:<pubkey>:<d-tag>", "<extra-cost>"],  // Shipping options or collection, MAY appear multiple times
    ["a", "30405:<pubkey>:<d-tag>"]  // Product collection
  ]
}
```

#### Notes
1. Product Configuration:
   - Products can be simple, variable (with options), or variations of variable products
   - Digital products skip shipping requirements
   - Visibility controls product display status

2. Variable products: 
   - The parent or "root" product should use `variable` as value for `type`
   - The variations of the parent product should use `variation` as value for `type`. 
   - Variations MUST include an `a` tag pointing to the `variable` parent product.
 
2. Shipping Rules:
   - Shipping options can be defined directly by pointing to a shipping event, or inherited from collections
   - If the product specifies product-specific shipping, and also from a collection, shipping options MUST be merged.

3. Collections and Categories:
   - Products can refer to one o multiple collections using `a` tags, whether or not they are part of it, for discoverability purposes.
   - Categories ("t" tags) aid in discovery and organization

4. Location Support:
   - Optional location data aids in local marketplace features, they can point to a collection event to inherit it's value
   - Geohash enables precise location-based searches, they can point to a collection event to inherit it's value

### Product Collection (Kind: 30405)
A specialized event type using [NIP-51](51.md) like list format to organize related products into groups. Collections allow merchants or any user to create meaningful product groupings and share common attributes that products can also reference, establishing one-to-many relationships.

**Content**: Optional collection description

**Required tags**:
- `d`: Unique collection identifier
- `title`: Collection display name/title
- `a`: Product references `["a", "30402:<pubkey>:<d-tag>"]`
  - Multiple product references allowed
  - References must point to valid product listings

**Optional tags**:
- Display:
  - `image`: Collection banner/thumbnail URL
  - `summary`: Brief collection description

- Location:
  - `location`: Human-readable location string
  - `g`: Geohash for precise location lookup

- Reference Options:
  - `shipping_option`: Available shipping options `["shipping_option", "30406:<pubkey>:<d-tag>"]`, MAY appear multiple times

```jsonc
{
  "kind": 30405,
  "created_at": <unix timestamp>,
  "content": "<optional collection description>",
  "tags": [
    // Required tags
    ["d", "<collection identifier>"],
    ["title", "<collection name>"],
    ["a", "30402:<pubkey>:<d-tag>"],  // Product reference
    
    // Optional tags
    ["image", "<collection image URL>"],
    ["summary", "<collection description>"],
    
    // Location
    ["location", "<location string>"],
    ["g", "<geohash>"],
    
    // Reference Options
    ["shipping_option", "30406:<pubkey>:<d-tag>"],  // Available shipping options, MAY appear multiple times
  ]
}
```

#### Notes
1. Collection Management:
   - Collections can contain any number of products
   - Products can belong to multiple collections

2. Reference Model:
   - Collection settings (shipping, location, geohash) serve as references only
   - Products MUST explicitly reference collection resources to inherit collection attributes (e.g. shipping, location, geohash).
   - No automatic cascading of settings to products

3. Location Support:
   - Optional location data helps with marketplace organization
   - Enables geographic grouping of related products

### Drafts
Products and collections can be saved as private drafts while being prepared for publication. This allows merchants to work on listings before making them publicly visible. Implementation MUST follow [NIP-37](https://github.com/nostr-protocol/nips/blob/master/37.md) for draft management.

### Shipping Option (Kind: 30406)
A specialized event type for defining shipping methods, costs, and constraints. Shipping options can be published by merchants or third-party providers (delivery companies, DVMs, etc.) and referenced by product listings or collections.

**Content**: Optional human-friendly shipping description

**Required tags**:
- `d`: Unique shipping option identifier
- `title`: Display title for the shipping method
- `price`: Base cost array `[<base_cost>, <currency>]`
- `country`: Array of ISO 3166-1 alpha-2 country codes `[<code1>, <code2>, ...]`
- `service`: Service type ("standard", "express", "overnight", "pickup")

**Optional tags**:
- Extra details:
  - `carrier`: The name of the carrier that will be used for the delivery
- Time and Location:
  - `region`: Array of ISO 3166-2 region codes for which shipping method is available `[<code1>, <code2>, ...]`
  - `duration`: Delivery window `[<min>, <max>, <unit>]` using ISO 8601 duration units
    - min: Minimum delivery time
    - max: Maximum delivery time
    - unit: "H" (hours), "D" (days), "W" (weeks)
  - `location`: Physical address for pickup
  - `g`: Geohash for precise location

- Constraints:
  - `weight-min`: Minimum weight `[<value>, <unit>]` (ISO 80000-1)
  - `weight-max`: Maximum weight `[<value>, <unit>]`
  - `dim-min`: Minimum dimensions `[<l>x<w>x<h>, <unit>]`
  - `dim-max`: Maximum dimensions `[<l>x<w>x<h>, <unit>]`

- Price Calculations:
  - `price-weight`: Per weight pricing `[<price>, <unit>]`
  - `price-volume`: Per volume pricing `[<price>, <unit>]`
  - `price-distance`: Per distance pricing `[<price>, <unit>]`

```jsonc
{
  "kind": 30406,
  "created_at": <unix timestamp>,
  "content": "<optional shipping description>",
  "tags": [
    // Required tags
    ["d", "<shipping identifier>"],
    ["title", "<shipping method title>"],
    ["price", "<base_cost>", "<currency>"],
    ["country", "<ISO 3166-1 alpha-2>", "...", "..."],  // Array of country codes
    ["service", "<service-type>"],

    // Extra details
    ["carrier","<name of the carrier>"]
    
    // Time and Location
    ["region", "<ISO 3166-2 code>", "...", "..."],  // Array of region codes
    ["duration", "<min>", "<max>", "<unit>"],  // ISO 8601 duration units (H/D/W)
    ["location", "<address string>"],
    ["g", "<geohash>"],
    
    // Constraints
    ["weight-min", "<value>", "<unit>"],
    ["weight-max", "<value>", "<unit>"],
    ["dim-min", "<l>x<w>x<h>", "<unit>"],
    ["dim-max", "<l>x<w>x<h>", "<unit>"],
    
    // Price Calculations
    ["price-weight", "<price>", "<unit>"],
    ["price-volume", "<price>", "<unit>"],
    ["price-distance", "<price>", "<unit>"]
  ]
}
```

#### Implementation Examples

Local Pickup:
```jsonc
{
  "kind": 30406,
  "created_at": 1703187600,
  "content": "Downtown Store Pickup",
  "tags": [
    ["d", "downtown-pickup"],
    ["title", "Downtown Store Pickup"],
    ["price", "0", "USD"],
    ["country", "US"],
    ["region", "US-FL"],
    ["service", "pickup"],
    ["location", "123 Main St, Downtown, FL"],
    ["g", "dhwm9c4ws"]
  ]
}
```

Standard Shipping:
```jsonc
{
  "kind": 30406,
  "created_at": 1703187600,
  "content": "Standard Regional Shipping",
  "tags": [
    ["d", "standard-regional"],
    ["title", "Standard Shipping"],
    ["price", "5.99", "USD"],
    ["country", "US"],
    ["region", "US-FL"],
    ["service", "standard"],
    ["duration", "24", "72", "H"],  // 24-72 hours delivery window
    ["weight-max", "30", "kg"],
    ["dim-max", "120x60x60", "cm"],
    ["price-weight", "0.75", "USD", "kg"]
  ]
}
```

#### Notes
1. Event Management:
   - Create separate events for each distinct shipping option
   - Each option needs a unique `d` tag identifier
   - Merchants can reference third-party shipping options

2. Shipping Rules:
   - Physical pickup requires location and/or geohash
   - Weight/dimension constraints use ISO 80000-1 units

3. Client Behavior:
   - Group options by service type and location
   - Use geohash for distance-based sorting
   - Validate package constraints before offering options

## 4. Order Communication Flow and Payment Processing

Order processing and status updates use [NIP-17](17.md) encrypted direct messages, with three event kinds serving different purposes:

- Kind `14`: Regular communication between parties
  - General inquiries and responses
  - Order clarifications
  - Subject can be order ID or empty
  
- Kind `16`: Order processing and status. These messages include a `type` field that indicates the specific kind of message
  - Order creation and details. `type`: 1
  - Payment requests. `type`: 2
  - Status updates. `type`: 3
  - Shipping information. `type`: 4
  
- Kind `17`: Payment receipts and verification

Message direction is determined by the author, and `p` tag:
- Buyer → Merchant: event author is the buyer, `p` tag contains merchant's pubkey
- Merchant → Buyer: event author is the merchant, `p` tag contains buyer's pubkey

The payment request flow can operate in two modes:
1. Direct: Merchant processes requests manually. Payment request is initiated by the merchant
2. Service-assisted: Merchant's payment service handles requests. Payment request is initiated by the buyer

### Message Types
#### 1. Order Creation
Sent by buyer to initiate order process.

**Content:** (Optional) Human readable order notes or special requests

**Required tags:**
- `p`: Merchant's public key
- `subject`: Human-friendly subject line for order information
- `type`: Must be "1" to indicate order creation
- `order`: Unique identifier for the order
- `amount`: Total order amount in satoshis
- `item`: Product reference in format "30402:<pubkey>:<d-tag>" with quantity. MAY appear multiple times

**Optional tags:**
- `shipping`: Reference to shipping option "30406:<pubkey>:<d-tag>"
- `address`: Shipping address details
- `email`: Customer email for contact
- `phone`: Customer phone number for contact
- Other optional tags can be added with more details from the customer

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<merchant-pubkey>"],
    ["subject", "<order-info subject>"],
    ["type", "1"],  // Order creation
    ["order", "<order-id>"],  // Unique order identifier
    ["amount", "<total-amount-in-sats>"],
    
    // Order items (can repeat)
    ["item", "30402:<pubkey>:<d-tag>", "<quantity>"],
    
    // Shipping details
    ["shipping", "30406:<pubkey>:<d-tag>"],
    ["address", "<shipping-address>"],
    
    // Customer contact
    ["email", "<customer-email>"],  // Optional
    ["phone", "<customer-phone>"],  // Optional
  ],
  "content": "Order notes or special requests"
}
```
#### 2. Payment Request
There are two variants depending on payment processing mode: manual or automatic processing. After the buyer pays the payment request, they MUST send a payment receipt to the merchant using a kind:`17` dm.

##### Manual Processing (merchant → buyer)

In this mode, the merchant manually initiates the payment by sending a payment request to the buyer. This requires either:
- The merchant being online to process requests, or
- Having an automated system for processing payment requests, which can be run by the merchant or a service they rely on according to merchant preferences.

Important considerations:
- Merchant shouldn't have a recommended application in their merchant's preferences
- Final price may differ from the order creation time
- Merchants decide whether to honor original prices
- Buyers can cancel orders if they don't agree with price changes

**Content:** (Optional) Human readable payment instructions and notes

**Required tags:**
- `p`: Buyer's public key
- `subject`: Human-friendly subject line for order payment requests
- `type`: Must be "2" to indicate payment request
- `order`: The unique order identifier from the original order
- `amount`: Total payment amount in satoshis

**Optional tags:**
- `payment`: Payment method details, can appear multiple times for different options:
  - Lightning format: `["payment", "lightning", "<bolt11-invoice or lud16>"]`
  - Bitcoin format: `["payment", "bitcoin", "<btc-address>"]`
  - eCash format: `["payment", "ecash", "<cashu-req>"]`

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<buyer-pubkey>"],
    ["subject", "order-payment"],
    ["type", "2"],  // Payment request
    ["order", "<order-id>"],
    ["amount", "<total-amount-in-sats>"],
    
    // Payment options (can include multiple)
    ["payment", "lightning", "<bolt11-invoice|lud16>"],
    ["payment", "bitcoin", "<btc-address>"],
    ["payment", "ecash", "<cashu-req>"],
  ],
  "content": "Payment instructions and notes"
}
```

##### Automatic Processing (buyer → merchant)
In this mode, the merchant MUST set valid payment options in their kind:`0` event (such as `cashu` or `lud16`). The key difference is that the buyer initiates the payment request using information provided by the merchant. For merchants using `manual` payment preference, they SHOULD use [NIP-89](89.md) to specify their preferred payment processing service, which can then automatically handle payment requests on their behalf, as described in the merchant preferences section above.

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<merchant-pubkey>"],
    ["subject", "order-payment"],
    ["type", "2"],  // Payment request
    ["order", "<order-id>"],
    ["amount", "<total-amount-in-sats>"],
    
    // Payment details from service
    ["payment", "lightning", "<bolt11-invoice|bolt12-offer>"],
    ["payment", "bitcoin", "<btc-address>"],
    ["payment", "ecash", "<cashu-req>"],
  ],
  "content": "Service-generated payment details"
}
```

#### 3. Order Status Updates
Once the merchant receives payment, they MUST update the status to "confirmed". Status updates can be sent as soon as a new order is acknowledged, initially setting the status to "pending". The "pending" status is optional and can be skipped, starting directly with "confirmed" once payment is received.

**Content:** (Optional) Human readable status update

**Required tags:**
- `p`: Buyer's public key
- `subject`: Human-friendly subject line for status updates
- `type`: Must be "3" to indicate status update
- `order`: The original order identifier
- `status`: Current order status:
  - `pending`: Order received but awaiting payment
  - `confirmed`: Payment received and verified
  - `processing`: Order is being prepared
  - `completed`: Order fulfilled
  - `cancelled`: Order cancelled by either party

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<buyer-pubkey>"],
    ["subject", "order-info"],
    ["type", "3"],  // Status update
    ["order", "<order-id>"],
    
    // Status information
    ["status", "<order-status>"],  // pending|confirmed|processing|completed|cancelled
  ],
  "content": "Human readable status update"
}
```

Buyers may also send a status update to cancel an order, ideally before the status has been set to "confirmed":

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<merchant-pubkey>"],
    ["subject", "order-info"],
    ["type", "3"],  // Status update
    ["order", "<order-id>"],
    
    // Status information
    ["status", "<order-status>"],  // cancelled
  ],
  "content": "Human readable status update"
}
```

#### 4. Shipping Updates
Sent by merchant to provide delivery tracking and status information.

**Content:** (Optional) Human readable shipping status and tracking information

**Required tags:**
- `p`: Buyer's public key
- `subject`: Human-friendly subject line for shipping updates
- `type`: Must be "4" to indicate shipping update
- `order`: The original order identifier
- `status`: Current shipping status:
  - `processing`: Order is being prepared for shipping
  - `shipped`: Package has been handed to carrier
  - `delivered`: Successfully delivered to destination
  - `exception`: Delivery issue or delay encountered

**Optional tags:**
- `tracking`: Carrier's tracking number
- `carrier`: Name of shipping carrier
- `eta`: Expected delivery time as unix timestamp

```jsonc
{
  "kind": 16,
  "tags": [
    // Required tags
    ["p", "<buyer-pubkey>"],
    ["subject", "shipping-info"],
    ["type", "4"],  // Shipping update
    ["order", "<order-id>"],
    
    // Shipping details
    ["status", "<shipping-status>"],  // processing|shipped|delivered|exception
    ["tracking", "<tracking-number>"],
    ["carrier", "<carrier-name>"],
    ["eta", "<unix-timestamp>"],
  ],
  "content": "Shipping status and tracking information"
}
```

#### 5. General Communication
Used for any order-related messages (Kind 14)

```jsonc
{
  "kind": 14,
  "tags": [
    // Required tags
    ["p", "<recipient-pubkey>"],
    ["subject", "<order-id>"],  // Optional, can be empty
  ],
  "content": "General communication message"
}
```

#### 6. Payment Receipt
Sent by buyer to confirm payment completion. The receipt can include proof of payment from any payment system, including traditional fiat gateways.

**Content:** (Optional) Human readable payment confirmation details

**Required tags:**
- `p`: Merchant's public key
- `subject`: Human-friendly subject line for order receipt
- `order`: The original order identifier
- `payment`: Payment proof details (at least one required):
  - Generic format: `["payment", "<medium>", "<medium-reference>", "<proof>"]`
  - Common examples:
    - Lightning: `["payment", "lightning", "<invoice>", "<preimage>"]`
    - Bitcoin: `["payment", "bitcoin", "<address>", "<txid>"]`
    - eCash: `["payment", "ecash", "<mint-url>", "<proof>"]`
    - Fiat: `["payment", "fiat", "<some-id>", "<some-proof>"]`
- `amount`: Payment amount

```jsonc
{
  "kind": 17,
  "tags": [
    // Required tags
    ["p", "<merchant-pubkey>"],
    ["subject", "order-receipt"],
    ["order", "<order-id>"],
    
    // Payment proof (one required)
    ["payment", "<medium>", "<medium-reference>", "<proof>"],
    ["payment", "lightning", "<invoice>", "<preimage>"],
    ["payment", "bitcoin", "<address>", "<txid>"],
    ["payment", "ecash", "<mint-url>", "<proof>"],
    ["payment", "fiat", "<some-id>", "<some-proof>"],

    // Metadata
    ["amount", "<amount>"]
  ],
  "content": "Payment confirmation details"
}
```

#### Notes
1. Message Flow:
   - Receipts should include verifiable proofs

2. Payment Processing:
   - Manual mode provides more flexibility
   - Automatic mode enables faster processing and convenience
   - Multiple payment options can be offered

3. Status Tracking:
   - Use consistent status codes
   - Include timestamps for all updates
   - Provide clear user messages

## 5. Product Reviews (Kind: 31555)

Product reviews follow [NIP-85](https://github.com/nostr-protocol/nips/blob/b1432b705f553bde6c4eb5fcfde8525d2913b477/85.md) and [QTS](https://habla.news/u/arkinox@arkinox.tech/DLAfzJJpQDS4vj3wSleum) guidelines with additional marketplace-specific rating criteria. Reviews provide structured feedback about products, merchants, and the overall purchase experience.

**Content:** Detailed review text

**Required tags:**
- `d`: Reference to product `["d", "a:30402:<merchant-pubkey>:<product-d-tag>"]`
- `rating`: Primary rating `["rating", "<score>", "thumb"]`
  - score: 0 (negative) to 1 (positive)
  - "thumb" label MUST be present as primary rating

**Optional tags:**
- Additional Ratings:
  - `rating`: Category scores `["rating", "<score>", "<category>"]`
    - score: 0 to 1 (supports fractional values)
    - category: These are optional, some standard categories may include:
      - "value": Price vs quality
      - "quality": Product quality
      - "delivery": Shipping experience
      - "communication": Merchant responsiveness

```jsonc
{
  "kind": 31555,
  "created_at": <unix timestamp>,
  "tags": [
    // Required tags
    ["d", "a:30402:<merchant-pubkey>:<product-d-tag>"],
    ["rating", "1", "thumb"],  // Primary rating
    
    // Optional rating categories
    ["rating", "0.8", "value"],
    ["rating", "1.0", "quality"],
    ["rating", "0.6", "delivery"],
    ["rating", "0.9", "communication"]
  ],
  "content": "Detailed review text"
}
```

#### Rating Calculation
The final score combines the primary "thumb" rating (50% weight) with additional category ratings (50% combined weight):

```
Total Score = (Thumb × 0.5) + (0.5 × (∑(Category Ratings) ÷ Number of Categories))
```

#### Notes
1. Rating System:
   - Primary thumb rating is required, it determines the overall an overall rating.
   - Additional categories are optional
   - Scores support fractional values between 0-1
   - Custom categories can be added

## 6. Implementation Guidelines

### Payment Flow Details

#### Payment Preferences
Merchants can specify their payment preferences in their kind:`0` event using the `payment_preference` tag:
```
["payment_preference", "<manual | ecash | lud16>"]
```

If not present, it defaults to `manual`. The preferences are processed in this order of complexity:
1. Manual (default): Merchant provides payment requests directly
2. eCash: Ideally the merchant has a kind `10019` event to know what mint they prefer.
  - If the `10019` event is not present, payment can be made by sending the token embedded directly in the order receipt message from a previously set mint (whether default or user selected); otherwise, the merchant's preferred mint SHOULD be used.
3. Lightning: Requires `lud16` or related lightning fields in kind `0`

#### Payment Processing Scenarios

1. **Manual Processing**
   - Merchant initiates payment request
   - Used when no application is recommended or automatic preferences are set
   - Merchant must manually send payment requests
   - Buyer waits for merchant's payment instructions
   - Merchants can have their own service that listens for new orders and then sends the payment request

2. **Automatic Processing**
   - Buyer initiates payment request
   - Requires a valid `payment_preference` in merchant's kind `0`
   - Service-Based Processing processing if `payment_preference` is `manual` and the merchant have a recommended application
   - Supports automatic payments via:
     - eCash tokens (locked to merchant's pubkey)
     - Lightning (using merchant's `lud16` address)

3. **Service-Based Processing**
   - Merchant MUST set `payment_preference` to `manual`
   - Merchant SHOULD have a [NIP-89](89.md) kind `31989` event recommending their preferred service
   - Buyers can immediately request payment using the service
   - Service handles payment details and completion monitoring

For all scenarios:
- Buyer MUST send payment receipt after completion
- Message direction is determined by `p` tag
- Merchant's [NIP-89](89.md) application preferences SHOULD be respected

### Marketplace Application Role
Marketplace applications can optionally facilitate the order processing and payment request by:

1. Generating payment requests based on merchant preferences when buyers initiate orders
2. Verifying payments and generating receipts automatically by prompting the buyer to sign the event
3. Helping merchants managing inventory and order status updates
4. Coordinating shipping information
5. Price calculations

This provides a smoother user experience while maintaining the ability for direct merchant-buyer communication as a fallback mechanism.

### Notes and Considerations

1. Tags are used for all structured, machine-readable data to facilitate easier parsing and filtering.

2. The content field is reserved for human-readable messages and additional information that doesn't require machine parsing.

3. All timestamps should be in Unix format (seconds since epoch).

4. Order IDs should be used consistently across all related messages.

5. Message threading should follow [NIP-10](10.md) conventions when replies are needed.
