# Projects
This section documents my data science projects, research questions, and data stories I create throughout the semesters.
---
## Project 1
Do Pre-Season Fantasy Football Projections Predict Actual Performance?
1. Problem Definition

Research question: How accurately do pre-season fantasy football point projections predict players' actual end-of-season fantasy performance, and does that accuracy differ between quarterbacks and non-quarterback skill positions (RB/WR/TE)?

Context: Every fantasy football season begins with a wave of expert and model-driven projections — single-number point forecasts that tell fantasy managers who to draft and in what order. These projections function like a market consensus: they aggregate scouting reports, statistical models, and expert judgment into one number per player before a single snap of the season is played. A large body of sports-forecasting research treats this kind of pre-event consensus as a "wisdom of the crowd" problem — the question isn't whether any one prediction is perfect, but whether aggregated pre-season judgment reliably outperforms random chance once the season's variance (injuries, role changes, unexpected breakouts) plays out (Herzog & Hertwig, 2011; Madsen, 2025). Quarterbacks are a useful comparison group here because their scoring is driven by a narrower, more stable set of inputs (mostly passing volume, which is largely a function of a stable starting role) than skill-position players, whose fantasy output is more exposed to touchdown variance, target competition, and in-season role changes (Abadzic et al., 2024).

Why it matters: This question is directly relevant to anyone who plays fantasy football, but it also speaks to a broader analytics question: how much predictive value does pre-season expert consensus actually carry once a season's real-world variance is introduced? Fantasy managers, sportsbooks setting player propositions, and fantasy media companies whose business model depends on projection accuracy all have a stake in this answer.

2. Data Description

Key variables:

Variable	Conceptual definition	Operational definition
Projected fantasy points	The pre-season expectation of a player's full-season fantasy value	The projection field in the projections dataset — a single point total per player, PPR scoring, forecast before the 2020 season began
Actual fantasy points	The player's realized fantasy output over the season	fantasy_points_ppr, summed across all 2020 regular-season games, from nflverse
Position	The player's on-field role, used to group/stratify the analysis	pos (QB vs. RB/WR/TE), since scoring scale and volatility differ substantially by position
Projection error	How far off the pre-season forecast was, and in which direction	actual − projected; positive values indicate the player outperformed their projection, negative values indicate underperformance

Data sources:

Actual performance data: nflverse (nflreadpy.load_player_stats()), an open-source, community-maintained NFL statistics dataset. Pulled at the season summary level (summary_level="reg") for the 2020 regular season, which returns one row per player with fantasy points already aggregated across all games played.
Projected performance data: projections.json, pulled from the Fantasy Football Data Pros API (/api/projections). 200 player records, each containing player_name, pos, projection, and team.

Unit of analysis: One row = one NFL player's 2020 regular season, after the two sources are merged on player name.

Size and assumptions: The projections file contains 200 players; after merging with actual 2020 season stats and removing players whose name strings didn't exactly match between the two sources (see Data Cleaning), the analysis dataset is smaller than 200. Because projections.json predates the season, it necessarily reflects each source's pre-season assumptions about depth-chart roles, which the 2020 season itself sometimes upended (injuries, COVID-19-related roster disruptions, and in-season role changes were all unusually elevated in 2020 specifically).

3. Data Cleaning and Preparation

Cleaning happened in a few concrete stages, each with a decision behind it:

Loading and inspecting the projections JSON. Rather than assume its structure, I first printed the raw JSON to confirm its shape (player_name, pos, projection, team) before writing any merge logic. This avoided guessing at column names that didn't actually exist.
Pulling actual stats at the season level, not weekly. nflreadpy.load_player_stats() defaults to one row per player per week. I explicitly set summary_level="reg" so fantasy_points_ppr would already be a season total, matching the season-total shape of the projections data, rather than manually summing weekly rows myself.
Resolving a duplicate-column collision. The actual-stats data includes two name fields — player_name (abbreviated, e.g. "D. Henry") and player_display_name (full name). Since the projections data also uses player_name, merging without accounting for this caused pandas to silently rename both copies (player_name_x/player_name_y), which is a subtle failure mode worth documenting rather than hiding. I merged explicitly on player_display_name (actual data) against player_name (projections data) to sidestep the collision.
Choosing an inner join over an outer join. I merged with how="inner", which keeps only players present in both sources. This was a deliberate trade-off: an inner join guarantees every row in the analysis has both a real projection and a real outcome, but it silently drops any player whose name was formatted differently between the two sources (e.g., suffixes like "Jr."/"II", or punctuation differences). I checked the size of this drop explicitly by comparing the merged player set back against the full projections list, rather than assuming the join was clean.
Deriving the projection-error variable. difference = fantasy_points_ppr − projection was computed post-merge as the core analytical variable — this wasn't in either raw source and had to be constructed.
Splitting by position. QB and non-QB (RB/WR/TE) were analyzed as separate groups rather than pooled, since QB point totals run on a different scale than skill-position totals; pooling them would have made a single chart misleading (QBs would visually dominate).
4. Visualizations and Insights

Chart 1 — Top 25 non-QB players (RB/WR/TE), projected vs. actual 2020 PPR points

![Top 25 non-QB players: projected vs actual fantasy points, 2020]({{ site.baseurl }}/project1-chart1-nonqb.png)

Each row is one player; the blue dot marks their pre-season projection, the orange dot their actual season total. Almost every player in this top 25 finished above their projected fantasy output, the only exception was Mike Evans. Most players in the lower half of this group (ranked 15th–25th) landed within about 50 points of their projection, while several near the top posted breakout seasons roughly 100+ points above projection (Alvin Kamara and Davante Adams stand out).

Chart 2 — Top 25 QBs, projected vs. actual 2020 PPR points

![Top 25 QBs: projected vs actual fantasy points, 2020]({{ site.baseurl }}/project1-chart2-qb.png)

Same chart structure, restricted to quarterbacks. Unlike the skill-position chart, QB projections were much less consistent in direction: in the lower half of this list (ranked 15th–25th), almost every QB finished under their projection, while in the top 10 almost all finished above projection, the one exception being Lamar Jackson, who had been projected as the QB1 for the season.

5. Storytelling and Narrative

