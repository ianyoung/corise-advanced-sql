# Exercise 1

For our first exercise, we need to determine which customers are eligible to order from Virtual Kitchen, and which distributor will handle the orders that they place. We want to return the following information:

- Customer ID

- Customer first name
- Customer last name
- Customer email
- Supplier ID
- Supplier name
- Shipping distance in kilometers or miles (you choose)


## Step 1

We have 10,000 potential customers who have signed up with Virtual Kitchen. If the customer is able to order from us, then their city/state will be present in our database. Create a query in Snowflake that returns all customers that can place an order with Virtual Kitchen.

## Step 2

We have 10 suppliers in the United States. Each customer should be fulfilled by the closest distribution center. Determine which supplier is closest to each customer, and how far the shipment needs to travel to reach the customer. There are a few different ways to complete this step. **Use the customer's city and state to join to the us_cities resource table. Do not worry about zip code for this exercise.**

Order your results by the customer's last name and first name.

Before you submit your solutions to the exercise, please create a brief comment at the top of your SQL query to explain your approach.