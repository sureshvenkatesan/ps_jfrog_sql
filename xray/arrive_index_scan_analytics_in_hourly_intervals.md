
# Artifacts eventually indexed and scanned in series of intervals (Query1):

**Query1** splits the time window into 60 minute intervals and shows how many of artifacts that arrived (in Xray) in the interval were indexed or scanned in that interval or “eventually” in any future interval in the given UTC time window .

**Note:** All epoch times and the time window are in UTC 
```
select x.* from (
SELECT
    to_char(interval_start, 'YYYY-MM-DD HH24:MI') as interval_start_utc,
    count(*) FILTER (WHERE arrive_time_ms IS NOT NULL) as num_arrived,
    count(*) FILTER (WHERE index_time IS NOT NULL) as num_indexed,
    count(*) FILTER (WHERE scanned_time IS NOT NULL) as num_scanned
    
FROM
    generate_series(
        '2023-03-01 10:00'::timestamp,
        '2023-03-14 10:00'::timestamp,
        '60 minutes'
    ) as intervals(interval_start)
    LEFT JOIN root_files ON
    binary_manager_id = 'default'
        AND arrive_time_ms >= extract(epoch from interval_start)::bigint * 1000
        AND arrive_time_ms < extract(epoch from (interval_start + interval '60 minutes'))::bigint * 1000
GROUP BY interval_start
ORDER BY interval_start ) x
where x.num_arrived > 0;
```
---

**Query2**

You can get the artifacts for **Query1** including the arrival , when they were actually indexed and scanned  for the 60 minute intervals in the given UTC time window  from:
```
select x.* from (
SELECT
    to_char(interval_start , 'YYYY-MM-DD HH24:MI') as interval_start_utc, interval_start,
    repo, TRIM(CONCAT(COALESCE(path, ''), ' ', COALESCE(name, ''))) AS path_name,
    sha256 , 
    arrive_time_ms, to_char(to_timestamp(arrive_time_ms / 1000) AT TIME ZONE 'UTC', 'YYYY-MM-DD HH24:MI:SS.MS') as arrive_datetime_utc,
    index_time, to_char(to_timestamp(index_time) AT TIME ZONE 'UTC', 'YYYY-MM-DD HH24:MI:SS') AS index_datetime_utc,
    scanned_time, to_char(to_timestamp(scanned_time) AT TIME ZONE 'UTC', 'YYYY-MM-DD HH24:MI:SS') AS scanned_datetime_utc,
    (index_time - (arrive_time_ms / 1000)) as  arrive_to_index_sec , 
    (scanned_time - index_time) as  index_to_scanned_sec,
    (scanned_time - (arrive_time_ms / 1000)) as  arrive_to_scanned_sec  
FROM
    generate_series(
        '2023-03-01 10:00'::timestamp,
        '2023-03-14 10:00'::timestamp,
        '60 minutes'
    ) as intervals(interval_start)
    LEFT JOIN root_files ON
        binary_manager_id = 'default'
        AND arrive_time_ms >= extract(epoch from interval_start)::bigint * 1000
        AND arrive_time_ms < extract(epoch from (interval_start + interval '60 minutes'))::bigint * 1000
ORDER BY interval_start) x
where x.repo is not null;
```
---

# Artifacts indexed and scanned in that  interval itself (Query3):

**Query3** splits the time window into 60 minute intervals and shows how many of those arrived in the interval were  indexed or scanned in that interval itself. It will not count them even
if they were eventually indexed or scanned in a later interval in the given time window.

For example: Between "2022-02-09 15:00" and "2022-02-09 15:10" num_arrived is 4, num_indexed is 4 .
If num_scanned is 1 in this interval , **Query3** will show only 1  event if the remaining 3 of the num_arrived in  the first interval got scanned in the second interval   between "2022-02-09 15:10" and "2022-02-09 15:15" . Also num_scanned will count these 3 artifacts in the second or  later intervals. 

```
select x.* from (
SELECT
    to_char(interval_start, 'YYYY-MM-DD HH24:MI') as interval_start_utc,
    count(*) FILTER (WHERE arrive_time_ms IS NOT NULL) as num_arrived,
    count(*) FILTER (WHERE index_time IS NOT NULL
                     -- AND index_time <= extract(epoch from (interval_start + interval '60 minutes'))::bigint
                     AND (index_time >= extract(epoch from interval_start)::bigint) 
                     AND (index_time < extract(epoch from (interval_start + interval '60 minutes'))::bigint + 3600)
                    ) as num_indexed,
    count(*) FILTER (WHERE scanned_time IS NOT NULL 
                     AND (scanned_time >= extract(epoch from interval_start)::bigint) 
                     AND (scanned_time < extract(epoch from (interval_start + interval '60 minutes'))::bigint + 3600)
                    ) as num_scanned
FROM
    generate_series(
        '2023-03-01 10:00'::timestamp,
        '2023-03-14 10:00'::timestamp,
        '60 minutes'
    ) as intervals(interval_start)
    LEFT JOIN root_files ON
        binary_manager_id = 'default'
        AND arrive_time_ms >= extract(epoch from interval_start)::bigint * 1000
        AND arrive_time_ms < extract(epoch from (interval_start + interval '60 minutes'))::bigint * 1000
GROUP BY interval_start
ORDER BY interval_start
) x
where x.num_arrived > 0;
```

**Note:** 
The purpose of adding + 3600 in the conditions index_time < extract(epoch from (interval_start + interval '60 minutes'))::bigint + 3600 and scanned_time < extract(epoch from (interval_start + interval '60 minutes'))::bigint + 3600 is to include files that were indexed or scanned within the first hour of the next interval.

Here's why:

The generate_series function generates a series of intervals starting from interval_start and ending at interval_start + interval '60 minutes'. The query is counting the number of files that arrived, indexed, or scanned within these intervals.

When we filter the index_time and scanned_time using the condition <mark>index_time < extract(epoch from (interval_start + interval '60 minutes'))::bigint<mark> and <mark>scanned_time < extract(epoch from (interval_start + interval '60 minutes'))::bigint<mark>, respectively, we're excluding files that were indexed or scanned after the end of the interval. However, there could be some files that were indexed or scanned within the first hour of the next interval, and we want to include those files in the count as well.

So we add + 3600 to the end of the interval to extend the window by 1 hour, which allows us to include files that were indexed or scanned within the first hour of the next interval.

In summary, the purpose of + 3600 in the conditions is to extend the indexing and scanning windows by 1 hour, so that files that were indexed or scanned within the first hour of the next interval can be included in the count

---