Looking at both charts together, the skill-position group was noticeably more accurate to project than the QB group: almost every non-QB player in the top 25 over-achieved their projection, while QBs showed a much wider and less consistent spread, with a cluster of lower-ranked QBs underperforming and a cluster of top-ranked QBs overperforming. A few things are worth keeping in mind when reading this pattern: some players who missed the top 25 entirely may have suffered season-ending injuries, which would depress their actual total independent of how good their underlying projection was. A plausible explanation for the bigger QB spread is that quarterback scoring depends on a wider set of inputs than skill positions do — QBs are scored on passing yards and touchdowns plus turnovers like interceptions and fumbles, while skill positions are scored mainly on catches/rushes, yards, and touchdowns. More moving parts in the scoring formula means more ways for a projection to miss. Coming back to the research question: the skill-position group's projections held up reasonably well against actual results, while QB projections were noticeably less reliable, with bigger gaps between projected and actual and more players landing under their projection near the middle of the group.

6. Limitations, Ethics, and Reflection
Single-season scope. This analysis covers only the 2020 season, which was an unusual year (COVID-19 roster disruptions, opt-outs, and schedule irregularities). Findings here may not generalize to a "normal" season.
Name-matching data loss. The inner join on player name drops any player whose name string didn't match exactly between the two sources. This isn't random — it may disproportionately affect players with suffixes, hyphenated names, or less common name formats, which is a form of selection bias worth naming rather than ignoring.
Unknown projection methodology. The projections.json source doesn't document how its projections were generated (which model, which assumptions, how recent before the season). Without that, it's impossible to know whether a "miss" reflects a bad model, incomplete pre-season information (e.g., an injury that happened after projections were finalized), or normal variance.
PPR-only scoring. Both the projections and actual data are compared under PPR scoring; a league using standard or half-PPR scoring could show different results, since receptions are not evenly distributed across positions (RBs/WRs/TEs benefit far more from PPR than QBs).
What I'd explore next: Running this across multiple seasons (not just 2020) to see whether the QB vs. skill-position predictability gap holds up consistently, and comparing multiple independent projection sources against each other rather than just one.
7. Code and Transparency

Code repository: file:///Users/loganwilson9/Library/CloudStorage/OneDrive-UniversityofNorthCarolinaatCharlotte/DTSC%202/project_1.html

Data sources:

nflverse / nflreadpy: github.com/nflverse/nflreadpy
Projections dataset: Fantasy Football Data Pros API

