# Assignment 4: Functional Design

## Problem Statement

### Problem Domain: Campus Secondhand Fashion

The domain is college students exchanging fashion items within their campus communities. This includes buying, selling, and swapping clothes and accessories among students. I am connected to this domain because I have personally observed that students need affordable and convenient ways to refresh their wardrobes during seasonal changes, dorm moves, or campus events. I have also personally purchased items from Facebook marketplaces and Facebook groups exclusively for college students. I have come to realize that campus communities provide a trusted environment where exchanges can be arranged safely and locally.

### Problem: Clothes Exchange is Messy and Risky for Students

Students who want to rehome clothing or discover bargains often face disorganized workflows and systems. Public resale apps involve shipping, scams, and irrelevant listings. Campus Facebook groups are unstructured and hard to search, with offers scattered across comment threads and no real filtering available. Informal messaging lacks accountability and often leads to lost conversations, missed pickups and wasted time. These difficulties prevent students from efficiently participating in secondhand fashion exchanges.

### Stakeholders

* Student seller: A student who posts items to sell or exchange and manages bids.
* Student buyer: A student who browses items and places bids to buy.
* Campus community moderator: A person or small group of people responsible for safety reports and handling misconduct reports.
* College sustainability office: Their end-of-year bins are less likely to be overflowing or messy if students can swap clothes in an orderly way.
* Payment and pickup facilitator: External tools such as Venmo, PayPal, or cash that complete the transaction.

### Evidence and Comparables

