SELECT 
     ad_date, campaign_id,
     sum (spend) AS "загальна сума витрат",
     sum (impressions) AS "кількість показів",
     sum (clicks) AS "кількість кліків",
     sum (value) AS "загальний VALUE конверсій",
     sum (spend) / sum (clicks) as CPC,
     sum (spend) / (sum (impressions)/1000) as CPM,
     sum (clicks)*100 / sum (impressions) as CTR,
     (sum (value) - sum (spend))*100 / sum (spend) as ROMI
FROM facebook_ads_basic_daily 
WHERE clicks > 0 AND impressions/1000 > 0 AND spend > 0
GROUP BY ad_date, campaign_id
ORDER BY ad_date;

SELECT 
     campaign_id, sum(spend) > 500000,
     (sum (value) - sum (spend))*100 / sum (spend) as ROMI     
FROM facebook_ads_basic_daily
GROUP BY campaign_id
HAVING sum(spend) > 500000;

WITH fabd_fc_fa_gabd AS (
SELECT
     fabd.ad_date, 
     fc.campaign_name,
     fa.adset_name,
     fabd.spend,
     fabd.impressions,
     fabd.reach,
     fabd.clicks,
     fabd.leads,
     fabd.value
FROM  facebook_ads_basic_daily AS fabd 
LEFT JOIN facebook_campaign AS fc ON fabd.campaign_id = fc.campaign_id
LEFT JOIN facebook_adset AS fa ON fa.adset_id = fabd.adset_id

UNION ALL 

SELECT
     gabd.ad_date,
     gabd.campaign_name,
     gabd.adset_name,
     gabd.spend,
     gabd.impressions,
     gabd.reach,
     gabd.clicks,
     gabd.leads,
     gabd.value
FROM google_ads_basic_daily AS gabd)

SELECT
     ad_date,
     campaign_name,
     adset_name,
     round(sum(spend)::NUMERIC/100,2) AS total_spend,
     sum(impressions) AS total_impressions,
     sum(clicks) AS total_clicks,
     round(sum(value)::NUMERIC/100,2) AS total_value
FROM fabd_fc_fa_gabd
GROUP BY ad_date,
         campaign_name,
         adset_name;

WITH fabd_fc_fa_gabd AS (
SELECT
     fabd.ad_date, 
     fabd.url_parameters,
     fabd.spend,
     fabd.impressions,
     fabd.reach,
     fabd.clicks,
     COALESCE (fabd.leads,0) AS leads,
     fabd.value
FROM  facebook_ads_basic_daily AS fabd 
LEFT JOIN facebook_campaign AS fc ON fabd.campaign_id = fc.campaign_id
LEFT JOIN facebook_adset AS fa ON fa.adset_id = fabd.adset_id

UNION ALL 

SELECT
     gabd.ad_date,
     gabd.url_parameters,
     gabd.spend,
     gabd.impressions,
     gabd.reach,
     gabd.clicks,
     gabd.leads,
     gabd.value
FROM google_ads_basic_daily AS gabd)

SELECT
     ad_date AS дата,
     case
        when lower(substring(url_parameters, 'utm_campaign=([^\&]+)')) != 'nan'
        then decode_url_part(lower(substring(url_parameters, 'utm_campaign=([^\&]+)')))
      end as utm_campaign,
     round(sum(spend)::NUMERIC/100,2) AS total_spend,
     sum(impressions) AS total_impressions,
     sum(clicks) AS total_clicks,
     round(sum(value)::NUMERIC/100,2) AS total_value,
     CASE WHEN sum(impressions)=0
          THEN NULL
          ELSE sum(clicks)*100/sum(impressions)
      END  AS CTR, 
     CASE WHEN sum(clicks)=0
          THEN NULL
          ELSE round(sum(spend)::NUMERIC/100,2)/sum(clicks)
      END  AS CPC,
     CASE WHEN sum(impressions)=0
          THEN NULL
          ELSE (round(sum(spend)::NUMERIC/100,2)/sum(impressions))*1000
      END  AS CPM,
     CASE WHEN round(sum(spend)::NUMERIC/100,2)=0
          THEN NULL
          ELSE 
      (round(sum(value)::NUMERIC/100,2)-round(sum(spend)::NUMERIC/100,2))*100/round(sum(spend)::NUMERIC/100,2)
      END  AS ROMI
FROM fabd_fc_fa_gabd
GROUP BY ad_date,
         utm_campaign;

