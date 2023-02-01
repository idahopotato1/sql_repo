## AZURE SQL

## Presentation Stock Change

```sql
SELECT mpi.division_id, mpi.district_cd, mpi.store, mpi.upc, mpi.item_desc, mpi.user_id, mpi.business_date, spi.start_pi, epi.end_pi, mpi.pi_change
FROM (
     SELECT DISTINCT str.division_id, str.district_cd, store, upc, item_desc, user_id, business_date, SUM(pi_change) AS pi_change
     FROM scmrep.store_manual_pi mpi
     INNER JOIN scmrep.lu_store_finance_om str ON str.store_id = mpi.store
     INNER JOIN scmrep.lu_day_merge dy ON dy.d_date = mpi.business_date
     WHERE dy.week_id between 202201 and 202253
          --AND mpi.upc IN ${hivevar:upc}
          AND str.store_id = 177
     GROUP BY str.division_id, str.district_cd, store, upc, item_desc, user_id, business_date) mpi
INNER JOIN (
     SELECT store, upc, user_id, business_date, start_pi, adjustment_timestamp,
          ROW_NUMBER() OVER (PARTITION BY store, upc, user_id, business_date ORDER BY adjustment_timestamp ASC) AS rn
     FROM scmrep.store_manual_pi mpi
     INNER JOIN scmrep.lu_store_finance_om str ON str.store_id = mpi.store
     INNER JOIN scmrep.lu_day_merge dy ON dy.d_date = mpi.business_date
     WHERE dy.week_id between 202201 and 202253
          -- AND mpi.upc IN ${hivevar:upc}
          AND str.store_id = 177
) spi ON mpi.store = spi.store AND mpi.upc = spi.upc AND mpi.user_id = spi.user_id AND mpi.business_date = spi.business_date AND spi.rn = 1
INNER JOIN (
     SELECT store, upc, user_id, business_date, end_pi, adjustment_timestamp,
          ROW_NUMBER() OVER (PARTITION BY store, upc, user_id, business_date ORDER BY adjustment_timestamp DESC) AS rn
     FROM scmrep.store_manual_pi mpi
     INNER JOIN scmrep.lu_store_finance_om str ON str.store_id = mpi.store
     INNER JOIN scmrep.lu_day_merge dy ON dy.d_date = mpi.business_date
     WHERE dy.week_id between 202201 and 202253
        --  AND mpi.upc IN ${hivevar:upc}
          AND str.store_id = 177
) epi ON mpi.store = epi.store AND mpi.upc = epi.upc AND mpi.user_id = epi.user_id AND mpi.business_date = epi.business_date AND epi.rn = 1
```

## Warehouse to Store Order History

```sql
SELECT store.division_id, dept_section_id, store.district_cd, store.store_id, dtl.orig_cic_id, cic.internet_item_dsc, chrono.week_id,
     SUM(CASE WHEN hdr.order_type_cd = 'FAR' THEN 1 ELSE 0 END) AS far_order_count,
     SUM(CASE WHEN hdr.order_type_cd = 'FAR' THEN COALESCE(dtl.order_qty, COALESCE(dtl.exe_demand_qty, 0)) ELSE 0 END) AS far_cases_ordered,
     SUM(CASE WHEN hdr.order_type_cd = 'FAR' THEN COALESCE(dtl.exe_ship_qty, COALESCE(dtl.modify_qty, 0)) ELSE 0 END) AS far_cases_shipped,
     SUM(CASE WHEN NOT hdr.order_type_cd = 'FAR' THEN 1 ELSE 0 END) AS store_add_count,
     SUM(CASE WHEN NOT hdr.order_type_cd = 'FAR' THEN COALESCE(dtl.order_qty, COALESCE(dtl.exe_demand_qty, 0)) ELSE 0 END) AS store_add_cases_ordered,
     SUM(CASE WHEN NOT hdr.order_type_cd = 'FAR' THEN COALESCE(dtl.exe_ship_qty, COALESCE(dtl.modify_qty, 0)) ELSE 0 END) AS store_add_cases_shipped,
     SUM(COALESCE(dtl.firm_order_qty, 0)) AS firm_qty
FROM scmrep.ov_order_hdr_hist hdr
INNER JOIN scmrep.lu_day_merge chrono ON chrono.d_date = hdr.schedule_delivery_dt
INNER JOIN scmrep.lu_store_finance_om store ON store.store_id = hdr.dest_facility_id
INNER JOIN scmrep.ov_order_dtl_hist dtl ON dtl.order_hdr_sk = hdr.order_hdr_sk AND dtl.order_dt = hdr.order_dt
INNER JOIN (SELECT DISTINCT dept_section_id, CAST(corp_item_cd AS BIGINT) AS corp_item_cd, internet_item_dsc FROM scmrep.lu_cic) cic ON cic.corp_item_cd = dtl.orig_cic_id
WHERE store.division_id = 30
     AND store.store_id = 177
     AND chrono.week_id IN (select week_id from scmrep.lu_day_merge where d_date = current_date group by 1)
     AND cic.dept_section_id IN (301, 302, 313, 318, 327, 335, 338, 314, 323, 324, 325, 317)
GROUP BY store.division_id, dept_section_id, store.district_cd, store.store_id, dtl.orig_cic_id, cic.internet_item_dsc, chrono.week_id
ORDER BY store.division_id, dept_section_id, store.district_cd, store.store_id, dtl.orig_cic_id, cic.internet_item_dsc, chrono.week_id
```

