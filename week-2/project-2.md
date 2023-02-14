# Project 2

## Brief

Virtual Kitchen has an emergency! 

We shipped several meal kits without including fresh parsley, and our customers are starting to complain. We have identified the impacted cities, and we know that 25 of our customers did not get their parsley. That number might seem small, but Virtual Kitchen is committed to providing every customer with a great experience.

**Our management has decided to provide a different recipe for free (if the customer has other preferences available), or else use grocery stores in the greater Chicago area to send an overnight shipment of fresh parsley to our customers. We have one store in Chicago, IL and one store in Gary, IN both ready to help out with this request.**

Last night, our on-call developer **created a query to identify the impacted customers and their attributes in order to compose an offer to these customers to make things right.** But the developer was paged at 2 a.m. when the problem occurred, and she created a fast query so that she could go back to sleep.

You review her code today and decide to reformat her query so that she can catch up on sleep.

Here is the query she emailed you. **Refactor it to apply a consistent format, and add comments that explain your choices.** We are going to review different options in the lecture, so if you are willing to share your refactored query with the class, then let us know!

```sql
select 
    first_name || ' ' || last_name as customer_name,
    ca.customer_city,
    ca.customer_state,
    s.food_pref_count,
    (st_distance(us.geo_location, chic.geo_location) / 1609)::int as chicago_distance_miles,
    (st_distance(us.geo_location, gary.geo_location) / 1609)::int as gary_distance_miles
from vk_data.customers.customer_address as ca
join vk_data.customers.customer_data c on ca.customer_id = c.customer_id
left join vk_data.resources.us_cities us 
on UPPER(rtrim(ltrim(ca.customer_state))) = upper(TRIM(us.state_abbr))
    and trim(lower(ca.customer_city)) = trim(lower(us.city_name))
join (
    select 
        customer_id,
        count(*) as food_pref_count
    from vk_data.customers.customer_survey
    where is_active = true
    group by 1
) s on c.customer_id = s.customer_id
    cross join 
    ( select 
        geo_location
    from vk_data.resources.us_cities 
    where city_name = 'CHICAGO' and state_abbr = 'IL') chic
cross join 
    ( select 
        geo_location
    from vk_data.resources.us_cities 
    where city_name = 'GARY' and state_abbr = 'IN') gary
where 
    ((trim(city_name) ilike '%concord%' or trim(city_name) ilike '%georgetown%' or trim(city_name) ilike '%ashland%')
    and customer_state = 'KY')
    or
    (customer_state = 'CA' and (trim(city_name) ilike '%oakland%' or trim(city_name) ilike '%pleasant hill%'))
    or
    (customer_state = 'TX' and (trim(city_name) ilike '%arlington%') or trim(city_name) ilike '%brownsville%')
```

![vk-customers-missing-parsley-1](/Users/ian/code/github/ianyoung/corise-advanced-sql/assets/vk-customers-missing-parsley-1.png)



## Evaluation

So the query above identifies the 25 impacted customers and retrieves their relevant attributes so that:

1. They may receive a free recipe (if they have one listed in their preferences) or
2. They may be sent an overnight shipment of fresh parsley from a store in either Gary, IN or Chicago, IL.

The above query is working correctly but is in need of a refactor now the 2 a.m. emergency is over.

Required information (columns):

- **Customer Name**
- **Customer City**
- **Customer State**
- **Count of food preferences** *(to see if they preferred receipes on record)*
- **Distance from Chicago store in miles** *(to see which store is closest)*
- **Distances from Gary store in miles** *(to see which store is closest)*



## Observations

#### Modularity

- Subqueries can be broken out as CTE's.
  - Customer food preferences.
  - Geo-location of the store in Gary.
  - Geo-location of the store in Chicago.
- Customer details can also be broken out into a CTE to tidy it up.
- There `where` clause looking for affected cities can be pulled out into a CTE

#### Consistency

- Table aliasing is not applied consistently. Sometimes it uses `as` sometimes it doesn't.
- Functions such as `trim` and `upper` should be written in a consistent casing. Some are uppercase.

#### Simplicity

- `ltrim()` and `rtrim()` can be replaced with `trim()`.

#### Style

Opionionated preferences on style.

- `join` should be written more specifically as `inner join`.
- The `on` of joins should be on a new line and indented.
- Line lengths should be kept to under 120 characters.
- Use `concat` instead of `||` for better readability. (Although `||` is a standard SQL operator)



## Solution

After understanding what the query is doing we can try to clean things up. It's often easiest to start by looking at the subqueries and joins to see what can be extracted.

### Customer Details

The first three columns can be obtained fairly simply from the `customer_data` and `customer_address` tables. We will include `customer_id` so that we can use this as a join later on:

```sql
-- Customer details
with customer_details as (
    select
        customer_data.customer_id,
        concat(customer_data.first_name, ' ', customer_data.last_name) as customer_name,
        trim(customer_address.customer_city) as customer_city,
        trim(customer_address.customer_state) as customer_state
    from vk_data.customers.customer_data as customer_data
    inner join vk_data.customers.customer_address as customer_address
        on customer_data.customer_id = customer_address.customer_id
)
```

As well as extracting into a CTE we have:

- Used full table references for easier comprehension
- Consistently used `as` for table aliases
- Kept everything lowercase, including functions
- Included the full name of the join (`inner join`)

### Customer Food Preferences

We need to know if the customer has any other recipe preferences on profile by returning a count. This can also be extracting fairly simply as a CTE from a single table, making sure they are active:

```sql
-- Customer food preferences (of active customers)
, customer_food_prefs as (
    select
        customer_id,
        count(*) as food_pref_count
    from vk_data.customers.customer_survey
    where is_active = true
    group by customer_id
)
```

Once again we are applying the same rules for consistency.

