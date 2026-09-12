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

Code repository: github.com/lwils144-dev/data-structures-portfolio

Data sources:

nflverse / nflreadpy: github.com/nflverse/nflreadpy
Projections dataset: Fantasy Football Data Pros API

AI usage disclosure: Generative AI (Claude, Sonnet 4.5, accessed via Anthropic's Claude in Cowork mode, September 2026) was used throughout this project's development in the following ways: (1) troubleshooting Python errors during data loading and merging (e.g., diagnosing a ModuleNotFoundError caused by a Python 3.13/pandas version conflict, and a KeyError caused by a duplicate player_name column after merging); (2) helping guide the writing and iteration of pandas merge logic and seaborn/matplotlib visualization code based on the actual column names and data structure of the files used; (3) assisting with the project outline and drafting portions of this write-up, which were reviewed, edited, and personalized by me. No AI tool was used to generate or fabricate data, results, or citations — all data comes from the cited APIs, and all cited sources were independently verified to exist.

Key Academic References

Abadzic, A., Cheun, J., & Patel, M. (2024). Data analysis on predicting the top 12 fantasy football players by position. SMU Data Science Review, 8(2), Article 7. https://scholar.smu.edu/datasciencereview/vol8/iss2/7

Herzog, S. M., & Hertwig, R. (2011). The wisdom of ignorant crowds: Predicting sport outcomes by mere recognition. Judgment and Decision Making, 6(1), 58–72. https://doi.org/10.1017/S1930297500002096

Madsen, J. K. (2025). Goal-line oracles: Exploring accuracy of wisdom of the crowd for football predictions. PLOS ONE, 20(1), e0312487. https://doi.org/10.1371/journal.pone.0312487

Butler, D., Butler, R., & Eakins, J. (2021). Expert performance and crowd wisdom: Evidence from English Premier League predictions. European Journal of Operational Research, 288(1), 170–182. https://doi.org/10.1016/j.ejor.2020.05.034


