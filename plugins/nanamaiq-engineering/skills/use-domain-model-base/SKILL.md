---
name: use-domain-model-base
description: Create, modify, move, or review domain models while enforcing the shared DomainModel identity and audit contract. Use for any concrete class or record under src/Domain, mapping into domain models, or changes to common model properties such as Id, CreatedAt, CreatedBy, UpdatedAt, and UpdatedBy.
---

# Use domain model base

## Apply the base-model rule

- Make every concrete type in the Domain assembly inherit `NanamaIq.[YourProjectName].Domain.Models.DomainModel`, directly or indirectly (where `[YourProjectName]` is your actual project name in namespace).
- Keep `DomainModel` in `src/Domain/Models/DomainModel.cs`.
- Reuse its `Id`, `CreatedAt`, `CreatedBy`, `UpdatedAt`, and `UpdatedBy` properties. Never redeclare them on derived models.
- Keep business-specific properties on the derived model.
- Use records for entity models unless repository conventions establish a stronger reason not to.

The architecture test in `src/Infrastructure.Tests/Domain/DomainModelConventionTests.cs` enforces inheritance for every concrete Domain type. Do not weaken the scan or add exceptions for a domain model.

## Map external JSON

Treat external content and ENGINE metadata as separate ownership boundaries:

1. Deserialize structured output from Microsoft Foundry agents into the derived model with `JsonSerializerDefaults.Web`.
2. Add `JsonPropertyName` when the external property name is not covered by web camelCase, such as `published_date`.
3. Assign `Id` and audit properties inside the ENGINE workflow rather than trusting external values.
4. Preserve model-specific validation before persistence.

## Change common properties carefully

Add a property to `DomainModel` only when every domain model owns the same concept. When changing the base contract, update all workflows, persistence mappings, API and MCP contracts, and tests that consume it. Do not move analysis fields, source metadata, URLs, or other model-specific data into the base merely because more than one model uses it.

## Validate

1. Search `src/Domain` for concrete classes or records that bypass `DomainModel` or redeclare its properties.
2. Run `dotnet build src/[YourSolutionName].slnx` (replace `[YourSolutionName]` with your actual solution name).
3. Run `dotnet test src/[YourSolutionName].slnx --no-build` (replace `[YourSolutionName]` with your actual solution name).
4. Confirm the domain-model convention test discovers every concrete Domain type.
