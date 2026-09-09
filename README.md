# polinrider-monitor

Private watchdog: hourly scans EVERY branch of EVERY repo you can access for PolinRider, strips/heals what it finds, and re-adds the guard workflow to any repo that lost it. Findings are logged as issues here.

Setup: add a repo secret `GH_PAT` = a classic token with `repo`+`workflow` scope.
