# sql-use-joins-without-loops

The situation is we need to obtain special order IDs from table ```order_special``` in the Oracle database first; then find these orders' corresponding information from ```order_summary``` and ```order_detail```, and obtain the orders of each customer to within 7 days from the first order date; finaly calculate the total sales.

I found an original SQL code as following.
```sql
declare
  total_sales integer;
  sales integer;

begin
  
  total_sales := 0;
  
  for i_row in (
    select sm.customer_id, min(sm.order_date) as order_date
    from order_summary sm
    left join order_detail det
      on sm.order_id = det.order_id
    where sm.order_id in (select distinct order_id from order_special where {conditions_special})
      and {conditions_summary}
      and {conditions_detail}
    group by sm.customer_id
    )
  loop
    
    select nvl(sum(det.sales), 0) into sales
    from order_detail det
    left join order_summary sm
      on det.order_id = sm.order_id
    where sm.customer_id = i_row.customer_id
      and sm.order_date >= i_row.order_date
      and sm.order_date <= to_char(to_date(i_row.order_date, 'yyyy/mm/dd') + 6, 'yyyy/mm/dd')
      and {conditions_summary}
      and {conditions_detail}
    ;
    
    total_sales := total_sales + sales;
    
  end loop;
  
  dbms_output.put_line('total sales = ' || total_sales);
  
end;
```


However, this can be done simply by a query with joins instead of loops.
```sql
select sum(det.sales) as total_sales
from
  (
  select sm.customer_id,
    min(sm.order_date) as order_date_1st,
    to_char( to_date( min( sm.order_date ), 'yyyy/mm/dd' ) + 6, 'yyyy/mm/dd' ) as order_date_7days
  from
    (
    select distinct order_id
    from order_special
    where {conditions_special}
    ) sp
  inner join order_summary sm
    on sp.order_id = sm.order_id
  inner join order_detail det
    on sp.order_id = det.order_id
  where 1=1
    and {conditions_summary}
    and {conditions_detail}
  group by sm.customer_id
  ) id
inner join order_summary sm
  on 1=1
    and id.customer_id = sm.customer_id
    and sm.order_date between id.order_date_1st and id.order_date_7days
inner join order_detail det
  on sm.order_id = det.order_id
where 1=1
  and {conditions_summary}
  and {conditions_detail}
```

Additionally, using subqueries within joins would be more efficient than those within where clauses.
