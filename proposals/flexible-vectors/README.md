## Flexible vectors proposal

This proposal tracks candidate instructions as additional "tiers", while first tier is considered working "MVP" set. The intent is keep track of experimental operations which need to be be evaluated while maintining working core functionaility.

- [FlexibleVectors.md](FlexibleVectors.md) - all types, instructions with straight-forward lowering (MVP)
- [FlexibleVectorsSecondTier.md](FlexibleVectorsSecondTier.md) - self-contained instructions with performance concerns
- [FlexibleVectorsThirdTier.md](FlexibleVectorsThirdTier.md) - instructions that affect other instructions; without this tier vector length is a runtime constant
