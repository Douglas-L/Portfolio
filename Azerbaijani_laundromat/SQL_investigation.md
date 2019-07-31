
# 1. Writing Queries

1.1. Fetch names of all companies who are within jurisdiction GB, VG or TR.
```sql
SELECT name_norm
 FROM entities
WHERE jurisdiction IN ('GB', 'VG', 'TR') 
ORDER BY name_norm; 
```

1.2. Which year saw the most amount of euros flow through the bank, how much?
```sql
SELECT SUBSTR(date, 1, 4) as YEAR
 , ROUND(SUM(amount_eur), 2) AS amount_eur
 FROM transactions
GROUP BY YEAR
ORDER BY amount_eur DESC 
LIMIT 1;
-- 2013 with 2,198,809,314.7 euros
``` 

1.2a. Which day saw the most amount of euros flow through the bank?
```sql 
SELECT date
 , ROUND(SUM(amount_eur), 2) AS amount_eur
 FROM transactions
GROUP BY date
ORDER BY amount_eur DESC 
LIMIT 1;
-- 2013-11-28, 30,177,270.17
``` 

1.3. Fetch the data below
```sql
SELECT e.name
 , e.type 
 , e.jurisdiction
 , a.account
 FROM entities e 
    JOIN accounts a ON e.entity_id = a.name
ORDER BY e.name DESC 
LIMIT 5;
```
# 2. Analyzing Queries

2.1. Analyzing count difference 
```sql
SELECT COUNT(*) 
 FROM  accounts a
    LEFT JOIN entities e ON a.name = e.entity_id;
-- 3706 
SELECT COUNT(*) 
 FROM  entities e
    LEFT JOIN accounts a ON a.name = e.entity_id;
-- 3768 

SELECT COUNT(entity_id)
 FROM entities e
WHERE NOT EXISTS (SELECT 1 FROM accounts a
                   WHERE a.name = e.entity_id);
-- 62
``` 
The difference is because there are entities that do not have a corresponding account, but they will show up when entities is the left table. The third query finds these rows. 

2.2. Explain what data is fetched by this query. 
```sql
SELECT amount_eur
    , COUNT(*)
 FROM transactions 
GROUP BY amount_eur 
ORDER BY count(*) DESC;
```
This query fetches the number of times a specific euro amount that was sent. This may be useful for analyzing potential money laundering because one sign of money laundering is rapid movement of money in and out through the bank, particularly if an account receiving a specific amount quickly sends the same amount back out.

# 3. Business Case 

Yes, there is evidence of money laundering occurring within the bank as there is a variety of irregular transactions. 

First, there are a number of transaction amounts that are repeated, as much as 83 times for 20000 euros (Q2.2). This could be a coincidence since round numbers can be convenient for anyone, but a closer look reveals that only 5 payer accounts are responsible for the 83 transactions (Q3.1.1). These companies are LCM Alliance, Metastar, Polux, Hilux and Farberlex. Furthermore, the individual transactions show these payments are for different contracts so it's odd the payer managed to get these round number payments(Q3.1.2). Other notable patterns include recurring payer-beneficiary pairs and irregular payment time intervals, so let's dig deeper. 
![query3.1.2](./fl_312.png)

The most active pair is found to be 3642 (BAKTELEKOM MMC) as payer and 1686(HILUX SERVICES LP) as beneficiary, with 436 transactions (Q3.2.1). What stood out even more is that the transactions have an average value of 2.15mm euros and were being sent with only an average of 1.3 days between transactions (Q3.2.2), with some even being on the same day. The rapid and large amounts of money sent require further investigation and the varying amounts sent and days between transactions make it even more suspicious (Q3.2.3). The purpose of the payments is only noted to be some sort of fee without further detail.  
![query3.2.2](./fl_323.png)

Another sign for money laundering are two making multiple transactions on the same day. In a normal business context, it's unusual to pay twice or more on the same day, though it can happen. Payer 0 (LCM Alliance) takes it to the extreme as it paid beneficiary 6 (LOTA SALES LLP) more than once a day 324 times. Looking at the individual transactions like before, these are irregular payments, both by frequency and amount (Q3.2.2).
![query3.3.2](./fl_332.png)

In summary, three signs of money laundering are occurring are:
1. Select companies involved in sending round number amounts frequently 
2. Large amounts of money moved at short time intervals between two companies
3. Multiple transactions between two companies on the same day 

Next steps would be checking whether the suspicious accounts have a valid reason for their behavior and generalizing these findings to create monitoring rules to flag other transactions. 

Appendix: Queries Used

2.2
```sql
SELECT amount_eur
    , COUNT(*)
 FROM transactions 
GROUP BY amount_eur 
ORDER BY count(*) DESC;
```
3.1.1.
```sql
SELECT DISTINCT payer_account
FROM (SELECT id, date, payer_account, beneficiary_account, purpose
        FROM transactions 
        WHERE amount_eur = 20000
        ORDER BY Date);
```
3.1.2.
```sql
SELECT id, date, payer_account, beneficiary_account, purpose
        FROM transactions 
        WHERE amount_eur = 20000
        ORDER BY Date
```   
3.2.1.
```sql
SELECT payer_account
    , beneficiary_account
    , COUNT(1) num_trans
FROM transactions
GROUP BY 1,2
HAVING COUNT(1) > 1
ORder by num_trans dESC;
```
3.2.2.
```sql
SELECT avg(amount_eur) as avg_amount
 , avg(days_since_last_transaction) as avg_days_between_trans
 FROM (
    SELECT id
        , date
        , payer_account
        , beneficiary_account
        , amount_eur
        , purpose
        , julianday(date) - julianday(LAG(date,1) OVER (ORDER BY date))  as days_since_last_transaction
    FROM transactions 
    WHERE (payer_account = 1686 AND beneficiary_account = 3642)
    OR (payer_account = 3642 AND beneficiary_account = 1686)
    ORDER BY Date) s ;
```

3.2.3.
```sql
SELECT id
        , date
        , payer_account
        , beneficiary_account
        , amount_eur
        , amount_orig
        , purpose
        , julianday(date) - julianday(LAG(date,1) OVER (ORDER BY date))  as days_since_last_transaction
    FROM transactions 
    WHERE (payer_account = 1686 AND beneficiary_account = 3642)
    OR (payer_account = 3642 AND beneficiary_account = 1686)
    ORDER BY Date;
```
3.3.1.
```sql
SELECT payer_account 
, beneficiary_account
, COUNT(cnt)as days_multi_trans
FROM (
    SELECT payer_account
        , beneficiary_account
        , date
        , COUNT(1) cnt
    FROM transactions
    GROUP BY 1,2,3
    HAVING COUNT(1) > 1) s
    GROUP BY payer_account
    ORDER BY days_multi_trans DESC;
```
3.3.2.
```sql
SELECT id
, date
, payer_account
, beneficiary_account
, amount_eur
, purpose
, julianday(date) - julianday(LAG(date,1) OVER (ORDER BY date))
FROM transactions 
WHERE (payer_account = 0 AND beneficiary_account = 6)
  OR (payer_account = 6 AND beneficiary_account = 0)
ORDER BY Date ;
```