WITH romi_by_month AS (

WITH fb_g_monthly AS (

WITH fabd_fc_fa_gabd AS (

SELECT
     fabd.ad_date, 
     fabd.url_parameters,
     fabd.spend,
     fabd.impressions,
     fabd.reach,
     fabd.clicks,
     COALESCE (fabd.leads,0) AS leads,
     fabd.value
FROM  facebook_ads_basic_daily AS fabd 
LEFT JOIN facebook_campaign AS fc ON fabd.campaign_id = fc.campaign_id
LEFT JOIN facebook_adset AS fa ON fa.adset_id = fabd.adset_id

UNION ALL 

SELECT
     gabd.ad_date,
     gabd.url_parameters,
     gabd.spend,
     gabd.impressions,
     gabd.reach,
     gabd.clicks,
     gabd.leads,
     gabd.value
FROM google_ads_basic_daily AS gabd)

SELECT
     date_trunc ('month' , ad_date) AS ad_date_trunced,
     case
        when lower(substring(url_parameters, 'utm_campaign=([^\&]+)')) != 'nan'
        then decode_url_part(lower(substring(url_parameters, 'utm_campaign=([^\&]+)')))
      end as utm_campaign,
     round(sum(spend)::NUMERIC/100,2) AS total_spend,
     sum(impressions) AS total_impressions,
     sum(clicks) AS total_clicks,
     round(sum(value)::NUMERIC/100,2) AS total_value,
     CASE WHEN sum(impressions)=0
          THEN NULL
          ELSE sum(clicks)*100/sum(impressions)
      END  AS CTR, 
     CASE WHEN sum(clicks)=0
          THEN NULL
          ELSE round(sum(spend)::NUMERIC/100,2)/sum(clicks)
      END  AS CPC,
     CASE WHEN sum(impressions)=0
          THEN NULL
          ELSE (round(sum(spend)::NUMERIC/100,2)/sum(impressions))*1000
      END  AS CPM,
     CASE WHEN round(sum(spend)::NUMERIC/100,2)=0
          THEN NULL
          ELSE 
      (round(sum(value)::NUMERIC/100,2)-round(sum(spend)::NUMERIC/100,2))*100/round(sum(spend)::NUMERIC/100,2)
      END  AS ROMI
FROM fabd_fc_fa_gabd
GROUP BY ad_date_trunced,
         utm_campaign)
           
SELECT ad_date_trunced AS ad_month,
       utm_campaign AS utm_campaign_m,
       sum(total_spend) AS total_spend_m,
       sum(total_impressions) AS total_impressions_m,
       sum(total_clicks) AS total_clicks_m,
       sum(total_value) AS total_value_m,
       LAG (CASE 
          WHEN sum(total_impressions) !=0
          THEN sum(total_clicks)*100/sum(total_impressions)
        END) OVER (ORDER BY ad_date_trunced) AS CTR_m_ago,
       CASE 
          WHEN sum(total_impressions) !=0
          THEN sum(total_clicks)*100/sum(total_impressions)
        END  AS CTR_m,
        CASE 
          WHEN sum(total_clicks) !=0
          THEN round(sum(total_spend)::NUMERIC/100,2)/sum(total_clicks)
        END  AS CPC_m,
       LAG (CASE
          WHEN sum(total_impressions) !=0
          THEN (round(sum(total_spend)::NUMERIC/100,2)/sum(total_impressions))*1000
              END) OVER (ORDER BY ad_date_trunced)  AS CPM_m_ago,
        CASE
          WHEN sum(total_impressions) !=0
          THEN (round(sum(total_spend)::NUMERIC/100,2)/sum(total_impressions))*1000
        END  AS CPM_m,
        LAG (CASE
                WHEN round(sum(total_spend)::NUMERIC/100,2) !=0
                THEN 
(round(sum(total_value)::NUMERIC/100,2)-round(sum(total_spend)::NUMERIC/100,2))*100/round(sum(total_spend)::NUMERIC/100,2)
              END) OVER (ORDER BY ad_date_trunced)  AS ROMI_month_ago,
        CASE
          WHEN round(sum(total_spend)::NUMERIC/100,2) !=0
          THEN 
(round(sum(total_value)::NUMERIC/100,2)-round(sum(total_spend)::NUMERIC/100,2))*100/round(sum(total_spend)::NUMERIC/100,2)
        END  AS ROMI_m 
FROM fb_g_monthly        
GROUP BY ad_month,
         utm_campaign_m)
         
SELECT rbm.ad_month, 
       rbm.utm_campaign_m,
       CASE 
           WHEN sum(rbm.CTR_m_ago) !=0
           THEN sum(rbm.CTR_m) / sum (rbm.CTR_m_ago)
        END   AS CTR_ch,
        CASE 
           WHEN sum(rbm.CPM_m_ago) !=0
           THEN sum(rbm.CPM_m)/sum(rbm.CPM_m_ago)
         END AS CPM_ch,
       (sum(rbm.ROMI_m) / sum(ROMI_month_ago)) AS ROMI_ch
FROM romi_by_month AS rbm       
GROUP BY rbm.ad_month,
         rbm.utm_campaign_m
ORDER BY rbm.ad_month;       


         
- 👋 Hi, I’m @VolodymyrLnnk
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
VolodymyrLnnk/VolodymyrLnnk is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
