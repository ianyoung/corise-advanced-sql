## Exploring the Dataset

### US Cities

`vk_data.resources.us_cities` is an external resource so let's have a quick look at the columns and the row data: 

```sql
select * from us_cities
order by 2, 3 asc
limit 10
```

![us-cities](/Users/ian/Downloads/us-cities.png)

We only need the city and state combination but we'll include a count of the matching pairs to see if there are duplicates.

```sql
select 
    city_name as city, 
    upper(trim(state_abbr)) as state, 
    count(*)
from 
    us_cities
group by 
    1,2
order by 
    3 desc
```

To guard against any mismatches `upper()` and `trim()` are used to clean the values from any extra whitespace and make sure they are all capitalised. This is particularly important if you think you'll later be joining on these values.

<img src="/Users/ian/Code/github/ianyoung/corise-advanced-sql/assets/us-cities-count.png" alt="us-cities-count" style="zoom:50%;" />

From here we can see that the count is greater than 1 for a number of city/state pairs. This should be a 1 to 1 match. 

We can use [row_number() and partition by](https://docs.snowflake.com/en/sql-reference/functions/row_number.html) (a window function) in Snowflake to give a ranking for each city/state pair:

```sql
select
    upper(trim(city_name)) as city,
    upper(state_abbr) as state,
    lat,
    long,
    row_number() over (partition by upper(city_name), upper(state_abbr) order by 1) as rank
from
    vk_data.resources.us_cities
```

![us-cities-rank](/Users/ian/Code/github/ianyoung/corise-advanced-sql/assets/us-cities-rank.png)

The final part is to make sure we only return the first value where there are multiple rankings. A `where` statement would not work on the `rank` column because the query is still processing so we'd either need to pull it out to a CTE and select only the row with `rank = 1`:

```sql
with city_state_rank as (
    select
        upper(trim(city_name)) as city,
        upper(state_abbr) as state,
        lat,
        long,
        row_number() over (partition by upper(city_name), upper(state_abbr) order by 1) as rank
    from
        vk_data.resources.us_cities
)

select * from city_state_rank
where rank = 1
```

Or in Snowflake we can use the [qualify clause](https://docs.snowflake.com/en/sql-reference/constructs/qualify.html). 

```sql
select
    upper(trim(city_name)) as city,
    upper(state_abbr) as state,
    lat,
    long
from
    vk_data.resources.us_cities
qualify row_number() over (partition by upper(city_name), upper(state_abbr) order by 1) = 1
```

![us-cities-ranked-by-1](/Users/ian/Code/github/ianyoung/corise-advanced-sql/assets/us-cities-ranked-by-1.png)

The results set is now trimmed, cleaned, and only returns a single value for each city/state pair.

## References

- [Row_number Snowflake documentation](https://docs.snowflake.com/en/sql-reference/functions/row_number.html)
- [SQL Partion By clause](https://blog.quest.com/when-and-how-to-use-the-sql-partition-by-clause/) - Gives a good explanation
- [Qualify - Snowflake documentation](https://docs.snowflake.com/en/sql-reference/constructs/qualify.html)

















