-- Part 2 - Segmentation



DROP TABLE IF EXISTS efood2022-357817.main_assessment.ck_orders_1;
CREATE TABLE efood2022-357817.main_assessment.ck_orders_1
AS


SELECT

a.user_id
, a.monetary_value
, a.avg_monetary_value
, a.frequency
, a.recency
, a.last_order_date
, a.max_order_date
, CASE 
	WHEN b.breakfast_flag >=1 THEN 1 ELSE 0 END AS breakfast_buyer_flag
, ntile(4) over (ORDER BY a.recency DESC) AS recency_decile
, ntile(4) over (ORDER BY a.frequency) AS frequency_decile
, ntile(4) over (ORDER BY a.avg_monetary_value) AS monetary_decile

FROM 

(SELECT 
user_id
, round(SUM(amount), 1) AS monetary_value
, round(AVG(amount), 1) AS avg_monetary_value
, COUNT(order_id) AS frequency
, MAX(order_timestamp) AS last_order_date
, (SELECT MAX(order_timestamp) FROM `efood2022-357817.main_assessment.orders1` ) AS max_order_date
, date_diff( (SELECT MAX(order_timestamp) FROM `efood2022-357817.main_assessment.orders1` ), MAX(order_timestamp) , DAY) AS recency

FROM `efood2022-357817.main_assessment.orders1`
GROUP BY 
user_id

) a


LEFT JOIN 
( 
SELECT 
a.user_id
, a.breakfast_flag

FROM 
(
SELECT user_id, SUM(CASE WHEN cuisine = 'Breakfast' THEN 1 ELSE 0 END) AS breakfast_flag  FROM `efood2022-357817.main_assessment.orders1`
GROUP BY user_id
) a

) b

ON a.user_id = b.user_id




DROP TABLE IF EXISTS efood2022-357817.main_assessment.ck_orders_2;
CREATE TABLE efood2022-357817.main_assessment.ck_orders_2
AS

SELECT 

a.*
, CASE 
	WHEN a.RFM_combination IN ('1111', '1112', '1113', '1121', '1122', '1123', '1124',  '1132', '1211', '1212', '1114', '1141', '1131') THEN 'almost_inactive_with_breakfast'
	WHEN a.RFM_combination IN ('0111', '0112', '0113', '0121', '0122', '0123', '0124', '0132', '0211', '0212', '0114', '0141', '0131') THEN 'almost_inactive_without_breakfast'
	
	WHEN a.RFM_combination IN ('1133', '1134', '1143', '1244', '1243', '1144') THEN 'big_spenders_slipping_away_with_breakfast'
	WHEN a.RFM_combination IN ('0133', '0134', '0143', '0244', '0243', '0144') THEN 'big_spenders_slipping_away_without_breakfast'
	
	WHEN a.RFM_combination IN ('1131', '1142', '1241', '1242', '1142', '1241', '1242') THEN 'low_spenders_slipping_away_with_breakfast'
	WHEN a.RFM_combination IN ('0131', '0142', '0241', '0242', '0142', '0241', '0242') THEN 'low_spenders_slipping_away_without_breakfast'

	WHEN a.RFM_combination IN ('1311', '1411', '1331', '1312', '1313', '1314', '1323', '1324', '1412', '1413', '1414', '1421', '1423', '1424' ) THEN 'new_customers_with_breakfast'
	WHEN a.RFM_combination IN ('0311', '0411', '0331', '0312', '0313', '0314', '0323', '0324', '0412', '0413', '0414', '0421', '0423', '0424' ) THEN 'new_customers_without_breakfast'

	WHEN a.RFM_combination IN ('1222', '1223', '1233', '1213', '1214', '1221', '1224', '1231', '1232', '1234') THEN 'potential_churners_with_breakfast'
	WHEN a.RFM_combination IN ('0222', '0223', '0233',  '0213', '0214', '0221', '0224', '0231', '0232', '0234') THEN 'potential_churners_without_breakfast'

	WHEN a.RFM_combination IN ( '1333', '1321', '1422', '1332', '1432', '1341', '1342',  '1322') THEN 'active_with_breakfast'
	WHEN a.RFM_combination IN ( '0333', '0321', '0422', '0332', '0432', '0341', '0342', '0322') THEN 'active_without_breakfast'


	WHEN a.RFM_combination IN ('1433', '1434', '1443', '1444', '1431', '1441', '1442', '1334', '1343', '1344') THEN 'loyal_with_breakfast'
	WHEN a.RFM_combination IN ('0433', '0434', '0443', '0444', '0431', '0441', '0442', '0334', '0343', '0344') THEN 'loyal_with_breakfast'

ELSE 'null' END AS segment 

FROM

( SELECT 

user_id
, avg_monetary_value
, breakfast_buyer_flag
, recency_decile
, frequency_decile
, monetary_decile
, concat(breakfast_buyer_flag , recency_decile , frequency_decile , monetary_decile ) AS RFM_combination

FROM efood2022-357817.main_assessment.ck_orders_1 ) a


------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


-- Outcome of segmentation

