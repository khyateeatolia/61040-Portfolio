# Assignment 2: Functional Design

## Problem Statement

### Problem Domain: Campus Secondhand Fashion
The domain is college students exchanging fashion items within their campus communities. This includes buying, selling, and swapping clothes and accessories among students. I am connected to this domain because I have personally observed that students need affordable and convenient ways to refresh their wardrobes during seasonal changes, dorm moves, or campus events. Campus communities also provide a trusted environment where exchanges can be arranged safely and locally.

### Problem: Clothes Exchange is Messy and Risky for Students
Students who want to rehome clothing or discover bargains often face disorganized workflows. Public resale apps involve shipping, scams, and irrelevant listings. Campus Facebook groups are unstructured and hard to search, with offers scattered across comment threads. Informal messaging lacks accountability and often leads to lost conversations and wasted time. These difficulties prevent students from efficiently participating in secondhand fashion exchanges.

### Stakeholders
- Student seller: A student who posts items to sell or exchange and manages bids.  
- Student buyer: A student who browses items and places bids.  
- Platform operator: The service that manages authentication, notifications, and moderation.  
- Campus community moderator: A person or small group responsible for safety reports and handling misconduct.  
- Payment and pickup facilitator: External tools such as Venmo, PayPal, or cash that complete the transaction.  

### Evidence and Comparables
- **Poshmark**. This platform is widely used for secondhand fashion and demonstrates demand for structured listing features.  
- **Depop**. This social marketplace is popular among younger users and highlights the usefulness of feeds and tags.  
- **Mercari**. This app emphasizes classification and shipping, showing different tradeoffs compared to local pickups.  
- **Rumie College Marketplace**. This app focuses on campus-only exchanges and proves the value of school-based verification.  
- **University exchange groups**. Facebook and other networks already host informal student groups, which validates the need for a safer and more structured solution.  
- **Research on the secondhand fashion market**. Reports show that circular fashion and resale markets are rapidly expanding, which supports the problem definition.  

---

## Application Pitch

### Application Name: CampusCloset
CampusCloset is an application that provides students with a simple and trusted place to exchange clothes and accessories within their campus communities.

### Motivation
The application reduces the mess and risks of informal secondhand clothing exchanges by creating a safe, student-only environment where transactions are transparent and easy to arrange.

### Key Features
**Verified School Sign Up.** Students must register using a school email address. This requirement ensures that all users are verified members of academic communities. The feature increases trust and makes arranging pickups safer.

**Best Offer Listings and Bidding.** Each listing includes a name, description, tags such as washed, pre owned, or new with tags, and optional minimum price. Buyers place bids that are visible to sellers in a transparent history. Sellers can choose when and to whom to sell, but the app does not enforce acceptance of the highest bidder. This feature reduces confusion and provides structure that informal channels lack.

**Feed of Latest Listings.** The application maintains a feed where users can see all new listings in time order and filter them by tags such as size or category. This feature reduces search friction and encourages discovery, which is essential for making the marketplace lively and useful.

**Direct Messaging for Pickup and Payment.** Each listing has a messaging thread where buyers and sellers can communicate. They can arrange pickup times, note payment methods, and mark items as sold. This feature consolidates communication in one place and prevents confusion that often occurs when conversations are scattered.

---

## Concept Design

### UniversityAuth

**Purpose**  
Provide verified identities for users based on school email addresses.  

**Types**  
- EmailAddress  
- UserId  
- VerificationToken  

**State**  
- pendingVerifications: Map[EmailAddress → VerificationToken]  
- users: Map[UserId → {email: EmailAddress, displayName: String, avatarUrl: Url, verifiedAt: Timestamp}]  

**Actions**  
- requestVerification(email: EmailAddress) → VerificationToken  
- confirmVerification(token: VerificationToken) → UserId  
- lookupUser(id: UserId) → UserRecord  

**Notifications**  
- UserVerified(UserId)  

---

### ItemListing

**Purpose**  
Represent an item being sold or exchanged by a user.  

**Types**  
- ListingId  
- UserId  
- Tag  
- CurrencyAmount  