* **Research on the secondhand fashion market** ([ThredUp 2024 Resale Report](https://www.thredup.com/resale)). Industry reports consistently show that circular fashion and resale markets are rapidly expanding. This growth provides evidence that student users would benefit from having a trusted platform focused specifically on their context.
* **Poshmark** ([poshmark.com](https://poshmark.com)). Poshmark is a widely used platform for secondhand fashion, which demonstrates that there is sustained demand for structured listing features in online resale marketplaces.
* **Depop** ([depop.com](https://www.depop.com)). Depop is a social resale marketplace that is especially popular among younger users. Its design highlights the importance of having a feed-based interface and descriptive tags for clothing items.
* **Mercari** ([mercari.com](https://www.mercari.com)). Mercari focuses heavily on classification and shipping, which illustrates different tradeoffs compared to local pickup exchanges. This shows how design choices around logistics impact user experience.
* **Rumie College Marketplace** ([rumie.com](https://rumie.com/college-marketplace)). Rumie provides a campus-focused exchange platform that requires school-based verification. This supports the idea that restricting access to students increases trust and safety.
* **University Exchange Groups** ([Wellesley Facebook group](https://www.facebook.com/groups/217708995001165/)). Many universities have informal exchange groups hosted on platforms like Facebook. These groups validate the need for a safer, more structured, and campus-specific solution rather than relying on loosely moderated communities.

---

## Application Pitch

### Application Name:

CampusCloset is an application that provides students with a simple and trusted place to exchange clothes, shoes, and accessories within their campus communities.

### Motivation

The application reduces the mess and risks of informal secondhand clothing exchanges by creating a safe, student-only environment where transactions are transparent and easy to arrange.

### Key Features

**Verified School Sign Up.** Students must register using a school email address. This requirement ensures that all users are verified members of academic communities. The feature increases trust and makes arranging pickups safer.

**Feed of Latest Listings.** The application maintains a feed where users can see all new listings in time order and filter them by tags such as price, category, or recency. This feature reduces search friction and encourages discovery, which is essential for making the marketplace lively and useful.

**Best Offer Listings and Bidding.** Each listing includes a name, description, tags such as washed, pre-owned, or new with tags, and optional minimum price. Buyers place bids that are visible to sellers and other viewers in a transparent history log. Sellers can choose when and to whom to sell, but the app does not enforce acceptance of the highest bidder. This feature reduces confusion and provides structure that informal channels lack.

**Direct Messaging for Pickup and Payment.** Each listing has a private messaging thread where buyers and sellers can communicate. They can arrange pickup times, share addresses and contact details, note payment methods, and mark items as sold. This feature consolidates communication in one place and prevents confusion that often occurs when conversations are scattered.

**Profile Pages.** Each user has a profile page tied to their verified account. Profiles display usernames, avatars, bios, and all listings created by that user in time order. If a user has no listings, their profile page explicitly states this. Profile pages make it easier to view a seller’s history and reputation.

---

# 3 — Concept design (4 focused concepts)

I follow the Essence of Software guidance: each concept specifies *purpose*, *types*, *state*, *actions*, and *notifications/side-effects*.

---

## Concept A — `UserAccount`

**Purpose:**
Authenticate students via school email, assign them a unique username and `UserId`, and maintain their profile page (including their listings).

**Types:**

* `EmailAddress`
* `UserId`
* `Username`
* `VerificationToken`

**State:**

* `pending_verifications: Map<EmailAddress, VerificationToken>` — tokens awaiting confirmation
* `users: Map<UserId -> {
    email: EmailAddress,
    username: Username,
    displayName: String,
    avatarUrl: Url,
    verifiedAt: Timestamp
  }]`
* `profiles: Map<UserId -> {
    bio?: String,
    listings: List<ListingId>
  }]`

**Actions:**

* `request_verification(email: EmailAddress) -> VerificationToken`
* `confirm_verification(token: VerificationToken, username: Username) -> UserId`
* `lookup_user(id: UserId) -> UserRecord`
* `view_profile(userId: UserId) -> ProfileView`

**Notifications / Side-effects:**

* `UserVerified(UserId)` emitted when confirmation succeeds
* Email with verification link sent on `request_verification`

**Notes:**

* Each `UserId` is bound to a unique `username`.
* Profile pages list the user’s items in chronological order.
* If a user has no listings, the profile displays “This user has no active listings.”
* Profiles are visible to logged-in users, supporting trust and reputation.

---

## Concept B — `ItemListing`

**Purpose:**
Represent an item for sale or exchange and track its lifecycle (active, withdrawn, sold).

**Types:**

* `ListingId`
* `UserId` (seller)
* `Tag`
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
    currentHighestBid?: BidId,
    bidLog: List<{bidder: UserId, amount: CurrencyAmount, timestamp: Timestamp}>
  }]`

**Actions:**

* `create_listing(seller, title, description, photos, tags, minAsk?) -> ListingId`
* `update_listing(listingId, fields)`
* `set_status(listingId, status)`
* `accept_bid(listingId, bidId)`

**Notifications:**

* `ListingCreated(ListingId)`
* `ListingUpdated(ListingId)`
* `ListingSold(ListingId, BidId)`

**Notes:**

* Each listing shows the seller’s unique `username`.
* Each listing includes both the `currentHighestBid` and a full `bidLog`.
* Listings automatically attach to the seller’s profile in `UserAccount`.

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

* `bids_by_listing: Map<ListingId -> List<BidId>>`
* `bid_records: Map<BidId -> {bidder, listing, amount, timestamp}>`

**Actions:**

* `place_bid(bidder, listingId, amount) -> BidId`
* `withdraw_bid(bidId, bidder)`
* `get_bids(listingId) -> [BidRecord]`
* `get_current_high(listingId) -> BidId?`

**Notifications:**

* `BidPlaced(ListingId, BidId)`

**Notes:**

* Bids remain stored in full history.
* Winning bids are referenced back in `ItemListing`.

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

* `start_thread(user, listingId) -> ThreadId`
* `post_message(threadId, user, text, attachments?)`
* `mark_pickup_complete(threadId, user)`
* `flag_message(threadId, messageId, reason)`

**Notifications:**

* `NewMessage(ThreadId, Message)`

**Notes:**

* Threads always reference the associated listing.
* Pickup completion signals transaction closure.
* Moderators can review flagged content.

---

## Concept E — `Feed`

**Purpose:**
Maintain a browsable and filterable view of available listings for all users.

**Types:**

* `FeedView`
* `ListingId`
* `Tag`

**State:**

* `feedIndex: List[ListingId]` (ordered by createdAt)
* `tagIndex: Map[Tag -> List[ListingId]]`
* `priceIndex: Map[PriceRange -> List[ListingId]]`

**Actions:**

* `get_latest(n: Int) -> FeedView`
* `filter_by_tag(tag: Tag) -> FeedView`
* `filter_by_price(min: CurrencyAmount, max: CurrencyAmount) -> FeedView`
* `refresh_feed()`

**Notifications:**

* `FeedUpdated`

**Notes:**

* Users can filter the feed by time, tag, or price.
* Feed entries show seller usernames and current bid highlights.

---

# 4 — Essential synchronizations

---

### Sync 1 — AuthRequired for Listings and Bids

* **When:** A user attempts `create_listing` or `place_bid`.
* **Where:** `UserAccount` + `ItemListing` + `Bidding`.
* **Then:** Verify `UserId` exists and has `verifiedAt != null`. If not verified, reject action.
* **Rationale:** Ensures only verified student accounts can create listings or bid.

---

### Sync 2 — Place Bid → Update Listing and Profile

* **When:** `Bidding.place_bid` completes.
* **Where:** `Bidding` + `ItemListing` + `UserAccount`.
* **Then:**

  * Append bid to `bid_records` and `bidLog`.
  * If higher than current, update `ItemListing.currentHighestBid`.
  * Emit `BidPlaced`.
  * Show updated bids in listing and profile views.
* **Rationale:** Keeps listings’ active bids transparent and ties bid visibility to seller profiles.

---

### Sync 3 — Accept Bid → Close Listing and Notify Buyer

* **When:** `ItemListing.accept_bid` is called.
* **Where:** `ItemListing` + `Bidding` + `MessagingThread` + `UserAccount`.
* **Then:**

  * Set listing status to Sold.
  * Emit `ListingSold`.
  * Notify buyer via `MessagingThread`.
  * Update seller’s profile page so the item moves from active to sold history.
* **Rationale:** Ensures closure across all connected concepts and visible state in profile.

---

### Sync 4 — Feed Refresh on Listing Events

* **When:** `ItemListing.create_listing` or `update_listing`.
* **Where:** `ItemListing` + `Feed` + `UserAccount`.
* **Then:**

  * Add or update listing in feed indexes.
  * Add listing to seller’s profile.
  * Emit `FeedUpdated`.
* **Rationale:** Keeps feed and profile pages synchronized with listing lifecycle.

---

### Sync 5 — Flag Message → Moderator Workflow

* **When:** `MessagingThread.flag_message` is called.
* **Where:** `MessagingThread` + Moderator.
* **Then:**

  * Create moderation case.
  * Notify moderator for review.
* **Rationale:** Supports safety and accountability in user interactions.

---

The five concepts together form the foundation for CampusCloset. `UserAccount` ensures that only verified students with unique usernames can act in the system and provides public profile pages. `ItemListing` manages the representation and lifecycle of items, while `Bidding` tracks transparent bid histories. `MessagingThread` supports structured communication, and `Feed` provides searchable and filterable access to items. Synchronizations ensure that actions flow smoothly between concepts without breaking modularity.

---

### UX designs
![Image 1](https://github.com/user-attachments/assets/4858e6ea-68fd-44b6-b637-f07214d26ec6)

![Image 2](https://github.com/user-attachments/assets/ae70e8fc-2a77-48cd-bf66-3ab64d92b89c)

![Image 3](https://github.com/user-attachments/assets/1ebcf4eb-8776-4785-b7dd-f37b952deb74)


---

## User Journey

Meet Maya, a sophomore living in a college dorm. As the semester begins, she realizes she no longer needs the extra desk chair she bought last year. At the same time, she is looking for an affordable winter coat. Maya has heard about the campus marketplace app and decides to give it a try.

**Step 1: Signing Up**  
Maya downloads the app and signs up with her school email address. She confirms her account through the verification link she receives. She chooses a username that will appear on her profile and listings. Now she has a verified profile (see Sketch 3: User Profile Page).

**Step 2: Listing an Item**  
Maya goes to her profile and selects the option to create a new listing. She uploads a photo of her old cocktail, writes a short description, sets the price to twenty-five dollars, and tags it under “Clothes.” Once submitted, the dress appears on her profile page and also shows up in the public feed for others to see (see Sketch 1: Feed).

**Step 3: Browsing the Feed**  
While checking the feed, Maya uses the filters to narrow results to “Clothing” under twenty dollars. She immediately spots a post for a winter coat from another student, listed at fifteen dollars (see Sketch 1: Feed with Filters). She clicks the listing to view more details and decides she is interested. She places her bid at twenty dollars. 

**Step 4: Starting a Conversation**  
The seller decides to give the coat to Maya, and Maya opens a messaging thread with the seller where she receives the details of the listing.  They agree to meet the next afternoon at the student center. 

**Step 5: Completing the Exchange**  
The next day, Maya meets the seller, tries on the coat, and finalizes the purchase. Back in the app, the seller marks the pickup as complete in the chat thread. The coat listing automatically updates to “Sold,” and it no longer appears in the feed.

**Outcome**  
In a single experience, Maya has sold her unused dress and purchased a coat, all within her campus community. She feels reassured by the verified profiles and enjoys the simplicity of browsing listings with filters. Her profile page now reflects her sold and active listings, making it easy for other students to trust her as a reliable community member.
