---
date: 2026-02-14
title: "The Code Citadel - The Merchant's Ledger"
layout: "page"
showDate: true
showEdit: true
showReadingTime: true
showTableOfContents: true
---

*In the [previous chronicle]({{< relref "/posts/the-code-citadel/2-roads-of-mud-and-stone/" >}}), we learned that the roads between our districts are treacherous - plagued by latency, packet loss, and the eternal choice between the blocking Courier and the asynchronous Pigeon. The Royal Travel Service learned to handle 5,000 bookings per day using circuit breakers and async messaging. But even with reliable communication, a deeper crisis awaits: the fragmentation of Truth itself. Lady Catherine's booking is about to fail in a new and terrifying way.*

---

## Prologue: The Ink of Authority

{{< figure src="The_Great_Ledger.jpg" alt="The High Treasurer and the singular Great Ledger" >}}

In the high tower of the Monolithic Keep, there sat a man of terrifying power. He was not the King, nor the General. He was the **High Treasurer**.

Before him lay **The Great Ledger** (The Relational Database).

In the era of the Monolith, "Truth" was a physical location. It was a single book, bound in iron, chained to a desk in a locked room. If the King wished to move 100 gold coins from the Tax Collector's vault to the Army's payroll, the Treasurer performed a sacred ritual:

1. He dipped his quill.
2. He crossed out "100" in the Tax column.
3. He wrote "100" in the Army column.

This action was **Atomic**. If the Treasurer suffered a heart attack after crossing out the Tax but before crediting the Army, the transaction did not "half-happen." The ink would be scraped away. The page would be burned. The gold never moved. This was the comfort of **ACID** (Atomicity, Consistency, Isolation, Durability). The Castle slept soundly because the Truth was singular, locked, and guarded by a single transaction manager.

But when we tore down the Castle to build the City (Microservices), we committed a heresy. We tore the Great Ledger apart.

---

## The Fragmented Truth

{{< figure src="The_Fragmented_Ledgers.jpg" alt="Each guild maintains its own separate ledger" >}}

In the Distributed City, there is no High Treasurer. There is no single book. Instead, every Guild has its own notebook.

The Royal Travel Service, now split across the City, maintains separate ledgers:

- **Payment Service** tracks traveler accounts and transactions
- **Booking Service** tracks journey reservations  
- **Inventory Service** tracks carriage and room availability
- **Notification Service** tracks sent confirmations

This is **Database-per-Service**. It is essential for autonomy - the Payment Guild can optimize their ledger for financial transactions without asking the Booking Guild - but it destroys the "Royal Truth" that Master Clerk Edmund maintained in his single Great Ledger.

This gave each team autonomy. But it created a terrifying problem: **There was no longer a single source of truth.**

### The Tale of Lady Catherine's Lost Booking

{{< figure src="The_Broken_Wagon_Transaction.jpg" alt="The merchant caught in an inconsistent state" >}}

Consider a simple business transaction: **Lady Catherine wants to book a journey to Rome.** The journey costs 200 gold. It requires a carriage seat and rooms at The Golden Lion in Florence and The Silver Stag in Lyon. In the Keep, Master Clerk Edmund handled this as one transaction in his Great Ledger. In the City, it is a distributed nightmare spanning three districts.

1. Lady Catherine gives 200 gold to the **Payment Service**. (Payment Service writes: *"-200 Gold"*).
2. The Payment Service sends a runner to the **Booking Service**.
3. The Booking Service checks availability. *All carriages to Rome are full.*

In the Keep, Edmund would have simply erased the gold deduction. But in the City, the Payment Service has already written the entry. It has already closed its book. The transaction has "committed" locally. We now have a system in an **Inconsistent State**: Lady Catherine has lost her gold, but she has no journey. To fix this, we cannot "rollback" time. We must move forward. We must generate a new transaction to fix the old one. We have entered the realm of **Distributed Transactions**.

**The Nightmare Scenario in Detail:**

