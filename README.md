# SQL-Project-Polina-Schulz
Hello, I am Polina Schulz, Data Analyst

Welcome to my SQL project!

In this repository, I present an SQL project that demonstrates my skills in working with SQL.

The project is structured so that you can directly view my SQL code. Additionally, I have included screenshots that display the results of the SQL queries.

The source data is publicly available in Google BigQuery and can be viewed live there.

Enjoy exploring, and feel free to reach out if you have any questions!

1) To begin, I will outline the data structure of the file.


select *   
from `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*` as ga4   
limit 10   
;
<img width="2579" height="928" alt="image" src="https://github.com/user-attachments/assets/e9e7c314-9761-4e1a-a6c4-cb1c13e06f7f" />

2) Filtering the data to include only the year 2021 and specific selected events.

select
timestamp_micros(event_timestamp) as event_timestamp
, user_pseudo_id 
, (select value.int_value from ga4.event_params where key = 'ga_session_id') as session_id
, event_name as event_name
, geo.country as country
, device.category as device_category
, traffic_source.source as source
, traffic_source.medium as medium
, traffic_source.name as campaign
from `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*` as ga4
where 
_TABLE_SUFFIX between '20210101' and '20211231'
and event_name in ('session_start', 'view_item', 'add_to_cart', 'begin_checkout', 'add_shipping_info', 'add_payment_info', 'purchase')
;

<img width="1470" height="921" alt="image" src="https://github.com/user-attachments/assets/5d967c4e-cc2c-4f22-9d84-3ff936801b96" />

3) Calculation of conversions by date and traffic channels.
with user_session_mapping as (
  select 
timestamp_micros(event_timestamp) as event_timestamp
, user_pseudo_id
|| cast((select value.int_value from ga4.event_params where key = 'ga_session_id') as string) as user_session_id
, event_name as event_name
, traffic_source.source as source
, traffic_source.medium as medium
, traffic_source.name as campaign
from `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*` as ga4
where 
event_name in ('session_start', 'add_to_cart', 'begin_checkout', 'purchase')
),

event_count as (
select
date (event_timestamp) as event_date
, source
, medium
, campaign
, count(distinct user_session_id) as user_sessions_count
, count (distinct case when event_name ='add_to_cart' then user_session_id end) as count_add_to_cart
, count (distinct case when event_name ='begin_checkout' then user_session_id end) as count_begin_checkout
, count (distinct case when event_name ='purchase' then user_session_id end) as count_purchase
from user_session_mapping
group by 1,2,3,4
)
select
event_date
, source
, medium
, campaign
, user_sessions_count
, count_add_to_cart / user_sessions_count as visit_to_cart
, count_begin_checkout / user_sessions_count as visit_to_checkout
, count_purchase / user_sessions_count as visit_to_purchase
from event_count
;
<img width="1669" height="882" alt="image" src="https://github.com/user-attachments/assets/072965b1-ac58-4bcd-886c-6beb86cc0f5d" />

4) Comparison of conversion rates between different landing pages.
with page_location as (

select
user_pseudo_id ||'_'||
cast((select value.int_value from unnest (event_params) where key = 'ga_session_id')as string) as user_session_id
, (select value.string_value from unnest (event_params) where key = 'page_location') as page_location_full_url
, (select value.int_value from unnest (event_params) where key = 'ga_session_id') as session_id
from `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where event_name ='session_start'
and  _TABLE_SUFFIX between '20200101' and '20201231'
), 

purchases as (
select
user_pseudo_id ||'_'||
cast((select value.int_value from unnest (event_params) where key = 'ga_session_id')as string) as user_session_id
, (select value.string_value from unnest (event_params) where key = 'page_location') as page_location_full_url
, (select value.int_value from unnest (event_params) where key = 'ga_session_id') as session_id
from `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where event_name ='purchase'
and  _TABLE_SUFFIX between '20200101' and '20201231'
)
select 
regexp_extract(pl.page_location_full_url, r"https?://[^/]+(/[^?#]*)") as page_path
, count (distinct pl.user_session_id) as number_unique_user_sessions
, count (distinct p.user_session_id) as number_purchases_for_unique_u_s
, count (distinct p.user_session_id) / count (distinct pl.user_session_id) as visit_to_purchase
from page_location as pl
left join purchases as p using (user_session_id)
where pl.page_location_full_url is not null
group by page_path
order by 2 desc
;
<img width="1062" height="852" alt="image" src="https://github.com/user-attachments/assets/f2d326db-0f8f-4b99-a01a-ffa06c775bd4" />

5) Checking the correlation between user engagement and purchases.
with user_sessions as (
  select
    user_pseudo_id ||
      cast((select value.int_value from unnest(event_params) where key = 'ga_session_id') as string)
      as user_session_id
    ,sum(coalesce(
      (select value.int_value from unnest(event_params) where key = 'engagement_time_msec'),0))
      as total_engagement_time
    ,case 
      when 
        sum(coalesce(
          safe_cast(
          (select value.string_value from unnest(event_params) where key = 'session_engaged') as integer), 0)
        ) > 0
      then 1
      else 0 end 
      as is_session_engaged
  from `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*` e
  group by 1
  order by 3 desc
), 

purchases as (
  select
    user_pseudo_id ||
      cast((select value.int_value from unnest(event_params) where key = 'ga_session_id') as string)
      as user_session_id
  from `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*` e
  where event_name = 'purchase'
)
select
  corr(s.total_engagement_time, case when p.user_session_id is not null then 1 else 0 end) as corr_engagement_time_to_purchase
  , corr(s.is_session_engaged,case when p.user_session_id is not null then 1 else 0 end) as corr_engaged_session_to_purchase
from user_sessions as s
left join purchases as p using (user_session_id)
;

<img width="980" height="242" alt="image" src="https://github.com/user-attachments/assets/245fa882-b969-4447-9533-abcf2af6a75f" />

