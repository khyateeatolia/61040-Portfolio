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

# 3 — Concept design (4 focused concepts)

I follow the Essence of Software guidance: each concept specifies *purpose*, *types*, *state*, *actions*, and *notifications/side-effects*. ([Essence criteria][9])

---

## Concept A — `UniversityAuth`

**Purpose:**  
Verify that a user is part of an academic community and provide an authenticated `UserId` that other concepts can depend on.

**Types:**  
* `EmailAddress`  
* `UserId`  
* `VerificationToken`  

**State:**  
* `pending_verifications: Map<EmailAddress, VerificationToken>` — pending tokens awaiting confirmation  
* `users: Map<UserId -> {email, displayName, avatarUrl, verifiedAt: Timestamp}]` — records of all verified users  

**Actions:**  
* `request_verification(email: EmailAddress) -> VerificationToken` — creates token, sends to email  
* `confirm_verification(token: VerificationToken) -> UserId` — finalizes verification, issues `UserId`  
* `lookup_user(id: UserId) -> UserRecord` — retrieve verified profile  

**Notifications / Side-effects:**  
* `UserVerified(UserId)` emitted when confirmation succeeds  
* Email with verification link sent on `request_verification`  

**Notes:**  
Allows any valid `.edu` or partner domain. To discourage spam, disposable addresses are rejected. Manual review may be flagged if suspicious activity is detected.

---

## Concept B — `ItemListing`

**Purpose:**  
Represent an item for sale or exchange and track its lifecycle (active, withdrawn, sold).

**Types:**  
* `ListingId`  
* `UserId` (seller)  
* `Tag` (controlled vocabulary for size, brand, condition)  
* `CurrencyAmount`  

**State:**  
* `listings: Map<ListingId -> {
    seller: UserId,
    title: String,
    description: String,
    photos: List<Url>,
    tags: List<Tag>,
    minAsk?: CurrencyAmount,
    createdAt: Timestamp,
    status: Enum{Active, Sold, Withdrawn},
    currentHighestBid?: BidId
}]`

**Actions:**  
* `create_listing(seller, title, description, photos, tags, minAsk?) -> ListingId`  
* `update_listing(listingId, fields)` (title, description, tags, minAsk, photos)  
* `set_status(listingId, status)`  
* `accept_bid(listingId, bidId)` (records winning bid and sets status Sold)  

**Notifications:**  
* `ListingCreated(ListingId)`  
* `ListingUpdated(ListingId)`  
* `ListingSold(ListingId, BidId)`  

**Notes:**  
Each listing must include at least one photo. Tags are standardized to support filtering in the feed. The listing keeps a pointer to the `currentHighestBid` for efficient display.

---

## Concept C — `Bidding`

**Purpose:**  
Allow buyers to place bids on active listings, track bidding history, and expose current top bids.

**Types:**  
* `BidId`  
* `ListingId`  
* `UserId`  
* `CurrencyAmount`  

**State:**  
* `bids_by_listing: Map<ListingId -> List<BidId>>` (time-ordered)  
* `bid_records: Map<BidId -> {bidder, listing, amount, timestamp}>`  

**Actions:**  
* `place_bid(bidder, listingId, amount) -> BidId`  
* `withdraw_bid(bidId, bidder)`  
* `get_bids(listingId) -> [BidRecord]`  
* `get_current_high(listingId) -> BidId?`  

**Notifications:**  
* `BidPlaced(ListingId, BidId)`  

**Notes:**  
Top bid is updated in `ItemListing` through synchronization (not direct mutation). Bids remain as a full historical record, even after listing closes.

---

## Concept D — `MessagingThread`

**Purpose:**  
Support structured communication between buyers and sellers around a listing, including pickup arrangements and moderation.

**Types:**  
* `ThreadId`  
* `ListingId`  
* `UserId`  
* `Message`  

**State:**  
* `threads: Map<ThreadId -> {listingId, participants: Set<UserId>, messages: List<Message>}]`  
* `Message = {sender: UserId, text: String, attachments?: List<Url>, timestamp: Timestamp}`  