**Step 1:** Payment Service receives the request
- Checks Lady Catherine's account: 200 gold available ✓
- Deducts 200 gold from her account
- Records transaction in Payment ledger
- **Commits the transaction**
- Sends message to Booking Service: "Payment confirmed, proceed with booking"

**Step 2:** Booking Service receives the message
- Attempts to reserve carriage seat
- All carriages to Rome are fully booked for the next month
- **Booking fails**

Now we have a catastrophic inconsistency:
- Payment Service ledger shows: Lady Catherine paid 200 gold ✓
- Booking Service ledger shows: No booking exists for Lady Catherine
- Lady Catherine's account: 200 gold missing
- Lady Catherine's journey: Doesn't exist

In the Keep, this could never happen. Edmund would have checked carriage availability *before* taking the payment. If no carriages were available, the transaction would never start.

But in the City, the Payment Service and Booking Service have separate ledgers. The Payment Service committed its transaction without knowing the Booking Service would fail.

**The Angry Noblewoman:** Lady Catherine arrives at the booking office, furious. *"You took my gold but gave me no journey! This is theft!"*

The clerk checks the Payment Service: *"Yes, you paid 200 gold."*
They check the Booking Service: *"But we have no record of your booking."*

The clerk is confused. Both statements are true. The system is in an inconsistent state. They have to manually refund Lady Catherine and apologize profusely. She takes her business to "Swift Journeys Ltd" and tells all her noble friends about the "incompetent" service.

This happens three more times that week. The reputation suffers. The engineers are in crisis mode, trying to figure out how to prevent this.

---

## The Two Generals Problem

{{< figure src="The_Two_Generals.jpg" alt="Two generals unable to achieve perfect coordination" >}}

Why can we not simply coordinate the Guilds? Why can't the King say, *"Nobody write in your books until **everyone** is ready?"*

This brings us to the **Two Generals Problem**, the only unsolvable problem in computer science.

Imagine two armies, led by **General A** (Service A) and **General B** (Service B), camped on opposite hills of a hostile city. They **must** attack simultaneously to win. If one attacks alone, he will be slaughtered. They communicate only via messengers (**The Network**) running through the enemy valley.

- **General A:** "Attack at dawn?"
- **General B:** "Agreed. Dawn."

General B sends the messenger back. But B worries: *What if the messenger is shot? If A doesn't get my 'Yes', he won't attack. I will attack alone and die.* So B needs a confirmation from A.

- **General A:** "I received your 'Yes'. Attack at dawn."

Now A worries: *What if B didn't hear my confirmation? He won't attack. I will die.* This infinite regress of "acknowledging the acknowledgment" proves that over an unreliable network, **Perfect Consensus is Impossible**. You can never be 100% sure that the other service is in the same state as you.

**The Coordination Nightmare:** The engineers tried to solve Lady Catherine's problem by adding coordination. Before the Payment Service commits, it should check with the Booking Service: "Can you fulfill this booking?"

But the network between services is unreliable. The Booking Service might respond *"Yes, carriage available"*, but the response gets lost in the network. The Payment Service times out, assumes *"No"*, and tells Lady Catherine "Booking failed." But the Booking Service thinks it said "Yes" and is waiting for payment confirmation. The system is now in an inconsistent state again.

The engineers realized they were fighting the Two Generals Problem. No amount of messaging could guarantee perfect coordination over an unreliable network. They needed a different approach.

---

## The Royal Decree: The Two-Phase Commit (2PC)

{{< figure src="The_Royal_Decree_2PC.jpg" alt="The King coordinating a two-phase commit" >}}

To solve this, early Architects tried to recreate the Monolith's safety using the **Two-Phase Commit (2PC)**. Think of this as the **Royal Decree**.

The King acts as the **Transaction Coordinator**. He summons the Guild Masters of the Mercers and the Wheelwrights. He holds them at gunpoint.

**Phase 1: The Prepare Phase (The Voting)** The King asks: *"Can you commit to this trade? Check your ledgers. Lock your vaults."*

- The Mercer puts a lock on Aldous's 50 gold. Nobody can touch it. He votes: **"Yes."**
- The Wheelwright checks his stock. He reserves the wheels. He votes: **"Yes."**

