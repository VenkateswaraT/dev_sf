# Platform Events — Considerations & Allocations (clean extract)

> **Source:** Salesforce *Platform Events Developer Guide*, v67.0 (Summer '26).
> Faithfully transcribed from `platform_events.pdf` — **Considerations** (printed pp. 114–121) and **Allocations + Error Status Codes** (printed pp. 132–143).
> **How to use:** this is your verify-and-check reference for the Active Reading Kit. Answer the interrogation-bank questions from memory *first*, then confirm/correct here. The exact numbers are what you cite in design reviews.

---

## Contents

**Part 1 — Considerations**
- [Defining platform events](#defining-platform-events)
- [Publishing platform events](#publishing-platform-events)
- [Subscribing with Processes and Flows](#subscribing-with-processes-and-flows)
- [Publishing & subscribing with Apex and APIs](#publishing--subscribing-with-apex-and-apis)
- [Decoupled publishing and subscription](#decoupled-publishing-and-subscription)
- [What's the difference between the Salesforce events?](#whats-the-difference-between-the-salesforce-events)

**Part 2 — Allocations**
- [Which platform events do allocations apply to?](#which-platform-events-do-allocations-apply-to)
- [Common platform event allocations](#common-platform-event-allocations)
- [Default allocations for publishing and delivery](#default-allocations-for-publishing-and-delivery)
- [Add-on license](#increase-allocations-with-an-add-on-license)
- [Monitoring usage against allocations](#monitoring-usage-against-your-allocations)
- [Standard-volume platform event allocations](#standard-volume-platform-event-allocations-legacy)
- [Error status codes](#platform-event-error-status-codes)

---
---

# Part 1 — Considerations

Special behaviors related to defining, publishing, and subscribing to platform events.

## Defining platform events

**Field-level security.** All platform event fields are read only by default, and you can't restrict access to a particular field. Field-level security permissions don't apply — the event message contains all fields.

**Enforcement of field attributes.** Platform event records are validated so that custom-field attributes are enforced: the **Required** and **Default** attributes, the precision of number fields, and the maximum length of text fields.

**Permanent deletion of event definitions.** Deleting an event definition permanently removes it — it can't be restored. Delete the associated triggers first. Published events that use the definition are also deleted.

**Renaming event objects.** Delete the associated triggers before renaming. If the event name changes after clients have subscribed, subscribed clients must **resubscribe** to the updated topic. Re-add your trigger for the renamed event object.

**No associated tab.** Platform events have no tab — you can't view event records in the Salesforce UI.

**No SOQL support.** You can't query event messages with SOQL.

**No record page support in Lightning App Builder.** Your platform events appear in the object list, but you can't create a Lightning record page for them because event records aren't available in the UI.

**Platform events in package uninstall.** When uninstalling a package with *"Save a copy of this package's data for 48 hours after uninstall"* enabled, platform events **aren't** exported.

**Event volume in package installs/upgrades.** Installing a managed or unmanaged package that contains a **standard-volume** platform event causes the event type to be saved as **high volume** in the subscriber org. Upgrading a managed package doesn't change the event volume in the subscriber org.

**No support in Professional and Group editions.** Platform events aren't supported there; installing a package containing platform event objects **fails** in those orgs.

**Permissions.** To define a custom platform event you need **Customize Application**. For publishing/subscribing permissions, see *Platform Event Permissions*.

## Publishing platform events

**Publishing in read-only mode.** During read-only mode (Salesforce maintenance), publishing **standard-volume** events throws an exception and the events aren't published. Publishing **high-volume** events in read-only mode sometimes fails when the event schema isn't up to date in Salesforce.

**High-volume platform event persistence.** High-volume events are temporarily persisted to and served from an **industry-standard distributed system** during the retention period. A distributed system doesn't have the semantics/guarantees of a transactional database, so Salesforce **can't provide a synchronous response** for a publish request. Events are queued and buffered, and Salesforce attempts to publish asynchronously. In rare cases an event message might not be persisted on the initial or subsequent attempts — meaning it **isn't delivered to subscribers and isn't recoverable**.

**At-least-once publishing and duplicate events.** Asynchronous publishing uses the **at-least-once** model, *not* exactly-once. If the system hits an internal error while publishing the queued event, it retries; in rare cases no publish acknowledgment is received, so the same event is published more than once. (If acknowledgments arrive as expected, no duplicates are published.)

> A **duplicate event** has the **same `EventUuid`**, a **different `ReplayId`**, and the **same payload**. Handle duplicates in your subscriber — e.g., implement **deduplication using `EventUuid`**.

## Subscribing with Processes and Flows

**Supported standard events.** Processes and flows can subscribe to custom platform events and these standard events:
`AIPredictionEvent` · `BatchApexErrorEvent` · `FlowExecutionErrorEvent` · `FOStatusChangedEvent` · `OrderSummaryCreatedEvent` · `OrderSumStatusChangedEvent` · `PlatformStatusAlertEvent`

**Infinite loops and limits.** Publishing an event from a process/flow that also creates that same event triggers itself. To avoid an endless loop, make sure the new event message's field values **don't meet the filter criteria** of the associated criteria node.

**Subscriptions related list.** The platform event's detail page shows which entities are waiting for its messages, with a link to each subscribed process. Waiting flow interviews appear as one "Process" subscriber.

**Uninstalling events.** Before uninstalling a package that includes a platform event: delete interviews waiting for its messages, and deactivate processes that reference it.

**Einstein predictions.** Every Einstein prediction generates an `AIPredictionEvent`. Use event condition filters to trigger only on predictions for a specific object. Beware loops: if your process/flow updates a field used by an Einstein prediction, Einstein re-runs and writes back new results → a new `AIPredictionEvent` → re-triggers you. Only update fields **not** used in Einstein predictions.

**Platform Event–Triggered flows** cannot call another flow via the **Subflow** element.

### Event processes (process-only)
- **Apex actions:** you can't use an event reference to set an sObject variable in the Apex class.
- **Email alerts:** can't use values from platform event messages. Workaround: build an autolaunched flow that takes each event field as an input variable and sends the email via a Send Email action; call that flow from the process, mapping event fields to flow variables.
- **Flows actions:** you can't use an event reference to set a record variable in the flow (even when the platform event is the record variable's object). Workaround: create an input variable per event field in the flow, then map event references to those variables when adding the Flows action.
- **Packaging event processes:** the associated object isn't included automatically — advise subscribers to create it, or add it to the package manually.

### Formulas & filtering (flows)
- **Formulas:** to reference a platform event in a flow formula, pass the event data into a record variable in the **Pause** element, then reference the field in that record variable.
- **Event condition values:** when filtering, only the **first 765 bytes** of the condition value are used (fewer characters with multi-byte characters).

## Publishing & subscribing with Apex and APIs

**Only `after insert` triggers** are supported — event notifications can't be updated, only inserted (published).

**Infinite trigger loop.** Publishing an event from a trigger on the same event object fires the trigger in an infinite loop.

**Empty-string text fields in Apex.** Publishing in Apex with a Text field set to `''` delivers the field as **`null`**, not empty string. (Empty string *is* preserved via APIs, flows, and processes.)

**`OwnerId` of new records.** In a platform event trigger, records you create get `OwnerId` = **Automated Process** by default. To set another value, configure the trigger to run as another user (via `PlatformEventSubscriberConfig`), or set `ownerId` explicitly on the new record:

```apex
Opportunity newOpp = new Opportunity(
    OwnerId = customerOrder.createdById,
    AccountId = acc.Id,
    StageName = 'Qualification',
    Name = 'A ' + customerOrder.Product_Name__c + ' opportunity for ' + acc.name,
    CloseDate = Date.today().addDays(7));
```
For cases and leads you can also use assignment rules (`AssignmentRuleHeader` / DML options).

**Changing the Opportunity `OwnerId`.** If a trigger updates opportunity `OwnerId` while **opportunity splits** are enabled and runs as the default Automated Process user, a set of splits totaling **0%** is created — invalid (must be 100%), causing validation errors on later updates (e.g., Amount, Owner). Fix: run the trigger as a **different user** via `PlatformEventSubscriberConfig`.

**No email from a platform event trigger** with the default Automated Process running user — the sender (Automated Process) has no email address, so `Messaging.SingleEmailMessage` can't send. Change the running user to send email.

**Replaying past events.** You can replay past events via **Pub/Sub API or Streaming API (CometD)** — **but not Apex** and other subscribers.

**Salesforce maintenance & sandbox refresh.** Rarely, maintenance (data-center migration, instance refresh) or a **sandbox refresh** can **reset the stream** of retained high-volume events — those events are no longer replayable, and the **`ReplayId` before the activity has no relation to `ReplayId` after**. Site switches *don't* affect the retained stream — **except** Apex triggers used as **parallel subscriptions** that lag behind: they can't continue processing pre-switch events and resume from events published **after** the switch.

**MuleSoft Salesforce Connector after Hyperforce migration / sandbox refresh** can hit an **invalid Replay ID** error if the Mule app accesses a Replay ID no longer valid on the new instance.

**Millisecond precision in DateTime fields.** For events delivered to **CometD** clients (JSON), DateTime is `YYYY-MM-DDTHH:mm:ss.sssZ` (with ms). API v42.0 and earlier omit ms. For events delivered to **Apex triggers**, DateTime fields **don't** include millisecond precision.

**Apex trigger subscriptions disabled in inactive orgs.** If an org becomes inactive, all Apex trigger subscriptions stop and are disabled — no processing of incoming or missed messages. After reactivation, new Apex trigger subscriptions start when a message is next published.

## Decoupled publishing and subscription

When publish behavior is **Publish Immediately**, the event is published **outside** a Lightning Platform database transaction. Publishing and subscription are **decoupled** — the subscriber can't assume an action from the publishing transaction is committed before the event is received.

> **Note:** This decoupled behavior **doesn't apply** to events whose publish behavior is **Publish After Commit**.

**Publisher doesn't respect transaction boundaries.** With Publish Immediately, a record the publisher creates *after* publishing might not be committed before the subscriber receives the event — so a subscriber lookup can fail. Example:
1. A process publishes an event and creates a Task.
2. A trigger on Task delays the commit of the Task record.
3. A second process (subscribed) receives the event, looks up the new Task, and errors because the record isn't committed yet:
   > *"MyProcess process is configured to start when a MyEvent platform event message occurs. A MyEvent message occurred, but the process didn't start because no records in your org match the values specified in the process's Object node."*

Conversely, a record a *subscriber* creates after receiving an event might not be immediately findable (processing is async and can be slow for complex logic).

**Event published from a trigger.** An `after insert` trigger that publishes a **Publish Immediately** event can have the event processed before the trigger's record is committed — a subscribed process looking up that record fails.

> **Solution (both cases):** set publish behavior to **Publish After Commit**. The event is published only after the transaction commits, so the subscriber can find the record.

## What's the difference between the Salesforce events?

**Custom events** (generate & deliver custom messages):
- **Custom Platform Events** — secure, scalable, customizable notifications within Salesforce or from external sources; **you define the fields/schema**; publish/subscribe on-platform or externally.
- **Generic Events** — custom events with **arbitrary payloads**; you **can't define the schema**.

**Data events** (tied to Salesforce records):
- **Change Data Capture events** — published for record and field changes.
- **PushTopic events** — track field changes, tied to Salesforce records.

*(For a custom-vs-data comparison, see "Streaming Event Features" in the Streaming API Developer Guide.)*

**Standard events (security, Apex, monitoring):**
- **Asset Token Events** (`AssetTokenEvent`) — monitor OAuth 2.0 asset-token flow completions for a connected device.
- **Batch Apex Error Events** (`BatchApexErrorEvent`) — catch errors during batch Apex, including uncatchable ones like Apex limit exceptions.
- **Real-Time Event Monitoring** — standard events to monitor user activity in real time (e.g., `LoginEventStream` for logins).

**Event-like features** — features that *look* like streaming events but aren't event notifications.

---
---

# Part 2 — Allocations

Allocations for platform event definitions, publishing/subscribing, and delivery to Pub/Sub API clients, CometD clients, empApi Lightning components, and event relays.

## Which platform events do allocations apply to?

| Event Type | Counts toward **publishing** allocation | Counts toward **delivery** allocation & add-on entitlement |
|---|:---:|:---:|
| **Custom events** that you define | ✅ Yes | ✅ Yes |
| **Standard events** (see *Standard Platform Event Object List*) | ❌ No | ⚠️ Per event — check the *"Event Delivery Allocation Enforced"* section in each event's reference doc |
| **Real-Time Event Monitoring** events | ❌ No | ❌ No |

> When allocations aren't enforced, **system protection limits** apply.

## Common platform event allocations

Apply to **both** standard-volume and high-volume events.

| Description | Performance & Unlimited | Enterprise | Developer | Professional (with API Add-On) |
|---|--:|--:|--:|--:|
| Max **platform event definitions** per org ¹ | 100 | 50 | 5 | 5 |
| Max **concurrent CometD clients** (subscribers), all channels & event types ² | 2,000 | 1,000 | 20 | 20 |
| Max **Process Builder processes + flows** that can subscribe to a platform event | 4,000 | 4,000 | 4,000 | 5 |
| Max **active** Process Builder processes + flows that can subscribe | 2,000 | 2,000 | 2,000 | 5 |
| Max **custom channels** for all Platform Events events (except RTEM) | 100 | 100 | 100 | 100 |
| Max **custom channels** for all Change Data Capture events | 100 | 100 | 100 | 100 |
| Max **custom channels** for Real-Time Event Monitoring events | 3 | 3 | 3 | 3 |
| Max **distinct custom platform events** added to a channel as channel members ³ | 50 | 50 | 5 | 5 |
| Max **Real-Time Event Monitoring events** added to a channel as channel members ³ | 10 | 10 | 10 | 10 |
| Max **event message size** you can publish ⁴ | 1 MB | 1 MB | 1 MB | 1 MB |

¹ Events from an installed managed package **share** the org's platform-event-definition allocation.
² The concurrent-client allocation applies to **CometD** and to **all** event types (platform, change, PushTopic, generic). It does **not** apply to non-CometD clients (Apex triggers, flows, processes). **empApi uses CometD** and consumes this allocation — each logged-in empApi user = **one** client (multiple tabs share one connection = still one client). A client exceeding the allocation gets an error and can't subscribe until a connection frees up.
³ If the same event is added to multiple channels, it's counted **once** toward the allocation.
⁴ Objects with hundreds of custom fields or many long-text-area fields can hit this; the publish call then errors.

## Default allocations for publishing and delivery

Apply when the org has **no add-on licenses** — enforced, can't be exceeded.

- **Publishing allocation** = event messages you can send to the event bus **per hour** by **any** method (Apex, Pub/Sub API and other APIs, flows, processes).
- **Delivery allocation** = event messages delivered in a **24-hour** period to **Pub/Sub API, CometD, empApi components, and event relays**. **Excludes** non-API subscribers — **Apex triggers, flows, and Process Builder processes don't count** against delivery.
- The **delivery allocation is shared** between **high-volume platform events and Change Data Capture events**.

**Delivery usage is combined across all subscribers.** Delivered events are counted **per subscribed client** (including event relays) and summed. *Example:* Unlimited Edition, 50,000/24 h allocation. 20,000 messages delivered to **two** clients = **40,000** consumed → 10,000 remaining in the window.

**Rolling limits.** Hourly publishing usage = publishes in the last hour; daily delivery usage = delivered events in the last 24 hours. Publishing limit is checked when an event is **published**; delivery limit is checked when an event is **received**.

### Default allocation numbers

| Description | Performance & Unlimited | Enterprise & Professional (with API Add-On) | Developer |
|---|--:|--:|--:|
| **Event Delivery** — max delivered messages in last 24 h, shared by all clients | 50,000 | 25,000 | 10,000 |
| **Event Delivery for Salesforce Order Management** (with OM license) | 100 | 100 | 100 |
| **Event Delivery for BYO Channel for Messaging / CCaaS** (per license; with Digital Engagement, Contact Center, or Einstein 1 Service) | 25,000 | 25,000 | 25,000 |
| **Event Publishing** — max messages published per hour | 250,000 | 250,000 | 50,000 |

- **Delivery** applies to: Pub/Sub API · CometD · empApi Lightning component · Event relays. **Doesn't apply to:** Apex triggers · Flows · Process Builder processes.
- **Publishing** applies to all methods: Apex · Pub/Sub API · REST API · SOAP API · Bulk API · Flows · Process Builder processes.

> **Publishing with REST/SOAP/Bulk API also consumes daily API request allocations.** Publishing with **Pub/Sub API, Apex, flows, or processes does not** consume daily API request allocations.

**Why is publishing higher than delivery?** Different client sets. Publishing applies to all publish methods; delivery applies only to a subset (not Apex triggers/flows/processes). E.g., **Apex publish + Apex trigger subscribe → only publishing counts** (you can process far more). **Apex publish + Pub/Sub API subscribe → both count.**

> **Note:** Even though Apex triggers/flows/processes don't count against delivery, their processing **rate** depends on subscriber processing time and event volume — slower processing means longer to reach the tip of the stream.

### If you exceed an allocation
- **Exceed hourly publishing:** the publish call fails with **`LIMIT_EXCEEDED`**. Events aren't published or queued — wait for usage to drop, then republish.
- **Exceed delivery:** an error is returned and the **subscription is disconnected**.
  - **CometD:** `403::Organization total events daily limit exceeded` (Bayeux `/meta/connect`).
  - **Pub/Sub API:** error code `sfdc.platform.eventbus.grpc.subscription.limit.exceeded`, message *"You have exceeded the event delivery limit for your org."*
  - **Recovery:** keep the subscriber disconnected temporarily; 24 h usage decreases over time; events received meanwhile are stored for the **72-hour** retention period; resume from where you left off using the **Replay ID** (Pub/Sub API and CometD). For persistently high volume, buy an add-on.

**To reduce delivery consumption:** use **stream filtering** (custom channels) to receive only relevant events, and remove unnecessary subscribers (each delivery to each subscriber counts).

## Increase allocations with an add-on license

Purchase a **Platform Events add-on** (contact your Salesforce Account Rep). It moves event **delivery** to a **monthly usage-based entitlement** model, allows usage spikes, and also raises **publishing**:

- **+100,000 delivered events/day** (3 million/month) as a usage-based entitlement.
- **+25,000 published events/hour.**
- Daily delivery isn't strictly enforced — a **grace allocation** (higher than purchased) lets subscribers keep receiving; Salesforce may adjust grace at any time.
- Entitlement **resets monthly** after your contract start date.
- Computed for **production orgs only** (not sandbox/trial).
- Overages monitored on a **calendar month** from contract start; monthly monitoring entitlement = **daily allocation × 30**.

**Example — one high-volume add-on license:**

| Description | Performance & Unlimited | Enterprise & Professional (with API Add-On) |
|---|---|---|
| **Event Delivery** entitlement (shared by all clients) | **Last 24 h:** 150,000 (50 K org + 100 K add-on + grace) · **Monthly:** 4.5 M (1.5 M org + 3 M add-on) | **Last 24 h:** 125,000 (25 K org + 100 K add-on + grace) · **Monthly:** 3.75 M (0.75 M org + 3 M add-on) |
| **Event Publishing** (per hour) | 275,000 (250 K org + 25 K add-on) | 275,000 (250 K org + 25 K add-on) |

*Delivery entitlement applies to Pub/Sub API, CometD, empApi, event relays — not Apex triggers/flows/processes. The monthly entitlement is returned in the limits REST API resource.*

## Monitoring usage against your allocations

**In Setup:** Quick Find → **Platform Events** → **Event Allocations** section. If you bought the add-on, the **grace allocation** and **monthly** usage also show. Usage-based entitlements appear under **Company Information → Usage-based Entitlements**.

| Metric | Default allocation | With add-on license |
|---|---|---|
| **Event Delivery** (CometD, Pub/Sub API, empApi, event relays) | Daily last-24 h via REST API limits: **`DailyDeliveredPlatformEvents`**; via Apex: `System.OrgLimit` → `DailyDeliveredPlatformEvents` (updates within minutes) | Daily as at left; **Monthly** in Setup → Platform Events → Event Allocations, REST limits **`MonthlyPlatformEvents`** (API ≤47.0) / **`MonthlyPlatformEventsUsageEntitlement`** (API ≥48.0, updates once/day); usage-based entitlement in Company Information |
| **Event Publishing** (per hour) | REST API limits → **`HourlyPublishedPlatformEvents`** | Same: **`HourlyPublishedPlatformEvents`** |

**Hourly delivery via REST API:** call the `limits` resource hourly; the difference between two hourly reads = events delivered in that hour. Example — 40,000 remaining at 12:00 PM, 38,500 at 1:00 PM → **1,500 delivered** to API subscribers in that hour.

```json
// GET /services/data/vXX.0/limits
{
  "DailyDeliveredPlatformEvents" : {
    "Max" : 50000,
    "Remaining" : 40000
  }
}
```

**Granular metrics:** SOQL on **`PlatformEventUsageMetric`** — separate/combined platform-event and CDC metrics, broken down by event name, client ID, event type, usage type, and time segment (CometD and Pub/Sub API clients, empApi, event relays). See *Enhanced Usage Metrics*.

## Standard-volume platform event allocations (legacy)

For standard-volume events defined in **API v44.0 and earlier**.

> **Important:** You can **no longer define new** standard-volume custom platform events. New platform events are **high volume** by default. **Standard-volume custom platform events retire in Summer '27** — migrate existing ones before retirement.

| Description | Performance & Unlimited | Enterprise | Developer & Professional (with API Add-On) |
|---|--:|--:|--:|
| **Event Delivery** — max delivered messages in last 24 h, shared by all CometD clients ¹ | 50,000 | 25,000 | 10,000 |
| **Event Publishing** — max messages published per hour | 100,000 | 100,000 | 1,000 |

Exceeding delivery returns `403::Organization total events daily limit exceeded` (Bayeux `/meta/connect`). Standard-volume messages generated after exceeding are stored in the event bus and retrievable within the **24-hour retention window**. Monitor via REST limits: delivery = **`DailyStandardVolumePlatformEvents`**, publishing = **`HourlyPublishedStandardVolumePlatformEvents`**.

¹ Add-on increases daily delivered events by **+100,000** (e.g., Unlimited 50,000 → 150,000); multiple add-ons allowed. Salesforce recommends CometD delivery **not exceed 5 million/day**.

## Platform event error status codes

Returned in the `SaveResult` when publishing fails.

**Synchronous errors** (returned immediately in the publish call result):

| Status code | Meaning |
|---|---|
| **`LIMIT_EXCEEDED`** | Published messages exceeded the **hourly publishing limit**, or the test limit for events published from an Apex **test** context. |
| **`PLATFORM_EVENT_PUBLISHING_UNAVAILABLE`** | Publishing failed because a service was temporarily unavailable. Try again later. |
| **`PLATFORM_EVENT_ENCRYPTION_ERROR`** | Couldn't publish due to an encryption problem — an org misconfiguration or a general encryption-service error. |

Where the code appears: **Apex** → `Database.SaveResult` → `Database.Error`. **SOAP API** → `SaveResult`. **REST API** → `errors` field in the JSON.

> **Note:** In **flows**, if an event fails for an event-triggered flow, the flow **never starts** — so no flow error email is sent. Debug via errors generated by the platform event.

---

*End of extract. Cross-check against the PDF's printed pages 114–121 and 132–143 if any number looks off; allocation figures change by release.*
