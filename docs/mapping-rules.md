# Provider mapping rules

Read this before you add or change a provider's service codes or tracking statuses. It covers where the code lives, how we translate a provider's codes into our own service levels and statuses, the edge cases we've run into so far, and what a reviewer checks before a mapping change is merged.

## Why these rules exist

Every delivery provider speaks its own language. One provider says `OUT_FOR_DELIVERY`, another says `WITH_COURIER`, and a third sends a numeric event code with a free-text description. Globomantics Shipping hides all of that behind a small set of service levels and shipment statuses, so the rest of the app only has to understand one vocabulary.

That only works if every provider is mapped the same way. A provider that maps a customs hold to `InTransit` makes a stuck parcel look healthy. A provider that maps a failed attempt to `OnHold` tells the customer the wrong story. These rules keep the mappings consistent between providers, between people, and between sessions with an agent.

## Files

Each provider has three files in `Globomantics.Shipping/Providers/<Provider>/`:

- `<Provider>Adapter.cs` implements `IProviderAdapter` and is registered in `Program.cs`.
- `<Provider>ServiceCodes.cs` lists the services we can book, as `ServiceOption` records.
- `<Provider>TrackingStatuses.cs` maps the provider's tracking codes to a `ShipmentStatus`.

The adapter only wires things together. It exposes `ProviderId`, `DisplayName`, `Services` and `TrackingStatuses`, and it should not contain any mapping logic of its own. If you find yourself writing an `if` in an adapter, the mapping belongs in one of the other two files.

`ProviderId` is a lowercase, hyphenated version of the provider's name, like `harbor-post` or `havsgren-lines`. It's stored on every shipment and used in URLs, so never change it once a provider is live.

Keep service code and tracking status changes in separate commits, so they can be reviewed and merged independently.

## Service codes

### The basics

- Copy each code exactly as the spec spells it, including the prefix. Don't fix what looks like a typo in a provider's code: if the provider sends it that way, that's the code.
- Use the provider's own name for the service in `Name`, prefixed with the provider's short name if the spec doesn't already do that (for example "Harbor Post Economy").
- `TransitDays` is the spec's number, not the level's typical value.
- Keep the services in the same order as the spec, so a reviewer can compare them side by side.

### Choosing a service level

Map each service to the closest existing `ServiceLevel`, based on the transit days in the spec:

| Transit days in the spec | ServiceLevel |
|---|---|
| 0 (delivered the same day) | `SameDay` |
| 1 | `Express` |
| 2 or 3 | `Standard` |
| 4 or more | `Economy` |

Go by the number, not by the name. A provider that calls a two-day service "Express" still maps to `Standard`. A provider that calls a one-day service "Priority" maps to `Express`.

If the spec gives a range, like "2 to 4 days", use the upper end. We'd rather promise a little less than we can deliver.

If a service has no transit days at all in the spec (some providers leave them out for international services), stop and ask. Don't guess a number.

### Services we don't add

- Services the provider marks as deprecated or "for existing customers only".
- Return services. Returns are booked through a separate flow and aren't part of the service list.
- Add-ons like insurance, signature on delivery, or Saturday delivery. Those aren't services in our model, even when the provider gives them a code.

## Tracking statuses

### The basics

- Tracking status dictionaries use `StringComparer.OrdinalIgnoreCase`. Provider feeds don't guarantee the case.
- Copy each code exactly as the spec spells it, including underscores.
- Never map a code to `ShipmentStatus.Unknown`. Unknown is what we show for codes that aren't mapped, and the Providers page lists those codes so we can spot them. Mapping a code to Unknown on purpose hides it from that list.
- Several codes can map to the same status. That's expected: most providers have more codes than we have statuses.
- If a code doesn't fit any status, stop and ask. Don't add a new `ShipmentStatus` on your own: every status shows up across the whole app.

### What each status means

| ShipmentStatus | Use it when | Shown as | Needs attention |
|---|---|---|---|
| `Booked` | The shipment is registered with the provider, but the provider doesn't have the parcel yet. | Booked | No |
| `PickedUp` | The provider has collected the parcel from the sender. | Picked up | No |
| `InTransit` | The parcel is moving through the provider's network: hubs, depots, line haul, sorting. | In transit | No |
| `OutForDelivery` | The parcel is with the person or vehicle that will deliver it today. | Out for delivery | No |
| `Delivered` | The recipient has the parcel, or has everything they need to collect it. | Delivered | No |
| `DeliveryFailed` | A delivery attempt was made and didn't succeed. | Delivery failed | Yes |
| `OnHold` | The parcel can't move until someone does something: customs, the sender, or the recipient. | On hold | Yes |
| `Returned` | The parcel is going back to the sender, or is already there. | Returned to sender | Yes |

"Needs attention" comes from `ShipmentStatusExtensions.NeedsAttention`, and those shipments are the ones someone has to look at. Getting that column wrong is the most expensive mistake you can make in a mapping, because it either hides a problem or creates work for nothing.

### Rules for common codes

