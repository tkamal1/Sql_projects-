
-- 1. How many unique customers are there?

	select  count(customer_id) as unique_customer_id
	    from
	(select customer_id
	from customer_joining_info
	group by customer_id ) as t;

-- 2. How many unique customers are coming from each region?

    select   region_id, count
        from
    (select  customer_id,region_id,
        count(region_id) over (partition by region_id) as count
    from
	(select customer_id,region_id
	from customer_joining_info ) as t) as t2
	    group by  region_id, count;

-- 3. How many unique customers are coming from each area?

	select   area_id,count_area
	    from
	(select customer_id,area_id,
	       count(area_id)over(partition by area_id) as count_area
	from customer_joining_info ) as t
	group by area_id, count_area;

-- 4. What is the total amount for each transaction type?

	select  txn_type,  total_amount
	    from
	(select txn_type ,txn_amount,
	       sum(txn_amount)over(partition by txn_type) as total_amount
	from customer_transactions) as t
	group by txn_type, total_amount;

-- 5. For each month - how many customers make more than 1 deposit and 1 withdrawal in a single month?


	select month, COUNT(customer_id) AS num_customers
	    from
	(SELECT customer_id, month
	    from
	(select customer_id, month(txn_date)as month,txn_type
	from customer_transactions) as t
	 WHERE txn_type = 'deposit'
	    GROUP BY customer_id, month
	    HAVING COUNT(*) > 1
	 INTERSECT
	 SELECT customer_id, month
	    from
	(select customer_id, month(txn_date)as month,txn_type
	from customer_transactions) as t1
	 WHERE txn_type = 'withdrawal'
	    GROUP BY customer_id, month
	    HAVING COUNT(*) > 1
	    ) as t2
	GROUP BY month
	order by month asc ;


-- 6. What is closing balance for each customer?


    select customer_id,
        sum( case
                    when   txn_type = 'deposit'
                        then +txn_amount
                    else
                         txn_amount
                    end ) as closing_balance
        from
	(select *,
	    dense_rank() over (partition by customer_id order by  customer_id,txn_date asc ) as rnk
	from customer_transactions) as t
	    group by customer_id;

-- 7.What is the closing balance for each customer at the end of the month?



	select customer_id,date_format(txn_date ,'%y-%m') as month,
	       sum( case
	                    when   txn_type = 'deposit'
	                        then +txn_amount
	                    else
	                         txn_amount
	                    end ) as closing_balance_of_the_month
	from customer_transactions
	group by customer_id,date_format(txn_date ,'%y-%m')
	order by customer_id asc;


-- 8. Please show the latest 5 days total withdraw amount.

	select sum(txn_amount) as sum
	    from
	(select  *
	FROM
	(
	  SELECT *, dense_rank() OVER (ORDER BY txn_date desc ) AS rownum
	
	  from customer_transactions
	  where txn_type like 'withdrawal'
	) AS subquery
	where rownum between 1 and 5 )as t;

-- 9. Find out the total deposit amount for every five days consecutive series. You can assume 1 week = 5 days
	-- Please show the result week wise total amount.


	select  week, sum
	    from
	(select txn_date,week,
	       sum(txn_amount)over(partition by week) as sum
	    from
	(select *,
	      ceiling(dense_rank() over ( order by txn_date)/5) as week
	from customer_transactions
	where  txn_type like 'deposit') as t) as t2
	group by  week, sum;


-- 10. Please compare every weeks total deposit amount by the following previous week.

	select week,(total_deposit-new_rnk) as compare
	    from
	(select t2.*,lag(total_deposit) over() as new_rnk
	    from
	(select week,sum(txn_amount) as total_deposit
	    from
	(select t.*,dense_rank() over (order by day ) as rnk,
	       dense_rank() over (order by day)/5 as divided_by_5,
	       ceiling( dense_rank() over (order by day)/5) as week
	    from
	(select *,
	       day(txn_date) as day
	from customer_transactions
	where txn_type ='deposit') as t ) as t1
	group by week )as t2 )as t3
