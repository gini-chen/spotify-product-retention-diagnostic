(Spotify Product Retention Diagnostic)
# Audience Analytics: Does Engagement Persist After Streaming Spikes?
**Artist Profile:** [Spotify](https://open.spotify.com/artist/3akIq2A3BNOLtMa9X5C81M?si=lL9GJGooSA-z0i-iJOlgcg)

**Role Scope:** Product Analytics | Audience Engagement | Growth Insights  
**Tools:** SQL (PostgreSQL), DBeaver

## Main Question
A streaming spike can bring attention to an independent artist, but does that attention translate into sustained listening activity?

## Dataset & Preparation
The dataset contains **1,051 daily artist-level observations from January 1, 2024, through November 16, 2026**, covering **63,137 streams**. Each row represents one day of activity across my catalog.
[View dataset](data/audience_timeline.csv) 

## SQL Analysis

### Query 1 — February 4, 2025 had the most streams, but fewer listeners than January 23

**Question:** Did the days with the most streams also reach the most listeners?

**SQL**

```sql
select date, streams, listeners as daily_listeners, saves from database
order by streams desc
limit 5;
```

**Results**

| Date | Streams | Daily listeners | Saves |
| --- | ---: | ---: | ---: |
| February 4, 2025 | 1,345 | 655 | 115 |
| January 23, 2025 | 992 | 780 | 88 |
| May 26, 2024 | 948 | 94 | 4 |
| March 14, 2024 | 882 | 4 | 1 |
| May 24, 2024 | 853 | 138 | 9 |

**Finding:** February 4 had the highest stream count, while January 23 reached more listeners. More plays did not always mean more people listening.


### Query 2 — Average daily listeners fell approximately 65% from week 1 to week 4

**Question:** How did listening and saving activity change during the following four weeks?

I compared the seven days before the February 4 peak with four consecutive seven-day periods afterward, excluding the peak day.

**SQL**

```sql
with clean_data as 
(select cast(date as date) as date, cast(streams as integer) as streams, 
cast(listeners as integer) as listeners, cast(saves as integer) as saves
from database), 
peak as 
(select date as spike_date from clean_data
order by streams desc, date
limit 1), 
periods (period_name, first_day, last_day) as 
(values ('Before spike', -7, -1),
        ('Week 1 after',  1,  7),
        ('Week 2 after',  8, 14),
        ('Week 3 after', 15, 21),
        ('Week 4 after', 22, 28)) 
select p.period_name, count(a.date)days_oberved, round(avg(streams), 1) avg_streams, 
round(avg(listeners),1) avg_listeners, sum(saves)total_saves, 
round(100.0 * sum(a.saves) / nullif(sum(a.streams), 0), 2) saves_per_100_streams
from peak s
cross join periods p
left join clean_data a
    on a.date between s.spike_date + p.first_day
                  and s.spike_date + p.last_day
group by p.period_name
order by p.period_name;

```

**Results**

| Period | Days observed | Average daily streams | Average daily listeners | Total saves | Saves per 100 streams |
| --- | ---: | ---: | ---: | ---: | ---: |
| Before spike | 7 | 367.4 | 155.0 | 95 | 3.69 |
| Week 1 after | 7 | 214.6 | 86.4 | 74 | 4.93 |
| Week 2 after | 7 | 16.4 | 15.6 | 2 | 1.74 |
| Week 3 after | 7 | 30.7 | 29.6 | 5 | 2.33 |
| Week 4 after | 7 | 31.4 | 30.1 | 12 | 5.45 |

**Finding:** Average daily listeners declined approximately **65% from week 1 to week 4**, with the sharpest drop in week 2 and a partial recovery afterward. Each period contains seven observed days.
Week 4 had a higher saves-per-100-streams ratio than week 1, but total saves fell from 74 to 12. This ratio measures saves relative to plays, not the percentage of listeners who saved.

### Query 3 — Followers remained 27 higher after 28 days, while the active audience declined

**Question:** Were follower totals and active audience counts still higher 28 days later?

**SQL**
```sql
with clean_data as (
select cast(date as date) as date, cast(streams as integer) as streams,
cast(followers as integer) as followers,
cast("monthly listeners" as integer) as monthly_listeners,
cast("monthly active listeners" as integer)as monthly_active_listeners
from database),
peak as 
(select date as spike_date from clean_data
order by streams desc, date
limit 1)
select d.date, p.spike_date, d.followers, d.followers - first_value(d.followers)over (order by d.date) net_fol_since_spike,
d.monthly_listeners, d.monthly_active_listeners
from clean_data d
cross join peak p 
where d.date in (p.spike_date, p.spike_date + 7, p.spike_date + 28)
order by d.date;
```

**Results**

| Snapshot | Date | Followers | Net change since spike | Monthly listeners | Monthly active listeners |
| --- | --- | ---: | ---: | ---: | ---: |
| Spike day | February 4, 2025 | 243 | 0 | 1,706 | 60 |
| Day 7 | February 11, 2025 | 271 | +28 | 1,848 | 70 |
| Day 28 | March 4, 2025 | 270 | +27 | 1,053 | 32 |

**Finding:** After 28 days, followers remained **11.1% above** the spike-day total, while monthly active listeners were **46.7% below** it. These percentages compare day 28 with the spike day.




## Further Exploration: Catalog Activity Around Releases

My releases span English-language Ballad Pop, Mandopop, and Mandopop / Pop-Rock. I also explored whether catalog activity differed around these releases.

**Question:** How did streaming volume, saves, and playlist adds differ across comparable release windows?



| | From Me to You | Missing Pieces |
|---|---|---|
|| <img width="100" alt="From Me to You" src="https://github.com/user-attachments/assets/f07166cb-94dc-495f-8ce3-a607993c27c3" /> | <img width="100" alt="Missing Pieces" src="https://github.com/user-attachments/assets/7c607941-91fe-408e-82d0-afb1e10b609f" /> |
| Release date | May 24, 2024 | September 19, 2025 |
| Style | Mandopop | Mandopop / Pop-Rock |
| Language | Mandarin | Mandarin |
| Observation window | May 24–June 22, 2024 | September 19–October 18, 2025 |
| Catalog streams | 12,099 | 3,146 |
| Saves | 42 | 23 |
| Playlist adds | 67 | 102 |

**Finding:** The “From Me to You” window had more catalog streams and saves, while the “Missing Pieces” window had more playlist adds.

**Interpretation:** These windows showed different engagement patterns. However, the comparison cannot establish whether genre, promotion, audience size, or another factor explains the differences.