- Registration, pre-advice, label created, manifested: `Booked`.
- Collected, accepted, picked up from sender, dropped off at a service point by the sender: `PickedUp`.
- Repeated hub scans are all `InTransit`. So are depot arrivals, sorting, line haul and "in transit to destination country".
- With courier, on the van, out for delivery, loaded for delivery: `OutForDelivery`.
- Neighbour, locker, and pickup point deliveries are `Delivered`. The recipient either has the parcel or has a code to collect it.
- Customs holds and address queries are `OnHold`. So is "waiting for payment of duties".
- A failed attempt is `DeliveryFailed`, even if the provider will try again tomorrow. The next attempt moves it back to `OutForDelivery`.
- A parcel on its way back to the sender is `Returned`, even before it arrives.
- Damaged or lost parcels: stop and ask. We handle those case by case with the provider.

### Worked example: Harbor Post

Harbor Post was the first provider we mapped, and its files are the reference for every provider after it.

| Harbor Post code | ShipmentStatus | Why |
|---|---|---|
| `PRE_ADVICE` | `Booked` | Registered, parcel not collected yet |
| `ACCEPTED` | `PickedUp` | Harbor Post has the parcel |
| `IN_TRANSIT` | `InTransit` | Moving through the network |
| `ARRIVED_AT_DEPOT` | `InTransit` | A depot scan is still transit, not delivery |
| `OUT_FOR_DELIVERY` | `OutForDelivery` | With the driver today |
| `DELIVERED` | `Delivered` | With the recipient |
| `DELIVERED_TO_PICKUP_POINT` | `Delivered` | The recipient has what they need to collect it |
| `NOT_DELIVERED` | `DeliveryFailed` | An attempt was made |
| `HELD` | `OnHold` | Waiting for someone to act |
| `RETURNED` | `Returned` | Going back to the sender |

And its services:

| Harbor Post code | Transit days | ServiceLevel |
|---|---|---|
| `HP-ECO` | 4 | `Economy` |
| `HP-PRI` | 2 | `Standard` |
| `HP-EXP` | 1 | `Express` |

Note that `HP-PRI` is called "Priority" but maps to `Standard`, because the spec says two days.

### Edge cases we've run into

- **The same code means different things for different providers.** `DELIVERED` is easy, but `HELD` at one provider can mean customs, and at another it can mean "held at the depot for collection". Read the spec's description, not just the code.
- **Codes that only occur on some routes.** Some codes only appear on certain lanes, like customs codes for shipments outside the EU. Map them anyway. They'll show up eventually.
- **Codes the spec doesn't list.** Sometimes a provider sends a code that isn't in their spec. Don't add it based on one shipment. Ask the provider for an updated spec first, and leave it unmapped until then: the Providers page keeps showing it.
- **Codes that change meaning between spec versions.** Check the spec version in the provider's doc under `docs/providers/`. If you're updating mappings for a new version, list every code whose meaning changed in the pull request.
- **Final statuses.** `Delivered` and `Returned` are final for a journey. If a provider sends a hub scan after a delivery, that's a new journey (usually a return), and the provider's spec should say how they mark it.

## Tests

- Each mapping file has a test file in `Globomantics.Shipping.Tests/Providers/`, named `<Provider>ServiceCodesTests.cs` or `<Provider>TrackingStatusesTests.cs`.
- Add one `InlineData` row per code, in the same order as the spec.
- Add a test that checks the total count matches the spec. If the counts don't match, the test should fail, even if every individual row passes.
- Add one test that sends a code in lowercase and checks it still maps. That's what catches a dictionary without the case-insensitive comparer.
- `AdapterConventionTests` picks up new adapters automatically. Don't add them there.
- Don't test `Unknown` for unmapped codes in the provider's test file. That behavior belongs to `ProviderAdapterExtensions`.

## Before you finish

- Run `dotnet test`, and make sure every test passes, not just the new ones.
- Open the Providers page in the app and check that the provider shows its services and its tracking codes.
- List any code in the spec you weren't sure how to map, and why.
- If you skipped a service on purpose (deprecated, returns, add-ons), list it with the reason.

## Pull request checklist

A reviewer checks these before approving a mapping change:

- [ ] Codes are copied exactly from the spec, in the spec's order.
- [ ] Service levels follow the transit days table, not the service names.
- [ ] No code maps to `Unknown`.
- [ ] The "needs attention" statuses (`DeliveryFailed`, `OnHold`, `Returned`) are only used where the spec's description supports them.
- [ ] Each test file has one row per code, a count test, and a lowercase test.
- [ ] Service code and tracking status changes are in separate commits.
- [ ] The pull request lists every code the author wasn't sure about.
- [ ] `dotnet test` passes in CI.

## Glossary

- **Spec:** the provider's integration document under `docs/providers/`. It's the source of truth for codes, names and transit days.
- **Service:** something we can book with a provider, like "Harbor Post Express".
- **Service level:** our own speed category for a service: `Economy`, `Standard`, `Express` or `SameDay`.
- **Tracking code:** what a provider sends in its tracking feed to say where a parcel is.
- **Shipment status:** our own status for a shipment, shown everywhere in the app.
- **Onboarding:** a provider that's registered but doesn't have both its services and its tracking codes mapped yet.
