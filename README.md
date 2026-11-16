# Globomantics Shipping

Globomantics Shipping is a small shipping app used in the Pluralsight course **Token Management and Memory in GitHub Copilot**. It books parcels with delivery providers and shows where each one is.

Every provider speaks its own language. Harbor Post says `OUT_FOR_DELIVERY`, Havsgren Lines says `WITH_COURIER`. Globomantics Shipping translates each provider's service and tracking codes into its own statuses through a small adapter per provider. The team is now onboarding a second provider, Havsgren Lines. Its integration spec is in `docs/providers/havsgren-lines-spec.md`.

This repository starts from the first commit of [fekberg/globomantics-shipping](https://github.com/fekberg/globomantics-shipping), the shared sample for the AI Ready: Coding Agents with GitHub Copilot path. It deliberately has no `.github/copilot-instructions.md`: the course builds one.

## What you need

- [.NET 10 SDK](https://dotnet.microsoft.com/download)
- The GitHub Copilot app, signed in, on a paid Copilot plan (Copilot Memory needs one)

## Run it

```bash
dotnet run --project Globomantics.Shipping
```

Then open http://localhost:5080. Shipments live in memory, so restarting the app resets the sample data.

## Run the tests

```bash
dotnet test
```

## How it's organized

```
Globomantics.Shipping/
  Providers/                    one folder per delivery provider
    HarborPost/                 adapter, service codes, tracking statuses
    HavsgrenLines/              being onboarded, nothing mapped yet
  Shipments/                    shipments, statuses, and the in-memory store
  Pages/                        the Razor Pages UI
Globomantics.Shipping.Tests/    tests, mirroring the app's folders
docs/providers/                 provider integration specs
.github/workflows/              CI
```

The Providers page in the app shows every code Globomantics Shipping understands, and lists any tracking codes it has received but can't translate yet.

## Disclaimer

This demo uses fictional names and people. No affiliation with or endorsement by any real company.
