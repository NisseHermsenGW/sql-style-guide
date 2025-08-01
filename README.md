
# SQL style guide

This style guide is based on the work of [Matt Mazur](https://mattmazur.com/) and specifically tailored to the requirements of the
[DBT](https://www.getdbt.com/) warehouse platform. [SQLFluff](https://sqlfluff.com/) settings for a DBT project can be found in `.sqlfluff`.

TODO:
* prefix generic column names with table name, e.g. `payment_amount` instead of `amount`, to avoid confusion between similar metrics based on entity?

## Example

Here's a non-trivial query to give you an idea of what this style guide looks like in the practice:

```sql
with hubspot_interest as (

    select
        -- ids
        contact_id,

        -- properties
        email,
        timestamp_millis(property_beacon_interest) as expressed_interest_at
    
    from hubspot.contact
    
    where 
        property_beacon_interest is not null

), 

support_interest as (

    select
        -- ids
        conversation.conversation_id,

        -- properties
        conversation.email,
        conversation.created_at as expressed_interest_at

    from helpscout.conversation
    inner join helpscout.conversation_tag using(conversation_id)
    
    where
        conversation_tag.tag = 'beacon-interest'

), 

combined_interest as (

    select * from hubspot_interest
    union all
    select * from support_interest

),

first_interest as (

    select
        email,
        min(expressed_interest_at) as expressed_interest_at
    
    from combined_interest
    
    group by 1

)

select * from first_interest
```

In addition, here's a example for a DBT project (including [Jinja](https://jinja.palletsprojects.com/en/stable/) invocation):

```sql
with customers as (

    select * from {{ ref('stg_jaffle_shop__customers') }}

),

orders as (

    select * from {{ ref('stg_jaffle_shop__orders') }}

),

orders_grouped_by_customer as (

    select
        -- ids
        customer_id,

        -- properties
        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_data,
        count(order_id) as number_of_orders

    from orders

    group by 1

),

customers_joined_on_orders as (

    select
        -- ids
        customers.customer_id,

        -- properties
        customers.first_name,
        customers.last_name,

        coalesce(orders_grouped_by_customer.number_of_orders, 0)
            as number_of_orders,

        orders_grouped_by_customer.first_order_date,
        orders_grouped_by_customer.most_recent_order_data

    from customers
    left join orders_grouped_by_customer using (customer_id)
    
    where
        customers.lifetime_value > 1000

)

select * from customers_joined_on_orders
```


## Guidelines

### Use lowercase SQL

It's just as readable as uppercase SQL and you won't have to constantly be holding down a shift key.

```sql
-- Good
select * from users

-- Bad
SELECT * FROM users

-- Bad
Select * From users
```

### Put each selected column on its own line

When selecting columns, always put each column name on its own line and never on the same line as `select`. For multiple columns, it's easier to read when each column is on its own line. And for single columns, it's easier to add additional columns without any reformatting (which you would have to do if the single column name was on the same line as the `select`).

```sql
-- Good
select 
    id

from users 

-- Good
select 
    id,
    email

from users 

-- Bad
select id
from users 

-- Bad
select id, email
from users 
```

### Put whiteline before `from`, `where` and `order` / `group by`:

Group the sources, condition and grouping / ordering sections of code by adding whitelines between them.

```sql
-- Good
select
    customers.name,
    sum(orders.amount) as total_large_spendings

from orders
inner join customers using (customer_id)

where
    orders.amount > 1000

group 1
order by 2

-- Bad
select
    customers.name,
    sum(orders.amount) as total_large_spendings
from orders
inner join customers using (customer_id)
where
    orders.amount > 1000
group 1
order by 2

```

### Annotate attributes by group

For complex and/or long queries, group and annotate the table attributes by the following types:
* `ids`, includes PK, Surrogate Keys and FKs.
* `properties`, includes the remaining existing and derived attributes.

```sql
-- Good
select
    -- ids
    product_id,
    supplier_id
    
    -- properties
    product_name,
    product_type,
    product_description,
    (price / 100.0)::float as product_price,

    case
        when type = 'jaffle' then 1
        else 0
    end as is_food_item,
    
    case
        when type = 'beverage' then 1
        else 0
    end as is_drink_item

from source

-- Okay
select
    product_id,
    supplier_id
    product_name,
    product_type,
    product_description,
    (price / 100.0)::float as product_price,

    case
        when type = 'jaffle' then 1
        else 0
    end as is_food_item,
    
    case
        when type = 'beverage' then 1
        else 0
    end as is_drink_item

from source

```

### `select *`

When selecting `*` it's fine to include the `*` next to the `select` and also fine to include the `from` on the same line, assuming no additional complexity like `where` conditions:

```sql
-- Good
select * from users 

-- Good too
select *
from users

-- Bad
select * from users where email = 'name@example.com'
```

### Indenting where conditions

Similarly, conditions should always be spread across multiple lines to maximize readability and make them easier to add to. Operators should be placed at the end of each line:

```sql
-- Good
select * from users

where 
    email = 'example@domain.com'

-- Good
select * from users

where 
    email like '%@domain.com' and 
    created_at >= '2021-10-08'

-- Bad
select * from users
         
where email = 'example@domain.com'

-- Bad
select * from users
         
where 
    email like '%@domain.com' and created_at >= '2021-10-08'

-- Bad
select * from users
         
where 
    email like '%@domain.com' 
    and created_at >= '2021-10-08'
```

### Left align SQL keywords

Some IDEs have the ability to automatically format SQL so that the spaces after the SQL keywords are vertically aligned. This is cumbersome to do by hand (and in my opinion harder to read anyway) so I recommend just left aligning all of the keywords:

```sql
-- Good
select 
    id,
    email

from users

where 
    email like '%@gmail.com'

-- Bad
select id, email

  from users
  
 where email like '%@gmail.com'
```

### Use single quotes

Some SQL dialects like BigQuery support using double quotes, but for most dialects double quotes will wind up referring to column names. For that reason, single quotes are preferable:

```sql
-- Good
select * from users
         
where 
    email = 'example@domain.com'

-- Bad
select * from users
         
where 
    email = "example@domain.com"
```

If your SQL dialect supports double quoted strings and you prefer them, just make sure to be consistent and not switch between single and double quotes.

### Use `!=` over `<>`

Simply because `!=` reads like "not equal" which is closer to how we'd say it out loud.

```sql
-- Good
select 
    count(*) as paying_users_count

from users

where 
    plan_name != 'free'
```

### Commas should be at the the end of lines

```sql
-- Good
select
    id,
    email

from users

-- Bad
select
    id
    , email

from users
```

While the commas-first style does have some practical advantages (it's easier to spot missing commas and results in cleaner diffs), I'm just not a huge fan of how they look so prefer commas-last.

### Avoid spaces inside of parenthesis

```sql
-- Good
select * from users
         
where 
    id in (1, 2)

-- Bad
select * from users
         
where 
    id in ( 1, 2 )
```

### Break long lists of `in` values into multiple indented lines

```sql
-- Good
select * from users
         
where 
    email in (
        'user-1@example.com',
        'user-2@example.com',
        'user-3@example.com',
        'user-4@example.com'
    )
```

### Table names should be a plural snake case of the noun

```sql
-- Good
select * from users

-- Good
select * from visit_logs

-- Bad
select * from user

-- Bad
select * from visitLog
```

### Column names should be snake_case

```sql
-- Good
select
    id,
    email,
    timestamp_trunc(created_at, month) as signup_month

from users

-- Bad
select
    id,
    email,
    timestamp_trunc(created_at, month) as SignupMonth

from users
```

### Column name conventions

* ID fields should be suffixed with `_id`. For example, `customer_id`, `order_id`. **_This should be true for both PK and FK fields._**
* Boolean fields should be prefixed with `is_`, `has_`, or `does_`. For example, `is_customer`, `has_unsubscribed`, etc.
* Date-only fields should be suffixed with `_date`. For example, `report_date`.
* Date+time fields should be suffixed with `_at`. For example, `created_at`, `posted_at`, etc.

### Column order conventions

Put the primary key first, followed by foreign keys, then by all other columns. If the table has any system columns (`created_at`, `updated_at`, `is_deleted`, etc.), put those last.

```sql
-- Good
select
    id,
    name,
    created_at

from users

-- Bad
select
    created_at,
    name,
    id,

from users
```

### Use `using` in simple key to key joins instead of `on`:

```sql
-- Good
select
    ...

from orders
inner join customers using (customer_id)

-- Bad
select
    ...

from orders
inner join customers on orders.customer_id = customers.customer_id

```

### Include `inner` for inner joins

Better to be explicit so that the join type is crystal clear:

```sql
-- Good
select
    users.email,
    sum(charges.amount) as total_revenue

from users
inner join charges on users.id = charges.user_id

-- Bad
select
    users.email,
    sum(charges.amount) as total_revenue

from users
join charges on users.id = charges.user_id
```

### For join conditions, put the table that was referenced first immediately after the `on`

By doing it this way it makes it easier to determine if your join is going to cause the results to fan out:

```sql
-- Good
select
    ...

from users
left join charges on users.id = charges.user_id
-- primary_key = foreign_key --> one-to-many --> fanout
  
select
    ...

from charges
left join users on charges.user_id = users.id
-- foreign_key = primary_key --> many-to-one --> no fanout

-- Bad
select
    ...

from users
left join charges on charges.user_id = users.id
```

### Single join conditions should be on the same line as the join

```sql
-- Good
select
    users.email,
    sum(charges.amount) as total_revenue

from users
inner join charges on users.id = charges.user_id

group by 1

-- Bad
select
    users.email,
    sum(charges.amount) as total_revenue

from users
inner join charges
on users.id = charges.user_id

group by 1
```

When you have multiple join conditions, place each one on their own indented line:

```sql
-- Good
select
    users.email,
    sum(charges.amount) as total_revenue

from users
inner join charges on 
    users.id = charges.user_id and
    refunded = false

group by 1
```

### Avoid aliasing table names most of the time

It can be tempting to abbreviate table names like `users` to `u` and `charges` to `c`, but it winds up making the SQL less readable:

```sql
-- Good
select
    users.email,
    sum(charges.amount) as total_revenue

from users
inner join charges on users.id = charges.user_id

-- Bad
select
    u.email,
    sum(c.amount) as total_revenue

from users u
inner join charges c on u.id = c.user_id
```

Most of the time you'll want to type out the full table name.

There are two exceptions:

If you need to join to a table more than once in the same query and need to distinguish each version of it, aliases are necessary.

Also, if you're working with long or ambiguous table names, it can be useful to alias them (but still use meaningful names):

```sql
-- Good: Meaningful table aliases
select
  companies.com_name,
  beacons.created_at

from stg_mysql_helpscout__helpscout_companies companies
inner join stg_mysql_helpscout__helpscout_beacons_v2 beacons on companies.com_id = beacons.com_id

-- OK: No table aliases
select
  stg_mysql_helpscout__helpscout_companies.com_name,
  stg_mysql_helpscout__helpscout_beacons_v2.created_at

from stg_mysql_helpscout__helpscout_companies
inner join stg_mysql_helpscout__helpscout_beacons_v2 on stg_mysql_helpscout__helpscout_companies.com_id = stg_mysql_helpscout__helpscout_beacons_v2.com_id

-- Bad: Unclear table aliases
select
  c.com_name,
  b.created_at

from stg_mysql_helpscout__helpscout_companies c
inner join stg_mysql_helpscout__helpscout_beacons_v2 b on c.com_id = b.com_id
```

### Include the table when there is a join, but omit it otherwise

When there are no join involved, there's no ambiguity around which table the columns came from so you can leave the table name out:

```sql
-- Good
select
    id,
    name

from companies

-- Bad
select
    companies.id,
    companies.name

from companies
```

But when there are joins involved, it's better to be explicit so it's clear where the columns originated:

```sql
-- Good
select
    users.email,
    sum(charges.amount) as total_revenue

from users
inner join charges on users.id = charges.user_id

-- Bad
select
    email,
    sum(amount) as total_revenue

from users
inner join charges on users.id = charges.user_id

```

### Always rename aggregates and function-wrapped arguments

```sql
-- Good
select 
    count(*) as total_users

from users

-- Bad
select 
    count(*)

from users

-- Good
select 
    timestamp_millis(property_beacon_interest) as expressed_interest_at

from hubspot.contact

where 
    property_beacon_interest is not null

-- Bad
select
    timestamp_millis(property_beacon_interest)

from hubspot.contact

where
    property_beacon_interest is not null
```

### Be explicit in boolean conditions

```sql
-- Good
select * from customers 
         
where 
    is_cancelled = true

-- Good
select * from customers 
         
where 
    is_cancelled = false

-- Bad
select * from customers 

where 
    is_cancelled

-- Bad
select * from customers 

where 
    not is_cancelled
```

### Use `as` to alias column names

```sql
-- Good
select
    id,
    email,
    timestamp_trunc(created_at, month) as signup_month

from users

-- Bad
select
    id,
    email,
    timestamp_trunc(created_at, month) signup_month

from users
```

### Group using column names or numbers, but not both

I prefer grouping by number.

```sql
-- Good
select 
    user_id, 
    count(*) as total_charges

from charges

group by 1

-- Ok
select 
    user_id, 
    count(*) as total_charges

from charges

group by user_id

-- Bad
select
    timestamp_trunc(created_at, month) as signup_month,
    vertical,
    count(*) as users_count

from users

group by 1, vertical
```

### Take advantage of lateral column aliasing when grouping by name

Only when grouping by column names.

```sql
-- Good
select
  timestamp_trunc(com_created_at, year) as signup_year,
  count(*) as total_companies

from companies

group by signup_year

-- Bad
select
  timestamp_trunc(com_created_at, year) as signup_year,
  count(*) as total_companies

from companies

group by timestamp_trunc(com_created_at, year)
```

### Grouping columns should go first

```sql
-- Good
select
  timestamp_trunc(com_created_at, year) as signup_year,
  count(*) as total_companies

from companies

group by signup_year

-- Bad
select
  count(*) as total_companies,
  timestamp_trunc(com_created_at, year) as signup_year

from mysql_helpscout.helpscout_companies

group by signup_year
```

### Aligning case/when statements

Each `when` should be on its own line (nothing on the `case` line) and should be indented one level deeper than the `case` line. The `then` can be on the same line or on its own line below it, just aim to be consistent.
Furthermore, each case statement should be surrounded by whitelines for readability.

```sql
-- Good
select
    event_id,
    location,
    
    case
        when event_name = 'viewed_homepage' then 'Homepage'
        when event_name = 'viewed_editor' then 'Editor'
        else 'Other'
    end as page_name

from events

-- Good too
select
    event_id,
    location,
    
    case
        when event_name = 'viewed_homepage'
            then 'Homepage'
        when event_name = 'viewed_editor'
            then 'Editor'
        else 'Other'            
    end as page_name

from events

-- Bad 
select
    event_id,
    location,
    case when event_name = 'viewed_homepage' then 'Homepage'
        when event_name = 'viewed_editor' then 'Editor'
        else 'Other'        
    end as page_name

from events
```

### Use CTEs, not subqueries

Avoid subqueries; CTEs will make your queries easier to read and reason about.

When using CTEs, pad the query with new lines. 

If you use any CTEs, always `select *` from the last CTE at the end. That way you can quickly inspect the output of other CTEs used in the query to debug the results.

Closing CTE parentheses should use the same indentation level as `with` and the CTE names.

```sql
-- Good
with ordered_details as (

    select
        user_id,
        name,
        row_number() over (partition by user_id order by date_updated desc) as details_rank
    
    from billingdaddy.billing_stored_details

),

first_updates as (

    select 
        user_id, 
        name
    
    from ordered_details
    
    where 
        details_rank = 1

)

select * from first_updates

-- Bad
select 
    user_id, 
    name

from (
    select
        user_id,
        name,
        row_number() over (partition by user_id order by date_updated desc) as details_rank
    
    from billingdaddy.billing_stored_details
) ranked

where 
    details_rank = 1
```

### Use meaningful CTE names

```sql
-- Good
with ordered_details as (

-- Bad
with d1 as (
```

### Window functions

Leave it all on its own line:

```sql
-- Good
select
    user_id,
    name,
    row_number() over (partition by user_id order by date_updated desc) as details_rank

from billingdaddy.billing_stored_details

-- Okay
select
    user_id,
    name,
    row_number() over (
        partition by user_id
        order by date_updated desc
    ) as details_rank

from billingdaddy.billing_stored_details
```

## Credits

This style guide was inspired in part by:

* [Fishtown Analytics' dbt Style Guide](https://github.com/fishtown-analytics/corp/blob/master/dbt_coding_conventions.md#sql-style-guide)
* [KickStarter's SQL Style Guide](https://gist.github.com/fredbenenson/7bb92718e19138c20591)
* [GitLab's SQL Style Guide](https://about.gitlab.com/handbook/business-ops/data-team/sql-style-guide/)