**Phase 2: The Commit Phase (The Execution)** The King sees two "Yes" votes. He shouts: **"COMMIT!"** Both Guilds write the transaction permanently and unlock their vaults.

**The Royal Travel Service's 2PC Implementation:** The CTO decided to implement Two-Phase Commit for Lady Catherine's bookings. A **Booking Coordinator Service** orchestrates the transaction:

**Phase 1: Prepare**
1. Coordinator asks Payment Service: "Can you deduct 200 gold from Lady Catherine?"
   - Payment Service locks the 200 gold
   - Responds: "Yes, ready to commit"

2. Coordinator asks Booking Service: "Can you reserve carriage to Rome?"
   - Booking Service locks the seat
   - Responds: "Yes, ready to commit"

**Phase 2: Commit**
All services voted "Yes", so Coordinator sends: **"COMMIT!"**
- Payment Service: Permanently deducts 200 gold, unlocks account
- Booking Service: Permanently reserves seat, unlocks carriage

**Success!** Lady Catherine gets her booking, and all ledgers are consistent.

**The Failure Mode:** 

{{< figure src="The_Coordinator_Failure.jpg" alt="The coordinator fails, leaving resources locked" >}}

This sounds safe, but it is fragile. 

What if, after voting "Yes", the Wheelwright's guild hall is struck by lightning (Server Crash)? The Mercer is still holding the lock on Aldous's gold. He is waiting for the King's command. But the King is waiting for the Wheelwright. The entire economy **Blocks**. 

While the lock is held, Aldous cannot spend his gold elsewhere. The system freezes. In a city of 100 services, if one service hangs during a 2PC, the entire transaction chain halts. The Royal Decree provides **Consistency**, but it sacrifices **Availability** (**The CAP Theorem**). In a busy medieval market, we cannot afford to stop trading every time a Wheelwright takes a nap.

**The Black Friday Disaster:** During the biggest sale day, thousands of travelers tried to book journeys simultaneously. The 2PC coordinator was overwhelmed:

- Payment Service had 5,000 accounts locked, waiting for coordinator
- Booking Service had 3,000 carriage seats locked, waiting for coordinator
- Travelers couldn't make any bookings because resources were locked
- When Coordinator timed out, locks remained held
- Services didn't know whether to commit or rollback

The platform was effectively frozen for 2 hours. Travelers saw spinning wheels. Bookings failed. The King sent an angry letter: *"Your travel service was completely unavailable during the busiest shopping day of the year. This is unacceptable!"*

The CTO realized: 2PC provides consistency, but it sacrifices availability. In a high-traffic travel platform, availability is more important than perfect consistency. They needed a different approach.

---

## The Merchant's Deal: The Saga Pattern

{{< figure src="The_Merchant_of_Prato.jpg" alt="Francesco Datini managing distributed transactions" >}}

The wise Architect looks not to the King, but to the Merchants. Men like **Francesco Datini**, the "Merchant of Prato" (1335–1410), ran vast multinational empires without a central database. They had branches in Florence, Pisa, Avignon, and Barcelona. How did they keep their books in sync? They didn't. At least, not instantly. They used **Sagas**.

A Saga is a sequence of local transactions. Instead of one big lock, we break the trade into steps.

{{< figure src="The_Saga_Steps.jpg" alt="Sequential steps of a saga transaction" >}}

1. **Mercer:** Deduct 50 Gold. (Commit). Send message to Wheelwright.
2. **Wheelwright:** Allocate Wheels. (Commit). Send message to Logistics.
3. **Logistics:** Ship Wagon. (Commit).

**The Royal Travel Service's Saga Implementation:** After the Black Friday disaster, the engineers redesigned Lady Catherine's booking flow as a Saga. No more coordinator. No more locks. Each service commits its local transaction immediately.

**The New Booking Flow:**

**Step 1: Payment Service**
- Receives booking request for Lady Catherine's York to Rome journey
- Deducts 200 gold from her account
- **Commits immediately** (no locks, no waiting)
- Records: "Payment received for booking-12345"
- Publishes event: "Payment Completed" (booking-12345, 200 gold)