## Store Order and polltime

```sql
select distinct
str.loadid,
str.ordertype,
str.divisionid,
str.districtid,
str.storeid,
cic.u_section,
str.polltimestamp,
str.corporationitemcode,
str.upc,
str.corporationitemcode,
cic.descr,
str.itemorderqtyeaches,
str.itemorderqtywhseunits
from scmrep.inbound_from_jda_storeorder str
--join scmrep.corporate_item itm
--on itm.corporate_item_cd = str.corporationitemcode
join scmrep.jda_cic cic
on cic.u_cic_code =  str.corporationitemcode
join scmrep.jda_assortment ast
on ast.u_cic_code = str.corporationitemcode
where cast(str.POLLTIMESTAMP as date) = '2022-08-15'
and str.STOREID in ('0177')
and u_data_batch_cd in ('0117','0170','0119','0101','0168','0123','0133','0135')
```

## Tag Subscription

```sql
SELECT DISTINCT
TAG.DEPT_SECTION_ID
, TAG.CORP_ITEM_CD
, UPC_ID, CIC.INTERNET_ITEM_DSC
, CIC.PACK_WHSE_QTY
, CIC. NUMERIC_SIZE_QTY
, CIC.SIZE_UOM_CD
, TAG.LOC_ID
, TAG.AISLE_NBR
, TAG.SECT_NBR
, TAG.SHELF_CD
, max(source_last_updt_ts)
FROM SCMREP.TAG_SUBSCRIPTION TAG
JOIN SCMREP.LU_CIC CIC
ON CIC.CORP_ITEM_CD = TAG.CORP_ITEM_CD
WHERE TAG.STORE_ID = 177
--and source_last_updt_ts = max(source_last_updt_ts)
--and corp_item_cd in (9900038,9014962,2011658,2020403,9152218,2900013,9151571,2020809,9014972,9150355,2020043,9050236,2010741,2011796,9101008,9012301,2011525,9014255,9011118,9010943,9050117,9100566,2020253,9150899,9050313,9150693,9015920,9014968,9018498,30031468,30010042,30040061,30030416,30010902,74401173,30030578,30030130,30010051,30010903,30030470,30031225,74400906,30040094,30030955,88660894,88661232,88290110,88661616,86930414,84740054,38450027,39500369,38250451,38250151,39550022,38250149,38250095,74450602,74500019,74450190,1050525,1014390,1050088,1050721,1052303,1050659,1050173,1050163,1050089,1054026,1056324,1052400,1050253,1050707,1051599,1060873,1054601,1059634,1051089,1051172,1052352,1052678,1058988,1055451,1050258,1011222,1056317,1059564,1011272,1050015,1059025,1061149,1055443,1056322,1061151,1053679,1051102,1052062,1050643,42050710,42050332,42011366,48050110,42200044,47050674,42011754,42100034,48200224,48100252,42200070,48020113,45010196,48010184,42200067,47050173,48050107,42050640,48011217,42011102,42011717,42050970,42052001,42011141,42050022,42050360,48040124,48020437,45010315,47010663,48050157,48050030,48013414,42051549,48100302,42050018,42051591,42011032,42051601,42052064,48050082,42012037,42050334,42050476,42051022,36150034,36450227,36304174,36300150,36301337,36100051,36302913,36050352,36100369,36050095,36050275,36050359,3300349,8051060,3300403,8050980,3200792,3200255,3300398,8050230,3250537,3200781,8010404,3200137,3200217,3200086,3200662,3100363,3250239,8100081,8200213,3300570,8250009,3200136,3300346,3300338,3100295,3100116,3300327,3350154,8100086,8150129,8100052,8010933,3200804,8200035,3200471,3100294,74100627,75800302,75800274,28350414,13100090,7150016,28202443,25100014,15200001,22400027,25100225,13450001,15100351,13010071,4050054,25100155,25100129,23010838,6041112,22500021,13050220,22010059,28350494,22200016,5850056,22100290,4100012,7050005,7150017,13150320,15150070,25050142,23010217,25100051,24010025,25100028,23010629,22400013,5250237,14151944,6041109,4100133,5850133,6040263,4100063,28200382,13500347,14150407,17200143,4050192,4050297,28350231,14201304,7200109,25300068,21300226,13050184,5300152,13050173,14200364,14200398,15200117,7050018,15200002,5201193,5400690,28350026,14200565,13500345,14150405,28200289,22150044,7010372,14250040,25150014,6060157,11011247,20020071,19010837,27050488,27050300,20020953,20070078,26330194,11010589,20070186,20080005,11010416,20030006,20020555,11010941,19020024,11100311,19090116,11014456,11101807,26321114,20070546,20070535,20030051,26250271,11010990,26250168,20060257,20070305,20060274,20030258,10050779,19150399,26320364,11101906,20060101,19030930,20020277,19031680,20070138,20070070,11100057,26150184,20010028,19050746,27101512,19030452,20050417,20070059,20060420,20020028,29011108,26400505

--)
and TAG.DEPT_SECTION_ID in (301,302,313,318,327,335,338,314,323,324,325,317,311,312,322)
group by
TAG.DEPT_SECTION_ID
, TAG.CORP_ITEM_CD
, UPC_ID, CIC.INTERNET_ITEM_DSC
, CIC.PACK_WHSE_QTY
, CIC. NUMERIC_SIZE_QTY
, CIC.SIZE_UOM_CD
, TAG.LOC_ID
, TAG.AISLE_NBR
, TAG.SECT_NBR
, TAG.SHELF_CD
```