SELECT 
segment
, SUM(avg_monetary_value) AS sum_value
, COUNT(user_id) AS total_obs
FROM  efood2022-357817.main_assessment.ck_orders_2
GROUP BY 
segment



------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


-- Query 1


DROP TABLE IF EXISTS efood2022-357817.main_assessment.ck_query1;
CREATE TABLE efood2022-357817.main_assessment.ck_query1
AS 

SELECT 
city
, COUNT(order_id) AS total_orders

FROM `efood2022-357817.main_assessment.orders1` 
GROUP BY 
city
HAVING COUNT(order_id) > 1000  -- All cities that exceed 1000 orders



-- Breakfast orders

DROP TABLE IF EXISTS efood2022-357817.main_assessment.ck_query1_2;
CREATE TABLE efood2022-357817.main_assessment.ck_query1_2
AS 


SELECT

c.city
, c.user_id 
, c.amount_breakfast
, c.orders_breakfast
, CASE WHEN c.orders_breakfast > 3 THEN 1 ELSE 0 END AS orders_3_flag



FROM 

(SELECT 

a.city
, b.user_id
, SUM(b.amount) AS amount_breakfast
, COUNT(b.order_id) AS orders_breakfast


FROM efood2022-357817.main_assessment.ck_query1 a
LEFT JOIN 
(SELECT * FROM `efood2022-357817.main_assessment.orders1` WHERE cuisine = 'Breakfast')  b
ON a.city = b.city


GROUP BY 
a.city
, b.user_id


) c


-- efood orders

DROP TABLE IF EXISTS efood2022-357817.main_assessment.ck_query1_3;
CREATE TABLE efood2022-357817.main_assessment.ck_query1_3
AS 


SELECT

c.city
, c.user_id 
, c.amount_efood
, c.orders_efood
, CASE WHEN c.orders_efood > 3 THEN 1 ELSE 0 END AS orders_3_flag_efood



FROM 

(SELECT 

a.city
, b.user_id
, SUM(b.amount) AS amount_efood
, COUNT(b.order_id) AS orders_efood


FROM efood2022-357817.main_assessment.ck_query1 a
LEFT JOIN  `efood2022-357817.main_assessment.orders1`  b
ON a.city = b.city


GROUP BY 
a.city
, b.user_id


) c



-- Both breakfast and efood orders

DROP TABLE IF EXISTS efood2022-357817.main_assessment.ck_query1_4;
CREATE TABLE efood2022-357817.main_assessment.ck_query1_4
AS 


SELECT 
c.city
, c.total_amount_breakfast / c.total_orders_breakfast AS breakfast_basket
, c.total_amount_efood / c.total_orders_efood AS efood_basket

, c.total_orders_breakfast / c.total_users_breakfast AS breakfast_freq
, c.total_orders_efood / c.total_users_efood AS efood_freq

, c.users_3_breakfast / c.total_users_breakfast AS breakfast_users3freq_perc
, c.users_3_efood / c.total_users_efood AS efood_users3freq_perc


FROM 
(SELECT 

a.city
, COUNT(a.user_id) AS total_users_breakfast
, SUM(a.amount_breakfast)AS total_amount_breakfast
, SUM(a.orders_breakfast) AS total_orders_breakfast
, SUM(a.orders_3_flag) AS users_3_breakfast

, COUNT(b.user_id) AS total_users_efood
, SUM(b.amount_efood)AS total_amount_efood
, SUM(b.orders_efood) AS total_orders_efood
, SUM(b.orders_3_flag_efood) AS users_3_efood

FROM efood2022-357817.main_assessment.ck_query1_2 a
LEFT JOIN efood2022-357817.main_assessment.ck_query1_3 b
ON a.city = b.city

GROUP BY 
a.city

) c

ORDER BY total_orders_breakfast DESC 
limit 5



SELECT * FROM efood2022-357817.main_assessment.ck_query1_4



------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

-- Query 2


DROP TABLE IF EXISTS efood2022-357817.main_assessment.ck_query2;
CREATE TABLE efood2022-357817.main_assessment.ck_query2
AS 


SELECT 
a.city
, a.user_id
, a.total_orders
, row_number() over (PARTITION BY  a.city ORDER BY a.total_orders DESC) AS rn 


FROM 

(SELECT 

city
, user_id
, COUNT(order_id) AS total_orders

FROM `efood2022-357817.main_assessment.orders1`  

GROUP BY 
city
, user_id ) a



DROP TABLE IF EXISTS efood2022-357817.main_assessment.ck_query2_1;
CREATE TABLE efood2022-357817.main_assessment.ck_query2_1
AS 

SELECT 
c.city
, (c.total_orders_10 / c.total_orders ) * 100 AS orders_perc


FROM 
(
SELECT 
a.city
, SUM(a.total_orders) AS total_orders_10
, SUM(b.total_orders) AS total_orders

FROM (SELECT * FROM efood2022-357817.main_assessment.ck_query2 WHERE rn <=10 ) a 
LEFT JOIN 
(SELECT 

city
, COUNT(order_id) AS total_orders

FROM `efood2022-357817.main_assessment.orders1`  

GROUP BY 
city
) b

ON a.city = b.city

GROUP BY a.city ) c



