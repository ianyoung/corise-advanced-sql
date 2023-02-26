# Project 4 Instructions

For this week's project, we will work with another sample tech exercise from the `SNOWFLAKE_SAMPLE_DATA` database. We will use the `TPCH_SF1` schema to complete the exercise.

**Instructions that were provided to the candidate**

We need to develop a report to analyze **AUTOMOBILE** customers who have placed **URGENT** orders. We expect to see one row per customer, with the following columns:

* **C_CUSTKEY**
* **LAST_ORDER_DATE**: The date when the last URGENT order was placed
* **ORDER_NUMBERS**: A comma-separated list of the order_keys for the three highest dollar urgent orders
* **TOTAL_SPENT**: The total dollar amount of the three highest orders
* **PART_1_KEY**: The identifier for the part with the highest dollar amount spent, across all urgent orders 
* **PART_1_QUANTITY**: The quantity ordered
* **PART_1_TOTAL_SPENT**: Total dollars spent on the part 
* **PART_2_KEY**: The identifier for the part with the second-highest dollar amount spent, across all urgent orders  
* **PART_2_QUANTITY**: The quantity ordered
* **PART_2_TOTAL_SPENT**: Total dollars spent on the part 
* **PART_3_KEY**: The identifier for the part with the third-highest dollar amount spent, across all urgent orders 
* **PART_3_QUANTITY**: The quantity ordered
* **PART_3_TOTAL_SPENT**: Total dollars spent on the part 

The output should be sorted by **LAST_ORDER_DATE** descending.

There are two parts to this exercise, and you can choose in which order you would like to complete them.  

To submit your work, please create **one text file** (can live in Google Docs, GitHub, etc.) with the SQL code for Part 1, followed by the answer to Part 2. If you are submitting as a notebook, then you can format Part 2 so that it appears as a comment block.

1. Create a query to provide the report requested. Your query should have a LIMIT 100 when you submit it for review. Remember that you are creating this as a tech exercise for a job evaluation. Your query should be well-formatted, with clear names and comments.

2. Review the candidate's tech exercise below, and provide a one-paragraph assessment of the SQL quality. Provide examples/suggestions for improvement if you think the candidate could have chosen a better approach.

Do you agree with the results returned by the query?

Is it easy to understand?

Could the code be more efficient?


<details>
    <summary>Candidates submission:</summary>

    ```sql
    with urgent_orders as (
        select
            o_orderkey,
            o_orderdate,
            c_custkey,
            p_partkey,
            l_quantity,
            l_extendedprice,
            row_number() over (partition by c_custkey order by l_extendedprice desc) as price_rank
        from snowflake_sample_data.tpch_sf1.orders as o
        inner join snowflake_sample_data.tpch_sf1.customer as c on o.o_custkey = c.c_custkey
        inner join snowflake_sample_data.tpch_sf1.lineitem as l on o.o_orderkey = l.l_orderkey
        inner join snowflake_sample_data.tpch_sf1.part as p on l.l_partkey = p.p_partkey
        where c.c_mktsegment = 'AUTOMOBILE'
            and o.o_orderpriority = '1-URGENT'
        order by 1, 2),

    top_orders as (
        select
            c_custkey,
            max(o_orderdate) as last_order_date,
            listagg(o_orderkey, ', ') as order_numbers,
            sum(l_extendedprice) as total_spent
        from urgent_orders
        where price_rank <= 3
        group by 1
        order by 1)

    select 
        t.c_custkey,
        t.last_order_date,
        t.order_numbers,
        t.total_spent,
        u.p_partkey as part_1_key,
        u.l_quantity as part_1_quantity,
        u.l_extendedprice as part_1_total_spent,
        u2.p_partkey as part_2_key,
        u2.l_quantity as part_2_quantity,
        u2.l_extendedprice as part_2_total_spent,
        u3.p_partkey as part_3_key,
        u3.l_quantity as part_3_quantity,
        u3.l_extendedprice as part_3_total_spent
    from top_orders as t
    inner join urgent_orders as u on t.c_custkey = u.c_custkey
    inner join urgent_orders as u2 on t.c_custkey = u2.c_custkey
    inner join urgent_orders as u3 on t.c_custkey = u3.c_custkey
    where u.price_rank = 1 and u2.price_rank = 2 and u3.price_rank = 3
    order by t.last_order_date desc
    limit 100
    ```

</details>





