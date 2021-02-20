## Flexible vectors proposal

Instructions and types are broken up into three tiers, with each tier viable without the ones that follow. This is done to simplify testing, but also to provide a priority queue for inclusion. The intent is to have working core functionaility, while testing additional insturctions, therefore higher tiers are tentative and should be eiliminated at some point. Additional tiers can change while testing.

- [FlexibleVectors.md](FlexibleVectors.md) - all types, instructions with straight-forward lowering
- [FlexibleVectorsSecondTier.md](FlexibleVectorsSecondTier.md) - self-contained instructions with performance concerns
- [FlexibleVectorsThirdTier.md](FlexibleVectorsThirdTier.md) - instructions that affect other instructions; without this tier vector length is a runtime constant
