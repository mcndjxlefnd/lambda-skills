# TSPLIB Instance Sources

Primary mirror for EUC_2D and ATT format TSPLIB instances:

https://raw.githubusercontent.com/coin-or/jorlib/master/jorlib-core/src/test/resources/tspLib/tsp/

This is the test-data directory of the coin-or/jorlib repository. It contains
standard instances in TSPLIB95 format. Tested working: att48, eil51, st70,
pr76, berlin52, lin318.

Alternative mirrors that returned 404s as of 2026-05:
- GitHub raw files under mastqe/tsplib (account deleted or repo privated)
- Several other personal-mirror repos — unreliable, no SLA

Format note: Most instances are EUC_2D (2D Euclidean). ATT instances use
ATT edge weight type (pseudo-Euclidean with different rounding). Both are
supported by HAGA's TSPLIBParser.

Instances are saved to `data/tsplib/` in the HAGA repo (tracked).
