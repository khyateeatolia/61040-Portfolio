# Assignment 2: Functional Design

## Problem Statement

### Problem Domain: Campus Secondhand Fashion
The domain is college students exchanging fashion items within their campus communities. This includes buying, selling, and swapping clothes and accessories among students. I am connected to this domain because I have personally observed that students need affordable and convenient ways to refresh their wardrobes during seasonal changes, dorm moves, or campus events. I have also personally purchased items from Facebook marketplaces and Facebook groups exclusively for college students. I have come to realize that campus communities provide a trusted environment where exchanges can be arranged safely and locally.

### Problem: Clothes Exchange is Messy and Risky for Students
Students who want to rehome clothing or discover bargains often face disorganized workflows and systems. Public resale apps involve shipping, scams, and irrelevant listings. Campus Facebook groups are unstructured and hard to search, with offers scattered across comment threads and no real filtering available. Informal messaging lacks accountability and often leads to lost conversations, missed pickups and wasted time. These difficulties prevent students from efficiently participating in secondhand fashion exchanges.

### Stakeholders
- Student seller: A student who posts items to sell or exchange and manages bids.  
- Student buyer: A student who browses items and places bids to buy. 
- Campus community moderator: A person or small group of people responsible for safety reports and handling misconduct reports.
- College sustainability office: Their end-of-year bins are less likely to be overflowing/messy if students can swap clothes orderly. 
- Payment and pickup facilitator: External tools such as Venmo, PayPal, or cash that complete the transaction.  

### Evidence and Comparables