### Cities

Although I'm not too familiar with American geography I will presume the developer is right in her marking of the affected cities and states. We can retrieve the relevant city details fairly easily and then mark the affected cities so we can filter on them later. This can be done with a simple case statement:

```sql
-- Cities (with affected cities labelled)
, cities as (
    select
        cities.city_name,
        state_abbr,
        cities.geo_location,
        -- Label affected citites to provide something to filter on later
        case
            when
                (state_abbr = 'KY' and trim(city_name) ilike any('%concord%', '%georgetown%', '%ashland%')) then true
            when
                (state_abbr = 'CA' and (trim(city_name) ilike any('%oakland%', '%pleasant hill%'))) then true
            when
                (state_abbr = 'TX' and (trim(city_name) ilike '%arlington%') or trim(city_name) ilike '%brownsville%') then true
            else false end as affected_city
    from vk_data.resources.us_cities as cities
)
```

### Store Locations

For customers who don't have any saved recipe preferences we need to send them an emergency overnight shipment of parsley. Our stores in Gary, IN or Chicago, IL are on standby, we just need to figure out which of them is closest to the customer. We first need to retrieve their geo-location:

```sql
-- Geo-location of the store in Gary
, gary_store as (
    select geo_location
    from cities
    where city_name = 'GARY' and state_abbr = 'IN'
)

-- Geo-location of the store in Chicago
, chicago_store as (
    select geo_location
    from cities
    where city_name = 'CHICAGO' and state_abbr = 'IL'
)
```

It's easier to break each one out into their own CTE so we can reference them later on.

### The Main Query

Finally, after extracting the individual bits of information we can make the main query much easier to understand. The query itself (the result) will itself be a CTE:

```sql
-- Final result
, final_result as (
    select
        customer_details.customer_name,
        customer_details.customer_city,
        customer_details.customer_state,
        customer_food_prefs.food_pref_count,
        cast((st_distance(cities.geo_location, chicago_store.geo_location) / 1609) as int) 
            as chicago_distance_miles,
        cast((st_distance(cities.geo_location, gary_store.geo_location) / 1609) as int) 
            as gary_distance_miles
    from customer_details
    inner join 
        customer_food_prefs
        on customer_details.customer_id = customer_food_prefs.customer_id
    left join cities
        on upper(customer_details.customer_state) = upper(cities.state_abbr)
        and lower(customer_details.customer_city) = lower(cities.city_name)
    cross join gary_store
    cross join chicago_store
    where affected_city = true
)
```

Table names have been used instead of aliases for easier interpretation, everything has been kept to lowercase, and line lengths have been trimmed to a readable length. Consistency has been applied throughout. The end result is now much easier to understand and maintain.

### Combined

Here is the query in full. 

```sql
-- Customer details
with customer_details as (
    select
        customer_data.customer_id,
        concat(customer_data.first_name, ' ', customer_data.last_name) as customer_name,
        trim(customer_address.customer_city) as customer_city,
        trim(customer_address.customer_state) as customer_state
    from vk_data.customers.customer_data as customer_data
    inner join vk_data.customers.customer_address as customer_address
        on customer_data.customer_id = customer_address.customer_id
)

-- Customer food preferences (of active customers)
, customer_food_prefs as (
    select
        customer_id,
        count(*) as food_pref_count
    from vk_data.customers.customer_survey
    where is_active = true
    group by customer_id
)

-- Cities (with affected cities labelled)
, cities as (
    select
        cities.city_name,
        state_abbr,
        cities.geo_location,
        -- Label affected citites to provide something to filter on later
        case
            when
                (state_abbr = 'KY' and trim(city_name) ilike any('%concord%', '%georgetown%', '%ashland%')) then true
            when
                (state_abbr = 'CA' and (trim(city_name) ilike any('%oakland%', '%pleasant hill%'))) then true
            when
                (state_abbr = 'TX' and (trim(city_name) ilike '%arlington%') or trim(city_name) ilike '%brownsville%') then true
            else false end as affected_city
    from vk_data.resources.us_cities as cities
)

-- Geo-location of the store in Gary
, gary_store as (
    select geo_location
    from cities
    where city_name = 'GARY' and state_abbr = 'IN'
)

-- Geo-location of the store in Chicago
, chicago_store as (
    select geo_location
    from cities
    where city_name = 'CHICAGO' and state_abbr = 'IL'
)

-- Final result
, final_result as (
    select
        customer_details.customer_name,
        customer_details.customer_city,
        customer_details.customer_state,
        customer_food_prefs.food_pref_count,
        cast((st_distance(cities.geo_location, chicago_store.geo_location) / 1609) as int) 
            as chicago_distance_miles,
        cast((st_distance(cities.geo_location, gary_store.geo_location) / 1609) as int) 
            as gary_distance_miles
    from customer_details
    inner join 
        customer_food_prefs
        on customer_details.customer_id = customer_food_prefs.customer_id
    left join cities
        on upper(customer_details.customer_state) = upper(cities.state_abbr)
        and lower(customer_details.customer_city) = lower(cities.city_name)
    cross join gary_store
    cross join chicago_store
    where affected_city = true
)

-- Output everything from the final_result
select * from final_result
order by customer_name
```

We still get our 25 results but now the results are ordered by customer name:

![vk-customers-missing-parsley-1](/Users/ian/code/github/ianyoung/corise-advanced-sql/assets/vk-customers-missing-parsley-1.png)



*Note: It may be somewhat controversial but although I'm definitely no fan of leading commas in `select` statements I do find them useful when constructing multiple CTEs. It's very easy to forget the trailing commas when the `with` statement is out of view and each CTE is separated with line breaks for readability. It also helps having the comma on the same line as the proceeding CTE so they can easily be commented out when testing each individual part.*





