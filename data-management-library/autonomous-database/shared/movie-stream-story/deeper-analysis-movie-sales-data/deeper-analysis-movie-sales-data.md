﻿# Deeper Analysis of Movie Sales Data

## Introduction

**This lab is optional**. This lab is aimed at people who are accustomed to working with spreadsheets and are comfortable creating sophisticated formulas within their worksheets. In this lab we explore how to use the **`SQL MODEL`** clause to make SQL more spreadsheet-like in terms of inserting new rows and new calculations into a query.

Estimated Lab Time: 10 minutes

### Objectives

- Learn how to combine existing rows within a query to create new rows

- Understand how to define new calculations using a spreadsheet-like syntax


### Going A Little Deeper

Sometimes we will find that the data is organized in just the way we want it to be! In many cloud-based data warehouses, we are locked in to only viewing data in terms of the way it is stored. Making changes to the structure and organization of our data when it's in a spreadsheet is really easy - we can insert new rows and new columns to add new content.

Wouldn't it be great if our data warehouse offered the flexibility to add completely new rows to our data set that were derived from existing rows - effectively giving us the chance to build our own  **dimension**  values. In general data warehousing terminology this is known as adding **custom aggregates** to our result set.

What if we want to group the days of week into two new custom aggregates, effectively adding two new rows within our query?

