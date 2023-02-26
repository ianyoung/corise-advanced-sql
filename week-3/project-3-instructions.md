# Project 3 Instructions

The Virtual Kitchen developers are making some changes to the search functionality on the website. After gathering customer feedback, they want to change the recipe suggestion algorithm in order to improve the customer experience.

We have a beta version of the website available and have opened it for use by a small number of customers. Next week we plan to increase this number from 200 customers to 5,000. To ensure everything is ready for the test, we have implemented logging and are saving results to a table in Snowflake called vk_data.events.website_activity.

The table contains: 

- event_id: A unique identifier for the user action on the website
- session_id: The identifier for the user session
- user_id: The identifier for the logged-in user
- event_timestamp: Time of the event
- event_details: Details about the event in JSON — what action was performed by the user?

Once we expand the beta version, we expect the website_activity table to grow very quickly. While it is still fairly small, we need to develop a query to measure the impact of the changes to our search algorithm. Please create a query and review the query profile to ensure that the query will be efficient once the activity increases.

We want to create a daily report to track:

- Total unique sessions
- The average length of sessions in seconds
- The average number of searches completed before displaying a recipe 
- The ID of the recipe that was most viewed 

In addition to your query, please submit a short description of what you determined from the query profile and how you structured your query to plan for a higher volume of events once the website traffic increases.