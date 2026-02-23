replication steps
1. download epic games giveaway history from kaggle on https://www.kaggle.com/datasets/prajwaldongre/epic-games-free-giveaway-history-20182025?resource=download
2. run game id disambiguation (need steam api key)
3. clean up epic game name to steam id manually
3a. resolve multiple candidates
3b. manually add missing matches
4. scrape steam charts data given app ids
5. run empirical test scripts