**Step 2: Booking Service**
- Receives "Payment Completed" event
- Reserves carriage and rooms at The Golden Lion and The Silver Stag
- **Commits immediately**
- Records: "Booking confirmed for booking-12345"
- Publishes event: "Booking Confirmed" (booking-12345, York to Rome)

**Step 3: Notification Service**
- Receives "Booking Confirmed" event
- Sends confirmation to Lady Catherine
- **Commits immediately**

**Success!** The entire saga completes. Lady Catherine gets her booking. All services are *eventually consistent*.

**The Compensating Transaction (The Wergild)** 

{{< figure src="The_Compensating_Transaction.jpg" alt="Unwinding a failed saga with compensation" >}}

But what if Step 2 fails? The Wheelwright has no wheels. The system cannot magically "undo" Step 1. The Mercer has already committed the gold deduction. Instead, the Wheelwright sends a **Failure Event**: *"Purchase Failed."* The Mercer listens for this event. When he hears it, he executes a **Compensating Transaction**: *"Refund 50 Gold to Aldous."*

This effectively undoes the business impact of the first step.

- **The Risk:** For a few seconds (or minutes), the system was **Inconsistent**. Aldous had paid, but had no wagon. This is **Eventual Consistency**.
- **The Reward:** No locking. The Mercer deducted the gold and immediately went back to serving other customers. The system is highly **Available**.

**The Failed Booking Scenario:**

**Step 1:** Payment Service deducts 200 gold from Lady Catherine, commits immediately, publishes "Payment Completed"

**Step 2:** Booking Service receives "Payment Completed" event, attempts to reserve carriage, **Failure!** All carriages to Rome are full, publishes "Booking Failed"

**Step 3:** Payment Service (Compensating Transaction) receives "Booking Failed" event, executes compensation: Refund 200 gold to Lady Catherine, commits immediately, publishes "Refund Completed"

**Step 4:** Notification Service receives "Refund Completed" event, sends message to Lady Catherine: "Sorry, no availability. Your 200 gold has been refunded."

**The Inconsistency Window:** For about 2 seconds, Lady Catherine's account showed 200 gold deducted but no booking. If she checked her account during those 2 seconds, she'd be confused. But within 2 seconds, the refund completed, and her account was correct again.

This is **Eventual Consistency**. The system is temporarily inconsistent but eventually becomes consistent.

**The Trade-off:** The service accepted temporary inconsistency in exchange for no locks (high availability), no coordinator bottleneck (scalability), services can fail independently (resilience), and each service commits immediately (low latency).

### The Bill of Exchange: The Medieval Asynchronous Event

{{< figure src="The_Bill_of_Exchange.jpg" alt="A bill of exchange traveling across Europe" >}}

The historical equivalent of the Saga message was the **Bill of Exchange**. A merchant in Florence would write a bill ordering his agent in Bruges to pay a debt. This bill would travel across the Alps (The Network). If the agent in Bruges refused to pay (Transaction Failure), he would draft a formal legal document called a **Protest**. 

{{< figure src="The_Protest_Document.jpg" alt="A protest document triggering compensation" >}}

This Protest would travel back to Florence, triggering a "Compensating Transaction" - the original merchant would have to reimburse the buyer, plus interest and courier fees. The medieval economy *ran* on eventual consistency. It accepted that the view of the ledger in Florence and the view in Bruges would differ by three weeks (Network Latency). They reconciled the books later.

**The Royal Travel Service's Real-World Saga Complexity:**

The actual booking saga was more complex than the simple example:

**The Full Booking Saga:**

1. **Payment Service:** Deduct gold → Publish "Payment Completed"
2. **Booking Service:** Reserve carriage → Publish "Carriage Reserved"
3. **Inventory Service:** Reserve inn room → Publish "Room Reserved"
4. **Partner Service:** Notify inn of booking → Publish "Inn Notified"
5. **Loyalty Service:** Award travel points → Publish "Points Awarded"
6. **Notification Service:** Send confirmation email → Publish "Email Sent"

