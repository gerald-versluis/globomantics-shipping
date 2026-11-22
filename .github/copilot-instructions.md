# Globomantics Shipping

This is a .NET 10 Razor Pages app for fictional parcel shipping providers. It uses an in-memory shipment store and translates provider codes into shared service levels and shipment statuses.

Architecture:
- `Globomantics.Shipping/Pages/` contains the UI and page handlers.
- `Globomantics.Shipping/Providers/` contains provider integrations behind `IProviderAdapter`.
- `Globomantics.Shipping/Shipments/` contains domain models and in-memory state.
- `Globomantics.Shipping.Tests/` contains xUnit tests.
- `docs/providers/` contains provider specifications.

Build and run:
- `dotnet build Globomantics.Shipping.slnx`
- `dotnet test`
- `dotnet run --project Globomantics.Shipping`

Conventions:
- Use provider specs as the source of truth for provider-specific codes and mappings.
- Keep provider integration behavior behind `IProviderAdapter`; register each adapter explicitly in `Program.cs`.
- Preserve case-insensitive tracking-code lookups and the existing unknown-status behavior.
- Keep provider IDs stable and unique; do not rename or reuse one.
- Match provider service and tracking codes exactly; map the result to shared domain types.
- Follow existing Razor Pages, dependency-injection, and immutable-domain patterns.
- Keep shared state thread-safe and use `TimeProvider` for time-dependent behavior.
- Update the relevant mirrored tests and provider documentation for integration changes.
- Preserve the in-memory demo behavior unless persistence is explicitly requested.