## Order Closed in Route to Stores

```sql
select
    hdr.order_dt
    , hdr.com_received_ts
    , hdr.com_extract_ts
    , hdr.schedule_cutoff_ts
    , hdr.bomb_dt
    , hdr.schedule_pick_dt
    , hdr.com_schedule_delivery_dt
    , hdr.schedule_delivery_dt
    , hdr.inv_billing_dt
    , hdr.inv_dt
    , hdr.order_hdr_sk
    , hdr.dest_div_nbr
    , hdr.rog_cd
    , hdr.dest_facility_id  -- 4-character store number
    , hdr.ship_div_nbr
    , hdr.ship_facility_id  -- warehouse code
    , hdr.order_nbr
    , hdr.inv_nbr
    , hdr.register_id
    , hdr.com_order_nbr
    , hdr.exe_order_nbr
    , hdr.order_type_cd
    , hdr.order_sts_cd
    , dtl.orig_cic_id
    , dtl.upc_cntry_id
    , dtl.upc_sys_id
    , dtl.upc_manuf_id
    , dtl.upc_sales_id
    , dtl.item_dsc
    , dtl.order_qty
    , dtl.exe_ship_qty
from scmrep.ov_order_hdr_hist hdr
join scmrep.ov_order_dtl_hist dtl
    on dtl.order_dt = hdr.order_dt
    and dtl.order_hdr_sk = hdr.order_hdr_sk
where
    hdr.dest_facility_id = '0161'   -- 4-character store number
    and hdr.delivery_dt = current_date
    and dtl.orig_cic_id = 36300065
```

## Store Current PI

```sql
SELECT DISTINCT
U_STORE,
UPC.DEPT_SECTION_ID,
PI.AVAILDATE,
PI.U_CIC_CODE,
UPC.UPC_ID,
UPC.INTERNET_ITEM_DSC,
PI.QTY,
PI.U_PS_QTY,
last_updt_ts
FROM HIVE_METASTORE.SCMREP.JDA_INVENTORY_STORE PI
JOIN HIVE_METASTORE.SCMREP.LU_UPC UPC
ON UPC.CORP_ITEM_CD = PI.U_CIC_CODE
join hive_metastore.scmrep.lu_store_finance_om str
on str.store_id = pi.u_store
where AVAILDATE  =  CURRENT_DATE() -2
and str.store_id = 3441
```

## Future Store Orders

```sql
select distinct
str.loadid,
str.ordertype,
str.divisionid,
str.districtid,
str.storeid,
cic.u_section,
str.polltimestamp,
str.corporationitemcode,
str.upc,
str.corporationitemcode,
cic.descr,
str.itemorderqtyeaches,
str.itemorderqtywhseunits
from scmrep.inbound_from_jda_storeorder str
--join scmrep.corporate_item itm
--on itm.corporate_item_cd = str.corporationitemcode
join scmrep.jda_cic cic
on cic.u_cic_code =  str.corporationitemcode
join scmrep.jda_assortment ast
on ast.u_cic_code = str.corporationitemcode
where cast(str.POLLTIMESTAMP as date) = '2022-08-15'
and str.STOREID in ('0177')
and u_data_batch_cd in ('0117','0170','0119','0101','0168','0123','0133','0135')
--and u_data_batch_cd in ('0101')
--select distinct u_data_batch_cd
```