**Failure at Step 3:** Inn has no rooms available

**Compensating Transactions (in reverse order):**
1. **Partner Service:** Cancel inn notification (no-op, nothing to cancel)
2. **Inventory Service:** (Failed here, nothing to compensate)
3. **Booking Service:** Release carriage reservation → Publish "Carriage Released"
4. **Payment Service:** Refund gold → Publish "Refund Completed"
5. **Notification Service:** Send "Booking Failed" email

**The Complexity:** Each step needed a compensating transaction. The engineers had to carefully design:
- What happens if a compensating transaction fails?
- What if the network fails during compensation?
- How do we ensure compensations happen in the right order?
- How do we handle partial compensations?

**The Saga Orchestration Service:** You built a dedicated service to manage saga state:
- Tracks which steps completed
- Triggers compensating transactions on failure
- Retries failed compensations
- Provides visibility into in-flight sagas
- Alerts engineers when sagas get stuck

This added complexity, but it gave you the availability and scalability you needed.

---

## Choosing Your Truth

When building the Code Citadel, you must choose your definition of Truth.

**1. The Strong Consistency of the Keep (ACID/2PC)** 

Use this only when you must. If you are moving money between two bank accounts *inside the same bank*, use a Monolith or a distributed lock. The cost of error is too high. You cannot allow money to vanish, even for a second.

**2. The Eventual Consistency of the City (BASE/Sagas)** 

Use this for almost everything else. If a user buys a book, it is okay if the "Recommendation Service" doesn't know about it for 5 seconds. It is okay if the "Inventory Count" is slightly off for a minute, as long as we have a process (a Saga) to handle the backorder if we oversell.

**The Royal Travel Service's Final Architecture:**

After two years of painful learning, the service settled on a hybrid approach:

**Strong Consistency (2PC) for:**
- **Financial settlements with partners:** When paying The Golden Lion their share of revenue, there cannot be inconsistencies.

**Eventual Consistency (Sagas) for:**
- **Traveler bookings:** The vast majority of transactions. Sagas provide the availability and scalability needed.
- **Notifications:** Messages can be delayed without breaking the business.

**The Results After Two Years:**

**Metrics:**
- 50,000 bookings per day (vs. 500 in the Keep under Master Clerk Edmund)
- 99.5% availability (vs. 95% with 2PC)
- Average booking time: 3 seconds (vs. 8 seconds with 2PC)
- Inconsistency window: Average 2 seconds (acceptable)

---

**The Lesson of the Merchant** 

Francesco Datini left behind 150,000 letters. His "Database" was a chaotic stream of asynchronous messages flying across Europe. He knew that he could never know the *exact* state of his empire at this very second. He operated on **Trust** and **Reconciliation**. He knew that eventually, the letters would arrive. Eventually, the books would balance.

The Royal Travel Service adopted Datini's wisdom. The CFO initially resisted: "How can we run a business if we don't know the exact state of our finances at any moment?"

The CTO explained: "We know the state within 2 seconds. For a travel business, that's fast enough. And in exchange, we can handle 50,000 bookings per day instead of 500. We can stay available during peak season instead of crashing."

We must learn to let go of the Royal Truth. We must learn to live with the Merchant's ambiguity. In the Microservices City, the Ledger is not a stone tablet. It is a story told by a thousand pigeons, arriving one by one.

**The King's Assessment:** Two years after moving to the City, the King reviewed the travel service:

"The ledgers are no longer perfect. Sometimes, for a few seconds, the books don't balance. But the service is available when my subjects need it. The Royal Travel Service handles the Spring Festival without crashing. It serves travelers across the entire kingdom, not just the capital. The temporary inconsistency is a small price to pay for the scale and reliability now provided."

*Next in the Chronicle: [**The Great Fire**]({{< relref "/posts/the-code-citadel/4-the-great-fire/" >}}) - We explore how to design for catastrophe. When a tiny service catches fire, how do we prevent it from burning down the entire city?*
