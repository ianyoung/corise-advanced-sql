# Project 3

Instructions can be [viewed here](/week-3/project-3-instructions.md).

## Query

```sql
/*
 * A daily report to track:
 * 
 * 1. Total unique sessions
 * 2. The average length of sessions in seconds.
 * 3. The average number of searches completed before displaying a recipe.
 * 4. The name of the recipe that was most viewed.
 */

-- select * from vk_data.events.website_activity limit 1000

with events as (

	select
    	event_id,
        session_id,
        event_timestamp,
        trim(parse_json(event_details):"recipe_id", '"') as recipe_id,
        trim(parse_json(event_details):"event", '"') as event_type
    from vk_data.events.website_activity
    group by 1, 2, 3, 4, 5
    
)

, grouped_sessions as (

	select
    	session_id,
        min(event_timestamp) as min_event_timestamp,
        max(event_timestamp) as max_event_timestamp,
        iff(
        	count_if(event_type = 'view_recipe') = 0, null,
        	round(count_if(event_type = 'search') / count_if(event_type = 'view_recipe'))
        ) as searches_per_recipe_view
    from events
    group by session_id
    
)

, fav_recipe as (

	select
    	date(event_timestamp) as event_day,
        recipe_id,
        count(*) as total_views
    from events
    where recipe_id is not null
    group by 1, 2
    qualify row_number() over (partition by event_day order by total_views desc) = 1
    
)

, result as (

	select
    	date(min_event_timestamp) as event_day,
        count(session_id) as total_sessions,
        round(avg(datediff('sec', min_event_timestamp, max_event_timestamp))) as avg_session_length_sec,
        max(searches_per_recipe_view) as avg_searches_per_recipe_view,
        max(recipe_name) as favourite_recipe
    from grouped_sessions
    inner join fav_recipe on date(grouped_sessions.min_event_timestamp) = fav_recipe.event_day
    inner join vk_data.chefs.recipe using(recipe_id)
    group by 1

)

select * from result
order by 2
```

## Results

![Week 3 query results](/assets/week-3-query-results.png)

## Profile

![Week 3 query profile](/assets/week-3-query-profile.png)


## Performance

The query profile showed a query duration of 956ms. The aggregations involved with grouping the sessions was the most expensive part at 67%. This has been broken out as a CTE but may not scale well as the number of events grows over time.