AI usage disclosure: Generative AI (Claude, Sonnet 4.5, accessed via Anthropic's Claude in Cowork mode, September 2026) was used throughout this project's development in the following ways: (1) troubleshooting Python errors during data loading and merging (e.g., diagnosing a ModuleNotFoundError caused by a Python 3.13/pandas version conflict, and a KeyError caused by a duplicate player_name column after merging); (2) helping guide the writing and iteration of pandas merge logic and seaborn/matplotlib visualization code based on the actual column names and data structure of the files used; (3) assisting with the project outline and drafting portions of this write-up, which were reviewed, edited, and personalized by me. No AI tool was used to generate or fabricate data, results, or citations — all data comes from the cited APIs, and all cited sources were independently verified to exist.

Key Academic References

Abadzic, A., Cheun, J., & Patel, M. (2024). Data analysis on predicting the top 12 fantasy football players by position. SMU Data Science Review, 8(2), Article 7. https://scholar.smu.edu/datasciencereview/vol8/iss2/7

Herzog, S. M., & Hertwig, R. (2011). The wisdom of ignorant crowds: Predicting sport outcomes by mere recognition. Judgment and Decision Making, 6(1), 58–72. https://doi.org/10.1017/S1930297500002096

Madsen, J. K. (2025). Goal-line oracles: Exploring accuracy of wisdom of the crowd for football predictions. PLOS ONE, 20(1), e0312487. https://doi.org/10.1371/journal.pone.0312487

Butler, D., Butler, R., & Eakins, J. (2021). Expert performance and crowd wisdom: Evidence from English Premier League predictions. European Journal of Operational Research, 288(1), 170–182. https://doi.org/10.1016/j.ejor.2020.05.034




## Project 2
Can Last Season's Stats Predict This Season's All-NBA Teams?

*Logan Wilson · UNC Charlotte · Machine-learning classification project*

#### At a glance

| | |
|---|---|
| **Question** | Using only a player's stats from the previous season, can a model pick the 15 players who will make the All-NBA 1st, 2nd and 3rd Teams? |
| **Data** | 8,512 player-seasons (2000–2024 stats → 2001–2025 teams) from Basketball-Reference |
| **Best model** | Regularized logistic regression, ranked into three teams of five |
| **Result** | **8.7 of 15** correct players per season on six held-out seasons (2020–2025), vs. 8.0 for the best baseline |
| **Biggest lesson** | The model knows who is *good*. It can't know who will *stay healthy*, and under the 65-game rule, health decides eligibility. |

---

### 1. Problem Definition

**Prediction problem.** At the end of every NBA season, a panel of 100 media members votes on the All-NBA Teams: five players on the 1st Team, five on the 2nd Team and five on the 3rd Team. My question is whether those 15 players can be predicted **before the season starts**, using only what happened the season before.

**Target variable.** `tier_next`: a player's All-NBA result in season *t*, coded **0 = none, 1 = 3rd Team, 2 = 2nd Team, 3 = 1st Team**.

**Classification or regression?** Classification. The outcome is one of four ordered categories, not a number on a continuous scale. It is also a heavily **imbalanced** problem, because only about 4% of player-seasons lead to a selection. I handle this by having each model produce a *score* for every player, then ranking players within a season: the top 5 become the predicted 1st Team, the next 5 the 2nd Team and the next 5 the 3rd Team. That guarantees exactly 15 picks every year, like the real vote.

**Who benefits?**
* **Teams and front offices.** All-NBA selections carry real money. Under the collective bargaining agreement, making All-NBA can make a player eligible for a "supermax" (Designated Veteran) extension (Adams, 2024). Teams planning cap space two years out want to know how likely that is.
* **Players and agents,** for the same contract reasons.
* **Media, fans and sportsbooks** that set preseason award odds.

**Why it's worth investigating.** Most award-prediction projects use *same-season* stats, which really just reverse-engineers the vote after the fact. Using the *previous* season makes this a true forecast. It's harder, but it's the version a front office could actually use in October.

---

### 2. Background and Context

**How the teams are chosen.** All-NBA voting is done by media members, not a formula, so any model is really predicting **human judgment**. Two rule changes from the 2023 collective bargaining agreement matter for recent seasons (Adams, 2024):
1. **Positionless teams (starting 2023-24).** Before that, each team needed 2 guards, 2 forwards and 1 center. Now voters can pick any five players.
2. **The 65-game rule.** A player must appear in at least 65 games (with a 20-minute minimum in most of them) to be *eligible* for All-NBA, with narrow exceptions for season-ending injuries.

**What earlier research says.**
* **Voters reward production, and they used to think in positions.** Rocha da Silva and Canas Rodrigues (2022) used LASSO logistic regression on Basketball-Reference per-game stats and found that voters valued different stats for different positions, for example points for guards and forwards but not for centers. They argued position labels were distorting the selections, the same concern that led the NBA to go positionless. This supports using **logistic regression** and **box-score production** features.
* **Reputation matters less for All-NBA than for All-Star.** McMahan and Shor (2024) found that past All-Star status snowballs into future All-Star selections, but that this "cumulative advantage" disappears for All-NBA, which voters treat as a merit award. For my project, past All-NBA status should help mainly because it reflects **persistent talent**, not because voters favor famous names.
* **Team success and individual stats both drive award voting.** In MVP voting, Coleman et al. (2008) found that individual production and team success, especially large improvements in team wins, explained who got votes. That's why I include **team win %**.
* **Award prediction is a rare-event problem.** Albert et al. (2022) predicted All-Star selections with random forests, AdaBoost and neural networks and stressed the severe class imbalance. Raw accuracy looked high even when the model missed many actual stars. That is why I don't use accuracy as my main metric and why I compare a **random forest** with logistic models.

---

### 3. Data Description

**Source.** All player and team stats and past All-NBA teams come from **Basketball-Reference.com** (Sports Reference, n.d.). I downloaded them through Sumitro Datta's public GitHub mirror of Basketball-Reference's season tables (Datta, 2026). The notebook downloads the tables directly from that mirror, pinned to its April 2026 version, so anyone can rebuild the dataset. The mirror was last updated in April 2026, before the 2026 teams were announced, so I entered the **2025-26 All-NBA Teams** by hand from the official NBA.com announcement (NBA.com Staff, 2026). I use them only for a final out-of-sample test.

| Table | One row = |
|---|---|
| Player per-game stats | one player's season (points, rebounds, assists, games, age, …) |
| Player advanced stats | one player's season (PER, TS%, usage, Win Shares, BPM, VORP, …) |
| Team summaries | one team's season (wins, losses) |
| End-of-season teams | one All-NBA / All-Defense / All-Rookie selection |
| All-Star selections | one All-Star selection |

**Unit of analysis.** One row = **one player's season *t − 1***, paired with his All-NBA result in season *t*. For example, Nikola Jokić's 2023-24 stats are paired with his 2024-25 result (1st Team).

**Size.** 8,512 labeled player-seasons covering **25 target seasons (2001–2025)** and **375 All-NBA selections** (125 per team). There are two more unlabeled batches: 2024-25 stats (to predict 2026) and 2025-26 stats (to forecast 2027).

**Target.** `tier_next` (0–3), described above.

**Available features.** About 60 raw columns: per-game box score stats, shooting splits, advanced impact metrics (PER, WS, WS/48, BPM, OBPM, DBPM, VORP), usage, age, games, minutes, team record, and award history.

**Assumptions and restrictions.**
* **Eligibility filter.** I keep a player-season if he played **at least 500 minutes** *or* made All-NBA in the last three years. This removes deep-bench players with no realistic chance and tiny samples that produce broken percentages. It still keeps injured stars (e.g., Stephen Curry after playing only 5 games in 2019-20). **All 375 selections from 2001–2025 are kept.** No rookie made All-NBA in that span, so every selected player has a previous season.
* **Retirements count as "not selected."** If a player has no season *t* (retired, injured all year, left the league), his target is 0. A preseason forecast couldn't know that either.
* **Missed seasons are invisible.** Stats only describe games that were played. A player coming off an injury looks worse than he is.

---

### 4. Data Understanding and Exploration

**Summary statistics.** Future All-NBA players look very different the season before:

| Previous-season average | Not selected next year | Selected next year |
|---|---|---|
| Points per game | 9.9 | **22.2** |
| True shooting % | .54 | **.57** |
| PER | 13.9 | **23.1** |
| Win Shares | 3.3 | **9.8** |
| Box Plus/Minus | −0.7 | **+5.1** |
| VORP | 0.7 | **4.5** |
| Team win % | .49 | **.59** |
| Games played | 64.6 | **69.9** |

**The target is extremely imbalanced.**

![Bar chart of next-season All-NBA outcomes: 95.6% none and 1.5% for each team]({{ site.baseurl }}/project2-fig1-target-distribution.png)

About **95.6%** of player-seasons lead to no selection. A lazy model that predicts "nobody makes it" would be **95.8% accurate on the test seasons and find zero All-NBA players.** So instead of accuracy I use metrics built for finding rare cases (Section 7).

**Last year's All-NBA players usually come back, but not always.**

![Stacked bars showing where each All-NBA tier lands the next season]({{ site.baseurl }}/project2-fig2-repeat-rates.png)

**83%** of 1st-Teamers make an All-NBA team again the next season (59% on the 1st Team again). Only **39%** of 3rd-Teamers do. Among players who were *not* on an All-NBA team, only **1.9%** make one the next year. So past status is a strong signal, and it gives me a tough baseline to beat: "just pick last year's teams."

**Future selections stand out statistically.**

![Box plots of VORP, BPM, PPG and team win % for selected vs not-selected players]({{ site.baseurl }}/project2-fig3-feature-separation.png)

VORP and Box Plus/Minus separate the groups most clearly: the middle half of future All-NBA players sits above almost every non-selected player. Points per game separates them with more overlap. Team win % shifts upward but overlaps a lot.

**Outliers.** The lowest dots in the "Selected" group are mostly stars coming off injury-shortened seasons: Curry (2020), Kawhi Leonard (2018), Paul George (2015). They are real observations, not errors, so I kept them. They're exactly the cases where previous-season stats should struggle.

**Several features measure nearly the same thing.**

![Correlation heatmap of the candidate features]({{ site.baseurl }}/project2-fig4-correlation-heatmap.png)

VORP, Win Shares, BPM and PER are all correlated above 0.8 with each other. Minutes and points are correlated at 0.85. Having many near-duplicate features makes logistic regression coefficients unstable and hard to read, so the correlation map directly shaped the feature selection below.

**How the exploration shaped my decisions:**
1. Imbalance → rank-based evaluation (hits out of 15) and average precision instead of accuracy.
2. Strong repeat rates → include past All-NBA status as a feature, and use "run it back" as a baseline.
3. Redundant impact metrics → keep only two (VORP and BPM).
4. Injury outliers → keep them, and study them in the error analysis.

---

### 5. Data Preparation and Feature Selection

**Cleaning steps**
* **Traded players.** A player traded mid-season has a row for each team *plus* a combined row. Keeping all of them would count him two or three times. I kept only the combined row and saved his "main team" (the team he played the most games for) to attach a team win %.
* **Duplicates.** After that fix there are zero duplicate player-seasons, which the notebook checks with an `assert`.
* **Missing values.** The only gaps in the raw data were shooting percentages and PER for players with almost no minutes. The 500-minute filter removes all of them, leaving **no missing values** in the final features. The notebook still includes a season-median imputation step in case future data has gaps.
* **Target creation.** Each player-season is joined to the same player's All-NBA result one year later.

**Feature engineering**
* **Era-adjusted box score stats.** Scoring has risen a lot. Teams averaged about 95 points a game in the early 2000s and 110–116 in the 2020s, so 25 PPG meant more in 2004 than in 2024. I converted points, rebounds and assists into **z-scores within each season**, so each player is compared with his peers that year.
* **`vorp_change`:** the change in VORP from two seasons ago to last season (improving or declining?).
* **`all_nba_last3`:** the number of All-NBA selections in the last three seasons.

**Final 14 features:** age (next season), games played, points z, rebounds z, assists z, true shooting %, usage %, BPM, VORP, VORP change, team win %, this season's All-NBA tier, All-NBA picks in the last 3 years, and All-Star this season.

**Excluded, and why:** Win Shares and PER (near-duplicates of VORP/BPM); minutes per game (a near-duplicate of points, and it reflects role more than quality); steals, blocks and WS/48 (little extra information once BPM is included); raw per-game stats (replaced by era-adjusted z-scores). To check these choices, I ran the same cross-validation with **all 23 candidate features**. It did not do better (logistic regression average precision 0.650 vs. 0.657 with the 14), so the simpler set won.

**Encoding and scaling.** No text categories remained. All-NBA tier is already an ordered number (0–3), and All-Star is 0/1. For the logistic models, features are **standardized** inside a scikit-learn pipeline so the scaler is fit only on training data. The random forest doesn't need scaling.

**Training and evaluation split: by time, not at random.** A random split would let the model learn from 2023 and then "predict" 2012, which peeks at the future. It would also put the same player's neighboring seasons in both halves.

| Split | Target seasons | Rows | Selections |
|---|---|---|---|
| **Train** | 2001–2019 | 6,345 | 285 |
| **Test** (held out until the end) | 2020–2025 | 2,167 | 90 |
| **Live check** | 2026 | – | 15 |

I picked 2020–2025 as the test window on purpose. It includes the COVID-shortened season and the new positionless, 65-game-rule era, the conditions a model would face today.

**Preventing data leakage**
1. Every feature comes from season *t − 1* or earlier. Next-season games played are used **only** in the error analysis, never as an input.
2. Scalers are fit inside pipelines on training data only.
3. The within-season z-scores only use other players from the same *past* season.
4. Tuning used cross-validation inside the training years. The test seasons were scored **once**, after every choice was locked in.

---

### 6. Baseline and Model Development

**Baselines**
1. **Majority class:** predict "no All-NBA" for everyone. It shows why accuracy is misleading.
2. **Scoring leaders:** pick last season's top 15 scorers in points per game. It's what a casual fan might do.
3. **Run it back:** pick last year's All-NBA players in the same order (1st Team first), and fill any open spots with top scorers. This is the strongest baseline because it uses what voters already decided.

**Models**

| Model | Why it's appropriate |
|---|---|
| **Logistic regression** (any All-NBA team vs. none) | The standard model for a yes/no outcome. It gives a probability for every player and interpretable coefficients, and prior research used it for this exact problem. |
| **Ordinal logistic regression** (None < 3rd < 2nd < 1st) | The model from my original proposal. It uses the *order* of the teams directly, and each player's score is his expected tier. |
| **Random forest** (any team vs. none) | Can capture non-linear patterns and interactions (for example, scoring mattering more on winning teams) without scaling, and it's often strong on tabular data. |

**Hyperparameter tuning.** I used **forward-chaining cross-validation** within the training years. Each fold trains on every season *before* a validation block and tests on that block (2011–12, 2013–14, 2015–16, 2017–19), which mimics forecasting the future. I tuned on **average precision**.
* Logistic regression: regularization strength C ∈ {0.01, 0.1, 1, 10} × class weighting {none, balanced}. **Best: C = 0.01, balanced** (strong regularization, which makes sense with correlated features).
* Random forest: 18 combinations of tree depth {4, 8, unlimited}, minimum leaf size {1, 5, 20} and features per split {√p, 50%}. **Best: depth 8, leaf size 5, √p features.**

**Fair comparison.** All models used the same 14 features, the same training seasons, the same tuning folds, the same ranking-into-teams rule and the same untouched test seasons.

---

### 7. Model Evaluation and Selection

**Metrics, and what they measure**
* **Hits per 15:** of the model's 15 picks, how many actually made *any* All-NBA team, averaged over the six test seasons. Because both lists have exactly 15 names, this is both precision and recall. It's the easiest metric to explain.
* **Exact per 15:** right player *and* right team (1st, 2nd or 3rd).
* **Average precision (AP):** how well the whole ranking puts real selections near the top. A random ranking scores about 0.04 (the positive rate), and a perfect one scores 1.0. It's the standard metric for rare events.
* **ROC AUC:** the chance that a random future All-NBA player is ranked above a random non-selected player. I show it for context, but it looks flattering on imbalanced data.
* **4-class accuracy:** shown only to prove it's useless here.

**Test results (2020–2025, six seasons)**

| Model | Hits / 15 | Exact / 15 | Avg. precision | ROC AUC | Accuracy |
|---|---|---|---|---|---|
| Majority class | 0.0 | 0.0 | – | 0.52 | **95.8%** |
| Baseline: scoring leaders | 7.8 | 3.2 | 0.461 | 0.954 | 94.7% |
| Baseline: run it back | 8.0 | **5.7** | 0.542 | 0.965 | 95.5% |
| **Logistic regression** | **8.7** | 4.3 | **0.609** | **0.972** | 95.3% |
| Ordinal logistic | 8.5 | 4.7 | 0.594 | 0.970 | 95.3% |
| Random forest | **8.7** | 5.3 | 0.583 | 0.970 | 95.6% |

Look at the accuracy column: the model that finds **nobody** has the highest accuracy. That is why accuracy isn't used to pick a winner.

![Grouped bars of hits and exact hits per model]({{ site.baseurl }}/project2-fig5-model-comparison.png)

![Line chart of correct picks per test season]({{ site.baseurl }}/project2-fig6-hits-by-season.png)

**Final model: logistic regression.** The evidence:
1. It had the **highest average precision** on the test seasons (0.609) and **tied for the most hits** (8.7 of 15).
2. It was also best in **cross-validation** inside the training years (AP 0.657, 9.4 hits vs. 0.624 and 8.7 for the random forest), so the test result isn't a one-off.
3. It **matched or beat the "run it back" baseline in all six test seasons** (the scoring-leaders baseline edged it in 2022 and 2023).
4. Its coefficients can be explained to someone with no machine-learning background.

**Why the models perform so similarly, and why the flexible one didn't win.** With only ~285 positive examples in training and a signal that is mostly "good players tend to stay good," a weighted sum of stats captures nearly everything. The random forest's extra flexibility mainly fits noise, which shows up as lower average precision. It picked the *same number* of correct players as logistic regression in every single season. The ordinal model didn't help either: its extra tier structure doesn't add much, because the ranking step already turns any score into three teams.

**Tradeoffs.** No model beats "run it back" on **exact team placement** (5.7 of 15). If you already have the right names, last year's order is a very good guess at who goes on the 1st Team. The models are better at the harder, more valuable task: **finding who belongs among the 15**, including players who weren't on last year's teams. A practical tool could combine the two: use the model to pick the 15 names and past status to order them.

---

### 8. Model Interpretation and Insights

**What the model learned.**

![Logistic regression coefficients]({{ site.baseurl }}/project2-fig7-lr-coefficients.png)

Holding the other stats fixed, a player's chances go **up** with rebounding, VORP, scoring, assists, usage, BPM, recent All-NBA picks and team success. They go **down** with **age**, which has the largest negative coefficient: one standard deviation older (about 4 years) cuts the odds by roughly 40%. The rebounding effect makes sense in the Jokić/Giannis/Embiid era, when big men who rebound *and* create have dominated the voting.

**A caution about reading coefficients.** "All-NBA tier this season" has a coefficient near zero. That doesn't mean last year's selection is irrelevant. Its information is already carried by the correlated "All-NBA picks in last 3 years" (r = 0.82) and VORP, so the credit is split. To measure what each *kind* of information is worth, I shuffled related features **together** on the test data and measured how much average precision dropped (grouped permutation importance):

![Grouped permutation importance for both models]({{ site.baseurl }}/project2-fig7b-grouped-importance.png)

**Box-score production, overall impact and status carry both models.** Everything else (age, team win %, efficiency, games played, trend) matters very little. The logistic model leans hardest on production, while the random forest spreads its weight evenly across production, impact and status.

**Where it works and where it fails: confusion matrix.**

![Confusion matrix of predicted vs actual team]({{ site.baseurl }}/project2-fig8-confusion-matrix.png)

* **1st Team is easy:** 19 of 30 actual 1st-Teamers were predicted 1st Team, and only 4 were missed entirely.
* **2nd and 3rd Team are hard:** 34 of those 60 players were missed completely. The bottom of the All-NBA list changes a lot from year to year.

**Error analysis: who does it miss, and why?** Over the six test seasons, the model made 38 wrong picks and missed 38 real selections.
* **Wrong picks are mostly about health.** **25 of the 38** players it picked who *didn't* make it played **fewer than 65 games the next season** (or none at all), which makes them ineligible or at least hurts their case. Examples: Stephen Curry in 2020 (5 games), Kevin Durant in 2020 (Achilles, 0 games), Joel Embiid in 2025 (19 games). The median wrong pick played 51 games the next season, compared with 68 for correct picks. The model identified *talent* correctly. It can't see injuries coming.
* **Misses are mostly breakouts.** **24 of the 38** selections it missed were players with **no All-NBA pick in the previous three years**: first-timers such as Jayson Tatum (2020), Ja Morant (2022) and Luka Dončić (2020). Their big leap happened *during* the season being predicted.

**Example predictions: 2025 (model trained only on 2001–2019).** Nine of the top 15 were correct: Jokić, Giannis, Shai Gilgeous-Alexander, Tatum, LeBron James, Tyrese Haliburton, Donovan Mitchell, Jalen Brunson and Anthony Edwards. Of the six misses, Embiid (19 games), Luka (50), Anthony Davis (51) and Wembanyama (46) all missed major time.

**True out-of-sample test: the 2026 teams.** I refit the final model on all 25 labeled seasons and asked it to predict the 2025-26 teams from 2024-25 stats. These labels weren't used anywhere else.

| Pred. | Player | Predicted | Actual | Games in 2025-26 |
|---|---|---|---|---|
| 1 | Nikola Jokić | 1st | **1st** ✅ | 65 |
| 2 | Giannis Antetokounmpo | 1st | – | 36 |
| 3 | Shai Gilgeous-Alexander | 1st | **1st** ✅ | 68 |
| 4 | Luka Dončić | 1st | **1st** ✅ | 64 |
| 5 | Jayson Tatum | 1st | – | 16 |
| 6 | Domantas Sabonis | 2nd | – | 19 |
| 7 | LeBron James | 2nd | – | 60 |
| 8 | Victor Wembanyama | 2nd | 1st ✅ | 64 |
| 9 | Anthony Davis | 2nd | – | 20 |
| 10 | Anthony Edwards | 2nd | – | 61 |
| 11 | Cade Cunningham | 3rd | 1st ✅ | 64 |
| 12 | Karl-Anthony Towns | 3rd | – | 75 |
| 13 | Tyrese Haliburton | 3rd | – | 0 |
| 14 | Alperen Şengün | 3rd | – | 72 |
| 15 | Jalen Brunson | 3rd | 2nd ✅ | 74 |

**6 of 15** were correct. That's the model's worst season, and the simple scoring-leaders baseline did slightly better (7). The reason is clear: **7 of the 9 wrong picks played 61 or fewer games**, several because of major injuries (Tatum's Achilles, Haliburton's Achilles, Davis and Sabonis). Meanwhile, the model ranked 2026 breakouts such as Jalen Johnson, Chet Holmgren and Jalen Duren far down its list. The 2026 teams were an unusually young, new group, exactly the case previous-season stats handle worst.

#### Predicted vs. actual for every player who made All-NBA

The charts above look at the model's *picks*. This section flips it around: for each of the **105 players who actually made All-NBA** from 2020 to 2026, where did the model rank him? A predicted rank of 1–5 means the model had him on the 1st Team, 6–10 the 2nd Team, 11–15 the 3rd Team, and anything above 15 means the model left him off.

![Dot chart of the model's predicted rank for every actual All-NBA player, 2020–2026]({{ site.baseurl }}/project2-fig9-actual-players-rank.png)

**31 of the 35** actual 1st-Teamers (dark dots) were among the model's top 15. It has little trouble finding the league's best few players. The misses are mostly **2nd- and 3rd-Teamers** ranked 20th to 60th, and the labeled names show the two patterns from the error analysis. Some are **breakouts** the previous season couldn't show: Tatum in 2020 (ranked 64th), Morant in 2022, Cunningham in 2025. Others are **older veterans** the age effect pushed down, like Chris Paul in 2020 and 2021.

| Season | Exact team | Right player, different team | Missed | Found (of 15) |
|---|---|---|---|---|
| 2020 | 5 | 4 | 6 | **9** |
| 2021 | 7 | 2 | 6 | **9** |
| 2022 | 3 | 5 | 7 | **8** |
| 2023 | 4 | 3 | 8 | **7** |
| 2024 | 4 | 6 | 5 | **10** |
| 2025 | 3 | 6 | 6 | **9** |
| 2026 (live check) | 3 | 3 | 9 | **6** |
| **Total** | **29** | **29** | **47** | **58 of 105** |

![Lollipop chart of predicted rank for each actual 2026 All-NBA player]({{ site.baseurl }}/project2-fig10-2026-actual-vs-predicted.png)

In the 2026 live check, the model ranked four of the five real 1st-Teamers in its top 8 and Cunningham 11th. But the nine 2nd- and 3rd-Teamers it missed were ranked 18th to 58th. They were young players who took a big jump in 2025-26 (Jalen Johnson, Chet Holmgren, Jalen Duren), players coming off injury-shortened seasons (Tyrese Maxey played 52 games in 2024-25 and Kawhi Leonard 37), and a 37-year-old Kevin Durant, whom the age effect pushed down.

**Full player-by-player results** (click a season to expand):

<details markdown="1">
<summary><b>2020 All-NBA Teams</b>: predicted vs. actual</summary>

| Player | Actual team | Predicted team | Predicted rank | Result |
|---|---|---|---|---|
| Giannis Antetokounmpo | 1st Team | 1st Team | 1 | ✅ Exact team |
| James Harden | 1st Team | 1st Team | 2 | ✅ Exact team |
| Anthony Davis | 1st Team | 1st Team | 4 | ✅ Exact team |
| LeBron James | 1st Team | 2nd Team | 7 | 🟡 Right player, different team |
| Luka Dončić | 1st Team | nan | 19 | ❌ Missed |
| Nikola Jokić | 2nd Team | 2nd Team | 6 | ✅ Exact team |
| Damian Lillard | 2nd Team | 3rd Team | 12 | 🟡 Right player, different team |
| Kawhi Leonard | 2nd Team | 3rd Team | 13 | 🟡 Right player, different team |
| Pascal Siakam | 2nd Team | nan | 41 | ❌ Missed |
| Chris Paul | 2nd Team | nan | 44 | ❌ Missed |
| Russell Westbrook | 3rd Team | 1st Team | 3 | 🟡 Right player, different team |
| Rudy Gobert | 3rd Team | 3rd Team | 15 | ✅ Exact team |
| Ben Simmons | 3rd Team | nan | 17 | ❌ Missed |
| Jimmy Butler | 3rd Team | nan | 28 | ❌ Missed |
| Jayson Tatum | 3rd Team | nan | 64 | ❌ Missed |

</details>
<details markdown="1">
<summary><b>2021 All-NBA Teams</b>: predicted vs. actual</summary>

| Player | Actual team | Predicted team | Predicted rank | Result |
|---|---|---|---|---|
| Giannis Antetokounmpo | 1st Team | 1st Team | 1 | ✅ Exact team |
| Luka Dončić | 1st Team | 1st Team | 3 | ✅ Exact team |
| Nikola Jokić | 1st Team | 1st Team | 5 | ✅ Exact team |
| Kawhi Leonard | 1st Team | 2nd Team | 8 | 🟡 Right player, different team |
| Stephen Curry | 1st Team | nan | 24 | ❌ Missed |
| LeBron James | 2nd Team | 1st Team | 4 | 🟡 Right player, different team |
| Damian Lillard | 2nd Team | 2nd Team | 7 | ✅ Exact team |
| Joel Embiid | 2nd Team | 2nd Team | 9 | ✅ Exact team |
| Chris Paul | 2nd Team | nan | 38 | ❌ Missed |
| Julius Randle | 2nd Team | nan | 45 | ❌ Missed |
| Rudy Gobert | 3rd Team | 3rd Team | 12 | ✅ Exact team |
| Kyrie Irving | 3rd Team | 3rd Team | 14 | ✅ Exact team |
| Jimmy Butler | 3rd Team | nan | 17 | ❌ Missed |
| Paul George | 3rd Team | nan | 19 | ❌ Missed |
| Bradley Beal | 3rd Team | nan | 26 | ❌ Missed |

</details>
<details markdown="1">
<summary><b>2022 All-NBA Teams</b>: predicted vs. actual</summary>

| Player | Actual team | Predicted team | Predicted rank | Result |
|---|---|---|---|---|
| Nikola Jokić | 1st Team | 1st Team | 1 | ✅ Exact team |
| Giannis Antetokounmpo | 1st Team | 1st Team | 2 | ✅ Exact team |
| Luka Dončić | 1st Team | 1st Team | 3 | ✅ Exact team |
| Jayson Tatum | 1st Team | 3rd Team | 15 | 🟡 Right player, different team |
| Devin Booker | 1st Team | nan | 34 | ❌ Missed |
| Joel Embiid | 2nd Team | 1st Team | 5 | 🟡 Right player, different team |
| Kevin Durant | 2nd Team | 3rd Team | 11 | 🟡 Right player, different team |
| Stephen Curry | 2nd Team | 3rd Team | 12 | 🟡 Right player, different team |
| Ja Morant | 2nd Team | nan | 47 | ❌ Missed |
| DeMar DeRozan | 2nd Team | nan | 53 | ❌ Missed |
| LeBron James | 3rd Team | 2nd Team | 6 | 🟡 Right player, different team |
| Karl-Anthony Towns | 3rd Team | nan | 19 | ❌ Missed |
| Trae Young | 3rd Team | nan | 23 | ❌ Missed |
| Chris Paul | 3rd Team | nan | 29 | ❌ Missed |
| Pascal Siakam | 3rd Team | nan | 39 | ❌ Missed |

</details>
<details markdown="1">
<summary><b>2023 All-NBA Teams</b>: predicted vs. actual</summary>

| Player | Actual team | Predicted team | Predicted rank | Result |
|---|---|---|---|---|
| Giannis Antetokounmpo | 1st Team | 1st Team | 2 | ✅ Exact team |
| Luka Dončić | 1st Team | 1st Team | 3 | ✅ Exact team |
| Joel Embiid | 1st Team | 1st Team | 4 | ✅ Exact team |
| Jayson Tatum | 1st Team | 1st Team | 5 | ✅ Exact team |
| Shai Gilgeous-Alexander | 1st Team | nan | 33 | ❌ Missed |
| Nikola Jokić | 2nd Team | 1st Team | 1 | 🟡 Right player, different team |
| Stephen Curry | 2nd Team | 3rd Team | 12 | 🟡 Right player, different team |
| Jimmy Butler | 2nd Team | nan | 16 | ❌ Missed |
| Donovan Mitchell | 2nd Team | nan | 23 | ❌ Missed |
| Jaylen Brown | 2nd Team | nan | 35 | ❌ Missed |
| LeBron James | 3rd Team | 2nd Team | 6 | 🟡 Right player, different team |
| Domantas Sabonis | 3rd Team | nan | 17 | ❌ Missed |
| Julius Randle | 3rd Team | nan | 25 | ❌ Missed |
| Damian Lillard | 3rd Team | nan | 30 | ❌ Missed |
| De'Aaron Fox | 3rd Team | nan | 57 | ❌ Missed |

</details>
<details markdown="1">
<summary><b>2024 All-NBA Teams</b>: predicted vs. actual</summary>

| Player | Actual team | Predicted team | Predicted rank | Result |
|---|---|---|---|---|
| Nikola Jokić | 1st Team | 1st Team | 1 | ✅ Exact team |
| Giannis Antetokounmpo | 1st Team | 1st Team | 2 | ✅ Exact team |
| Luka Dončić | 1st Team | 1st Team | 3 | ✅ Exact team |
| Jayson Tatum | 1st Team | 1st Team | 5 | ✅ Exact team |
| Shai Gilgeous-Alexander | 1st Team | 2nd Team | 10 | 🟡 Right player, different team |
| Anthony Davis | 2nd Team | 3rd Team | 13 | 🟡 Right player, different team |
| Kevin Durant | 2nd Team | 3rd Team | 14 | 🟡 Right player, different team |
| Kawhi Leonard | 2nd Team | nan | 21 | ❌ Missed |
| Anthony Edwards | 2nd Team | nan | 32 | ❌ Missed |
| Jalen Brunson | 2nd Team | nan | 41 | ❌ Missed |
| Domantas Sabonis | 3rd Team | 2nd Team | 6 | 🟡 Right player, different team |
| LeBron James | 3rd Team | 2nd Team | 8 | 🟡 Right player, different team |
| Stephen Curry | 3rd Team | 2nd Team | 9 | 🟡 Right player, different team |
| Tyrese Haliburton | 3rd Team | nan | 19 | ❌ Missed |
| Devin Booker | 3rd Team | nan | 22 | ❌ Missed |

</details>
<details markdown="1">
<summary><b>2025 All-NBA Teams</b>: predicted vs. actual</summary>

| Player | Actual team | Predicted team | Predicted rank | Result |
|---|---|---|---|---|
| Nikola Jokić | 1st Team | 1st Team | 1 | ✅ Exact team |
| Giannis Antetokounmpo | 1st Team | 1st Team | 4 | ✅ Exact team |
| Shai Gilgeous-Alexander | 1st Team | 2nd Team | 6 | 🟡 Right player, different team |
| Jayson Tatum | 1st Team | 2nd Team | 7 | 🟡 Right player, different team |
| Donovan Mitchell | 1st Team | 3rd Team | 11 | 🟡 Right player, different team |
| LeBron James | 2nd Team | 2nd Team | 9 | ✅ Exact team |
| Jalen Brunson | 2nd Team | 3rd Team | 13 | 🟡 Right player, different team |
| Anthony Edwards | 2nd Team | 3rd Team | 15 | 🟡 Right player, different team |
| Stephen Curry | 2nd Team | nan | 17 | ❌ Missed |
| Evan Mobley | 2nd Team | nan | 30 | ❌ Missed |
| Tyrese Haliburton | 3rd Team | 2nd Team | 10 | 🟡 Right player, different team |
| Karl-Anthony Towns | 3rd Team | nan | 24 | ❌ Missed |
| James Harden | 3rd Team | nan | 45 | ❌ Missed |
| Jalen Williams | 3rd Team | nan | 46 | ❌ Missed |
| Cade Cunningham | 3rd Team | nan | 54 | ❌ Missed |

</details>
<details markdown="1">
<summary><b>2026 All-NBA Teams (live check, model refit on 2001–2025)</b>: predicted vs. actual</summary>

| Player | Actual team | Predicted team | Predicted rank | Result |
|---|---|---|---|---|
| Nikola Jokić | 1st Team | 1st Team | 1 | ✅ Exact team |
| Shai Gilgeous-Alexander | 1st Team | 1st Team | 3 | ✅ Exact team |
| Luka Dončić | 1st Team | 1st Team | 4 | ✅ Exact team |
| Victor Wembanyama | 1st Team | 2nd Team | 8 | 🟡 Right player, different team |
| Cade Cunningham | 1st Team | 3rd Team | 11 | 🟡 Right player, different team |
| Jalen Brunson | 2nd Team | 3rd Team | 15 | 🟡 Right player, different team |
| Donovan Mitchell | 2nd Team | nan | 18 | ❌ Missed |
| Kevin Durant | 2nd Team | nan | 33 | ❌ Missed |
| Jaylen Brown | 2nd Team | nan | 41 | ❌ Missed |
| Kawhi Leonard | 2nd Team | nan | 46 | ❌ Missed |
| Jalen Johnson | 3rd Team | nan | 30 | ❌ Missed |
| Chet Holmgren | 3rd Team | nan | 40 | ❌ Missed |
| Tyrese Maxey | 3rd Team | nan | 42 | ❌ Missed |
| Jalen Duren | 3rd Team | nan | 45 | ❌ Missed |
| Jamal Murray | 3rd Team | nan | 58 | ❌ Missed |

</details>


**Just for fun: the 2027 forecast** (from 2025-26 stats). 1st Team: Jokić, Gilgeous-Alexander, Dončić, Wembanyama, Antetokounmpo. 2nd Team: Cunningham, Tatum, Jalen Johnson, Edwards, Şengün. 3rd Team: Duren, Mitchell, Towns, Brunson, Maxey. Based on the error analysis, the biggest risk is anyone who doesn't reach 65 games.

**What can be concluded:** previous-season production, impact and status identify **more than half** of next year's All-NBA players (8.7 of 15 on average), and **most of the 1st Team**. They do this better than simple rules of thumb.

**What can't be concluded:** the model does **not** show that any stat *causes* a player to be selected. It finds patterns in how voters have behaved. It also can't predict injuries, trades or breakout seasons, which explain most of its errors.

---

### 9. Limitations, Ethics, and Reflection

**Biases and gaps in the data**
* **Voter bias carries over.** The model learns from *media votes*. If voters have favored big markets, winning teams or familiar names, the model inherits and repeats those patterns. McMahan and Shor (2024) found less reputation bias in All-NBA than in All-Star voting, but "less" doesn't mean none.
* **Rule changes.** 23 of the 25 target seasons were voted under position rules and without the 65-game minimum. The model learned an older version of the award.
* **Box-score bias.** Stats like points and VORP favor players with the ball in their hands. Elite defenders and off-ball players are undervalued, both by the stats and possibly by voters.
* **Injuries are invisible.** A missed season looks like a bad season.

**Who could be affected by wrong predictions?**
* **False positive** (predicting a player *will* make it when he won't): a team might plan its salary cap around paying a supermax extension that never becomes available, or a bettor might lose money.
* **False negative** (missing a player who does make it): a team could be caught off guard when a player becomes supermax-eligible, and the player's value could be underestimated in trade or contract talks.
* The players themselves: if a model like this shaped contract offers, a young player on the rise (the kind this model misses most) could be underpaid.

**Is this model appropriate for real-world decisions?** Only as a **starting point, never as the deciding factor.** It's useful for a quick first look ("who are the realistic candidates next year?"), but it gets roughly 6 of 15 names wrong in an average year, and it's blind to health, trades and player development, which are the things a front office cares about most. Any real use should combine it with medical information, scouting and current-season data.

**What users should understand before relying on it:** the scores rank players *relative to each other* within a season. The model forecasts a *media vote*, not "true" player quality. Its errors are predictable: injured stars get overrated and breakout young players get underrated.

**What I'd explore next**
* **Games-played risk:** add each player's injury history or games missed over several seasons to estimate the chance he reaches 65 games.
* **Early-season updating:** re-run the model after 20 games with current-season stats, closer to how voters actually see the season.
* **Two-stage model:** use the model to pick the 15 names, then use past status (the "run it back" signal) to order them into teams, combining the strengths from Section 7.
* **Voting-share target:** predict each player's share of All-NBA voting points instead of just the team, which gives the model much more information to learn from.

**Reflection.** The most surprising result was how good the "run it back" baseline was. Almost half the work of predicting All-NBA is simply knowing who was great last year. The model's real value is in the other half, and what separates the hits from the misses is health.

---

### 10. Code and Transparency

**Code.** Jupyter Notebook with all code, output and charts: [project2_all_nba.ipynb](https://github.com/lwils144-dev/data-structures-portfolio/blob/main/project2_all_nba.ipynb)

**Tools.** Python 3, pandas, NumPy, scikit-learn, statsmodels and matplotlib.

**AI usage disclosure.** I used generative AI (**Claude, Anthropic; model identifier "claude-opus-5-5," accessed through Claude's Cowork mode in the desktop app, September 2026**) in this project for the following:
1. **Data sourcing:** finding a downloadable mirror of Basketball-Reference data after direct access to the site was blocked, and looking up the official 2026 All-NBA Teams.
2. **Code:** debugging the Python notebook (data cleaning, feature engineering, cross-validation, model training, charts).
3. **Writing:** drafting sections of this write-up, which I reviewed and edited.

No data or results were invented. Every number on this page comes from running the linked notebook on the cited data.

#### References

Adams, L. (2024, March 16). *Hoops Rumors glossary: 65-game rule.* Hoops Rumors. https://www.hoopsrumors.com/2024/03/hoops-rumors-glossary-65-game-rule.html

Albert, A. A., de Mingo López, L. F., Allbright, K., & Gómez Blas, N. (2022). A hybrid machine learning model for predicting USA NBA All-Stars. *Electronics, 11*(1), Article 97. https://doi.org/10.3390/electronics11010097

Coleman, B. J., DuMond, J. M., & Lynch, A. K. (2008). An examination of NBA MVP voting behavior: Does race matter? *Journal of Sports Economics, 9*(6), 606–627. https://doi.org/10.1177/1527002508320653

Datta, S. (2026). *bball-reference-datasets* [Data set]. GitHub. https://github.com/sumitrodatta/bball-reference-datasets

McMahan, P., & Shor, E. (2024). Status ambiguity and multiplicity in the selection of NBA awards. *Sociological Science, 11*, 680–706. https://doi.org/10.15195/v11.a25

NBA.com Staff. (2026, May 24). *2025-26 All-NBA Teams announced.* NBA.com. https://www.nba.com/news/2025-26-all-nba-teams-announced

Rocha da Silva, J. V., & Canas Rodrigues, P. (2022). All-NBA teams' selection based on unsupervised learning. *Stats, 5*(1), 154–171. https://doi.org/10.3390/stats5010011

Sports Reference. (n.d.). *Basketball-Reference.com.* Retrieved September 23, 2026, from https://www.basketball-reference.com/