- **Research on the secondhand fashion market** ([ThredUp 2024 Resale Report](https://www.thredup.com/resale)). Industry reports consistently show that circular fashion and resale markets are rapidly expanding. This growth provides evidence that student users would benefit from having a trusted platform focused specifically on their context.  
- **Poshmark** ([poshmark.com](https://poshmark.com)). Poshmark is a widely used platform for secondhand fashion, which demonstrates that there is sustained demand for structured listing features in online resale marketplaces.  
- **Depop** ([depop.com](https://www.depop.com)). Depop is a social resale marketplace that is especially popular among younger users. Its design highlights the importance of having a feed-based interface and descriptive tags for clothing items.  
- **Mercari** ([mercari.com](https://www.mercari.com)). Mercari focuses heavily on classification and shipping, which illustrates different tradeoffs compared to local pickup exchanges. This shows how design choices around logistics impact user experience.  
- **Rumie College Marketplace** ([rumie.com](https://rumie.com/college-marketplace)). Rumie provides a campus-focused exchange platform that requires school-based verification. This supports the idea that restricting access to students increases trust and safety.  
- **University Exchange Groups** ([example Facebook group](https://www.facebook.com/groups/217708995001165/)). Many universities have informal exchange groups hosted on platforms like Facebook. These groups validate the need for a safer, more structured, and campus-specific solution rather than relying on loosely moderated communities. 



---

## Application Pitch

### Application Name: 
CampusCloset is an application that provides students with a simple and trusted place to exchange clothes, shoes, and accessories within their campus communities.

### Motivation
The application reduces the mess and risks of informal secondhand clothing exchanges by creating a safe, student-only environment where transactions are transparent and easy to arrange.

### Key Features
**Verified School Sign Up.** Students must register using a school email address. This requirement ensures that all users are verified members of academic communities. The feature increases trust and makes arranging pickups safer.

**Feed of Latest Listings.** The application maintains a feed where users can see all new listings in time order and filter them by tags such as price or category. This feature reduces search friction and encourages discovery, which is essential for making the marketplace lively and useful.

**Best Offer Listings and Bidding.** Each listing includes a name, description, tags such as washed, pre-owned, or new with tags, and optional minimum price. Buyers place bids that are visible to sellers and other viewers in a transparent history. Sellers can choose when and to whom to sell, but the app does not enforce acceptance of the highest bidder. This feature reduces confusion and provides structure that informal channels lack.

**Direct Messaging for Pickup and Payment.** Each listing has a private messaging thread where buyers and sellers can communicate. They can arrange pickup times, share addresses/contact details, note payment methods, and mark items as sold. This feature consolidates communication in one place and prevents confusion that often occurs when conversations are scattered.

---

Perfect, thanks for clarifying. I’ve rewritten your **Concept Design** and **Essential Synchronizations** so that everything is in **proper concept syntax using backticks**. I also added filtering by **price and category** to the `Feed` concept. Here’s the cleaned version in GitHub Markdown:

```markdown
## Concept Design

### UniversityAuth

**Purpose**  
Provide verified identities for users based on school email addresses.  

```

Types
EmailAddress
UserId
VerificationToken

State
pendingVerifications: Map[EmailAddress → VerificationToken]
users: Map[UserId → {email: EmailAddress, displayName: String, avatarUrl: Url, verifiedAt: Timestamp}]

Actions
requestVerification(email: EmailAddress) → VerificationToken
confirmVerification(token: VerificationToken) → UserId
lookupUser(id: UserId) → UserRecord

Notifications
UserVerified(UserId)

```

---

### ItemListing

**Purpose**  
Represent an item being sold or exchanged by a user.  

```

Types
ListingId
UserId
Tag
CurrencyAmount

State
listings: Map[ListingId → {
seller: UserId,
title: String,
description: String,
photos: List[Url],
tags: List[Tag],
minAsk?: CurrencyAmount,
createdAt: Timestamp,
status: Enum{active, sold, withdrawn},
currentHighestBid?: BidId
}]

Actions
createListing(seller: UserId, title: String, description: String, photos: List[Url], tags: List[Tag], minAsk?: CurrencyAmount) → ListingId
updateListing(listingId: ListingId, fields: Map[String → Any])
setStatus(listingId: ListingId, status: Enum{active, sold, withdrawn})
acceptBid(listingId: ListingId, bidId: BidId)

Notifications
ListingCreated(ListingId)
ListingUpdated(ListingId)
ListingSold(ListingId, BidId)

```

---

### Bidding

**Purpose**  
Track and manage bids placed by users on listings.  

```

Types
BidId
ListingId
UserId
CurrencyAmount

State
bidsByListing: Map[ListingId → List[BidId]]
bidRecords: Map[BidId → {
bidder: UserId,
listing: ListingId,
amount: CurrencyAmount,
timestamp: Timestamp
}]

Actions
placeBid(bidder: UserId, listingId: ListingId, amount: CurrencyAmount) → BidId
withdrawBid(bidId: BidId, bidder: UserId)
getBids(listingId: ListingId) → List[BidRecord]
getCurrentHigh(listingId: ListingId) → BidId?

Notifications
BidPlaced(ListingId, BidId)

```

---

### MessagingThread

**Purpose**  
Enable communication between buyers and sellers for each listing.  

```

Types
ThreadId
ListingId
UserId
Message

State
threads: Map[ThreadId → {
listingId: ListingId,
participants: Set[UserId],
messages: List[Message]
}]
Message = {
sender: UserId,
text: String,
attachments?: List[Url],
timestamp: Timestamp
}

Actions
startThread(user: UserId, listingId: ListingId) → ThreadId
postMessage(threadId: ThreadId, user: UserId, text: String, attachments?: List[Url])
markPickupComplete(threadId: ThreadId, user: UserId)
flagMessage(threadId: ThreadId, messageId: Int, reason: String)

Notifications
NewMessage(ThreadId, Message)

```

---

### Feed

**Purpose**  
Maintain a browsable and filterable view of available listings for all users.  

```

Types
FeedView
ListingId
Tag
CurrencyAmount
Category

State
feedIndex: List[ListingId] (ordered by createdAt)
tagIndex: Map[Tag → List[ListingId]]
categoryIndex: Map[Category → List[ListingId]]
priceIndex: Map[ListingId → CurrencyAmount]

Actions
getLatest(n: Int) → FeedView
filterByTag(tag: Tag) → FeedView
filterByCategory(category: Category) → FeedView
filterByPriceRange(min: CurrencyAmount, max: CurrencyAmount) → FeedView
refreshFeed()

Notifications
FeedUpdated

```

---

## Essential Synchronizations

### AuthRequired
```

When a user performs any action in ItemListing, Bidding, MessagingThread, or Feed,
the system must verify that the user exists in UniversityAuth with a verified email.

```

### BidPlacementSync
```

Action: Bidding.placeBid
Effect: ItemListing updates currentHighestBid for the relevant ListingId
and notifies the seller.

```

### BidAcceptanceSync
```

Action: ItemListing.acceptBid
Effect: Bidding marks the winning BidId as accepted.
MessagingThread starts or updates a thread between seller and winning buyer.

```

### FeedRefreshSync
```

Action: ItemListing.createListing or ItemListing.updateListing
Effect: Feed.refreshFeed updates feedIndex, tagIndex, categoryIndex, and priceIndex
so the new listing is visible and filterable to users.

```

### MessageFlagSync
```

Action: MessagingThread.flagMessage
Effect: Notification sent to Platform Operator for moderation.

```
```

The five concepts together form the foundation for CampusCloset. UniversityAuth ensures that only verified students can act in the system. ItemListing manages the representation and lifecycle of items. Bidding independently tracks offers while maintaining transparent history. MessagingThread supports direct communication tied to items. Feed provides an accessible, ordered, and filterable view of what is available, which makes the marketplace lively and usable. Synchronizations connect these concepts in essential places without breaking their independence. This design follows the rubric by ensuring clear purposes, persistent state, independence of concepts, and explicit synchronizations.

---


