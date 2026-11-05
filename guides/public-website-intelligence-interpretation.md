# Interpret a public website intelligence report

A website's public code can reveal what a company measures, which products it promotes, and how parts of its customer journey work. Tracking containers are useful because they connect events to specific URLs, button labels, products, and values.

The report is an inventory of public evidence. It is not access to the company's analytics reports, advertising settings, customer database, or financial statements. This guide explains how to read the output of the [public website intelligence audit prompt](public-website-intelligence-prompt.md).

All identifiers, event names, prices, and scenarios used as examples below are hypothetical. Platform documentation was checked on 2026-11-05. Recheck the linked documentation when interpreting a later audit.

## How the technique works

A standard Google Tag Manager web installation includes an identifier such as `GTM-EXAMPLE`. The browser retrieves the published container's JavaScript from a URL shaped like this:

```text
https://www.googletagmanager.com/gtm.js?id=GTM-EXAMPLE
```

The delivered configuration may describe tags, trigger conditions, variables, and custom code. Reading it can reveal that a particular button sends an event to an advertising platform, or that a subscription CTA exists on a little-known landing page.

This is the published browser configuration. It does not provide editor access, unpublished workspace changes, analytics results, or all server-side behavior. Google documents how [server-side tagging processes data outside the browser](https://developers.google.com/tag-platform/tag-manager/server-side/intro).

Some sites embed tracking directly, deliver scripts through a first-party host, or load different tags according to consent and geography. Failure to find `GTM-` in the homepage is not evidence that tracking is absent.

## Evidence has different levels

| Evidence | What it establishes | What remains unknown |
| --- | --- | --- |
| A tag exists in a published container | The delivered configuration includes it. | Whether its trigger is reachable or the tag fires successfully. |
| A browser executes the tag | The tag runs under the observed conditions. | Whether a destination accepts the event. |
| A browser sends a tracking request | An outgoing request contains the observed fields. | Whether the platform counts, attributes, deduplicates, or rejects it. |
| A product string exists in a bundle | The delivered code contains that string. | Whether the feature is enabled, available, or maintained. |
| A public pricing page advertises a plan | The company publicly presents the offer. | Actual uptake, negotiated prices, margin, and availability to every customer. |
| A dated release note announces a feature | The company made that announcement at that time. | Whether every account received it then. |

Confidence belongs to a claim. An analyst can be highly confident that `team_plan_trial_started` exists and still have low confidence that a new team plan is about to launch.

Keep four categories separate in the report:

- **Observation.** The exact source fact.
- **Interpretation.** What the fact means in its technical context.
- **Hypothesis.** A plausible business explanation that needs more evidence.
- **Unknown.** Information the inspected sources do not establish.

## IDs, tags, events, and goals are different things

| Term | Meaning |
| --- | --- |
| Container ID | Identifies a GTM container. |
| Tag | A configured action that loads code or sends data when its conditions match. |
| Trigger | The conditions under which a tag is eligible to fire. |
| Variable | A value read or computed when the tag runs. |
| Event | An interaction or occurrence represented in tracking. |
| Destination | The analytics or advertising endpoint configuration that receives data. |
| Conversion action | An action the advertising platform is configured to count as a conversion. |
| Bidding goal | A goal used by the campaign's bidding configuration. |

A Google Ads conversion commonly combines an ID such as `AW-123456789` with a conversion label. Multiple actions can share an ID. Counting occurrences of `AW-` therefore does not count conversion actions. Some GTM configurations store the numeric ID without the prefix. [Google's conversion-tag documentation](https://developers.google.com/tag-platform/devguides/conversions) describes the ID, label, and optional value fields.

Likewise, a tracked action is not automatically a bidding objective. Google distinguishes primary and secondary actions, with additional rules for campaign goals and custom goals. The public tag does not establish those account settings. [Google's primary and secondary conversion guidance](https://support.google.com/google-ads/answer/11461796)

## Read the trigger before trusting the event name

Consider this hypothetical catalogue entry:

| Field | Observed configuration |
| --- | --- |
| Event | `booking_completed` |
| Trigger | Click text equals `Confirm booking` |
| Page condition | Path starts with `/quotes/` |
| Value | Absent |

The event observes a button click. It does not establish that the booking succeeded. The request might fail, validation might reject the form, or the visitor might abandon a later payment step.

Names such as `purchase`, `qualified_lead`, and `trial_started` are labels chosen by implementers. A successful transaction callback is stronger evidence of completion than a click trigger, but even client-side success needs context.

Trigger details can change the meaning:

- An exact `/signup` match differs from `/signup/`.
- A path condition without a hostname restriction may apply on several hosts that load the container.
- An initial-page-load trigger may miss client-side navigation.
- A history-change trigger may miss the initial page load.
- A text match can stop matching when copy or localization changes.
- A broad selector can include several unrelated buttons.
- An exclusion rule can prevent a seemingly eligible tag from firing.

These are investigation leads. They are not confirmed bugs until the relevant runtime behavior is tested.

## What a Microsoft Ads value of GBP 1 means

Suppose the report finds this Microsoft Advertising UET configuration:

```text
Action: Click
Category: CTA
Label: EnterDetails
Goal value: 1
Currency: GBP
Trigger: Click the Enter details button
```

The browser tag assigns a GBP 1 goal value to that event. It does not prove that the company receives one pound, spends one pound, or values an acquired customer at one pound.

Several explanations fit the same configuration:

| Explanation | Meaning | Can public code establish it alone? |
| --- | --- | --- |
| Nominal score | Each event contributes one unit expressed in a currency field. | No. |
| Placeholder | The implementer entered a value during setup without an economic model. | No. |
| Expected value | The business deliberately estimated the action's value at GBP 1. | No. |
| Legacy configuration | The value remains from an earlier implementation. | No. |

Microsoft conversion goals can use fixed revenue, variable revenue, or no revenue value. A matching goal's settings affect whether the browser-supplied amount contributes to reporting. [Microsoft's conversion-goal revenue reference](https://learn.microsoft.com/en-us/advertising/campaign-management-service/conversiongoalrevenue?view=bingads-13)

For a hypothetical calculation, assume 100 attributed events match a goal that accepts GBP 1 from each event. They could contribute GBP 100 of reported conversion value. That figure is not proof of GBP 100 in sales. Counting rules, attribution, and goal settings can make the number of reported conversions differ from the number of browser events.

The useful finding is that the company instrumented a particular step and assigned it a value. The reason for the value remains unknown unless another source explains it.

## Classify numbers before comparing them

| Number type | Hypothetical example | Safe interpretation | Unsupported conclusion |
| --- | --- | --- | --- |
| Assigned event value | `value: 500`, `currency: GBP` | This event carries a configured amount. | A lead costs GBP 500 or produces GBP 500 profit. |
| Dynamic transaction value | `value: order.total` | The tag reads a runtime field. | The actual amount, tax treatment, or margin. |
| Default value | A variable falls back to `1` | The fallback applies when the expected value is unavailable. | Every transaction is worth one unit. |
| Published price | GBP 49 per month | An advertised customer price. | Revenue after discounts, refunds, and taxes. |
| Annual-plan equivalent | GBP 40 per month, billed annually | A monthly presentation of annual pricing. | A monthly payment option at GBP 40. |
| Coupon identifier | `welcome20` | A promotion identifier. | Necessarily 20% off or GBP 20 off. |
| Credit | 5% toward the next booking | A future-use benefit under the offer. | Cash paid to the customer. |
| Affiliate share | 20% of platform commission | A share of the platform's commission. | 20% of the entire customer purchase. |
| Guarantee limit | Up to GBP 5,000 | An advertised coverage cap, subject to terms. | Typical compensation or expected customer value. |
| Comparison value | Benefits described as worth GBP 1,200 | The advertiser's valuation or comparison claim. | Its delivery cost or realized revenue. |
| Cookie duration | 90 days | Intended cookie retention. | The advertising account's attribution window. |
| Version or tag ID | Version `42` | A configuration identifier. | The launch date or chronological funnel order. |

Check units before formatting a value. A payment integration might use minor units, while an analytics tag uses major currency units. A bare `100` cannot be interpreted without its schema. An absent value is unknown, not zero.

Google also allows conversion values to be configured in the advertising account. A missing value in the browser does not mean the action has no value. [Google's conversion-value settings](https://support.google.com/google-ads/answer/13064107?hl=en)

## Why conversion values do not tell you what to bid

A configured lead value is different from acquisition cost. A competitor assigning GBP 800 to a demo does not establish that it pays GBP 800 for that demo.

Even a reliable expected lead value would not determine a profitable keyword bid without your click-to-lead rate, lead quality, costs, and required return.

As a hypothetical model, assume your expected contribution per qualified lead is GBP 200 and 5% of ad clicks become qualified leads. That implies GBP 10 of expected contribution per click before advertising cost and any costs excluded from the contribution calculation. It is not a recommended bid or a guarantee. A public tracking value does not supply those assumptions.

## How product discovery becomes credible

A strong discovery often follows several connected clues:

1. A tag references `business_plan_trial`.
2. Its trigger includes `/business` and a `Start team trial` button.
3. The public page describes shared workspaces and business billing.
4. A help article explains seat limits and eligibility.

Together, those sources support a business-plan finding. They still do not prove when the product launched.

Use these classifications:

| Classification | Evidence needed |
| --- | --- |
| Publicly marketed | A current public page describes the offering. |
| Publicly accessible | The relevant feature can be observed without bypassing access controls. |
| Shipped code, availability unknown | Business-specific code exists, but eligibility or activation is unverified. |
| Possible experiment or planned feature | A specific flag, variant, or future-facing message exists, but rollout is unresolved. |
| Historical residue | Older snapshots or contradictory current evidence support an outdated implementation. |
| False positive | The clue comes from generic library code, sample data, or an unrelated dependency. |

A feature flag's presence does not establish its current evaluation. A `waitlist` component inside an authentication SDK does not establish a company waitlist. A `v2` route does not establish a running A/B test.

Public pages absent from navigation can be useful discoveries. A public campaign page is not automatically secret or unreleased. A route returning HTTP 200 can also be a generic app shell, so inspect the content.

To claim a launch or change, compare dated evidence. Record the first time you observed it separately from the date it was created. Sitemap modification dates and asset names are supporting clues, not definitive chronology.

## What broader findings can reveal

| Source | Useful questions | Limits |
| --- | --- | --- |
| Pricing and checkout links | Which plans, billing periods, and upgrade paths are advertised? | Negotiated prices and conversion rates remain unknown. |
| Affiliate terms | How are partners rewarded, and for how long? | Published rewards are not blended acquisition cost. |
| Lead forms | Which fields qualify prospects, such as company size or budget? | A form field does not reveal sales scoring rules. |
| Sitemap and campaign pages | Which audiences, locations, services, and offers have dedicated content? | Content coverage does not establish traffic or active ad spend. |
| Help centre and changelog | Which workflows and integrations are documented? | Documentation can lag availability. |
| Client code | Which UI states, entitlements, routes, and business terms are shipped? | Dead code and shared components can mislead. |
| Cross-domain configuration | Which hosts were included in attribution setup? | Listed hosts may be old or inactive. |
| Telemetry outside GTM | Which app interactions have explicit instrumentation? | Instrumentation does not expose usage counts. |
| Careers pages | Which roles and skills are publicly sought? | A vacancy does not prove a roadmap commitment. |

Implementation clues are most useful when they explain a business observation. Naming a framework adds little unless it clarifies why navigation events differ from page loads or why a feature is visible in a shared bundle.

## Common report mistakes

| Mistake | Better reading |
| --- | --- |
| Counting every `AW-` match as a conversion | Deduplicate ID and label pairs. Distinguish base configuration from conversion tags. |
| Counting GTM listeners as marketing events | Separate internal support entries from business events. |
| Equating tracked events with bidding goals | Mark campaign goal settings as unknown. |
| Treating a click as a completed sale | Describe the trigger and its completion limitations. |
| Assuming duplicated initialization means duplicated conversions | Check actual event emissions and deduplication behavior. |
| Treating every number near `value` as money | Resolve its field, platform, units, and context. |
| Treating file order as funnel order | Reconstruct the journey from triggers and observed UX. |
| Treating a missing tag as missing tracking | Account for direct, server-side, offline, and imported events. |
| Reporting every library string as a product clue | Trace business-specific usage and corroborate it. |
| Reporting a potential trigger problem as a confirmed defect | Test the relevant behavior or label it a hypothesis. |
| Calling a discovered page a new launch | Find dated evidence or use "newly observed." |

## How to use the report

Start with findings that have both clear business relevance and independent evidence. A verified pricing tier or published affiliate programme is usually more actionable than an ambiguous event label.

Use event inventories to understand measurement priorities and generate questions. Use public offers to compare positioning. Use product clues to decide what deserves a closer look. Keep unresolved economics out of bidding decisions.

For later comparisons, preserve source URLs, timestamps, hashes, and short excerpts. Compare meaningful configuration rather than every byte of a vendor's generated runtime. A changed hash proves changed bytes, not a product launch. A removed tag does not prove a discontinued feature.

An effective report lets another reader trace every important conclusion to evidence and see exactly where interpretation begins.
