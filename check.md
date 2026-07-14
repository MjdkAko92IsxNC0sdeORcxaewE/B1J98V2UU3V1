# Torus machine readiness check

## Bound target

- Source repository: `incjanta/Torus`
- Source commit: `423b57b39f940fa21023ac5ab832d66d367b119e`
- Chain: `ethereum` (`1`)
- Live address: `0xaa390a37006e22b5775a34f2147f81ebd6a63641`
- Capture block: `25530134`
- Blueprint: `blueprints/torus_fund_reward.json`
- Paid impacts: fund extraction and reward extraction only

## Hard gates

- [x] Verified source and successful ACTE build are present.
- [x] Blueprint identity, scope files, ABI surfaces, and live target match.
- [x] `setup/live_context.json` is block-pinned and protocol-bound.
- [x] `repositories.json` is exactly `[]` before DeepWiki setup.
- [x] Old project queues are excluded from the prepared machine.
- [ ] Destination repository is public under `MjdkAko92IsxNC0sdeORcxaewE`.
- [ ] Every expected workflow appears in the GitHub Actions workflows API.
- [ ] Operator has configured required repository secrets before workflow launch.

## Live-context enrichment required

- [ ] event-derived mapping keys and current nonzero interval identifiers
- [ ] decoded recent value-moving event samples beyond the bounded 10k-block window
- [ ] Uniswap V3 LP tokenId/pool token0/token1/fee/liquidity/owed-fee state
- [ ] current and boundary interval rows around the latest build and burn indices
- [ ] TORUS, TITANX, WETH, and LP asset balances reconciled against accounting liabilities
- [ ] callable actor and role matrix for every privileged and public value-moving function

DeepWiki may use the current ACTE v3 snapshot for candidate generation. Missing values must remain unknown, and any candidate depending on them stays `NEEDS_LOCAL_PROOF` until refreshed live evidence and a local or pinned-fork test bind the impact.
