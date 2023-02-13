# Project 1

Reference ERD:

![virtual-kitchen-erd](/assets/virtual-kitchen-erd.webp)

**Create a query in Snowflake that returns all customers that can place an order with Virtual Kitchen.**

**1.1 Remove duplicates from the `us_cities` table.**

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

- `row_number()` is a window function. When used with `partition by` it provides a way to rank the city/state pairs.
- `qualify` is a Snowflake clause that allows us to filter the results to only return the first ranking city/state pair (avoiding duplicates).

**1.2 Select only customers that are eligable to order based on city and state**

```sql
with cities as (
    -- remove duplicate city and state cominbations
    select
        upper(trim(city_name)) as city_name,
        upper(state_abbr) as state_name,
        lat,
        long
    from vk_data.resources.us_cities
    qualify row_number() over (partition by upper(city_name), upper(state_abbr) order by 1) = 1
)

-- select * from cities

, customers as (
    -- selecting required columns
    select 
        customer_id,
        first_name as customer_first_name,
        last_name as customer_last_name,
        email as customer_email,
        c2.customer_city,
        c2.customer_state,
        cities.lat as customer_lat,
        cities.long as customer_long
    from vk_data.customers.customer_data as c1
    inner join vk_data.customers.customer_address as c2 using (customer_id)
    -- only serve relevant customers in supported locations
    inner join cities on (
        upper(trim(c2.customer_city)) = upper(cities.city_name) and
        upper(trim(c2.customer_state)) = upper(cities.state_name)
    )
)

select * from customers
```

- This matches the customer to the geo_location of their city
- The cleaning work on `us_cities` is now wrapped up into a CTE and referenced.
- We select all required information on the customer and `inner join` on the `cities` CTE to return only those customers who are eligable to order from us.

**1.3 Match the supplier to the geo_location of their city**

```sql
select
    supplier_id,
    supplier_name,
    upper(trim(supplier_city)) as supplier_city,
    upper(supplier_state) as supplier_state,
    c1.lat as supplier_lat,
    c1.long as supplier_long
from 
    vk_data.suppliers.supplier_info as s1
left join vk_data.resources.us_cities as c1 on
    upper(s1.supplier_city) = c1.city_name and
    upper(s1.supplier_state) = c1.state_abbr
```

![vk-supplier-location](/assets/vk-supplier-location.png)

**1.4 Return the data for the customer and the supplier rated as the closest. Sorted by last name and first name**

```sql
with cities as (
    -- remove duplicate city and state cominbations
    select
        upper(trim(city_name)) as city_name,
        upper(state_abbr) as state_name,
        lat,
        long
    from vk_data.resources.us_cities
    qualify row_number() over (partition by upper(city_name), upper(state_abbr) order by 1) = 1
)

-- select * from cities

, customers as (
    -- selecting required columns
    select 
        customer_id,
        first_name as customer_first_name,
        last_name as customer_last_name,
        email as customer_email,
        c2.customer_city,
        c2.customer_state,
        cities.lat as customer_lat,
        cities.long as customer_long
    from vk_data.customers.customer_data as c1
    inner join vk_data.customers.customer_address as c2 using (customer_id)
    -- only serve relevant customers in supported locations
    inner join cities on (
        upper(trim(c2.customer_city)) = upper(cities.city_name) and
        upper(trim(c2.customer_state)) = upper(cities.state_name)
    )
)

-- select * from customers

, suppliers as (
    select
        supplier_id,
        supplier_name,
        upper(trim(supplier_city)) as supplier_city,
        upper(supplier_state) as supplier_state,
        c1.lat as supplier_lat,
        c1.long as supplier_long
    from 
        vk_data.suppliers.supplier_info as s1
    left join vk_data.resources.us_cities as c1 on
        upper(s1.supplier_city) = c1.city_name and
        upper(s1.supplier_state) = c1.state_abbr
)

-- select * from suppliers limit 10

, final_result as (
    select 
        customer_id,
        customer_first_name,
        customer_last_name,
        customer_email,
        supplier_id,
        supplier_name,
        st_distance(
            st_makepoint(customers.customer_long, customers.customer_lat),
            st_makepoint(suppliers.supplier_long, suppliers.supplier_lat)) / 1000 as distance_in_km
    from
        customers
    -- get all suppliers for every customer
    cross join suppliers
    -- select the shortest distance only from the running query
    qualify 
        row_number() over (partition by customer_id order by distance_in_km) = 1
    order by
        customer_last_name, customer_first_name
)

select * from final_result
```

![vk-project1](/assets/vk-project1.png)

- The query in full broken up into subqueries.
- The suppliers table is cross joined to the customers table to get all suppliers for each customer.
- The distance between each supplier and customer is calculated and each combination is ranked from closest to furthest in kilometers.
- The data is returned for the customer and the supplier rated as the closest.
- Sorted by customer last name and first name.

*Note that although cross-join is used in this solution with a relative small dataset, in a much larger dataset this may not be feasible.*



