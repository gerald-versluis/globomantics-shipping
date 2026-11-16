# Havsgren Lines integration spec (v2.1)

Havsgren Lines moves parcels between its regional terminals overnight and hands them to local couriers for the last mile. Codes are uppercase in this spec, but the tracking feed doesn't guarantee the case.

## Service codes

| Code | Name | Transit days |
|---|---|---|
| HVL-ECO | Havsgren Economy | 5 |
| HVL-STD | Havsgren Standard | 2 |
| HVL-NXT | Havsgren Next Day | 1 |
| HVL-SDY | Havsgren Same Day | 0 |

## Tracking codes

| Code | Meaning |
|---|---|
| MANIFESTED | Shipment registered, waiting for collection |
| COLLECTED | Picked up from the sender |
| AT_HUB | Scanned at a Havsgren Lines terminal |
| IN_LINEHAUL | On the overnight line between terminals |
| CUSTOMS_HOLD | Held by customs, waiting for documents |
| WITH_COURIER | With the local courier for delivery today |
| DELIVERED | Delivered to the recipient |
| DELIVERED_NEIGHBOUR | Delivered to a neighbour, a card was left |
| IN_LOCKER | Delivered to a parcel locker, pickup code sent |
| ATTEMPT_FAILED | Delivery attempted, nobody home |
| ADDRESS_QUERY | Address can't be resolved, waiting for the sender |
| RETURNED_TO_SENDER | On its way back to the sender |

## Notes

- A parcel can be `AT_HUB` several times on one journey.
- `CUSTOMS_HOLD` only occurs on shipments to and from Norway.
- Locker deliveries count as delivered: the recipient has a pickup code.