**State**  
- listings: Map[ListingId → {seller: UserId, title: String, description: String, photos: List[Url], tags: List[Tag], minAsk?: CurrencyAmount, createdAt: Timestamp, status: Enum{active, sold, withdrawn}, currentHighestBid?: BidId}]  

**Actions**  
- createListing(seller: UserId, title: String, description: String, photos: List[Url], tags: List[Tag], minAsk?: CurrencyAmount) → ListingId  
- updateListing(listingId: ListingId, fields: Map[String → Any])  
- setStatus(listingId: ListingId, status: Enum{active, sold, withdrawn})  
- acceptBid(listingId: ListingId, bidId: BidId)  

**Notifications**  
- ListingCreated(ListingId)  
- ListingUpdated(ListingId)  
- ListingSold(ListingId, BidId)  

---

### Bidding

**Purpose**  
Track and manage bids placed by users on listings.  

**Types**  
- BidId  
- ListingId  
- UserId  
- CurrencyAmount  

**State**  
- bidsByListing: Map[ListingId → List[BidId]]  
- bidRecords: Map[BidId → {bidder: UserId, listing: ListingId, amount: CurrencyAmount, timestamp: Timestamp}]  

**Actions**  
- placeBid(bidder: UserId, listingId: ListingId, amount: CurrencyAmount) → BidId  
- withdrawBid(bidId: BidId, bidder: UserId)  
- getBids(listingId: ListingId) → List[BidRecord]  
- getCurrentHigh(listingId: ListingId) → BidId?  

**Notifications**  
- BidPlaced(ListingId, BidId)  

---

### MessagingThread

**Purpose**  
Enable communication between buyers and sellers for each listing.  

**Types**  
- ThreadId  
- ListingId  
- UserId  
- Message  

**State**  
- threads: Map[ThreadId → {listingId: ListingId, participants: Set[UserId], messages: List[Message]}]  
- Message = {sender: UserId, text: String, attachments?: List[Url], timestamp: Timestamp}  

**Actions**  
- startThread(user: UserId, listingId: ListingId) → ThreadId  
- postMessage(threadId: ThreadId, user: UserId, text: String, attachments?: List[Url])  
- markPickupComplete(threadId: ThreadId, user: UserId)  
- flagMessage(threadId: ThreadId, messageId: Int, reason: String)  

**Notifications**  
- NewMessage(ThreadId, Message)  

---

### Feed

**Purpose**  
Maintain a browsable and filterable view of available listings for all users.  

**Types**  
- FeedView  
- ListingId  
- Tag  

**State**  
- feedIndex: List[ListingId] (ordered by createdAt)  
- tagIndex: Map[Tag → List[ListingId]]  

**Actions**  
- getLatest(n: Int) → FeedView  
- filterByTag(tag: Tag) → FeedView  
- refreshFeed()  

**Notifications**  
- FeedUpdated  

---

## Essential Synchronizations

### AuthRequired
When a user performs any action in ItemListing, Bidding, MessagingThread, or Feed, the system must verify that the user exists in UniversityAuth with a verified email.

### BidPlacementSync
- Action: Bidding.placeBid  
- Effect: ItemListing updates currentHighestBid for the relevant ListingId and notifies the seller.

### BidAcceptanceSync
- Action: ItemListing.acceptBid  
- Effect: Bidding marks the winning BidId as accepted. MessagingThread starts or updates a thread between seller and winning buyer.

### FeedRefreshSync
- Action: ItemListing.createListing or ItemListing.updateListing  
- Effect: Feed.refreshFeed updates feedIndex and tagIndex so the new listing is visible to users.

### MessageFlagSync
- Action: MessagingThread.flagMessage  
- Effect: Notification sent to Platform Operator for moderation.

---

## Integrative Note
The five concepts together form the foundation for CampusCloset. UniversityAuth ensures that only verified students can act in the system. ItemListing manages the representation and lifecycle of items. Bidding independently tracks offers while maintaining transparent history. MessagingThread supports direct communication tied to items. Feed provides an accessible, ordered, and filterable view of what is available, which makes the marketplace lively and usable. Synchronizations connect these concepts in essential places without breaking their independence. This design follows the rubric by ensuring clear purposes, persistent state, independence of concepts, and explicit synchronizations.

---