- **new row 1 - Weekday** which consists of values for Tuesday(#3), Wednesday(#4), Thursday(#5)

- **new row 2 - Long Weekend** which consists of values for Monday(#2), Friday(#6), Saturday(#7) and Sunday(#1)

## Task 1: Revenue Analysis by Weekdays vs. Long Weekends

 **NOTE:** Different regions organize their day numbers in different ways. Oracle Database provides session settings that allow you to control these types of regional differences. In Germany, for example, the week starts on Monday, so that day is assigned as day number one.

1. Set our territory as being “United Kingdom” by using the following command:

    ```
    <copy>ALTER SESSION SET NLS_TERRITORY = "United Kingom";</copy>
    ```

2. Run the following query to see which day of the week is Monday:

   ```
    <copy>SELECT to_char(date'2018-01-01', 'd') day_number
    FROM dual;</copy>
    ```

    It will return the value of 1.

    This result is specific to the region United Kingdom where the start of the week is Monday.  In the United States, the day numbers start at one on Sunday. Therefore, it’s important to understand these regional differences. We can set the region to control the start of the week by using the **`ALTER SESSION SET`** command.

3. Set our territory as being “America” by using the following command:

    ```
    <copy>ALTER SESSION SET NLS_TERRITORY = America;</copy>
    ```

4. We can now check whether Monday is still the start of the week by simply re-running the same query:

    ```
    <copy>SELECT to_char(date'2018-01-01', 'd') day_number
    FROM dual;</copy>
    ```

    It will now return the value of 2.

    This is because in America the week starts on Sunday, making Monday day 2.

## Task 2: Revenue Analysis by Weekdays vs. Long Weekends

Now we know which day is the first day of the week we can move on. In spreadsheets, we can refer to values by referencing the row + column position such as A1 + B2. This would allow us to see more clearly the % contribution provided by each grouping so we can get some insight into the most heavily trafficked days for movie-watching. How can we do this?

1. Autonomous Data Warehouse has a unique SQL feature called the **`MODEL`** clause which creates a spreadsheet-like modeling framework over our data. If we tweak and extend the last query we can use the MODEL clause to add completely new rows (**Weekday** and **Long Weekend**) into our results:

    ```
    <copy>SELECT
    quarter_name,
    day_name,
    revenue
    FROM
    (SELECT
    quarter_name,
    day_dow,
    ROUND(SUM(actual_price),0) AS revenue
    FROM vw_movie_sales_fact
    WHERE year_name = '2020'
    GROUP BY quarter_name, day_name, day_dow
    ORDER BY quarter_name, day_dow)
    MODEL
    PARTITION BY (quarter_name)
    DIMENSION BY (day_dow)
    MEASURES(revenue revenue, 'Long Weekend' day_name, 0 contribution)
    RULES(
    revenue[8] = revenue[3]+revenue[4]+revenue[5],
    revenue[9] = revenue[1]+revenue[2]+revenue[6]+revenue[7],
    day_name[1] = 'Sunday',
    day_name[2] = 'Monday',
    day_name[3] = 'Tuesday',
    day_name[4] = 'Wednesday',
    day_name[5] = 'Thursday',
    day_name[6] = 'Friday',
    day_name[7] = 'Saturday',
    day_name[8] = 'Weekday',
    day_name[9] = 'Long Weekend'
    )
    ORDER BY quarter_name, day_dow;</copy>
    ```

2. This will generate the following output:

    ![Result of query using MODEL clause](images/lab-5b-step-2-substep-2.png " ")

See how easy it is to build upon existing discoveries using SQL to extend our understanding of the data! The concept of being able to add new rows using a spreadsheet-like approach within SQL is unique to Oracle. The MODEL clause creates two new rows that we identify as **day 8** and **day 9**. These new rows are assigned names -  day\_name\[8\] = 'Weekday' and day\_name\[9\] = 'Long Weekend'. The calculation of revenue for these two new rows uses a similar approach to many spreadsheets: revenue for day \[8\] is derived from adding together revenue for day \[3\]+ revenue for \[4\] + revenue for day \[5\].

## Task 3: Revenue and Contribution Analysis by Weekdays vs. Long Weekends

If we tweak and extend the last query we can expand the MODEL clause to also calculate contribution using a similar syntax to a spreadsheet:

    contribution[1] = trunc((revenue[1])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2)

This statement calculates the contribution for Sunday (day 1) by taking the revenue for day **1** and dividing it by the revenue from each of the seven days .

1. Run the following query to calculate revenue and contribution for each day including the new rows (Long Weekend and Weekday):

    ```
    <copy>SELECT
    quarter_name,
    day_name,
    contribution
    FROM
    (SELECT
    quarter_name,
    day_dow,
    SUM(actual_price) AS revenue
    FROM vw_movie_sales_fact
    WHERE year_name = '2020'
    GROUP BY quarter_name, day_name, day_dow
    ORDER BY quarter_name, day_dow)
    MODEL
    PARTITION BY (quarter_name)
    DIMENSION BY (day_dow)
    MEASURES(revenue revenue, 'Long Weekend' day_name, 0 contribution)
    RULES(
    revenue[8] = revenue[3]+revenue[4]+revenue[5],
    revenue[9] = revenue[1]+revenue[2]+revenue[6]+revenue[7],
    contribution[1] = trunc((revenue[1])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2),
    contribution[2] = trunc((revenue[2])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2),
    contribution[3] = trunc((revenue[3])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2),
    contribution[4] = trunc((revenue[4])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2),
    contribution[5] = trunc((revenue[5])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2),
    contribution[6] = trunc((revenue[6])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2),
    contribution[7] = trunc((revenue[7])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2),
    contribution[8] = trunc((revenue[3]+revenue[4]+revenue[5])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2),
    contribution[9] = trunc((revenue[1]+revenue[2]+revenue[6]+revenue[7])/(revenue[1]+revenue[2]+revenue[3]+revenue[4]+revenue[5]+revenue[6]+revenue[7])*100,2),
    day_name[2] = 'Monday',
    day_name[3] = 'Tuesday',
    day_name[4] = 'Wednesday',
    day_name[5] = 'Thursday',
    day_name[6] = 'Friday',
    day_name[7] = 'Saturday',
    day_name[1] = 'Sunday',
    day_name[8] = 'Weekday',
    day_name[9] = 'Long Weekend'
    )
    ORDER BY quarter_name, day_dow;</copy>
    ```

2. This will generate the following output:

    ![Result of query using MODEL clause](images/lab-5b-step-3-substep-2.png " ")


3. As with earlier examples, we can pivot the results and the final pivoted version of our code looks like this:

    ```
    <copy>SELECT *
    FROM
    (SELECT
    quarter_name,
    day_dow,
    day_name,
    contribution
    FROM
    (SELECT
    quarter_name,
    day_dow,
    SUM(actual_price) AS revenue
    FROM vw_movie_sales_fact
    WHERE year_name = '2020'
    GROUP BY quarter_name, day_name, day_dow
    ORDER BY quarter_name, day_dow)
    MODEL
    PARTITION BY (quarter_name)
    DIMENSION BY (day_dow)
    MEASURES(revenue revenue, 'Long Weekend' day_name, 0 contribution)
    RULES(
    revenue[8] = revenue[2]+revenue[3]+revenue[4],
    revenue[9] = revenue[1]+revenue[5]+revenue[6]+revenue[7],
    contribution[1] = trunc((revenue[1])/(revenue[1]+revenue[5]+revenue[6]+revenue[7]+revenue[2]+revenue[3]+revenue[4])*100,2),
    contribution[2] = trunc((revenue[2])/(revenue[1]+revenue[5]+revenue[6]+revenue[7]+revenue[2]+revenue[3]+revenue[4])*100,2),
    contribution[3] = trunc((revenue[3])/(revenue[1]+revenue[5]+revenue[6]+revenue[7]+revenue[2]+revenue[3]+revenue[4])*100,2),
    contribution[4] = trunc((revenue[4])/(revenue[1]+revenue[5]+revenue[6]+revenue[7]+revenue[2]+revenue[3]+revenue[4])*100,2),
    contribution[5] = trunc((revenue[5])/(revenue[1]+revenue[5]+revenue[6]+revenue[7]+revenue[2]+revenue[3]+revenue[4])*100,2),
    contribution[6] = trunc((revenue[6])/(revenue[1]+revenue[5]+revenue[6]+revenue[7]+revenue[2]+revenue[3]+revenue[4])*100,2),
    contribution[7] = trunc((revenue[7])/(revenue[1]+revenue[5]+revenue[6]+revenue[7]+revenue[2]+revenue[3]+revenue[4])*100,2),
    contribution[8] = trunc((revenue[3]+revenue[4]+revenue[5])/(revenue[1]+revenue[5]+revenue[6]+revenue[7]+revenue[2]+revenue[3]+revenue[4])*100,2),
    contribution[9] = trunc((revenue[1]+revenue[2]+revenue[6]+revenue[7])/(revenue[1]+revenue[5]+revenue[6]+revenue[7]+revenue[2]+revenue[3]+revenue[4])*100,2),
    day_name[2] = 'Monday',
    day_name[3] = 'Tuesday',
    day_name[4] = 'Wednesday',
    day_name[5] = 'Thursday',
    day_name[6] = 'Friday',
    day_name[7] = 'Saturday',
    day_name[1] = 'Sunday',
    day_name[8] = 'Weekday',
    day_name[9] = 'Long Weekend')
    ORDER BY quarter_name, day_dow)
    PIVOT
    (
    SUM(contribution) contribution
    FOR quarter_name IN('Q1-2020' as "Q1", 'Q2-2020' as "Q2", 'Q3-2020' as "Q3", 'Q4-2020' as "Q4")
    )
    ORDER BY day_dow;</copy>
    ```

4. The final output looks like this, where we can now see that over 60% of revenue is generated over those days within a Long Weekend! Conversely, the other three days in our week (Tuesday, Wednesday, Thursday) are generating nearly 40% of our weekly revenue, which means that on work/school nights we are still seeing strong demand for streaming movies. This type of information might be useful for our infrastructure team so they can manage their resources more effectively and our marketing team could use this information to help them drive new campaigns.

    ![Final query output using Pivot](images/lab-5b-step-3-substep-4.png " ")


### Recap

Let's quickly recap what has been covered in this lab:

- Explored power of Oracle's built in spreadsheet-like SQL Model clause to add new rows to our results

- Learned how to combine spreadsheet-like operations with other SQL features such as PIVOT


Please *proceed to the next lab*.

## **Acknowledgements**

- **Author** - Keith Laker, ADB Product Management
- **Adapted for Cloud by** - Richard Green, Principal Developer, Database User Assistance
- **Last Updated By/Date** - Keith Laker, August 2021