**Actions:**  
* `start_thread(user, listingId) -> ThreadId` (auto-includes seller and buyer)  
* `post_message(threadId, user, text, attachments?)`  
* `mark_pickup_complete(threadId, user)`  
* `flag_message(threadId, messageId, reason)`  

**Notifications:**  
* `NewMessage(ThreadId, Message)`  

**Notes:**  
Messages are anchored to listings for traceability. Moderators can review flagged content in context. Pickup completion is lightweight, signaling closure to both buyer and seller.

---

# 4 — Essential synchronizations

Following the recommended `when/where/then` form. Each synchronization links concepts without breaking modularity. ([Essence sync guidance][9])

---

### Sync 1 — AuthRequired for Listings and Bids

* **When:** A user attempts `create_listing` or `place_bid`.  
* **Where:** `UniversityAuth` + `ItemListing` + `Bidding`.  
* **Then:** Verify `UserId` exists and has `verifiedAt != null`. If not verified, reject action and emit `AuthRequired`.  
* **Rationale:** Prevents unverified or fake accounts from posting or bidding, ensuring trust in the campus marketplace.  

---

### Sync 2 — Place Bid → Update Listing and Notify Seller

* **When:** `Bidding.place_bid(bidder, listingId, amount)` completes.  
* **Where:** `Bidding` + `ItemListing` + `MessagingThread`.  
* **Then:**  
  * Append `BidId` to `bids_by_listing[listingId]`.  
  * If `amount` > `currentHighest.amount`, update `ItemListing.currentHighestBid`.  
  * Emit `BidPlaced(ListingId, BidId)` and notify seller.  
  * If buyer has not yet messaged seller, suggest starting a thread.  
* **Rationale:** Keeps bid history in `Bidding`, shows top bid in `ItemListing`, and alerts seller. Ensures smooth flow into negotiation.  

---

### Sync 3 — Accept Bid → Close Listing and Notify Buyer

* **When:** `ItemListing.accept_bid(listingId, bidId)` invoked.  
* **Where:** `ItemListing` + `Bidding` + `MessagingThread`.  
* **Then:**  
  * Set `ItemListing.status = Sold`.  
  * Record `winner = bid.bidder`.  
  * Emit `ListingSold(ListingId, BidId)`.  
  * Notify winning buyer and append a system message to the listing’s thread (“Bid accepted — arrange pickup”).  
  * Lock further bidding on that listing.  
* **Rationale:** Ensures consistent state transition, notifies buyer promptly, and funnels both parties into a messaging flow.  

---

### Sync 4 — Feed Refresh on Listing Events

* **When:** `ItemListing.create_listing` or `ItemListing.update_listing`.  
* **Where:** `ItemListing` + `Feed`.  
* **Then:**  
  * Refresh `feedIndex`, `tagIndex`, `categoryIndex`, and `priceIndex`.  
  * Emit `FeedUpdated` so users see the new or changed listing.  
* **Rationale:** Keeps feed views synchronized with the most recent listing data. Supports filtering by tag, category, and price.  

---

### Sync 5 — Flag Message → Moderator Workflow

* **When:** `MessagingThread.flag_message` is called.  
* **Where:** `MessagingThread` + `Platform operator`.  
* **Then:**  
  * Create a moderation case with the flagged message and thread context.  
  * Notify operator; optionally suspend sender until review.  
* **Rationale:** Provides safety oversight and ensures inappropriate behavior is handled quickly.  

The five concepts together form the foundation for CampusCloset. UniversityAuth ensures that only verified students can act in the system. ItemListing manages the representation and lifecycle of items. Bidding independently tracks offers while maintaining transparent history. MessagingThread supports direct communication tied to items. Feed provides an accessible, ordered, and filterable view of what is available, which makes the marketplace lively and usable. Synchronizations connect these concepts in essential places without breaking their independence. This design follows the rubric by ensuring clear purposes, persistent state, independence of concepts, and explicit synchronizations.

---


