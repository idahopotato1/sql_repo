# Pricing Execution

### Step 1: 
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_alw_hdr_dtl_all;
```

### Step 2:
```sql
CREATE or replace  TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_alw_hdr_dtl_all 
     (
      genrt_cic_id DECIMAL(12,0),
      corp_item_cd DECIMAL(8,0),
      group_id SMALLINT,
      corp INTEGER,
      division INTEGER,
      wds_division CHAR(2)  COLLATE 'en-ci' ,
      facility CHAR(4)  COLLATE 'en-ci' ,
      rog CHAR(4)  COLLATE 'en-ci' ,
      offer_number CHAR(8)  COLLATE 'en-ci' ,
      vend_num CHAR(6)  COLLATE 'en-ci' ,
      vend_sub_acnt CHAR(3)  COLLATE 'en-ci' ,
      cost_area SMALLINT,
      arrival_from_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      arrival_to_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      calendar_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      allow_type CHAR(1)  COLLATE 'en-ci' ,
      allow_amt DECIMAL(11,4),
      amt_in_cost DECIMAL(11,4),
      perform_1 CHAR(2)  COLLATE 'en-ci' ,
      dst_cntr CHAR(4)  COLLATE 'en-ci' ) COMMENT = '{"multiset": true}'
```

### Step 3:
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_alw_hdr_dtl_all
(       genrt_cic_id
,       corp_item_cd
,       group_id
,       corp
,       division
,       wds_division
,       facility
,       rog
,       offer_number
,       vend_num
,       vend_sub_acnt
,       cost_area
,       arrival_from_date
,       arrival_to_date
,       calendar_date
,       allow_type
,       allow_amt
,       amt_in_cost
,       perform_1
,       dst_cntr
)
SELECT DISTINCT
        vdd.CORPORATE_ITEM_INTEGRATION_ID
,       cic.corporate_item_cd as corp_item_cd
,       cic.smic_group_cd as Group_id
,       1    AS corp
,       rog.division_id     AS division
,       vdh.Res_Division_Id  AS wds_division
,       vdd.warehouse_id AS facility
,       rog.rog_id          AS rog
,       vdh.DEAL_OFFER_NBR AS offer_number
,       vdh.vendor_id  AS vend_num
,       vdh.VENDOR_SUB_ACCOUNT_ID AS vend_sub_acnt
,       vdh.Vendor_Cost_Area AS cost_area
,       vdh.SCHEDULE_ARRIVAL_START_DT  AS arrival_from_date
,       vdh.SCHEDULE_ARRIVAL_END_DT  AS arrival_to_date
-- ,       DATE '2021-07-01' AS calendar_date
--Akash:27/03/2019 : added logic to select report date for respective divisions
,       CASE
            WHEN (rog.division_id = '33' or rog.division_id = '34') THEN (current_date + """+str(east_gap)+""")::DATE
            ELSE (current_date + """+str(reg_gap)+""")::DATE
        END AS calendar_date
,       vdd.Allowance_Type_Cd AS allow_type
,       vdd.Flat_Allowance_Amt     
,       CASE
            WHEN vdd.Allowance_Type_Cd = 'C' THEN vdd.Flat_Allowance_Amt
            ELSE 0
        END AS amt_in_cost
,       vdd.FLAT_ALLOWANACE_PIND_CD  AS perform_1
,       COALESCE(whs.DISTRIBUTION_CENTER_ID, vdd.warehouse_id) AS dst_cntr
 FROM EDM_VIEWS_PRD.DW_VIEWS.VENDOR_DEAL_HEADER vdh
 INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.VENDOR_DEAL_DETAIL vdd
 on    vdd.VENDOR_DEAL_HEADER_INTEGRATION_ID = vdh.VENDOR_DEAL_HEADER_INTEGRATION_ID
 AND     vdd.DW_CURRENT_VERSION_IND = TRUE 
 AND     vdd.DW_LOGICAL_DELETE_IND = FALSE
 AND     vdh.DW_CURRENT_VERSION_IND = TRUE 
 AND     vdh.DW_LOGICAL_DELETE_IND = FALSE  
JOIN EDM_VIEWS_PRD.DW_VIEWS.Corporate_Item cic
 ON   VDD.Corporate_item_integration_id   = CIC.Corporate_item_integration_id
 AND   cic.DW_CURRENT_VERSION_IND = TRUE 
 AND     cic.DW_LOGICAL_DELETE_IND = FALSE
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.Supply_Chain_Item sci
ON   sci.warehouse_id = vdd.warehouse_id
AND     sci.Corporate_item_integration_id = cic.Corporate_item_integration_id
AND     sci.DW_CURRENT_VERSION_IND = TRUE 
AND     sci.DW_LOGICAL_DELETE_IND = false
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.Retail_Order_Group_Corporate_item rogci
ON      rogci.Ordering_Division_id = sci.division_id
AND     rogci.warehouse_id = sci.warehouse_id
AND     rogci.Corporate_item_integration_id = vdd.Corporate_item_integration_id
AND     rogci.DW_CURRENT_VERSION_IND = TRUE 
AND     rogci.DW_LOGICAL_DELETE_IND = false
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP AS rog
ON      rog.rog_id = rogci.rog_id
AND     rog.DW_CURRENT_VERSION_IND = TRUE 
AND     rog.DW_LOGICAL_DELETE_IND = false
LEFT    OUTER JOIN EDM_VIEWS_PRD.DW_VIEWS.WAREHOUSE whs
ON      whs.warehouse_id = vdd.warehouse_id
AND     whs.DW_CURRENT_VERSION_IND = TRUE 
AND     whs.DW_LOGICAL_DELETE_IND = false
-- WHERE   DATE '2021-07-01' BETWEEN h.SCHEDULE_ARRIVAL_START_DT AND h.SCHEDULE_ARRIVAL_END_DT
--Akash: 27/03/2019 : Replaced where clause with below lines
WHERE   (CASE
            WHEN (rog.division_id = '33' or rog.division_id = '34') THEN (current_date + """+str(east_gap)+""")::DATE
            ELSE (current_date + """+str(reg_gap)+""")::DATE
        END) BETWEEN vdh.SCHEDULE_ARRIVAL_START_DT AND vdh.SCHEDULE_ARRIVAL_END_DT
AND      vdd.active_ind <> 'I'
AND     vdd.Flat_Allowance_Amt > 0.01
AND     vdd.Allowance_Reason_Cd = '0'
AND     vdd.Allowance_Type_Cd <> 'A'
AND     vdd.FLAT_ALLOWANACE_PIND_CD <> '30'
AND     cic.Display_Item_Ind <> 'Y'
AND     cic.item_status_cd NOT IN ('D', 'X')
-- MAYANK (20190425) : commented below line for allowance in all the divisions
-- AND     rog.division_id = '27'
AND     cic.smic_group_cd IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,34,36,37,38,39,40,42,43,44,45,46,47,48,73,74,88,96,97,
49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,72,75,76,77,78,79,80,89,84,70,71,81,82,85,86,94,95);
```

### Step 4:
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_alw_final;
-- DROP_TBL: Add if exists in drop table
```

### Step 5:
```sql
CREATE or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_alw_final 
     (
      group_id SMALLINT,
      corp_item_cd DECIMAL(8,0),
      corp INTEGER,
      facility CHAR(4)  COLLATE 'en-ci' ,
      vend_num CHAR(6)  COLLATE 'en-ci' ,
      vend_sub_acnt CHAR(3)  COLLATE 'en-ci' ,
      cost_area SMALLINT,
      rog CHAR(4)  COLLATE 'en-ci' ,
      price_area_id INTEGER,
      alw_typ VARCHAR(1)  COLLATE 'en-ci' ,
      arrival_from_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      arrival_to_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      allow_amt DECIMAL(11,4),
      amt_in_cost DECIMAL(11,4),
      offer_number CHAR(8)  COLLATE 'en-ci' ) COMMENT = '{"multiset": true}' ;
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_MULTISET_OPTION - Remove MULTISET create table option
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
-- T_CREATE_TABLE_META - Add table metadata to table comment
```

### Step 6:
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_alw_final
(       group_id
,       corp_item_cd
,       corp
,       facility
,       vend_num
,       vend_sub_acnt
,       cost_area
,       rog
,       price_area_id
,       alw_typ
,       arrival_from_date
,       arrival_to_date
,       allow_amt
,       amt_in_cost
,       offer_number
)
With vendor_allow_hdr as
(
     select distinct
    1  as corporation_id,
    DEAL_OFFER_NBR as OFFER_NBR,
    ROG.Division_Id as division_cd,
    a.CURRENT_OFFER_IND
    FROM EDM_VIEWS_PRD.DW_VIEWS.VENDOR_DEAL_HEADER a
   INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP AS ROG 
   ON ROG.ROG_ID = A.ROG_CD AND ROG.DW_CURRENT_VERSION_IND = TRUE AND ROG.DW_LOGICAL_DELETE_IND = FALSE
   WHERE   a.DW_CURRENT_VERSION_IND = TRUE
    AND     a.DW_LOGICAL_DELETE_IND = false 
) ,
store_price_area as
(
     SELECT DISTINCT
     RSPA.section_cd AS loc_retail_sect_id
     ,RSPA.dw_first_effective_dt AS first_eff_dt
     ,RSPA.dw_last_effective_dt AS last_eff_dt
     ,RSPA.price_area_cd AS price_area_id
     , RSPA.FACILITY_INTEGRATION_ID
     , RT.FACILITY_NBR AS STORE_ID
     , RT.ROG_ID AS  ROG_ID
     FROM EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE_SECTION_PRICE_AREA AS RSPA
     INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE AS RT
     ON RSPA.FACILITY_INTEGRATION_ID = RT.FACILITY_INTEGRATION_ID
     AND RT.DW_LOGICAL_DELETE_IND = FALSE
     AND RT.DW_CURRENT_VERSION_IND = TRUE
     AND RSPA.DW_LOGICAL_DELETE_IND = FALSE
     AND RSPA.DW_CURRENT_VERSION_IND = TRUE
) ,
LU_STORE AS
(
Select Distinct
  F.Facility_nbr as store_id
  ,F.CLOSE_DT as closed_dt
  ,RT.rog_id
  ,RT.CORPORATION_ID
  FROM EDM_VIEWS_PRD.DW_VIEWS.FACILITY AS F
  INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE AS RT
  ON F.FACILITY_NBR = RT.FACILITY_NBR
  AND RT.DW_LOGICAL_DELETE_IND = FALSE
  AND RT.DW_CURRENT_VERSION_IND = TRUE
  AND F.DW_LOGICAL_DELETE_IND = FALSE
  AND F.DW_CURRENT_VERSION_IND = TRUE
)
SELECT  alw.group_id
,       alw.corp_item_cd
,       alw.corp
,       alw.facility
,       alw.vend_num
,       alw.vend_sub_acnt
,       alw.cost_area
,       alw.rog
,       spa.price_area_id
,       CASE
            WHEN alw.allow_type = 'T' AND alw.perform_1 = '20' THEN 'R'
            ELSE alw.allow_type
        END AS alw_typ
,       alw.arrival_from_date
,       alw.arrival_to_date
,       alw.allow_amt
,       alw.amt_in_cost
,       alw.offer_number
  
FROM    EDM_SANDBOX_PRD.MERCHAPPS.T_PE_ALW_HDR_DTL_ALL alw
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.VENDOR_COST_AREA_FACILITY vca
ON      vca.corporation_id    = alw.corp
AND     vca.division_id       = alw.division
AND     vca.vendor_id         = alw.vend_num
AND     vca.VENDOR_SUB_ACCOUNT_ID = alw.vend_sub_acnt
AND     vca.COST_AREA_CD       = alw.cost_area
AND     vca.on_hold_ind <> 'H'  
AND     vca.DW_CURRENT_VERSION_IND = TRUE
AND     vca.DW_LOGICAL_DELETE_IND = FALSE
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.Authorized_Backdoor_Receiving_Item  dsd
ON      dsd.Corporate_item_integration_id = alw.genrt_cic_id
AND     dsd.facility_integration_id = vca.facility_integration_id
AND     dsd.vendor_id = alw.vend_num
AND     dsd.Vendor_Sub_Account_Id = alw.vend_sub_acnt
AND     alw.calendar_date BETWEEN dsd.Authorized_Item_Start_Dt AND dsd.Authorized_Item_End_Dt
AND     dsd.DW_LOGICAL_DELETE_IND = FALSE
AND     dsd.DW_CURRENT_VERSION_IND = TRUE
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_ROG_REFERENCE urx
ON      urx.rog_id = alw.rog
AND     urx.Corporate_item_integration_id = alw.genrt_cic_id
AND     alw.calendar_date BETWEEN urx.DW_FIRST_EFFECTIVE_DT AND urx.DW_LAST_EFFECTIVE_DT
AND     urx.DW_LOGICAL_DELETE_IND = FALSE
AND     urx.DW_CURRENT_VERSION_IND = TRUE
INNER   JOIN store_price_area spa  
ON      spa.rog_id = urx.rog_id
AND     spa.loc_retail_sect_id = urx.retail_section_cd
AND     spa.FACILITY_INTEGRATION_ID = vca.FACILITY_INTEGRATION_ID
AND     alw.calendar_date BETWEEN spa.first_eff_dt AND spa.last_eff_dt
LEFT    OUTER JOIN vendor_allow_hdr nah
ON      nah.offer_nbr = alw.offer_number
AND     nah.division_cd = alw.division
AND     nah.current_offer_ind = 'Y'
WHERE   alw.cost_area > 0
GROUP   BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15
 
UNION
 
SELECT  alw.group_id
,       alw.corp_item_cd
,       alw.corp
,       alw.facility
,       alw.vend_num
,       alw.vend_sub_acnt
,       alw.cost_area
,       alw.rog
,       spa.price_area_id
,       CASE
            WHEN alw.allow_type = 'T' AND alw.perform_1 = '20' THEN 'R'
            ELSE alw.allow_type
        END AS alw_typ
,       alw.arrival_from_date
,       alw.arrival_to_date
,       alw.allow_amt
,       alw.amt_in_cost
,       alw.offer_number
FROM    EDM_SANDBOX_PRD.MERCHAPPS.T_PE_ALW_HDR_DTL_ALL alw
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_ROG_REFERENCE urx
ON      urx.rog_id = alw.rog
AND     urx.Corporate_item_integration_id = alw.genrt_cic_id
AND     alw.calendar_date BETWEEN urx.DW_FIRST_EFFECTIVE_DT AND urx.DW_LAST_EFFECTIVE_DT
AND     urx.DW_LOGICAL_DELETE_IND = FALSE
AND     urx.DW_CURRENT_VERSION_IND = TRUE
INNER   JOIN LU_STORE str
ON      str.rog_id = urx.rog_id
AND     str.closed_dt = '9999-12-31'::DATE
AND     str.corporation_id = 1
INNER   JOIN store_price_area spa  
ON      spa.rog_id = urx.rog_id
AND     spa.loc_retail_sect_id = urx.retail_section_cd
AND     spa.store_id = str.store_id
AND     alw.calendar_date BETWEEN spa.first_eff_dt AND spa.last_eff_dt
LEFT    OUTER JOIN vendor_allow_hdr nah
ON      TRIM(nah.offer_nbr) = TRIM(alw.offer_number)
AND     TRIM(nah.division_cd) = TRIM(division)::VARCHAR(2)
AND     nah.current_offer_ind = 'Y'
WHERE   alw.cost_area = 0
AND     (nah.offer_nbr IS NOT NULL
OR      (alw.wds_division <> '65' AND nah.offer_nbr IS NULL))
GROUP   BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15;
-- EXPR_FORMAT - Convert expression FORMAT/CAST_AS_FORMAT to TO_CHAR/TO_DATE
-- FUN_DATE_CAST - Reformat STRING-to-DATE casting
 
-- COLLECT STATISTICS ON EDM_BIZOPS_PRD..t_pe_alw_final INDEX (facility, corp_item_cd);
-- COLLECT_STATS - Comment out COLLECT STATISTICS. requires further review.
-- COLLECT STATISTICS ON EDM_BIZOPS_PRD..t_pe_alw_final COLUMN (corp, facility, corp_item_cd, rog, price_area_id, vend_num, vend_sub_acnt, cost_area);
-- COLLECT_STATS - Comment out COLLECT STATISTICS. requires further review.
```

### Step 7
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_cpn_adplan_all;
-- DROP_TBL: Add if exists in drop table
```

### Step 8
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_cpn_adplan_all 
     (
      division_id INTEGER,
      --rog_id INTEGER,
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      store_id INTEGER,
      upc_id DECIMAL(14,0),
      offer_id DECIMAL(13,0),
      OFFER_TYPE_CD CHAR(3)  COLLATE 'en-ci' ,
      COUPON_AMT DECIMAL(8,2),
      promo_method_cd CHAR(2)  COLLATE 'en-ci' ,
      promo_min_purch_qty DECIMAL(2,0),
      promo_lim_qty SMALLINT,
      promo_prc DECIMAL(7,2),
      promo_prc_fctr DECIMAL(2,0),
      first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}');
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

### Step 9
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_cpn_adplan_all
(       division_id
,       rog_cd
,       store_id
,       upc_id
,       offer_id
,       offer_type_cd
,       coupon_amt
,       promo_method_cd
,       promo_min_purch_qty
,       promo_lim_qty
,       promo_prc
,       promo_prc_fctr
,       first_eff_dt
,       last_eff_dt
)
SELECT DISTINCT str.division_id
,       RT.rog_id
,       str.facility_nbr
,       sipp.upc_nbr
,       sipp.PROMOTION_STORE_INTEGRATION_ID
,      CASE WHEN sipp.Coupon_type_cd = 'P' THEN 'PCP'
        WHEN sipp.Coupon_type_cd = 'E' THEN 'ECP' END AS Coupon_type_cd
,       sipp.COUPON_AMT
,       sipp.COUPON_METHOD_CD
,       sipp.MINIMUM_PURCHASE_QTY
,       sipp.COUPON_LIMIT_QTY
,       CASE WHEN COUPON_METHOD_CD In ('NE', 'NW') And ORIGINAL_COUPON_FCTR <> 0 Then ORIGINAL_COUPON_AMT
        WHEN COUPON_METHOD_CD In ('NE', 'NW') And ORIGINAL_COUPON_FCTR = 0 Then COUPON_AMT
        WHEN  COUPON_METHOD_CD Not In ('NE', 'NW') Then COUPON_AMT END AS PROMO_PRC
  
,       CASE WHEN COUPON_METHOD_CD In ('NE', 'NW') And ORIGINAL_COUPON_FCTR <> 0 Then ORIGINAL_COUPON_FCTR 
        WHEN COUPON_METHOD_CD In ('NE', 'NW') And ORIGINAL_COUPON_FCTR = 0 Then 1
        WHEN  COUPON_METHOD_CD Not In ('NE', 'NW') Then 0 END AS PROMO_PRC_FCTR
  
,       sipp.PROMOTION_START_DT
,       sipp.PROMOTION_END_DT
FROM    EDM_VIEWS_PRD.DW_VIEWS.PROMOTION_STORE sipp
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.FACILITY str
ON      str.Facility_integration_id = sipp.Facility_integration_id 
AND     str.close_dt > CURRENT_DATE
AND     str.DW_CURRENT_VERSION_IND = TRUE 
AND     str.DW_LOGICAL_DELETE_IND = FALSE
AND     sipp.DW_CURRENT_VERSION_IND = TRUE 
AND     sipp.DW_LOGICAL_DELETE_IND = FALSE 
Inner Join EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE AS RT
ON  RT.FACILITY_NBR = STR.FACILITY_NBR
AND RT.DW_CURRENT_VERSION_IND = TRUE 
AND RT.DW_LOGICAL_DELETE_IND = FALSE 
AND RT.corporation_id = 1
WHERE sipp.Coupon_type_cd IN ('E','P')
-- AND     DATE '2021-07-01' BETWEEN sipp.PROMOTION_START_DT AND sipp.PROMOTION_END_DT;
--Akash:27/03/2019: Replaced AND condition with below lines
   AND (CASE
            WHEN (str.division_id = '33' or str.division_id = '34') THEN (current_date + """+str(east_gap)+""")::DATE
            ELSE (current_date + """+str(reg_gap)+""")::DATE
        END) BETWEEN sipp.PROMOTION_START_DT AND sipp.PROMOTION_END_DT
```

### Step 10
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_reg_rtl;
-- DROP_TBL: Add if exists in drop table
```

### Step 11
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_reg_rtl 
     (
      division_id INTEGER,
       --rog_id INTEGER,
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      store_id INTEGER,
      upc_id DECIMAL(14,0),
      first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      reg_rtl_prc DECIMAL(7,2),
      reg_rtl_prc_fctr DECIMAL(2,0));
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

### Step 12
```sql
INSERT
INTO   EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_reg_rtl
(       division_id
,       rog_cd
,       store_id
,       upc_id
,       first_eff_dt
,       reg_rtl_prc
,       reg_rtl_prc_fctr
)
SELECT
        str.division_id
,       str.rog_cd
,       reg.store_id
,       reg.upc_id
,       reg.first_eff_dt
,       reg.reg_rtl_prc
,       reg.reg_rtl_prc_fctr
FROM    
(select distinct
prip.ITEM_PRICE_FCTR as reg_rtl_prc_fctr
,prip.ITEM_PRICE_AMT as reg_rtl_prc
,prip.DW_FIRST_EFFECTIVE_DT as first_eff_dt
,prip.UPC_NBR as upc_id
,rs.FACILITY_NBR as store_id
,prip.DW_LAST_EFFECTIVE_DT as last_eff_dt
from EDM_VIEWS_PRD.dw_views.PENDING_RETAIL_ITEM_PRICE prip
inner join EDM_VIEWS_PRD.dw_views.d1_upc upc
inner join   EDM_VIEWS_PRD.dw_views.RETAIL_STORE_SECTION_PRICE_AREA rsspa
INNER   JOIN EDM_VIEWS_PRD.dw_views.retail_store rs
on prip.rog_id = rs.rog_id
and prip.price_area_cd = rsspa.price_area_cd
and prip.upc_nbr = upc.upc_nbr
and upc.section_cd = rsspa.section_cd
and  rs.facility_integration_id = rsspa.facility_integration_id
where  rs.dw_current_version_ind = TRUE and rs.DW_logical_delete_ind = FALSE
and rsspa.dw_current_version_ind = TRUE and prip.dw_current_version_ind = TRUE
and prip.DW_logical_delete_ind= FALSE  and rsspa.DW_logical_delete_ind= FALSE
and  upc.DW_logical_delete_ind= FALSE
and rs.Corporation_Id=001
) reg
INNER JOIN
(
select distinct
f.Facility_Nbr as store_id
,f.Close_Dt as closed_dt
,rs.Corporation_Id as corporation_id
,f.Division_Id as division_id
,rs.Rog_Id as rog_cd
from EDM_VIEWS_PRD.dw_views.retail_store rs
INNER join EDM_VIEWS_PRD.dw_views.facility f
on f.facility_integration_id = rs.facility_integration_id
where rs.dw_current_version_ind = TRUE and rs.DW_logical_delete_ind= FALSE
AND f.dw_current_version_ind = TRUE AND f.DW_logical_delete_ind= FALSE
and rs.Corporation_Id = 001
) str
ON      str.store_id = reg.store_id
AND     str.closed_dt > CURRENT_DATE
AND     str.corporation_id = 001
QUALIFY ROW_NUMBER() OVER (PARTITION BY reg.store_id, reg.upc_id ORDER BY reg.first_eff_dt ASC, reg.last_eff_dt ASC) = 1;
-- COLLECT STATISTICS ON EDM_BIZOPS_PRD..t_pe_pending_reg_rtl INDEX (store_id, upc_id);
-- COLLECT_STATS - Comment out COLLECT STATISTICS. requires further review.
-- COLLECT STATISTICS ON EDM_BIZOPS_PRD..t_pe_pending_reg_rtl COLUMN (rog_cd, upc_id);
-- COLLECT_STATS - Comment out COLLECT STATISTICS. requires further review.
```

### Step 13
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_dsd_cost;
-- DROP_TBL: Add if exists in drop table
```

### Step 14
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_dsd_cost 
     (
      corporation_id VARCHAR(3)  COLLATE 'en-ci' ,
      division_id INTEGER,
      CORP_ITEM_CD INTEGER,
      VEND_NBR VARCHAR(6)  COLLATE 'en-ci' ,
      VEND_SUB_ACCT_NBR VARCHAR(3)  COLLATE 'en-ci' ,
      COST_AREA_ID SMALLINT,
      date_eff DATE  comment '{"FORMAT":"YY/MM/DD"}',
      cost_vend DECIMAL(9,4));
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

### Step 15
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_dsd_cost
(       corporation_id
,       division_id
,       corp_item_cd
,       vend_nbr
,       vend_sub_acct_nbr
,       cost_area_id
,       date_eff
,       cost_vend
)
SELECT  corporation_cd  AS corporation_id
,       division_cd ::INTEGER   AS division_id
,       corp_item_cd
,       vend_nbr
,       vend_sub_acct_nbr
,       cost_area_id
,       start_eff_dt    AS date_eff
,       vend_cst        AS cost_vend
FROM    EDM_VIEWS_PRD.DW_EDW_VIEWS.dsd_cost_pending  --- Using Data Share for Now as the EDM Equivalent Table has Issues and Delete logic is Missing From ITEMMASTERCOST BOD adn they are fixing it
WHERE   date_eff > CURRENT_DATE
QUALIFY ROW_NUMBER() OVER (PARTITION BY corporation_cd, division_cd, corp_item_cd, vend_nbr, vend_sub_acct_nbr, cost_area_id ORDER BY start_eff_dt ASC, vend_cst ASC) = 1;
-- FUN_CAST_OPTR - Reformat casting
 
-- COLLECT STATISTICS ON temp_tables.t_pe_pending_dsd_cost INDEX (division_id, corp_item_cd);
-- COLLECT_STATS - Comment out COLLECT STATISTICS. requires further review.
```

### Step 16
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_whse_cost;
-- DROP_TBL: Add if exists in drop table
```

### Step 17
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_whse_cost 
     (
      dst_cntr VARCHAR(4)  COLLATE 'en-ci' ,
      CORP_ITEM_CD INTEGER,
      date_eff DATE  comment '{"FORMAT":"YY/MM/DD"}',
      cost_vend DECIMAL(9,4));
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

### Step 18
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_whse_cost
(       dst_cntr
,       corp_item_cd
,       date_eff
,       cost_vend
)
SELECT  dst_cntr_cd     AS dst_cntr
,       corp_item_cd
,       start_eff_dt    AS date_eff
,       vend_cst        AS cost_vend
FROM    EDM_VIEWS_PRD.DW_EDW_VIEWS.whse_cost_pending      --- Using Data Share for Now as the EDM Equivalent Table has Issues and Delete logic is Missing From ITEMMASTERCOST BOD adn they are fixing it
QUALIFY ROW_NUMBER() OVER (PARTITION BY dst_cntr_cd, corp_item_cd ORDER BY start_eff_dt ASC, vend_cst ASC) = 1;
 
-- COLLECT STATISTICS ON temp_tables.t_pe_pending_whse_cost INDEX (dst_cntr, corp_item_cd);
-- COLLECT_STATS - Comment out COLLECT STATISTICS. requires further review.
```

### Step 19
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl ;
-- DROP_TBL: Add if exists in drop table
```

### Step 20
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl 
     (
      division_id INTEGER,
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      store_id INTEGER,
      upc_id DECIMAL(14,0),
      last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      reg_rtl_prc DECIMAL(7,2),
      reg_rtl_prc_fctr DECIMAL(2,0));
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

### Step 21
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl
(       division_id
--,       rog_id
,       rog_cd
,       store_id
,       upc_id
,       last_eff_dt
,       reg_rtl_prc
,       reg_rtl_prc_fctr
)
SELECT DISTINCT  RT.division_id
--,       str.rog_id  --Column Internal to LDW
,       RT.rog_id
,       RT.RETAIL_STORE_FACILITY_NBR
,       reg.upc_nbr
,       reg.DW_LAST_EFFECTIVE_DT
,       reg.ITEM_PRICE_AMT
,       reg.ITEM_PRICE_FCTR
FROM    EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE_UPC_PRICE reg
inner join edm_views_prd.DW_VIEWS.d1_retail_store rt 
on reg.facility_integration_id= rt.facility_integration_id
WHERE   REG.DW_CURRENT_VERSION_IND = TRUE
and reg.DW_LAST_EFFECTIVE_DT > current_date
  
QUALIFY ROW_NUMBER() OVER (PARTITION BY rt.RETAIL_STORE_FACILITY_NBR, reg.upc_nbr ORDER BY reg.DW_LAST_EFFECTIVE_DT DESC, reg.DW_FIRST_EFFECTIVE_DT DESC) = 1; 
 
-- COLLECT STATISTICS ON temp_tables.t_pe_hist_reg_rtl INDEX (store_id, upc_id);
-- COLLECT_STATS - Comment out COLLECT STATISTICS. requires further review.
-- COLLECT STATISTICS ON temp_tables.t_pe_hist_reg_rtl COLUMN (rog_cd, upc_id);
-- COLLECT_STATS - Comment out COLLECT STATISTICS. requires further review.
```

### Step 22
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_item_attr_1;
-- DROP_TBL: Add if exists in drop table
```

### Step 23
```sql
CREATE or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_item_attr_1 
     (
      division_id INTEGER,
      store_id INTEGER,
      genrt_cic_id DECIMAL(12,0),
      corp_item_cd DECIMAL(8,0),
      cic_dsc VARCHAR(40)  COLLATE 'en-ci' ,
      group_id SMALLINT,
      category_id SMALLINT,
      vend_conv_fctr SMALLINT,
      -- pack_whse_qty DECIMAL(7,2),
      pack_retail_qty DECIMAL(7,2),
      pack_dsc CHAR(30)  COLLATE 'en-ci' ,
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      dst_cntr VARCHAR(4)  COLLATE 'en-ci' ,
      cic_vend_cst DECIMAL(9,4),
      corporation_id BYTEINT,
      WHSE_CD VARCHAR(4)  COLLATE 'en-ci' ,
      status_dst CHAR(1)  COLLATE 'en-ci' ,
      cic_ib_cst DECIMAL(9,4),
      item_type_cd DECIMAL(3,0),
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      retail_status_cd CHAR(1)  COLLATE 'en-ci' ,
      loc_common_retail_cd DECIMAL(5,0),
      upc_id DECIMAL(13,0),
      snstv_tier_cd CHAR(1)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      VEND_NBR VARCHAR(6)  COLLATE 'en-ci' ,
      VEND_SUB_ACCT_NBR VARCHAR(3)  COLLATE 'en-ci' ,
      loc_retail_sect_id INTEGER,
      BUYER_NBR CHAR(2)  COLLATE 'en-ci' ,
      unit_prc_tbl_nbr SMALLINT,
-- ARIS: 20190213: added cost_area_id
      cost_area_id SMALLINT) COMMENT = '{"multiset": true}' ;
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- CREATE_MULTISET_OPTION - Remove MULTISET create table option
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
-- T_CREATE_TABLE_META - Add table metadata to table comment
```

### Step 24
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.T_PE_DSD_ITEM_ATTR_1
(       division_id
,       store_id
,       genrt_cic_id
,       corp_item_cd
,       cic_dsc
,       group_id
,       category_id
,       vend_conv_fctr
-- ,       pack_whse_qty
,       pack_retail_qty
,       pack_dsc
,       size_dsc
,       dst_cntr
,       cic_vend_cst
,       corporation_id
,       WHSE_CD
,       status_dst
,       cic_ib_cst
,       item_type_cd
,       rog_cd
,       retail_status_cd
,       loc_common_retail_cd
,       upc_id
,       snstv_tier_cd
,       DIV_PROMO_GRP_CD
,       VEND_NBR
,       VEND_SUB_ACCT_NBR
,       loc_retail_sect_id
,       BUYER_NBR
,       unit_prc_tbl_nbr
-- ARIS: 20190213: added cost_area_id
,       cost_area_id
)
SELECT DISTINCT  
str.division_id
,       str.store_id
,     cic.genrt_cic_id
,       cic.corp_item_cd
,       cic.cic_dsc
,       cic.group_id
,       cic.category_id
,       cic.vend_conv_fctr
-- ,       cic.pack_whse_qty
,       urx.pack_retail_qty
,       urx.pack_dsc
,       urx.size_dsc
,       wds.dst_cntr
,       vcc.vend_cst as cic_vend_cst
,       1 AS corporation_id
,       wds.whse_cd
,       wds.status_dst
,       vcc.ib_cst AS cic_ib_cst
,       cic.item_type_cd
,       str.rog_cd
,       wir.retail_status_cd
,       urx.loc_common_retail_cd
,       urx.upc_id
,       urx.snstv_tier_cd
,       urx.div_promo_grp_cd
--AKASH : 20190530: changed dsd.vend_nbr and sub_acct_nbr to vcc.vend_nbr and sub_acct_nbr
,       vcc.vend_nbr
,       vcc.vend_sub_acct_nbr
,       urx.loc_retail_sect_id
,       wds.buyer_nbr    -- wds.buyer_nbr
,       urx.unit_prc_tbl_nbr
-- ARIS: 20190213: added cost_area_id
,       vcc.cost_area_id
FROM (
SELECT  
       wcvi.corporation_id
,      wcvi.Corporate_item_integration_id as genrt_cic_id
,       sci.division_id as division_cd
,       sci.Distribution_center_id as dst_cntr
,       wcvi.warehouse_id as whse_Cd
,       wcvi.vendor_id as vend_nbr
,       wcvi.vendor_sub_acct_id  as vend_sub_acct_nbr
,       sci.Warehouse_Item_Status_Cd as status_dst    
,       wcvi.item_billing_cost_amt as ib_cst
,       wcvi.vendor_unit_cost_amt as vend_cst
,       sci.warehouse_item_status_cd
,       wcvi.vendor_conversion_fctr
,       sci.buyer_id as buyer_nbr
FROM    (
        SELECT  DISTINCT
                vwic.corporation_id
        ,       vwic.division_id
        ,       vwic.warehouse_id
        ,        vwic.vendor_unit_cost_amt
        ,       vwic.Corporate_item_integration_id
        ,       vwic.item_billing_cost_amt
        ,       vi.vendor_id
        ,       vi.vendor_sub_acct_id
        ,       vi.wims_sub_vendor_nbr
        ,       vi.vendor_conversion_fctr
        FROM    EDM_VIEWS_PRD.DW_VIEWS.vendor_warehouse_item_cost vwic
        INNER JOIN
                EDM_VIEWS_PRD.DW_VIEWS.vendor_item  vi
             ON vi.warehouse_id                   = vwic.warehouse_id
            AND vi.corporate_item_integration_id  = vwic.corporate_item_integration_id
            AND vi.vendor_id                      = vwic.vendor_id
            AND vi.vendor_sub_acct_id             = vwic.vendor_sub_acct_id
            AND vi.wims_sub_vendor_nbr            = vwic.wims_sub_vendor_nbr
            AND vwic.dw_current_version_ind = true AND vwic.dw_logical_delete_ind = FALSE
            AND vi.dw_current_version_ind = true AND vi.dw_logical_delete_ind   = FALSE
 
        ) wcvi
INNER JOIN
        EDM_VIEWS_PRD.DW_VIEWS.supply_chain_item  sci
    ON sci.warehouse_id                  = wcvi.warehouse_id
    AND sci.corporate_item_integration_id = wcvi.corporate_item_integration_id
    AND sci.DW_CURRENT_VERSION_IND = True AND sci.DW_LOGICAL_DELETE_IND = FALSE
  
 
where  sci.Warehouse_Item_Status_Cd NOT IN ('X', 'D')
AND     SUBSTR(wcvi.WAREHOUSE_ID, 1, 2) = 'DD'
 AND     wcvi.VENDOR_ID <> '021069'    
) wds
INNER   JOIN (
 SELECT DISTINCT
 c.Corporate_Item_Integration_Id as genrt_cic_id,
 c.Corporation_Id as Corporation_Id,
 c.CORPORATE_ITEM_CD as corp_item_cd,
 c.item_dsc as cic_dsc,
 c.SMIC_Group_Cd as group_id,
 c.smic_group_cd * 100 + c.smic_category_cd as category_id, --SMIC_Category
 c.Vendor_Conversion_Fctr as vend_conv_fctr, --Vendor_Item
 C.COMPARISON_ITEM_TYPE_CD  as item_type_cd, --Corporate_Item_UPC_ROG_Reference
 CASE WHEN c.Display_Item_Ind in ('') THEN 'N' WHEN c.Display_Item_Ind is NULL THEN 'N' ELSE c.Display_Item_Ind  END as display_ind
 FROM EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM c
 where c.Corporation_Id=1 
 AND c.SMIC_Group_Cd IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,34,36,37,38,39,40,42,43,44,45,46,47,48,73,74,88,96,97,
49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,72,75,76,77,78,79,80,89,84,70,71,81,82,85,86,94,95) 
 AND c.DW_LOGICAL_DELETE_IND = FALSE
 AND c.DW_CURRENT_VERSION_IND = TRUE
) cic
ON      cic.genrt_cic_id = wds.genrt_cic_id
AND     cic.corporation_id = 1
AND     cic.display_ind <> 'Y'
-- AKASH: 20190312 : added this join criteria
AND     WDS.VEND_NBR <> '021069'
        -- Exclude MCLANE COMPANY
INNER   JOIN (
SELECT DISTINCT
ROGCI.Retail_Status_Ind  as retail_status_cd,
sci.Division_Id as whse_division_id,
ROGCI.WAREHOUSE_ID as whse_cd,
ROGCI.Corporate_Item_Integration_id as genrt_cic_id,
ROGCI.Rog_Id as rog_id
FROM
EDM_VIEWS_PRD.DW_VIEWS.Retail_Order_Group_Corporate_item as ROGCI
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.SUPPLY_CHAIN_ITEM AS SCI
ON SCI.Corporate_Item_Integration_id = ROGCI.Corporate_Item_Integration_id
AND SCI.WAREHOUSE_ID = ROGCI.WAREHOUSE_ID
AND SCI.DW_LOGICAL_DELETE_IND = FALSE 
AND SCI. DW_CURRENT_VERSION_IND = TRUE
WHERE
ROGCI.DW_LOGICAL_DELETE_IND = FALSE 
AND ROGCI. DW_CURRENT_VERSION_IND = TRUE
) wir
ON      wir.whse_division_id = wds.division_cd
AND     wir.whse_cd = wds.whse_cd
AND     wir.genrt_cic_id = cic.genrt_cic_id
AND     wir.retail_status_cd <> 'D'
INNER   JOIN (
SELECT  distinct
     vdic.corporation_id 
,       vdic.division_id as division_cd
,       vdic.vendor_id as vend_nbr
,       vdic.vendor_sub_account_id as vend_sub_acct_nbr
,       vdic.corporate_item_integration_id  as Genrt_cic_id
,       vdic.vendor_cost_area_cd as cost_area_id
,       vdic.item_billing_cost_amt as ib_cst
,       vdic.vendor_unit_cost_amt as vend_cst
--,       vbr.terms_id
,       vdsdi.vendor_item_status_cd as item_status_ind
FROM    EDM_VIEWS_PRD.DW_VIEWS.vendor_dsd_item_cost   vdic
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.vendor_direct_store_delivery_item vdsdi
     ON vdsdi.vendor_id = vdic.vendor_id
    AND vdsdi.vendor_sub_account_id  = vdic.vendor_sub_account_id
    AND vdsdi.corporate_item_integration_id  = vdic.corporate_item_integration_id 
    AND vdsdi.dw_logical_delete_ind = FALSE AND vdsdi.DW_CURRENT_VERSION_IND= TRUE
    AND vdic.dw_logical_delete_ind = FALSE AND vdic.DW_CURRENT_VERSION_IND= TRUE
 
) vcc
ON      vcc.division_cd ::INTEGER = wir.whse_division_id
AND     vcc.genrt_cic_id = wds.genrt_cic_id
-- MAYANK : 20190604 : added the below condition to restrict the vendors with hold status
AND     vcc.item_status_ind <> 'H'
INNER   JOIN (
SELECT DISTINCT
cr.RETAIL_UNIT_PACK_NBR as pack_retail_qty,
cr.SHELF_UNIT_PACK_DSC as pack_dsc,
cr.SHELF_UNIT_SIZE_DSC  as size_dsc,
cr.Common_Retail_Cd  as loc_common_retail_cd,
cr.UPC_NBR as upc_id,
cri.Sensitive_tier_cd as snstv_tier_cd,
COALESCE(cm.common_item_group_cd,0)::NUMBER as div_promo_grp_cd,
cr.RETAIL_SECTION_CD as loc_retail_sect_id,
rp.Unit_Price_Table_Nbr as unit_prc_tbl_nbr,
cr.DW_LAST_EFFECTIVE_DT as last_eff_dt,
cr.ROG_ID as rog_id,
cr.Corporate_Item_Integration_Id as genrt_cic_id,
cr.STATUS_CD as retail_status_cd
  
FROM "EDM_VIEWS_PRD"."DW_VIEWS"."CORPORATE_ITEM_UPC_ROG_REFERENCE" cr
 
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP AS ROG
ON ROG.ROG_ID = CR.ROG_ID 
AND ROG.DW_LOGICAL_DELETE_IND = FALSE AND ROG.DW_CURRENT_VERSION_IND = TRUE  
AND cr.DW_LOGICAL_DELETE_IND = FALSE AND cr.DW_CURRENT_VERSION_IND = TRUE
 
LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.Common_Item_Group_item cm ON
cr.Corporate_Item_Integration_Id =cm.Corporate_Item_Integration_Id
AND CM.DIVISION_ID = ROG.DIVISION_ID
AND cm.DW_LOGICAL_DELETE_IND = FALSE AND cm.DW_CURRENT_VERSION_IND = TRUE
 
  
LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.CURRENT_RETAIL_ITEM_PRICE AS CRI --- left join this because CURRENT_RETAIL_ITEM_PRICE appears to be missing data 1/21/2023 modified
ON cr.rog_id = cri.rog_id 
AND cr.upc_nbr = cri.upc_nbr
AND CRI.DW_LOGICAL_DELETE_IND = FALSE AND cri.DW_CURRENT_VERSION_IND = TRUE
AND cr.DW_LOGICAL_DELETE_IND = FALSE AND cr.DW_CURRENT_VERSION_IND = TRUE
  
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.Retail_Order_Group_UPC rp ON
cr.rog_id = rp.rog_id and cr.upc_nbr = rp.upc_nbr 
AND rp.DW_CURRENT_VERSION_IND = TRUE 
and rp.DW_LOGICAL_DELETE_IND = FALSE
) urx
ON      urx.rog_id = wir.rog_id
AND     urx.genrt_cic_id = wir.genrt_cic_id
 
-- MAYANK : 20190527 :  adding this condition to stop UPCs with this status
AND     urx.retail_status_cd NOT IN ('X','D')
-- ARIS: 20190213: added this join criteria
 
 INNER   JOIN (
 SELECT DISTINCT
r.Division_Id as division_id,
f.Facility_Nbr as store_id,
r.Rog_Id as rog_id,
r.Rog_Id as rog_cd,
r.Corporation_Id as corporation_id,
f.Close_Dt as closed_dt
FROM EDM_VIEWS_PRD.DW_VIEWS.Facility f
INNER JOIN  EDM_VIEWS_PRD.DW_VIEWS.Retail_Store r ON
f.Facility_nbr =r.Facility_nbr
and r.dw_current_version_ind=TRUE and r.DW_LOGICAL_DELETE_IND=FALSE
and f.dw_current_version_ind=TRUE and f.DW_LOGICAL_DELETE_IND=FALSE
AND  closed_dt > CURRENT_DATE
 ) str
ON      str.rog_id = urx.rog_id
AND     str.corporation_id = 1
AND     str.division_id NOT IN (45, 58)
INNER   JOIN 
(
SELECT DISTINCT
a.Corporate_Item_Integration_Id as genrt_cic_id,
rs.FACILITY_NBR as store_id,
a.Vendor_Id as vend_nbr,
a.Vendor_Sub_Account_Id as vend_sub_acct_nbr,
a.Authorized_Item_Start_Dt as auth_start_dt,
a.Authorized_Item_End_Dt as auth_end_dt
FROM EDM_VIEWS_PRD.DW_VIEWS.Authorized_Backdoor_Receiving_Item a
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE rs ON
a.facility_integration_id=rs.facility_integration_id
and a.dw_current_version_ind=TRUE and a.DW_LOGICAL_DELETE_IND=FALSE
and rs.dw_current_version_ind=TRUE and rs.DW_LOGICAL_DELETE_IND=FALSE
) dsd
ON      dsd.genrt_cic_id = cic.genrt_cic_id
AND     dsd.store_id = str.store_id
AND     dsd.vend_nbr = vcc.vend_nbr
AND     dsd.vend_sub_acct_nbr = vcc.vend_sub_acct_nbr
AND     (CASE
            WHEN (str.division_id = '33' or str.division_id = '34') THEN (current_date + """+str(east_gap)+""")::DATE
            ELSE (current_date + """+str(reg_gap)+""")::DATE
        END) BETWEEN dsd.auth_start_dt AND dsd.auth_end_dt
 
WHERE   wds.status_dst NOT IN ('X', 'D')
AND     SUBSTR(wds.whse_cd, 1, 2) = 'DD'
AND     str.store_id < 5000
AND     cic.group_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,34,36,37,38,39,40,42,43,44,45,46,47,48,73,74,88,96,97,
49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,72,75,76,77,78,79,80,89,84,70,71,81,82,85,86,94,95);
```

### Step 25
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_whse_item_attr_1;
-- DROP_TBL: Add if exists in drop table
```

### Step 26
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_whse_item_attr_1 
     (
      division_id INTEGER,
      store_id INTEGER,
      genrt_cic_id DECIMAL(12,0),
      corp_item_cd DECIMAL(8,0),
      cic_dsc VARCHAR(40)  COLLATE 'en-ci' ,
      group_id SMALLINT,
      category_id SMALLINT,
      vend_conv_fctr SMALLINT,
      -- pack_whse_qty DECIMAL(7,2),
      pack_retail_qty DECIMAL(7,2),
      pack_dsc CHAR(30)  COLLATE 'en-ci' ,
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      dst_cntr VARCHAR(4)  COLLATE 'en-ci' ,
      cic_vend_cst DECIMAL(9,4),
      corporation_id BYTEINT,
      WHSE_CD VARCHAR(4)  COLLATE 'en-ci' ,
      status_dst CHAR(1)  COLLATE 'en-ci' ,
      cic_ib_cst DECIMAL(9,4),
      item_type_cd DECIMAL(3,0),
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      retail_status_cd CHAR(1)  COLLATE 'en-ci' ,
      loc_common_retail_cd DECIMAL(5,0),
      upc_id DECIMAL(13,0),
      snstv_tier_cd CHAR(1)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      VEND_NBR VARCHAR(6)  COLLATE 'en-ci' ,
      VEND_SUB_ACCT_NBR VARCHAR(3)  COLLATE 'en-ci' ,
      loc_retail_sect_id INTEGER,
      BUYER_NBR CHAR(2)  COLLATE 'en-ci' ,
      unit_prc_tbl_nbr SMALLINT);
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

### Step 27
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_whse_item_attr_1
(       division_id
,       store_id
,       genrt_cic_id
,       corp_item_cd
,       cic_dsc
,       group_id
,       category_id
,       vend_conv_fctr
-- ,       pack_whse_qty
,       pack_retail_qty
,       pack_dsc
,       size_dsc
,       dst_cntr
,       cic_vend_cst
,       corporation_id
,       WHSE_CD
,       status_dst
,       cic_ib_cst
,       item_type_cd
,       rog_cd
,       retail_status_cd
,       loc_common_retail_cd
,       upc_id
,       snstv_tier_cd
,       DIV_PROMO_GRP_CD
,       VEND_NBR
,       VEND_SUB_ACCT_NBR
,       loc_retail_sect_id
,       BUYER_NBR
,       unit_prc_tbl_nbr
)
SELECT DISTINCT   str.division_id
,       str.store_id
,       cic.genrt_cic_id
,       cic.corp_item_cd
,       cic.cic_dsc
,       cic.group_id
,       cic.category_id
,       cic.vend_conv_fctr
-- ,       cic.pack_whse_qty
,       urx.pack_retail_qty
,       urx.pack_dsc
,       urx.size_dsc
,       wds.dst_cntr_cd AS dst_cntr
,       wds.vend_cst AS cic_vend_cst
,       1 AS corporation_id
,       wds.whse_cd
,       wds.status_dst_cd   AS status_dst
,       wds.ib_cst AS cic_ib_cst
,       cic.item_type_cd
-- ,       rog.rog_id
,       rog.rog_cd
,       wir.retail_status_cd
,       urx.loc_common_retail_cd
 ,      urx.upc_id
,       urx.snstv_tier_cd
,       urx.div_promo_grp_cd
,       wds.vend_nbr
,       wds.vend_sub_acct_nbr
,       urx.loc_retail_sect_id
,       wds.buyer_nbr
,       urx.unit_prc_tbl_nbr
FROM(
SELECT  DISTINCT
       wcvi.corporation_id
,      wcvi.Corporate_item_integration_id as genrt_cic_id
,       sci.division_id as division_cd
,       sci.Distribution_center_id as dst_cntr_cd
,       wcvi.warehouse_id as whse_Cd
,       wcvi.vendor_id as vend_nbr
,       wcvi.vendor_sub_acct_id  as vend_sub_acct_nbr
,       sci.Warehouse_Item_Status_Cd as status_dst_cd     
,       wcvi.item_billing_cost_amt as ib_cst
,       wcvi.vendor_unit_cost_amt as vend_cst
,       sci.warehouse_item_status_cd
,       wcvi.vendor_conversion_fctr
,       sci.buyer_id as buyer_nbr
FROM    (
        SELECT  DISTINCT
                vwic.corporation_id
        ,       vwic.division_id
        ,       vwic.warehouse_id
        ,        vwic.vendor_unit_cost_amt
        ,       vwic.Corporate_item_integration_id
        ,       vwic.item_billing_cost_amt
        ,       vi.vendor_id
        ,       vi.vendor_sub_acct_id
        ,       vi.wims_sub_vendor_nbr
        ,       vi.vendor_conversion_fctr
        FROM    EDM_VIEWS_PRD.DW_VIEWS.vendor_warehouse_item_cost vwic
        INNER JOIN
                EDM_VIEWS_PRD.DW_VIEWS.vendor_item  vi
             ON vi.warehouse_id                   = vwic.warehouse_id
            AND vi.corporate_item_integration_id  = vwic.corporate_item_integration_id
            AND vi.vendor_id                      = vwic.vendor_id
            AND vi.vendor_sub_acct_id             = vwic.vendor_sub_acct_id
            AND vi.wims_sub_vendor_nbr            = vwic.wims_sub_vendor_nbr
            AND vwic.dw_current_version_ind = true AND vwic.dw_logical_delete_ind = FALSE
            AND vi.dw_current_version_ind = true AND vi.dw_logical_delete_ind   = FALSE
 
        ) wcvi
INNER JOIN
        EDM_VIEWS_PRD.DW_VIEWS.supply_chain_item  sci
    ON sci.warehouse_id                  = wcvi.warehouse_id
    AND sci.corporate_item_integration_id = wcvi.corporate_item_integration_id
    AND sci.DW_CURRENT_VERSION_IND = True AND sci.DW_LOGICAL_DELETE_IND = FALSE
  
 
where  sci.Warehouse_Item_Status_Cd NOT IN ('X', 'D')
AND     SUBSTR(wcvi.WAREHOUSE_ID, 1, 2) <> 'DD'
 AND     wcvi.VENDOR_ID <> '021069'
) wds
INNER   JOIN (
 SELECT DISTINCT
 c.Corporate_Item_Integration_Id as genrt_cic_id,
 c.Corporation_Id as Corporation_Id,
 c.CORPORATE_ITEM_CD as corp_item_cd,
 c.item_dsc as cic_dsc,
 c.SMIC_Group_Cd as group_id,
 c.smic_group_cd * 100 + c.smic_category_cd as category_id, --SMIC_Category
 c.Vendor_Conversion_Fctr as vend_conv_fctr, --Vendor_Item
 C.COMPARISON_ITEM_TYPE_CD  as item_type_cd, --Corporate_Item_UPC_ROG_Reference
 CASE WHEN c.Display_Item_Ind in ('') THEN 'N' WHEN c.Display_Item_Ind is NULL THEN 'N' ELSE c.Display_Item_Ind  END as display_ind
 FROM EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM c
 where c.DW_LOGICAL_DELETE_IND = FALSE
 AND c.DW_CURRENT_VERSION_IND = TRUE 
) cic
ON      cic.Genrt_cic_id = wds.genrt_cic_id
--AND     cic.non_merged_cic_ind = 'Y'
AND     cic.corporation_id = 1
AND     cic.display_ind <> 'Y'
AND cic.group_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,34,36,37,38,39,40,42,43,44,45,46,47,48,73,74,88,96,97,
49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,72,75,76,77,78,79,80,89,84,70,71,81,82,85,86,94,95)
-- AKASH: 20190312 : added this join criteria
        -- Exclude MCLANE COMPANY
INNER   JOIN (
SELECT DISTINCT
ROGCI.Retail_Status_Ind  as retail_status_cd,
sci.Division_Id as whse_division_id,
ROGCI.WAREHOUSE_ID as whse_cd,
ROGCI.Corporate_Item_Integration_id as genrt_cic_id,
ROGCI.Rog_Id as rog_id
FROM
EDM_VIEWS_PRD.DW_VIEWS.Retail_Order_Group_Corporate_item as ROGCI
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.SUPPLY_CHAIN_ITEM AS SCI
ON SCI.Corporate_Item_Integration_id = ROGCI.Corporate_Item_Integration_id
AND SCI.WAREHOUSE_ID = ROGCI.WAREHOUSE_ID
AND SCI.DW_LOGICAL_DELETE_IND = FALSE 
AND SCI. DW_CURRENT_VERSION_IND = TRUE
WHERE
ROGCI.DW_LOGICAL_DELETE_IND = FALSE 
AND ROGCI. DW_CURRENT_VERSION_IND = TRUE 
) wir
ON      wir.whse_division_id = wds.division_cd
AND     wir.whse_cd = wds.whse_cd
AND     wir.genrt_cic_id = cic.genrt_cic_id
AND     wir.retail_status_cd <> 'D'
 
 
 
INNER   JOIN (
 SELECT DISTINCT
cr.RETAIL_UNIT_PACK_NBR as pack_retail_qty,
cr.SHELF_UNIT_PACK_DSC as pack_dsc,
cr.SHELF_UNIT_SIZE_DSC  as size_dsc,
COALESCE(cr.Common_Retail_Cd,cic.Common_Retail_Cd)  as loc_common_retail_cd,
cr.UPC_NBR as upc_id,
cri.Sensitive_tier_cd as snstv_tier_cd,
COALESCE(cm.common_item_group_cd,0)::NUMBER as div_promo_grp_cd,
cr.RETAIL_SECTION_CD as loc_retail_sect_id,
rp.Unit_Price_Table_Nbr as unit_prc_tbl_nbr,
cr.DW_LAST_EFFECTIVE_DT as last_eff_dt,
cr.ROG_ID as rog_id,
cr.Corporate_Item_Integration_Id as genrt_cic_id,
cr.STATUS_CD as retail_status_cd
  
FROM "EDM_VIEWS_PRD"."DW_VIEWS"."CORPORATE_ITEM_UPC_ROG_REFERENCE" cr
 
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP AS ROG
ON ROG.ROG_ID = CR.ROG_ID 
AND ROG.DW_LOGICAL_DELETE_IND = FALSE AND ROG.DW_CURRENT_VERSION_IND = TRUE  
AND cr.DW_LOGICAL_DELETE_IND = FALSE AND cr.DW_CURRENT_VERSION_IND = TRUE
 
LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.Common_Item_Group_item cm ON
cr.Corporate_Item_Integration_Id =cm.Corporate_Item_Integration_Id
AND CM.DIVISION_ID = ROG.DIVISION_ID
AND cm.DW_LOGICAL_DELETE_IND = FALSE AND cm.DW_CURRENT_VERSION_IND = TRUE
 
LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.CURRENT_RETAIL_ITEM_PRICE AS CRI --- left join this because CURRENT_RETAIL_ITEM_PRICE appears to be missing data 1/21/2023 modified
ON CRI.Corporate_Item_Integration_Id = CR.Corporate_Item_Integration_Id 
AND cr.rog_id = cri.rog_id 
AND cr.upc_nbr = cri.upc_nbr
AND CRI.DW_LOGICAL_DELETE_IND = FALSE AND cri.DW_CURRENT_VERSION_IND = TRUE
AND cr.DW_LOGICAL_DELETE_IND = FALSE AND cr.DW_CURRENT_VERSION_IND = TRUE
  
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM AS CIC
ON CIC.CORPORATE_ITEM_INTEGRATION_ID = CR.CORPORATE_ITEM_INTEGRATION_ID
AND CIC.DW_LOGICAL_DELETE_IND = FALSE AND CIC.DW_CURRENT_VERSION_IND = TRUE
  
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.Retail_Order_Group_UPC rp ON
cr.rog_id = rp.rog_id and cr.upc_nbr = rp.upc_nbr 
AND rp.DW_CURRENT_VERSION_IND = TRUE 
and rp.DW_LOGICAL_DELETE_IND = FALSE
 
) urx
ON      urx.rog_id = wir.rog_id
AND     urx.genrt_cic_id = wir.genrt_cic_id
-- MAYANK : 20190527 :  adding this condition to stop UPCs with this status
AND     urx.retail_status_cd NOT IN ('X','D')
-- ARIS: 20190213: added this join criteria
 
 
INNER   JOIN (
SELECT DISTINCT
Rog_Id as rog_id,
Rog_Id as rog_cd
FROM
EDM_VIEWS_PRD.DW_VIEWS.Retail_Order_Group
where 
DW_LOGICAL_DELETE_IND = FALSE 
AND DW_CURRENT_VERSION_IND = TRUE 
) rog
ON      rog.rog_id = urx.rog_id
 
INNER   JOIN (
SELECT DISTINCT
r.Division_Id as division_id,
f.Facility_Nbr as store_id,
r.Rog_Id,
r.Corporation_Id as corporation_id,
f.Close_Dt as closed_dt
FROM EDM_VIEWS_PRD.DW_VIEWS.Facility f
INNER JOIN  EDM_VIEWS_PRD.DW_VIEWS.Retail_Store r ON
f.Facility_nbr =r.Facility_nbr
and r.dw_current_version_ind=TRUE and r.DW_LOGICAL_DELETE_IND=FALSE
and f.dw_current_version_ind=TRUE and f.DW_LOGICAL_DELETE_IND=FALSE
AND  closed_dt > CURRENT_DATE
) str
ON      str.rog_id = rog.rog_id
AND     str.corporation_id = 1
AND     str.division_id NOT IN (45, 58)
AND     str.store_id < 5000;
 
 
-- FUN_DATE_CAST - Reformat STRING-to-DATE casting
```

### Step 28
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_item_attr_2;
-- DROP_TBL: Add if exists in drop table
```

### Step 29
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_item_attr_2 
     (
      division_id INTEGER,
      store_id INTEGER,
      genrt_cic_id DECIMAL(12,0),
      corp_item_cd DECIMAL(8,0),
      cic_dsc VARCHAR(40)  COLLATE 'en-ci' ,
      group_id SMALLINT,
      category_id SMALLINT,
      vend_conv_fctr SMALLINT,
      -- pack_whse_qty DECIMAL(7,2),
      pack_retail_qty DECIMAL(7,2),
      pack_dsc CHAR(30)  COLLATE 'en-ci' ,
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      dst_cntr VARCHAR(4)  COLLATE 'en-ci' ,
      cic_vend_cst DECIMAL(9,4),
      corporation_id BYTEINT,
      WHSE_CD VARCHAR(4)  COLLATE 'en-ci' ,
      status_dst CHAR(1)  COLLATE 'en-ci' ,
      cic_ib_cst DECIMAL(9,4),
      item_type_cd DECIMAL(3,0),
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      retail_status_cd CHAR(1)  COLLATE 'en-ci' ,
      loc_common_retail_cd DECIMAL(5,0),
      upc_id DECIMAL(13,0),
      snstv_tier_cd CHAR(1)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      VEND_NBR VARCHAR(6)  COLLATE 'en-ci' ,
      VEND_SUB_ACCT_NBR VARCHAR(3)  COLLATE 'en-ci' ,
      loc_retail_sect_id INTEGER,
      BUYER_NBR CHAR(2)  COLLATE 'en-ci' ,
      unit_prc_tbl_nbr SMALLINT,
      price_area_id INTEGER,
      cost_area_id SMALLINT,
      buyer_nm CHAR(20)  COLLATE 'en-ci' ,
      vend_nm CHAR(40)  COLLATE 'en-ci' ,
      reg_rtl_prc DECIMAL(7,2),
      reg_rtl_prc_fctr DECIMAL(2,0),
      offer_id DECIMAL(13,0),
      COUPON_AMT DECIMAL(8,2),
      promo_method_cd CHAR(2)  COLLATE 'en-ci' ,
      promo_min_purch_qty DECIMAL(2,0),
      promo_lim_qty SMALLINT,
      promo_prc DECIMAL(7,2),
      promo_prc_fctr DECIMAL(2,0),
      first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_reg_rtl_prc DECIMAL(7,2),
      prtl_reg_rtl_prc_fctr DECIMAL(2,0),
      adplan_flag VARCHAR(1)  COLLATE 'en-ci' ,
      up_mult_factor DECIMAL(7,3),
      up_label_unit VARCHAR(5)  COLLATE 'en-ci' ,
      pending_cost DECIMAL(9,4),
      pending_cost_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
-- ARIS: 20190213: removed the following columns.  These will be calculated later at price area level
--      hist_reg_rtl_prc DECIMAL(7,2),
--      hist_reg_rtl_prc_fctr DECIMAL(2,0),
--      hist_last_eff_dt DATE FORMAT 'YY/MM/DD',
      alw_typ VARCHAR(1)  COLLATE 'en-ci' ,
      arrival_from_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      arrival_to_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      allow_amt DECIMAL(11,4),
      amt_in_cost DECIMAL(11,4));
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

### Step 30
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_item_attr_2
(       division_id
,       store_id
,       genrt_cic_id
,       corp_item_cd
,       cic_dsc
,       group_id
,       category_id
,       vend_conv_fctr
-- ,       pack_whse_qty
,       pack_retail_qty
,       pack_dsc
,       size_dsc
,       dst_cntr
,       cic_vend_cst
,       corporation_id
,       WHSE_CD
,       status_dst
,       cic_ib_cst
,       item_type_cd
,       rog_cd
,       retail_status_cd
,       loc_common_retail_cd
,       upc_id
,       snstv_tier_cd
,       DIV_PROMO_GRP_CD
,       VEND_NBR
,       VEND_SUB_ACCT_NBR
,       loc_retail_sect_id
,       BUYER_NBR
,       unit_prc_tbl_nbr
,       price_area_id
,       cost_area_id
,       buyer_nm
,       vend_nm
,       reg_rtl_prc
,       reg_rtl_prc_fctr
,       offer_id
,       COUPON_AMT
,       promo_method_cd
,       promo_min_purch_qty
,       promo_lim_qty
,       promo_prc
,       promo_prc_fctr
,       first_eff_dt
,       last_eff_dt
,       prtl_first_eff_dt
,       prtl_reg_rtl_prc
,       prtl_reg_rtl_prc_fctr
,       adplan_flag
,       up_mult_factor
,       up_label_unit
,       pending_cost
,       pending_cost_date
-- ARIS: 20190213: removed the following columns.  These will be calculated later at price area level
--,       hist_reg_rtl_prc
--,       hist_reg_rtl_prc_fctr
--,       hist_last_eff_dt
,       alw_typ
,       arrival_from_date
,       arrival_to_date
,       allow_amt
,       amt_in_cost
)
SELECT  tmp.division_id
,       tmp.store_id
,       tmp.genrt_cic_id
,       tmp.corp_item_cd
,       tmp.cic_dsc
,       tmp.group_id
,       tmp.category_id
,       tmp.vend_conv_fctr
-- ,       tmp.pack_whse_qty
,       tmp.pack_retail_qty
,       tmp.pack_dsc
,       tmp.size_dsc
,       tmp.dst_cntr
,       tmp.cic_vend_cst
,       tmp.corporation_id
,       tmp.WHSE_CD
,       tmp.status_dst
,       tmp.cic_ib_cst
,       tmp.item_type_cd
--,       tmp.rog_id
,       tmp.rog_cd
,       tmp.retail_status_cd
,       tmp.loc_common_retail_cd
,       tmp.upc_id
,       tmp.snstv_tier_cd
,       tmp.DIV_PROMO_GRP_CD
,       tmp.VEND_NBR
,       tmp.VEND_SUB_ACCT_NBR
,       tmp.loc_retail_sect_id
,       tmp.BUYER_NBR
,       tmp.unit_prc_tbl_nbr
,       spa.price_area_cd
,       vca.COST_AREA_CD
,       buy.buyer_nm
,       ven.VENDOR_NM
,       reg.ITEM_PRICE_AMT
,       reg.ITEM_PRICE_FCTR
,       cpn.offer_id
,       cpn.coupon_amt
,       cpn.promo_method_cd
,       cpn.promo_min_purch_qty
,       cpn.promo_lim_qty
,       cpn.promo_prc
,       cpn.promo_prc_fctr
,       cpn.first_eff_dt
,       cpn.last_eff_dt
,       prtl.first_eff_dt AS prtl_first_eff_dt
,       prtl.reg_rtl_prc  AS prtl_reg_rtl_prc
,       prtl.reg_rtl_prc_fctr   AS prtl_reg_rtl_prc_fctr
,       CASE WHEN adp.store_id IS NULL THEN ' ' ELSE 'Y' END AS adplan_flag
,       uni.UNIT_PRICE_MULTIPLICATION_FCTR  AS up_mult_factor
,       uni.UNIT_PRICE_LABEL_UNIT_CD  AS up_label_unit
,       COALESCE(dcst.cost_vend, wcst.cost_vend) AS pending_cost
,       COALESCE(dcst.date_eff, wcst.date_eff) AS pending_cost_date
-- ARIS: 20190213: removed the following columns.  These will be calculated later at price area level
--,       hrtl.reg_rtl_prc AS hist_reg_rtl_prc
--,       hrtl.reg_rtl_prc_fctr AS hist_reg_rtl_prc_fctr
--,       hrtl.last_eff_dt AS hist_last_eff_dt
,       alw.alw_typ
,       alw.arrival_from_date
,       alw.arrival_to_date
,       alw.allow_amt
,       alw.amt_in_cost
FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_item_attr_1 tmp
INNER   JOIN (
        SELECT  DISTINCT rsspa.facility_integration_id
,       rs.facility_nbr
,       rsspa.section_cd
,       rsspa.price_area_cd 
,       rs.rog_ID
        FROM    EDM_VIEWS_PRD.dw_views.RETAIL_STORE_SECTION_PRICE_AREA rsspa
        INNER   JOIN EDM_VIEWS_PRD.dw_views.retail_store rs
        ON      rs.facility_integration_id = rsspa.facility_integration_id
        inner join EDM_VIEWS_PRD.dw_views.Retail_Order_Group rog
        on rs.rog_ID=rog.rog_ID
        
        WHERE   rsspa.dw_current_version_ind = TRUE
        AND     rsspa.dw_logical_delete_ind = false 
        AND     rs.dw_current_version_ind = TRUE
        AND     rs.dw_logical_delete_ind = false
        
        and  rsspa.dw_last_effective_dt = '9999-12-31'::DATE
        and rog.dw_current_version_ind = TRUE
        AND rog.DW_LOGICAL_DELETE_IND = FALSE
        ) spa
ON     
try_to_number(spa.section_cd) = tmp.loc_retail_sect_id
AND   spa.facility_nbr = tmp.store_id
 
INNER   JOIN 
(
 Select Distinct  vc.Cost_area_cd , F.Facility_nbr,vc.VENDOR_ID ,vc.VENDOR_SUB_ACCOUNT_ID,vc.ON_HOLD_IND
From edm_views_prd.dw_views.VENDOR_COST_AREA_FACILITY vc
Inner join EDM_VIEWS_PRD.DW_VIEWS.FACILITY AS F
ON F.FACILITY_INTEGRATION_ID = VC.FACILITY_INTEGRATION_ID
AND     vc.DW_CURRENT_VERSION_IND = TRUE
AND     vc.DW_LOGICAL_DELETE_IND = FALSE
AND     F.DW_CURRENT_VERSION_IND = TRUE
AND     F.DW_LOGICAL_DELETE_IND = FALSE
)vca
ON      vca.VENDOR_ID = tmp.vend_nbr
and     vca.VENDOR_SUB_ACCOUNT_ID = tmp.vend_sub_acct_nbr
AND     vca.FACILITY_NBR = tmp.store_id
-- ARIS: 20190213: added cost_area_id to the join conditions
AND     vca.COST_AREA_CD = tmp.cost_area_id
AND     vca.ON_HOLD_IND <> 'H'
 
INNER   JOIN (
SELECT DISTINCT
  buyer_id,
  buyer_nm
FROM
EDM_VIEWS_PRD.DW_VIEWS.BUYER
where     dw_current_version_ind = TRUE
AND     DW_LOGICAL_DELETE_IND = FALSE
) buy
ON      buy.buyer_id = tmp.buyer_nbr
 
INNER   JOIN (
SELECT DISTINCT
VM.VENDOR_ID as vend_nbr,
VO.CORPORATION_ID as corporation_id,
VM.VENDOR_NM as VENDOR_NM 
FROM
EDM_VIEWS_PRD.DW_VIEWS.VENDOR_MASTER VM
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.VW_VENDOR_OUTLET VO
ON VM.VENDOR_ID = VO.VENDOR_ID
AND VO.DW_LOGICAL_DELETE_IND = FALSE
AND VO.DW_CURRENT_VERSION_IND = TRUE
AND VM.DW_LOGICAL_DELETE_IND = FALSE
AND VM.DW_CURRENT_VERSION_IND = TRUE
) ven
ON      ven.vend_nbr = tmp.vend_nbr
AND     ven.corporation_id = tmp.corporation_id
 
LEFT    OUTER JOIN (
  select distinct 
  f.FACILITY_NBR as FACILITY_NBR,
  rs.ITEM_PRICE_AMT as ITEM_PRICE_AMT,
  rs.ITEM_PRICE_FCTR as ITEM_PRICE_FCTR,
  rs.UPC_NBR as UPC_NBR,
  rs.DW_FIRST_EFFECTIVE_DT as DW_FIRST_EFFECTIVE_DT,
  rs.DW_LAST_EFFECTIVE_DT as DW_LAST_EFFECTIVE_DT
  from 
  EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE_UPC_PRICE rs
  join EDM_VIEWS_PRD.DW_VIEWS.facility f on
  rs.FACILITY_INTEGRATION_ID=f.FACILITY_INTEGRATION_ID
  and
  rs.DW_LOGICAL_DELETE_IND = FALSE
AND     rs.DW_CURRENT_VERSION_IND = TRUE
  and
  f.DW_LOGICAL_DELETE_IND = FALSE
AND     f.DW_CURRENT_VERSION_IND = TRUE
)  reg 
ON      reg.FACILITY_NBR = tmp.store_id
AND 
reg.UPC_NBR   = tmp.upc_id
AND     CURRENT_DATE BETWEEN reg.DW_FIRST_EFFECTIVE_DT AND reg.DW_LAST_EFFECTIVE_DT
 
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_cpn_adplan_all cpn
ON      cpn.store_id = tmp.store_id
AND     cpn.upc_id = tmp.upc_id
AND     cpn.offer_type_cd IN ('ECP', 'PCP')
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_reg_rtl prtl
ON      prtl.store_id = tmp.store_id
AND     prtl.upc_id   = tmp.upc_id
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_cpn_adplan_all adp
ON      adp.store_id = tmp.store_id
AND     adp.upc_id = tmp.upc_id
AND     adp.offer_type_cd IN ('AD', 'LTS')
 
LEFT    OUTER JOIN (
SELECT DISTINCT 
 UNIT_PRICE_TABLE_NBR,
 UNIT_PRICE_MULTIPLICATION_FCTR ,
UNIT_PRICE_LABEL_UNIT_CD
FROM 
EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP_UPC
where 
dw_current_version_ind = TRUE
and DW_LOGICAL_DELETE_IND = FALSE
) uni
ON      uni.UNIT_PRICE_TABLE_NBR = tmp.unit_prc_tbl_nbr
 
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_dsd_cost dcst
ON      dcst.corp_item_cd  = tmp.corp_item_cd
AND     dcst.vend_nbr      = tmp.vend_nbr
AND     dcst.vend_sub_acct_nbr = tmp.vend_sub_acct_nbr
-- ARIS: 20190213: change join from vca.cost_area_id to tmp.cost_area_id
AND     dcst.cost_area_id = tmp.cost_area_id
AND     dcst.division_id = tmp.division_id
 
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_pending_whse_cost wcst
ON      wcst.dst_cntr = tmp.dst_cntr
AND     wcst.corp_item_cd = tmp.corp_item_cd
-- ARIS: 20190213: removed the following table
--LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl hrtl
--ON      hrtl.store_id = tmp.store_id
--AND     hrtl.upc_id   = tmp.upc_id
 
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_alw_final alw
ON      alw.corp = tmp.corporation_id
AND     alw.facility = tmp.whse_cd
AND     alw.corp_item_cd = tmp.corp_item_cd
AND     alw.rog = tmp.rog_cd
AND     alw.price_area_id = spa.price_area_cd
AND     alw.vend_num = tmp.vend_nbr
AND     alw.vend_sub_acnt = tmp.vend_sub_acct_nbr
-- ARIS: 20190213: change join from vca.cost_area_id to tmp.cost_area_id
AND     alw.cost_area = tmp.cost_area_id
--jyoti:20190306-exclusion
WHERE 
--exclusion for seattle
NOT(tmp.division_id=27 AND tmp.rog_cd='SSEA' AND spa.price_area_cd=46)
AND NOT(tmp.division_id=27 AND tmp.rog_cd='SACG' AND spa.price_area_cd=9)
-- exclusion for denever
AND NOT(tmp.division_id=5 AND tmp.rog_cd ='SDEN' AND spa.price_area_cd in (3,14))
-- exclusion for shaws
AND NOT(tmp.division_id=33 AND tmp.rog_cd ='SEAS' AND spa.price_area_cd=1)
-- exclusion for Southern
AND NOT(tmp.division_id=20 AND tmp.rog_cd ='ADAL' AND spa.price_area_cd=6)
-- exclusion for Eastern
AND NOT(tmp.division_id=35 AND tmp.rog_cd ='SEAS' AND spa.price_area_cd=1)
-- exclusion for Portland
AND NOT(tmp.division_id=19 AND tmp.rog_cd ='SPRT' AND spa.price_area_cd in (61));
```

## Step 31
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_whse_item_attr_2;
-- DROP_TBL: Add if exists in drop table
```

## Step 32
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_whse_item_attr_2 
     (
      division_id INTEGER,
      store_id INTEGER,
      genrt_cic_id DECIMAL(12,0),
      corp_item_cd DECIMAL(8,0),
      cic_dsc VARCHAR(40)  COLLATE 'en-ci' ,
      group_id SMALLINT,
      category_id SMALLINT,
      vend_conv_fctr SMALLINT,
      -- pack_whse_qty DECIMAL(7,2),
      pack_retail_qty DECIMAL(7,2),
      pack_dsc CHAR(30)  COLLATE 'en-ci' ,
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      dst_cntr VARCHAR(4)  COLLATE 'en-ci' ,
      cic_vend_cst DECIMAL(9,4),
      corporation_id BYTEINT,
      WHSE_CD VARCHAR(4)  COLLATE 'en-ci' ,
      status_dst CHAR(1)  COLLATE 'en-ci' ,
      cic_ib_cst DECIMAL(9,4),
      item_type_cd DECIMAL(3,0),
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      retail_status_cd CHAR(1)  COLLATE 'en-ci' ,
      loc_common_retail_cd DECIMAL(5,0),
      upc_id DECIMAL(13,0),
      snstv_tier_cd CHAR(1)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      VEND_NBR VARCHAR(6)  COLLATE 'en-ci' ,
      VEND_SUB_ACCT_NBR VARCHAR(3)  COLLATE 'en-ci' ,
      loc_retail_sect_id INTEGER,
      BUYER_NBR CHAR(2)  COLLATE 'en-ci' ,
      unit_prc_tbl_nbr SMALLINT,
      price_area_id INTEGER,
      cost_area_id SMALLINT,
      buyer_nm CHAR(20)  COLLATE 'en-ci' ,
      vend_nm CHAR(40)  COLLATE 'en-ci' ,
      reg_rtl_prc DECIMAL(7,2),
      reg_rtl_prc_fctr DECIMAL(2,0),
      offer_id DECIMAL(13,0),
      COUPON_AMT DECIMAL(8,2),
      promo_method_cd CHAR(2)  COLLATE 'en-ci' ,
      promo_min_purch_qty DECIMAL(2,0),
      promo_lim_qty SMALLINT,
      promo_prc DECIMAL(7,2),
      promo_prc_fctr DECIMAL(2,0),
      first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_reg_rtl_prc DECIMAL(7,2),
      prtl_reg_rtl_prc_fctr DECIMAL(2,0),
      adplan_flag VARCHAR(1)  COLLATE 'en-ci' ,
      up_mult_factor DECIMAL(7,3),
      up_label_unit VARCHAR(5)  COLLATE 'en-ci' ,
      pending_cost DECIMAL(9,4),
      pending_cost_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
-- ARIS: 20190213: removed the following columns.  These will be calculated later at price area level
--      hist_reg_rtl_prc DECIMAL(7,2),
--      hist_reg_rtl_prc_fctr DECIMAL(2,0),
--      hist_last_eff_dt DATE FORMAT 'YY/MM/DD',
      alw_typ VARCHAR(1)  COLLATE 'en-ci' ,
      arrival_from_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      arrival_to_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      allow_amt DECIMAL(11,4),
      amt_in_cost DECIMAL(11,4));
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

## Step 33
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_whse_item_attr_2
(       division_id
,       store_id
,       genrt_cic_id
,       corp_item_cd
,       cic_dsc
,       group_id
,       category_id
,       vend_conv_fctr
-- ,       pack_whse_qty
,       pack_retail_qty
,       pack_dsc
,       size_dsc
,       dst_cntr
,       cic_vend_cst
,       corporation_id
,       WHSE_CD
,       status_dst
,       cic_ib_cst
,       item_type_cd
--,       rog_id
,       rog_cd
,       retail_status_cd
,       loc_common_retail_cd
,       upc_id
,       snstv_tier_cd
,       DIV_PROMO_GRP_CD
,       VEND_NBR
,       VEND_SUB_ACCT_NBR
,       loc_retail_sect_id
,       BUYER_NBR
,       unit_prc_tbl_nbr
,       price_area_id
,       cost_area_id
,       buyer_nm
,       vend_nm
,       reg_rtl_prc
,       reg_rtl_prc_fctr
,       offer_id
,       COUPON_AMT
,       promo_method_cd
,       promo_min_purch_qty
,       promo_lim_qty
,       promo_prc
,       promo_prc_fctr
,       first_eff_dt
,       last_eff_dt
,       prtl_first_eff_dt
,       prtl_reg_rtl_prc
,       prtl_reg_rtl_prc_fctr
,       adplan_flag
,       up_mult_factor
,       up_label_unit
,       pending_cost
,       pending_cost_date
-- ARIS: 20190213: removed the following columns.  These will be calculated later at price area level
--,       hist_reg_rtl_prc
--,       hist_reg_rtl_prc_fctr
--,       hist_last_eff_dt
,       alw_typ
,       arrival_from_date
,       arrival_to_date
,       allow_amt
,       amt_in_cost
)
SELECT  
tmp.division_id
,       tmp.store_id
,       tmp.genrt_cic_id
,       tmp.corp_item_cd
,       tmp.cic_dsc
,       tmp.group_id
,       tmp.category_id
,       tmp.vend_conv_fctr
-- ,       tmp.pack_whse_qty
,       tmp.pack_retail_qty
,       tmp.pack_dsc
,       tmp.size_dsc
,       tmp.dst_cntr
,       tmp.cic_vend_cst
,       tmp.corporation_id
,       tmp.WHSE_CD
,       tmp.status_dst
,       tmp.cic_ib_cst
,       tmp.item_type_cd
//,       tmp.rog_id 
,       tmp.rog_cd 
,       tmp.retail_status_cd
,       tmp.loc_common_retail_cd
,       tmp.upc_id
,       tmp.snstv_tier_cd
,       tmp.DIV_PROMO_GRP_CD
,       tmp.VEND_NBR
,       tmp.VEND_SUB_ACCT_NBR
,       tmp.loc_retail_sect_id
,       tmp.BUYER_NBR
,       tmp.unit_prc_tbl_nbr
,       spa.price_area_cd
,       0 ::SMALLINT AS cost_area_id
,       buy.buyer_nm
,       ven.VENDOR_NM
,       reg.ITEM_PRICE_AMT
,       reg.ITEM_PRICE_FCTR
,       cpn.offer_id
,       cpn.coupon_amt
,       cpn.promo_method_cd
,       cpn.promo_min_purch_qty
,       cpn.promo_lim_qty
,       cpn.promo_prc
,       cpn.promo_prc_fctr
,       cpn.first_eff_dt
,       cpn.last_eff_dt
,       prtl.first_eff_dt AS prtl_first_eff_dt
,       prtl.reg_rtl_prc  AS prtl_reg_rtl_prc
,       prtl.reg_rtl_prc_fctr   AS prtl_reg_rtl_prc_fctr
,       CASE WHEN adp.store_id IS NULL THEN ' ' ELSE 'Y' END AS adplan_flag
,       uni.UNIT_PRC_MULT_FCTR  AS up_mult_factor
,       uni.UNIT_PRC_LABEL_DSC  AS up_label_unit
,       wcst.cost_vend AS pending_cost
,       wcst.date_eff AS pending_cost_date
-- ARIS: 20190213: removed the following columns.  These will be calculated later at price area level
--,       hrtl.reg_rtl_prc AS hist_reg_rtl_prc
--,       hrtl.reg_rtl_prc_fctr AS hist_reg_rtl_prc_fctr
--,       hrtl.last_eff_dt AS hist_last_eff_dt
,       alw.alw_typ
,       alw.arrival_from_date
,       alw.arrival_to_date
,       alw.allow_amt
,       alw.amt_in_cost
FROM   EDM_SANDBOX_PRD.MERCHAPPS.T_PE_WHSE_ITEM_ATTR_1  tmp
INNER   JOIN (
        SELECT  rsspa.facility_integration_id
,       rs.facility_nbr
,       rsspa.section_cd
,       rsspa.price_area_cd
,       rs.rog_ID
        FROM    EDM_VIEWS_PRD.dw_views.RETAIL_STORE_SECTION_PRICE_AREA rsspa
        INNER   JOIN EDM_VIEWS_PRD.dw_views.retail_store rs
        ON      rs.facility_integration_id = rsspa.facility_integration_id
        inner join EDM_VIEWS_PRD.dw_views.Retail_Order_Group rog
        on rs.rog_ID=rog.rog_ID
        
        WHERE   rsspa.dw_current_version_ind = TRUE
    AND     rsspa.dw_logical_delete_ind = false 
    and     rs.dw_logical_delete_ind = false 
        AND     rs.dw_current_version_ind = TRUE
        
        and  rsspa.dw_last_effective_dt = '9999-12-31'::DATE
        and rog.dw_current_version_ind = TRUE
        AND     rog.DW_LOGICAL_DELETE_IND = FALSE
        ) spa
ON     
try_to_number(spa.section_cd) = tmp.loc_retail_sect_id
AND   
spa.facility_nbr = tmp.store_id
INNER   JOIN (
SELECT DISTINCT
  buyer_id,
  buyer_nm
FROM
EDM_VIEWS_PRD.DW_VIEWS.BUYER
where     dw_current_version_ind = TRUE
AND     DW_LOGICAL_DELETE_IND = FALSE
) buy
ON      buy.buyer_id = tmp.buyer_nbr
 
INNER   JOIN (
SELECT DISTINCT
VM.VENDOR_ID as vend_nbr,
VO.CORPORATION_ID as corporation_id,
VM.VENDOR_NM as VENDOR_NM 
FROM
EDM_VIEWS_PRD.DW_VIEWS.VENDOR_MASTER VM
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.VW_VENDOR_OUTLET VO
ON VM.VENDOR_ID = VO.VENDOR_ID
AND VO.DW_LOGICAL_DELETE_IND = FALSE
AND VO.DW_CURRENT_VERSION_IND = TRUE
AND VM.DW_LOGICAL_DELETE_IND = FALSE
AND VM.DW_CURRENT_VERSION_IND = TRUE
AND VO.CORPORATION_ID = 1
) ven
ON      ven.vend_nbr = tmp.vend_nbr
AND     ven.corporation_id = tmp.corporation_id
 
LEFT    OUTER JOIN (
  select distinct 
  f.FACILITY_NBR as FACILITY_NBR,
  rs.ITEM_PRICE_AMT as ITEM_PRICE_AMT,
  rs.ITEM_PRICE_FCTR as ITEM_PRICE_FCTR,
  rs.UPC_NBR as UPC_NBR,
  rs.DW_FIRST_EFFECTIVE_DT as DW_FIRST_EFFECTIVE_DT,
  rs.DW_LAST_EFFECTIVE_DT as DW_LAST_EFFECTIVE_DT
  from 
  EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE_UPC_PRICE rs
  join EDM_VIEWS_PRD.DW_VIEWS.facility f on
  rs.FACILITY_INTEGRATION_ID=f.FACILITY_INTEGRATION_ID
  and
  rs.DW_LOGICAL_DELETE_IND = FALSE
AND     rs.DW_CURRENT_VERSION_IND = TRUE
  and
  f.DW_LOGICAL_DELETE_IND = FALSE
AND     f.DW_CURRENT_VERSION_IND = TRUE
)  reg 
ON      reg.FACILITY_NBR = tmp.store_id
AND 
reg.UPC_NBR   = tmp.upc_id
AND     CURRENT_DATE BETWEEN reg.DW_FIRST_EFFECTIVE_DT AND reg.DW_LAST_EFFECTIVE_DT
 
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.T_PE_CPN_ADPLAN_ALL cpn
ON      cpn.store_id = tmp.store_id
AND     cpn.upc_id = tmp.upc_id
 
AND     cpn.offer_type_cd IN ('ECP', 'PCP')
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.T_PE_PENDING_REG_RTL prtl
ON      prtl.store_id = tmp.store_id
AND     prtl.upc_id   = tmp.upc_id
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.T_PE_CPN_ADPLAN_ALL adp
ON      adp.store_id = tmp.store_id
AND     adp.upc_id = tmp.upc_id
AND     adp.offer_type_cd IN ('AD', 'LTS')
LEFT    OUTER JOIN (
SELECT DISTINCT 
 UNIT_PRICE_TABLE_NBR AS UNIT_PRC_ID,
 UNIT_PRICE_MULTIPLICATION_FCTR AS UNIT_PRC_MULT_FCTR,
UNIT_PRICE_LABEL_UNIT_CD AS UNIT_PRC_LABEL_DSC
FROM 
EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP_UPC
where 
dw_current_version_ind = TRUE
and DW_LOGICAL_DELETE_IND = FALSE
) uni
ON      uni.UNIT_PRC_ID = tmp.unit_prc_tbl_nbr
 
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.T_PE_PENDING_WHSE_COST wcst
ON      wcst.dst_cntr = tmp.dst_cntr
AND     wcst.corp_item_cd = tmp.corp_item_cd
-- ARIS: 20190213: removed the following table
--LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl hrtl
--ON      hrtl.store_id = tmp.store_id
--AND     hrtl.upc_id   = tmp.upc_id
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.T_PE_ALW_FINAL alw
ON      alw.corp = tmp.corporation_id
AND     alw.facility = tmp.whse_cd
AND     alw.corp_item_cd = tmp.corp_item_cd
AND     alw.rog = tmp.rog_cd
AND     alw.price_area_id = spa.price_area_cd
AND     alw.vend_num = tmp.vend_nbr
AND     alw.vend_sub_acnt = tmp.vend_sub_acct_nbr
AND     alw.cost_area = 0
--jyoti-20190306-exclusion
WHERE 
NOT(tmp.division_id=27 AND tmp.rog_cd='SSEA' AND spa.price_area_cd=46)
AND NOT(tmp.division_id=27 AND tmp.rog_cd='SACG' AND spa.price_area_cd=9)
-- exclusion for denever
AND NOT(tmp.division_id=5 AND tmp.rog_cd ='SDEN' AND spa.price_area_cd in (3,14))
-- exclusion for shaws
AND NOT(tmp.division_id=33 AND tmp.rog_cd ='SEAS' AND spa.price_area_cd=1)
-- exclusion for Southern
AND NOT(tmp.division_id=20 AND tmp.rog_cd ='ADAL' AND spa.price_area_cd=6)
-- exclusion for Eastern
AND NOT(tmp.division_id=35 AND tmp.rog_cd ='SEAS' AND spa.price_area_cd=1)
-- exclusion for Portland
AND NOT(tmp.division_id=19 AND tmp.rog_cd ='SPRT' AND spa.price_area_cd in (61))
```

## Step 34
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl_pa;
-- DROP_TBL: Add if exists in drop table
```

## Step 35
```sql
CREATE or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl_pa 
     (
      division_id INTEGER,
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      price_area_id INTEGER,
      upc_id DECIMAL(13,0),
      hist_reg_rtl_prc DECIMAL(7,2),
      hist_reg_rtl_prc_fctr DECIMAL(2,0),
      hist_last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}') COMMENT = '{"multiset": true}' ;
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_MULTISET_OPTION - Remove MULTISET create table option
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
-- T_CREATE_TABLE_META - Add table metadata to table comment
```

## Step 36
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl_pa
(       division_id
--,       rog_id
,       rog_cd
,       price_area_id
,       upc_id
,       hist_reg_rtl_prc
,       hist_reg_rtl_prc_fctr
,       hist_last_eff_dt
)
SELECT  division_id
--,       rog_id
,       rog_cd
,       price_area_id
,       upc_id
,       hist_reg_rtl_prc
,       hist_reg_rtl_prc_fctr
,       hist_last_eff_dt
FROM    (
        SELECT  dt.division_id
        --,       dt.rog_id
        ,       dt.rog_cd
        ,       dt.price_area_id
        ,       dt.upc_id
        ,       rtl.reg_rtl_prc AS hist_reg_rtl_prc
        ,       rtl.reg_rtl_prc_fctr AS hist_reg_rtl_prc_fctr
        ,       rtl.last_eff_dt AS hist_last_eff_dt
        ,       COUNT(DISTINCT dt.store_id) AS store_count
        FROM    (
                SELECT  division_id
                --,       rog_id
                ,       rog_cd
                ,       store_id
                ,       price_area_id
                ,       upc_id
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_item_attr_2
                GROUP   BY division_id
                --,       rog_id
                ,       rog_cd
                ,       store_id
                ,       price_area_id
                ,       upc_id
                UNION
                SELECT  division_id
                --,       rog_id
                ,       rog_cd
                ,       store_id
                ,       price_area_id
                ,       upc_id
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_whse_item_attr_2
                GROUP   BY division_id
                --,       rog_id
                ,       rog_cd
                ,       store_id
                ,       price_area_id
                ,       upc_id
                ) dt
        INNER   JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl rtl
        ON      rtl.store_id = dt.store_id
        AND     rtl.upc_id = dt.upc_id
        GROUP   BY 1,2,3,4,5,6,7
        ) dt1
QUALIFY ROW_NUMBER() OVER (PARTITION BY division_id, rog_cd, price_area_id, upc_id ORDER BY hist_last_eff_dt DESC, store_count DESC, hist_reg_rtl_prc ASC) = 1;
```

## Step 37
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_3;
-- DROP_TBL: Add if exists in drop table
```

## Step 38
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_3 
     (
      division_id INTEGER,
      store_id INTEGER,
      corp_item_cd DECIMAL(8,0),
      cic_dsc VARCHAR(40)  COLLATE 'en-ci' ,
      group_id SMALLINT,
      category_id SMALLINT,
      vend_conv_fctr SMALLINT,
      pack_retail_qty DECIMAL(7,2),
      -- pack_whse_qty DECIMAL(7,2),
      pack_dsc CHAR(30)  COLLATE 'en-ci' ,
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      dst_cntr VARCHAR(4)  COLLATE 'en-ci' ,
      cic_vend_cst DECIMAL(9,4),
      corporation_id BYTEINT,
      WHSE_CD VARCHAR(4)  COLLATE 'en-ci' ,
      status_dst CHAR(1)  COLLATE 'en-ci' ,
      cic_ib_cst DECIMAL(9,4),
      item_type_cd DECIMAL(3,0),
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      retail_status_cd CHAR(1)  COLLATE 'en-ci' ,
      loc_common_retail_cd DECIMAL(5,0),
      upc_id DECIMAL(13,0),
      snstv_tier_cd CHAR(1)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      VEND_NBR VARCHAR(6)  COLLATE 'en-ci' ,
      VEND_SUB_ACCT_NBR VARCHAR(3)  COLLATE 'en-ci' ,
      loc_retail_sect_id INTEGER,
      BUYER_NBR CHAR(2)  COLLATE 'en-ci' ,
      unit_prc_tbl_nbr SMALLINT,
      price_area_id INTEGER,
      cost_area_id SMALLINT,
      buyer_nm CHAR(20)  COLLATE 'en-ci' ,
      vend_nm CHAR(40)  COLLATE 'en-ci' ,
      reg_rtl_prc DECIMAL(7,2),
      reg_rtl_prc_fctr DECIMAL(2,0),
      offer_id DECIMAL(13,0),
      COUPON_AMT DECIMAL(8,2),
      promo_method_cd CHAR(2)  COLLATE 'en-ci' ,
      promo_min_purch_qty DECIMAL(2,0),
      promo_lim_qty SMALLINT,
      promo_prc DECIMAL(7,2),
      promo_prc_fctr DECIMAL(2,0),
      first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_reg_rtl_prc DECIMAL(7,2),
      prtl_reg_rtl_prc_fctr DECIMAL(2,0),
      adplan_flag VARCHAR(1)  COLLATE 'en-ci' ,
      up_mult_factor DECIMAL(7,3),
      up_label_unit VARCHAR(5)  COLLATE 'en-ci' ,
      pending_cost DECIMAL(9,4),
      pending_cost_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      hist_reg_rtl_prc DECIMAL(7,2),
      hist_reg_rtl_prc_fctr DECIMAL(2,0),
      hist_last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      alw_typ VARCHAR(1)  COLLATE 'en-ci' ,
      arrival_from_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      arrival_to_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      allow_amt DECIMAL(11,4),
      amt_in_cost DECIMAL(11,4));
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

## Step 39
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_3
(       division_id
,       store_id
,       corp_item_cd
,       cic_dsc
,       group_id
,       category_id
,       vend_conv_fctr
,       pack_retail_qty
-- ,       pack_whse_qty
,       pack_dsc
,       size_dsc
,       dst_cntr
,       cic_vend_cst
,       corporation_id
,       WHSE_CD
,       status_dst
,       cic_ib_cst
,       item_type_cd
--,       rog_id
,       rog_cd
,       retail_status_cd
,       loc_common_retail_cd
,       upc_id
,       snstv_tier_cd
,       DIV_PROMO_GRP_CD
,       VEND_NBR
,       VEND_SUB_ACCT_NBR
,       loc_retail_sect_id
,       BUYER_NBR
,       unit_prc_tbl_nbr
,       price_area_id
,       cost_area_id
,       buyer_nm
,       vend_nm
,       reg_rtl_prc
,       reg_rtl_prc_fctr
,       offer_id
,       COUPON_AMT
,       promo_method_cd
,       promo_min_purch_qty
,       promo_lim_qty
,       promo_prc
,       promo_prc_fctr
,       first_eff_dt
,       last_eff_dt
,       prtl_first_eff_dt
,       prtl_reg_rtl_prc
,       prtl_reg_rtl_prc_fctr
,       adplan_flag
,       up_mult_factor
,       up_label_unit
,       pending_cost
,       pending_cost_date
,       hist_reg_rtl_prc
,       hist_reg_rtl_prc_fctr
,       hist_last_eff_dt
,       alw_typ
,       arrival_from_date
,       arrival_to_date
,       allow_amt
,       amt_in_cost
)
SELECT  tmp.division_id
,       tmp.store_id
,       tmp.corp_item_cd
,       tmp.cic_dsc
,       tmp.group_id
,       tmp.category_id
,       tmp.vend_conv_fctr
,       tmp.pack_retail_qty
-- ,       tmp.pack_whse_qty
,       tmp.pack_dsc
,       tmp.size_dsc
,       tmp.dst_cntr
,       tmp.cic_vend_cst
,       tmp.corporation_id
,       tmp.whse_cd
,       tmp.status_dst
,       tmp.cic_ib_cst
,       tmp.item_type_cd
--,       tmp.rog_id
,       tmp.rog_cd
,       tmp.retail_status_cd
,       tmp.loc_common_retail_cd
,       tmp.upc_id
,       tmp.snstv_tier_cd
,       tmp.div_promo_grp_cd
,       tmp.vend_nbr
,       tmp.vend_sub_acct_nbr
,       tmp.loc_retail_sect_id
,       tmp.buyer_nbr
,       tmp.unit_prc_tbl_nbr
,       tmp.price_area_id
,       tmp.cost_area_id
,       tmp.buyer_nm
,       tmp.vend_nm
,       tmp.reg_rtl_prc
,       tmp.reg_rtl_prc_fctr
,       tmp.offer_id
,       tmp.COUPON_AMT
,       tmp.promo_method_cd
,       tmp.promo_min_purch_qty
,       tmp.promo_lim_qty
,       tmp.promo_prc
,       tmp.promo_prc_fctr
,       tmp.first_eff_dt
,       tmp.last_eff_dt
,       tmp.prtl_first_eff_dt
,       tmp.prtl_reg_rtl_prc
,       tmp.prtl_reg_rtl_prc_fctr
,       tmp.adplan_flag
,       tmp.up_mult_factor
,       tmp.up_label_unit
,       tmp.pending_cost
,       tmp.pending_cost_date
,       reg.hist_reg_rtl_prc
,       reg.hist_reg_rtl_prc_fctr
,       reg.hist_last_eff_dt
,       tmp.alw_typ
,       tmp.arrival_from_date
,       tmp.arrival_to_date
,       tmp.allow_amt
,       tmp.amt_in_cost
FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_item_attr_2 tmp
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl_pa reg
ON      reg.rog_cd = tmp.rog_cd
AND      reg.price_area_id = tmp.price_area_id
AND     reg.upc_id = tmp.upc_id
GROUP   BY tmp.division_id
,       tmp.store_id
,       tmp.corp_item_cd
,       tmp.cic_dsc
,       tmp.group_id
,       tmp.category_id
,       tmp.vend_conv_fctr
,       tmp.pack_retail_qty
-- ,       tmp.pack_whse_qty
,       tmp.pack_dsc
,       tmp.size_dsc
,       tmp.dst_cntr
,       tmp.cic_vend_cst
,       tmp.corporation_id
,       tmp.whse_cd
,       tmp.status_dst
,       tmp.cic_ib_cst
,       tmp.item_type_cd
--,       tmp.rog_id
,       tmp.rog_cd
,       tmp.retail_status_cd
,       tmp.loc_common_retail_cd
,       tmp.upc_id
,       tmp.snstv_tier_cd
,       tmp.div_promo_grp_cd
,       tmp.vend_nbr
,       tmp.vend_sub_acct_nbr
,       tmp.loc_retail_sect_id
,       tmp.buyer_nbr
,       tmp.unit_prc_tbl_nbr
,       tmp.price_area_id
,       tmp.cost_area_id
,       tmp.buyer_nm
,       tmp.vend_nm
,       tmp.reg_rtl_prc
,       tmp.reg_rtl_prc_fctr
,       tmp.offer_id
,       tmp.COUPON_AMT
,       tmp.promo_method_cd
,       tmp.promo_min_purch_qty
,       tmp.promo_lim_qty
,       tmp.promo_prc
,       tmp.promo_prc_fctr
,       tmp.first_eff_dt
,       tmp.last_eff_dt
,       tmp.prtl_first_eff_dt
,       tmp.prtl_reg_rtl_prc
,       tmp.prtl_reg_rtl_prc_fctr
,       tmp.adplan_flag
,       tmp.up_mult_factor
,       tmp.up_label_unit
,       tmp.pending_cost
,       tmp.pending_cost_date
,       reg.hist_reg_rtl_prc
,       reg.hist_reg_rtl_prc_fctr
,       reg.hist_last_eff_dt
,       tmp.alw_typ
,       tmp.arrival_from_date
,       tmp.arrival_to_date
,       tmp.allow_amt
,       tmp.amt_in_cost
 
UNION ALL
 
SELECT  tmp.division_id
,       tmp.store_id
,       tmp.corp_item_cd
,       tmp.cic_dsc
,       tmp.group_id
,       tmp.category_id
,       tmp.vend_conv_fctr
,       tmp.pack_retail_qty
-- ,       tmp.pack_whse_qty
,       tmp.pack_dsc
,       tmp.size_dsc
,       tmp.dst_cntr
,       tmp.cic_vend_cst
,       tmp.corporation_id
,       tmp.whse_cd
,       tmp.status_dst
,       tmp.cic_ib_cst
,       tmp.item_type_cd
--,       tmp.rog_id
,       tmp.rog_cd
,       tmp.retail_status_cd
,       tmp.loc_common_retail_cd
,       tmp.upc_id
,       tmp.snstv_tier_cd
,       tmp.div_promo_grp_cd
,       tmp.vend_nbr
,       tmp.vend_sub_acct_nbr
,       tmp.loc_retail_sect_id
,       tmp.buyer_nbr
,       tmp.unit_prc_tbl_nbr
,       tmp.price_area_id
,       tmp.cost_area_id
,       tmp.buyer_nm
,       tmp.vend_nm
,       tmp.reg_rtl_prc
,       tmp.reg_rtl_prc_fctr
,       tmp.offer_id
,       tmp.COUPON_AMT
,       tmp.promo_method_cd
,       tmp.promo_min_purch_qty
,       tmp.promo_lim_qty
,       tmp.promo_prc
,       tmp.promo_prc_fctr
,       tmp.first_eff_dt
,       tmp.last_eff_dt
,       tmp.prtl_first_eff_dt
,       tmp.prtl_reg_rtl_prc
,       tmp.prtl_reg_rtl_prc_fctr
,       tmp.adplan_flag
,       tmp.up_mult_factor
,       tmp.up_label_unit
,       tmp.pending_cost
,       tmp.pending_cost_date
,       reg.hist_reg_rtl_prc
,       reg.hist_reg_rtl_prc_fctr
,       reg.hist_last_eff_dt
,       tmp.alw_typ
,       tmp.arrival_from_date
,       tmp.arrival_to_date
,       tmp.allow_amt
,       tmp.amt_in_cost
FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_whse_item_attr_2 tmp
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_hist_reg_rtl_pa reg
ON      reg.rog_cd = tmp.rog_cd
AND     reg.price_area_id = tmp.price_area_id
AND     reg.upc_id = tmp.upc_id
GROUP   BY tmp.division_id
,       tmp.store_id
,       tmp.corp_item_cd
,       tmp.cic_dsc
,       tmp.group_id
,       tmp.category_id
,       tmp.vend_conv_fctr
,       tmp.pack_retail_qty
-- ,       tmp.pack_whse_qty
,       tmp.pack_dsc
,       tmp.size_dsc
,       tmp.dst_cntr
,       tmp.cic_vend_cst
,       tmp.corporation_id
,       tmp.whse_cd
,       tmp.status_dst
,       tmp.cic_ib_cst
,       tmp.item_type_cd
--,       tmp.rog_id
,       tmp.rog_cd
,       tmp.retail_status_cd
,       tmp.loc_common_retail_cd
,       tmp.upc_id
,       tmp.snstv_tier_cd
,       tmp.div_promo_grp_cd
,       tmp.vend_nbr
,       tmp.vend_sub_acct_nbr
,       tmp.loc_retail_sect_id
,       tmp.buyer_nbr
,       tmp.unit_prc_tbl_nbr
,       tmp.price_area_id
,       tmp.cost_area_id
,       tmp.buyer_nm
,       tmp.vend_nm
,       tmp.reg_rtl_prc
,       tmp.reg_rtl_prc_fctr
,       tmp.offer_id
,       tmp.COUPON_AMT
,       tmp.promo_method_cd
,       tmp.promo_min_purch_qty
,       tmp.promo_lim_qty
,       tmp.promo_prc
,       tmp.promo_prc_fctr
,       tmp.first_eff_dt
,       tmp.last_eff_dt
,       tmp.prtl_first_eff_dt
,       tmp.prtl_reg_rtl_prc
,       tmp.prtl_reg_rtl_prc_fctr
,       tmp.adplan_flag
,       tmp.up_mult_factor
,       tmp.up_label_unit
,       tmp.pending_cost
,       tmp.pending_cost_date
,       reg.hist_reg_rtl_prc
,       reg.hist_reg_rtl_prc_fctr
,       reg.hist_last_eff_dt
,       tmp.alw_typ
,       tmp.arrival_from_date
,       tmp.arrival_to_date
,       tmp.allow_amt
,       tmp.amt_in_cost;
```

## Step 40:
```sql 
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_4;
-- DROP_TBL: Add if exists in drop table
```

## Step 41:
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_4 
     (
      division_id INTEGER,
      corp_item_cd DECIMAL(8,0),
      cic_dsc VARCHAR(40)  COLLATE 'en-ci' ,
      group_id SMALLINT,
      category_id SMALLINT,
      vend_conv_fctr SMALLINT,
      pack_retail_qty DECIMAL(7,2),
      -- pack_whse_qty DECIMAL(7,2),
      pack_dsc CHAR(30)  COLLATE 'en-ci' ,
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      dst_cntr VARCHAR(4)  COLLATE 'en-ci' ,
      cic_vend_cst DECIMAL(9,4),
      corporation_id BYTEINT,
      WHSE_CD VARCHAR(4)  COLLATE 'en-ci' ,
      status_dst CHAR(1)  COLLATE 'en-ci' ,
      cic_ib_cst DECIMAL(9,4),
      item_type_cd DECIMAL(3,0),
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      retail_status_cd CHAR(1)  COLLATE 'en-ci' ,
      loc_common_retail_cd DECIMAL(5,0),
      upc_id DECIMAL(13,0),
      snstv_tier_cd CHAR(1)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      VEND_NBR VARCHAR(6)  COLLATE 'en-ci' ,
      VEND_SUB_ACCT_NBR VARCHAR(3)  COLLATE 'en-ci' ,
      loc_retail_sect_id INTEGER,
      BUYER_NBR CHAR(2)  COLLATE 'en-ci' ,
      unit_prc_tbl_nbr SMALLINT,
      price_area_id INTEGER,
      cost_area_id SMALLINT,
      buyer_nm CHAR(20)  COLLATE 'en-ci' ,
      vend_nm CHAR(40)  COLLATE 'en-ci' ,
      reg_rtl_prc DECIMAL(7,2),
      reg_rtl_prc_fctr DECIMAL(2,0),
      offer_id DECIMAL(13,0),
      COUPON_AMT DECIMAL(8,2),
      promo_method_cd CHAR(2)  COLLATE 'en-ci' ,
      promo_min_purch_qty DECIMAL(2,0),
      promo_lim_qty SMALLINT,
      promo_prc DECIMAL(7,2),
      promo_prc_fctr DECIMAL(2,0),
      first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_reg_rtl_prc DECIMAL(7,2),
      prtl_reg_rtl_prc_fctr DECIMAL(2,0),
      adplan_flag VARCHAR(1)  COLLATE 'en-ci' ,
      up_mult_factor DECIMAL(7,3),
      up_label_unit VARCHAR(5)  COLLATE 'en-ci' ,
      pending_cost DECIMAL(9,4),
      pending_cost_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      hist_reg_rtl_prc DECIMAL(7,2),
      hist_reg_rtl_prc_fctr DECIMAL(2,0),
      hist_last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      alw_typ VARCHAR(1)  COLLATE 'en-ci' ,
      arrival_from_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      arrival_to_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      allow_amt DECIMAL(11,4),
      amt_in_cost DECIMAL(11,4));
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

## Step 42:
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_4
(       division_id
,       corp_item_cd
,       cic_dsc
,       group_id
,       category_id
,       vend_conv_fctr
,       pack_retail_qty
-- ,       pack_whse_qty
,       pack_dsc
,       size_dsc
,       dst_cntr
,       cic_vend_cst
,       corporation_id
,       whse_cd
,       status_dst
,       cic_ib_cst
,       item_type_cd
--,       rog_id
,       rog_cd
,       retail_status_cd
,       loc_common_retail_cd
,       upc_id
,       snstv_tier_cd
,       div_promo_grp_cd
,       vend_nbr
,       vend_sub_acct_nbr
,       loc_retail_sect_id
,       buyer_nbr
,       unit_prc_tbl_nbr
,       price_area_id
,       cost_area_id
,       buyer_nm
,       vend_nm
,       reg_rtl_prc
,       reg_rtl_prc_fctr
,       offer_id
,       COUPON_AMT
,       promo_method_cd
,       promo_min_purch_qty
,       promo_lim_qty
,       promo_prc
,       promo_prc_fctr
,       first_eff_dt
,       last_eff_dt
,       prtl_first_eff_dt
,       prtl_reg_rtl_prc
,       prtl_reg_rtl_prc_fctr
,       adplan_flag
,       up_mult_factor
,       up_label_unit
,       pending_cost
,       pending_cost_date
,       hist_reg_rtl_prc
,       hist_reg_rtl_prc_fctr
,       hist_last_eff_dt
,       alw_typ
,       arrival_from_date
,       arrival_to_date
,       allow_amt
,       amt_in_cost
)
SELECT  division_id
,       corp_item_cd
,       cic_dsc
,       group_id
,       category_id
,       vend_conv_fctr
,       pack_retail_qty
-- ,       pack_whse_qty
,       pack_dsc
,       size_dsc
,       dst_cntr
,       cic_vend_cst
,       corporation_id
,       whse_cd
,       status_dst
,       cic_ib_cst
,       item_type_cd
--,       rog_id
,       rog_cd
,       retail_status_cd
,       loc_common_retail_cd
,       upc_id
,       snstv_tier_cd
,       div_promo_grp_cd
,       vend_nbr
,       vend_sub_acct_nbr
,       loc_retail_sect_id
,       buyer_nbr
,       unit_prc_tbl_nbr
,       price_area_id
,       cost_area_id
,       buyer_nm
,       vend_nm
,       reg_rtl_prc
,       reg_rtl_prc_fctr
,       offer_id
,       COUPON_AMT
,       promo_method_cd
,       promo_min_purch_qty
,       promo_lim_qty
,       promo_prc
,       promo_prc_fctr
,       first_eff_dt
,       last_eff_dt
,       prtl_first_eff_dt
,       prtl_reg_rtl_prc
,       prtl_reg_rtl_prc_fctr
,       adplan_flag
,       up_mult_factor
,       up_label_unit
,       pending_cost
,       pending_cost_date
,       hist_reg_rtl_prc
,       hist_reg_rtl_prc_fctr
,       hist_last_eff_dt
,       alw_typ
,       arrival_from_date
,       arrival_to_date
,       allow_amt
,       amt_in_cost
FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_3
GROUP   BY division_id
,       corp_item_cd
,       cic_dsc
,       group_id
,       category_id
,       vend_conv_fctr
,       pack_retail_qty
-- ,       pack_whse_qty
,       pack_dsc
,       size_dsc
,       dst_cntr
,       cic_vend_cst
,       corporation_id
,       whse_cd
,       status_dst
,       cic_ib_cst
,       item_type_cd
--,       rog_id
,       rog_cd
,       retail_status_cd
,       loc_common_retail_cd
,       upc_id
,       snstv_tier_cd
,       div_promo_grp_cd
,       vend_nbr
,       vend_sub_acct_nbr
,       loc_retail_sect_id
,       buyer_nbr
,       unit_prc_tbl_nbr
,       price_area_id
,       cost_area_id
,       buyer_nm
,       vend_nm
,       reg_rtl_prc
,       reg_rtl_prc_fctr
,       offer_id
,       COUPON_AMT
,       promo_method_cd
,       promo_min_purch_qty
,       promo_lim_qty
,       promo_prc
,       promo_prc_fctr
,       first_eff_dt
,       last_eff_dt
,       prtl_first_eff_dt
,       prtl_reg_rtl_prc
,       prtl_reg_rtl_prc_fctr
,       adplan_flag
,       up_mult_factor
,       up_label_unit
,       pending_cost
,       pending_cost_date
,       hist_reg_rtl_prc
,       hist_reg_rtl_prc_fctr
,       hist_last_eff_dt
,       alw_typ
,       arrival_from_date
,       arrival_to_date
,       allow_amt
,       amt_in_cost;
```

## Step 43
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_5;
-- DROP_TBL: Add if exists in drop table
```

## Step 44
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_5 
     (
      division_id INTEGER,
      corp_item_cd DECIMAL(8,0),
      cic_dsc VARCHAR(40)  COLLATE 'en-ci' ,
      group_id SMALLINT,
      category_id SMALLINT,
      vend_conv_fctr SMALLINT,
      pack_retail_qty DECIMAL(7,2),
      -- pack_whse_qty DECIMAL(7,2),
      pack_dsc CHAR(30)  COLLATE 'en-ci' ,
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      dst_cntr VARCHAR(4)  COLLATE 'en-ci' ,
      cic_vend_cst DECIMAL(9,4),
      corporation_id BYTEINT,
      WHSE_CD VARCHAR(4)  COLLATE 'en-ci' ,
      status_dst CHAR(1)  COLLATE 'en-ci' ,
      cic_ib_cst DECIMAL(9,4),
      item_type_cd DECIMAL(3,0),
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      retail_status_cd CHAR(1)  COLLATE 'en-ci' ,
      loc_common_retail_cd DECIMAL(5,0),
      upc_id DECIMAL(13,0),
      snstv_tier_cd CHAR(1)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      VEND_NBR VARCHAR(6)  COLLATE 'en-ci' ,
      VEND_SUB_ACCT_NBR VARCHAR(3)  COLLATE 'en-ci' ,
      loc_retail_sect_id INTEGER,
      BUYER_NBR CHAR(2)  COLLATE 'en-ci' ,
      unit_prc_tbl_nbr SMALLINT,
      price_area_id INTEGER,
      cost_area_id SMALLINT,
      buyer_nm CHAR(20)  COLLATE 'en-ci' ,
      vend_nm CHAR(40)  COLLATE 'en-ci' ,
      reg_rtl_prc DECIMAL(7,2),
      reg_rtl_prc_fctr DECIMAL(2,0),
      COUPON_AMT DECIMAL(8,2),
      promo_method_cd CHAR(2)  COLLATE 'en-ci' ,
      promo_min_purch_qty DECIMAL(2,0),
      promo_lim_qty SMALLINT,
      promo_prc DECIMAL(7,2),
      promo_prc_fctr DECIMAL(2,0),
      first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_first_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      prtl_reg_rtl_prc DECIMAL(7,2),
      prtl_reg_rtl_prc_fctr DECIMAL(2,0),
      adplan_flag VARCHAR(1)  COLLATE 'en-ci' ,
      up_mult_factor DECIMAL(7,3),
      up_label_unit VARCHAR(5)  COLLATE 'en-ci' ,
      pending_cost DECIMAL(9,4),
      pending_cost_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      hist_reg_rtl_prc DECIMAL(7,2),
      hist_reg_rtl_prc_fctr DECIMAL(2,0),
      hist_last_eff_dt DATE  comment '{"FORMAT":"YY/MM/DD"}',
      alw_typ VARCHAR(1)  COLLATE 'en-ci' ,
      arrival_from_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      arrival_to_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      allow_amt DECIMAL(11,4),
      amt_in_cost DECIMAL(11,4));
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

## Step 45
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_5
(       division_id
,       corp_item_cd
,       cic_dsc
,       group_id
,       category_id
,       vend_conv_fctr
,       pack_retail_qty
-- ,       pack_whse_qty
,       pack_dsc
,       size_dsc
,       dst_cntr
,       cic_vend_cst
,       corporation_id
,       WHSE_CD
,       status_dst
,       cic_ib_cst
,       item_type_cd
--,       rog_id
,       rog_cd
,       retail_status_cd
,       loc_common_retail_cd
,       upc_id
,       snstv_tier_cd
,       DIV_PROMO_GRP_CD
,       VEND_NBR
,       VEND_SUB_ACCT_NBR
,       loc_retail_sect_id
,       BUYER_NBR
,       unit_prc_tbl_nbr
,       price_area_id
,       cost_area_id
,       buyer_nm
,       vend_nm
,       reg_rtl_prc
,       reg_rtl_prc_fctr
,       COUPON_AMT
,       promo_method_cd
,       promo_min_purch_qty
,       promo_lim_qty
,       promo_prc
,       promo_prc_fctr
,       first_eff_dt
,       last_eff_dt
,       prtl_first_eff_dt
,       prtl_reg_rtl_prc
,       prtl_reg_rtl_prc_fctr
,       adplan_flag
,       up_mult_factor
,       up_label_unit
,       pending_cost
,       pending_cost_date
,       hist_reg_rtl_prc
,       hist_reg_rtl_prc_fctr
,       hist_last_eff_dt
,       alw_typ
,       arrival_from_date
,       arrival_to_date
,       allow_amt
,       amt_in_cost
)
SELECT  dt.division_id
,       dt.corp_item_cd
,       dt.cic_dsc
,       dt.group_id
,       dt.category_id
,       dt.vend_conv_fctr
,       dt.pack_retail_qty
-- ,       dt.pack_whse_qty
,       dt.pack_dsc
,       dt.size_dsc
,       dt.dst_cntr
,       dt.cic_vend_cst
,       dt.corporation_id
,       dt.whse_cd
,       dt.status_dst
,       dt.cic_ib_cst
,       dt.item_type_cd
--,       dt.rog_id
,       dt.rog_cd
,       dt.retail_status_cd
,       dt.loc_common_retail_cd
,       dt.upc_id
,       dt.snstv_tier_cd
,       dt.div_promo_grp_cd
,       dt.vend_nbr
,       dt.vend_sub_acct_nbr
,       dt.loc_retail_sect_id
,       dt.buyer_nbr
,       dt.unit_prc_tbl_nbr
,       dt.price_area_id
,       dt.cost_area_id
,       dt.buyer_nm
,       dt.vend_nm
,       reg.reg_rtl_prc
,       reg.reg_rtl_prc_fctr
--,       promo.COUPON_AMT
,       (promo.promo_prc / COALESCE(NULLIF(promo.promo_prc_fctr,'0'), 1.00    
)) AS coupon_amt    
/* ARIS:  change from promo.COUPON_AMT to this new code */
--MAYANK : Change for Markdown Column
,       promo.promo_method_cd
,       promo.promo_min_purch_qty
,       promo.promo_lim_qty
,       promo.promo_prc
,       promo.promo_prc_fctr
,       promo.first_eff_dt
,       promo.last_eff_dt
,       prtl.prtl_first_eff_dt
,       prtl.prtl_reg_rtl_prc
,       prtl.prtl_reg_rtl_prc_fctr
,       dt.adplan_flag
,       dt.up_mult_factor
,       dt.up_label_unit
,       dt.pending_cost
,       dt.pending_cost_date
,       hreg.hist_reg_rtl_prc
,       hreg.hist_reg_rtl_prc_fctr
,       hreg.hist_last_eff_dt
,       dt.alw_typ
,       dt.arrival_from_date
,       dt.arrival_to_date
,       dt.allow_amt
,       dt.amt_in_cost
FROM    (
        SELECT  division_id
        ,       corp_item_cd
        ,       cic_dsc
        ,       group_id
        ,       category_id
        ,       vend_conv_fctr
        ,       pack_retail_qty
        -- ,       pack_whse_qty
        ,       pack_dsc
        ,       size_dsc
        ,       dst_cntr
        ,       cic_vend_cst
        ,       corporation_id
        ,       whse_cd
        ,       status_dst
        ,       cic_ib_cst
        ,       item_type_cd
        --,       rog_id
        ,       rog_cd
        ,       retail_status_cd
        ,       loc_common_retail_cd
        ,       upc_id
        ,       snstv_tier_cd
        ,       div_promo_grp_cd
        ,       vend_nbr
        ,       vend_sub_acct_nbr
        ,       loc_retail_sect_id
        ,       buyer_nbr
        ,       unit_prc_tbl_nbr
        ,       price_area_id
        ,       cost_area_id
        ,       buyer_nm
        ,       vend_nm
        ,       adplan_flag
        ,       up_mult_factor
        ,       up_label_unit
        ,       pending_cost
        ,       pending_cost_date
        ,       alw_typ
        ,       arrival_from_date
        ,       arrival_to_date
        ,       allow_amt
        ,       amt_in_cost
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_4
        GROUP   BY division_id
        ,       corp_item_cd
        ,       cic_dsc
        ,       group_id
        ,       category_id
        ,       vend_conv_fctr
        ,       pack_retail_qty
        -- ,       pack_whse_qty
        ,       pack_dsc
        ,       size_dsc
        ,       dst_cntr
        ,       cic_vend_cst
        ,       corporation_id
        ,       whse_cd
        ,       status_dst
        ,       cic_ib_cst
        ,       item_type_cd
        --,       rog_id
        ,       rog_cd
        ,       retail_status_cd
        ,       loc_common_retail_cd
        ,       upc_id
        ,       snstv_tier_cd
        ,       div_promo_grp_cd
        ,       vend_nbr
        ,       vend_sub_acct_nbr
        ,       loc_retail_sect_id
        ,       buyer_nbr
        ,       unit_prc_tbl_nbr
        ,       price_area_id
        ,       cost_area_id
        ,       buyer_nm
        ,       vend_nm
        ,       first_eff_dt
        ,       last_eff_dt
        ,       adplan_flag
        ,       up_mult_factor
        ,       up_label_unit
        ,       pending_cost
        ,       pending_cost_date
        ,       alw_typ
        ,       arrival_from_date
        ,       arrival_to_date
        ,       allow_amt
        ,       amt_in_cost
        ) dt
LEFT    OUTER JOIN (
        SELECT  rog_cd
        ,       upc_id
        ,       price_area_id
        ,       reg_rtl_prc
        ,       reg_rtl_prc_fctr
        FROM    (
                SELECT  rog_cd
                ,       upc_id
                ,       price_area_id
                ,       reg_rtl_prc
                ,       reg_rtl_prc_fctr
                ,       COUNT(1) AS cnt
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_3
                WHERE   reg_rtl_prc_fctr > 0
                GROUP   BY rog_cd
                ,       upc_id
                ,       price_area_id
                ,       reg_rtl_prc
                ,       reg_rtl_prc_fctr
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY rog_cd, upc_id, price_area_id ORDER BY cnt DESC) = 1
        ) reg
ON      reg.rog_cd = dt.rog_cd
AND     reg.upc_id = dt.upc_id
AND     reg.price_area_id = dt.price_area_id
LEFT    OUTER JOIN (
        SELECT  rog_cd
        ,       upc_id
        ,       price_area_Id
        ,       offer_id
        ,       COUPON_AMT
        ,       promo_method_cd
        ,       promo_min_purch_qty
        ,       promo_lim_qty
        ,       promo_prc
        ,       promo_prc_fctr
        ,       first_eff_dt
        ,       last_eff_dt
        FROM    (
                SELECT  rog_cd
                ,       upc_id
                ,       price_area_Id
                ,       offer_id
                ,       COUPON_AMT
                ,       promo_method_cd
                ,       promo_min_purch_qty
                ,       promo_lim_qty
                ,       promo_prc
                ,       promo_prc_fctr
                ,       first_eff_dt
                ,       last_eff_dt
                ,       COUNT(1) AS cnt
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_3
                WHERE   offer_id IS NOT NULL
                GROUP   BY rog_cd
                ,       upc_id
                ,       price_area_Id
                ,       offer_id
                ,       COUPON_AMT
                ,       promo_method_cd
                ,       promo_min_purch_qty
                ,       promo_lim_qty
                ,       promo_prc
                ,       promo_prc_fctr
                ,       first_eff_dt
                ,       last_eff_dt
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY rog_cd, upc_id, price_area_id ORDER BY cnt DESC, coupon_amt ASC, promo_prc ASC, offer_id ASC) = 1
        ) promo
ON      promo.rog_cd = dt.rog_cd
AND     promo.upc_id = dt.upc_id
AND     promo.price_area_id = dt.price_area_id
LEFT    OUTER JOIN (
        SELECT  rog_cd
        ,       upc_id
        ,       price_area_id
        ,       prtl_first_eff_dt
        ,       prtl_reg_rtl_prc
        ,       prtl_reg_rtl_prc_fctr
        FROM    (
                SELECT  rog_cd
                ,       upc_id
                ,       price_area_id
                ,       prtl_first_eff_dt
                ,       prtl_reg_rtl_prc
                ,       prtl_reg_rtl_prc_fctr
                ,       COUNT(1) AS cnt
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_3
                WHERE   prtl_reg_rtl_prc_fctr > 0
                GROUP   BY rog_cd
                ,       upc_id
                ,       price_area_id
                ,       prtl_first_eff_dt
                ,       prtl_reg_rtl_prc
                ,       prtl_reg_rtl_prc_fctr
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY rog_cd, upc_id, price_area_id ORDER BY cnt DESC, prtl_reg_rtl_prc ASC, prtl_first_eff_dt ASC) = 1
        ) prtl
ON      prtl.rog_cd = dt.rog_cd
AND     prtl.upc_id = dt.upc_Id
AND     prtl.price_area_id = dt.price_area_id
LEFT    OUTER JOIN (
        SELECT  rog_cd
        ,       upc_id
        ,       price_area_id
        ,       hist_reg_rtl_prc
        ,       hist_reg_rtl_prc_fctr
        ,       hist_last_eff_dt
        FROM    (
                SELECT  rog_cd
                ,       upc_id
                ,       price_area_id
                ,       hist_reg_rtl_prc
                ,       hist_reg_rtl_prc_fctr
                ,       hist_last_eff_dt
                ,       COUNT(1) AS cnt
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_3
                WHERE   hist_reg_rtl_prc_fctr > 0
                GROUP   BY rog_cd
                ,       upc_id
                ,       price_area_id
                ,       hist_reg_rtl_prc
                ,       hist_reg_rtl_prc_fctr
                ,       hist_last_eff_dt
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY rog_cd, upc_id, price_area_id ORDER BY cnt DESC, hist_last_eff_dt DESC, hist_reg_rtl_prc ASC) = 1
        ) hreg
ON      hreg.rog_cd = dt.rog_cd
AND     hreg.upc_id = dt.upc_Id
AND     hreg.price_area_id = dt.price_area_id;
-- NULLIFZERO: Translated NullifZero to nullif function
```

## Step 46:
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6;
-- DROP_TBL: Add if exists in drop table
```

## Step 47:
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6 
     (
      division_id INTEGER,
      promo_no_allowance CHAR(1)  COLLATE 'en-ci' ,
      allow_no_promo CHAR(1)  COLLATE 'en-ci' ,
      missing_allowance CHAR(1)  COLLATE 'en-ci' ,
      cost_change CHAR(1)  COLLATE 'en-ci' ,
      ad_plan CHAR(1)  COLLATE 'en-ci' ,
      less_10_Promo CHAR(1)  COLLATE 'en-ci' ,
      less_10_allowance CHAR(1)  COLLATE 'en-ci' ,
      greater_100_pass_through CHAR(1)  COLLATE 'en-ci' ,
      t_09_Retail CHAR(1)  COLLATE 'en-ci' ,
      lead_item CHAR(1)  COLLATE 'en-ci' ,
      Dominant_Price_Area CHAR(1)  COLLATE 'en-ci' ,
      OOB CHAR(1)  COLLATE 'en-ci' ,
      sskvi CHAR(1)  COLLATE 'en-ci' ,
      group_cd SMALLINT,
      group_nm CHAR(46)  COLLATE 'en-ci' ,
      SMIC CHAR(4)  COLLATE 'en-ci' ,
      SMIC_name CHAR(46)  COLLATE 'en-ci' ,
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      price_area_id INTEGER,
      PA_name CHAR(1)  COLLATE 'en-ci' ,
      pricing_role CHAR(1)  COLLATE 'en-ci' ,
      OOB_gap_id CHAR(20)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      loc_common_retail_cd DECIMAL(5,0),
      vendor_name VARCHAR(40)  COLLATE 'en-ci' ,
      VEND_NBR VARCHAR(6)  COLLATE 'en-ci' ,
      VEND_SUB_ACCT_NBR VARCHAR(3)  COLLATE 'en-ci' ,
      cost_area_id SMALLINT,
      Manuf CHAR(5)  COLLATE 'en-ci' ,
      upc_id DECIMAL(13,0),
      corp_item_cd DECIMAL(8,0),
      item_description VARCHAR(40)  COLLATE 'en-ci' ,
      DST VARCHAR(4)  COLLATE 'en-ci' ,
      FACILITY VARCHAR(4)  COLLATE 'en-ci' ,
      dst_stat CHAR(1)  COLLATE 'en-ci' ,
      rtl_stat CHAR(1)  COLLATE 'en-ci' ,
      buyer_nm CHAR(20)  COLLATE 'en-ci' ,
      vend_conv_fctr SMALLINT,
      t_pack_retail_qty DECIMAL(10,0),
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      Row_Offset CHAR(1)  COLLATE 'en-ci' ,
      UPC_13_Wk_Avg_Sales CHAR(1)  COLLATE 'en-ci' ,
      UPC_13_Wk_Avg_Qty CHAR(1)  COLLATE 'en-ci' ,
      UPC_13_Wk_Avg_RTL CHAR(1)  COLLATE 'en-ci' ,
      t_Rank CHAR(1)  COLLATE 'en-ci' ,
      pct_ACV_Stores CHAR(1)  COLLATE 'en-ci' ,
      CPC_13_Wk_Avg_Sales CHAR(1)  COLLATE 'en-ci' ,
      CPC_13_Wk_Avg_Qty CHAR(1)  COLLATE 'en-ci' ,
      CPC_13_Wk_Avg_RTL CHAR(1)  COLLATE 'en-ci' ,
      PND_Cost_Change_VND DECIMAL(15,4),
      PND_VEN_Date_Effective DATE  comment '{"FORMAT":"YY/MM/DD"}',
      New_Recc_Reg_Retail CHAR(1)  COLLATE 'en-ci' ,
      Vendor_Unit_Cost DECIMAL(10,3),
      Unit_Item_Billing_Cost DECIMAL(10,3),
      Prev_Retail_Price_Fctr DECIMAL(2,0),
      Previous_Retail_Price DECIMAL(7,2),
      Prev_Retail_Effective_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Pending_EDLP_Mult DECIMAL(2,0),
      Pending_EDLP_Retail DECIMAL(7,2),
      Pending_EDLP_Chg_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Pending_GP_pctg CHAR(1)  COLLATE 'en-ci' ,
      Reg_Retail_Price_Fctr DECIMAL(2,0),
      Reg_Retail DECIMAL(10,2),
      t_price_Per DECIMAL(10,3),
      t_Unit VARCHAR(5)  COLLATE 'en-ci' ,
      Reg_GP_pctg CHAR(1)  COLLATE 'en-ci' ,
      case_allow_count INTEGER,
      case_allow_amt DECIMAL(15,4),
      Case_Allow_amt_per_Unit DECIMAL(15,4),
      Case_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Case_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      S2S_Allow_count INTEGER,
      S2S_Allow_amt DECIMAL(15,4),
      S2S_Allow_amt_per_Unit DECIMAL(15,4),
      S2S_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      S2S_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Scan_Allow_count INTEGER,
      Scan_Allow_amt DECIMAL(15,4),
      Scan_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Scan_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Redem_Scan_Allow_count INTEGER,
      Redem_Allow_amt DECIMAL(15,4),
      Redem_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Redem_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Total_Allow_Unit CHAR(1)  COLLATE 'en-ci' ,
      Allowance_pctg CHAR(1)  COLLATE 'en-ci' ,
      Net_Cost_with_Allow CHAR(1)  COLLATE 'en-ci' ,
      Promo_Multiple DECIMAL(3,0),
      Promo_Price DECIMAL(10,4),
      Coupon_Method CHAR(2)  COLLATE 'en-ci' ,
      Min_Purch DECIMAL(2,0),
      Limit_Per_Txn SMALLINT,
      Promo_GP_pctg CHAR(1)  COLLATE 'en-ci' ,
      Net_Promo_Price DECIMAL(10,2),
      Price_Per DECIMAL(10,3),
      t2_Unit VARCHAR(5)  COLLATE 'en-ci' ,
      Markdown_pctg CHAR(1)  COLLATE 'en-ci' ,
      Mark_down DECIMAL(10,2),
      Promo_Start DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Promo_End DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Pass_Through CHAR(1)  COLLATE 'en-ci' ,
      NEW_Multiple CHAR(1)  COLLATE 'en-ci' ,
      NEW_Retail CHAR(1)  COLLATE 'en-ci' ,
      NEW_EDLP_GP_pctg CHAR(1)  COLLATE 'en-ci' ,
      NEW_Promo_Multiple CHAR(1)  COLLATE 'en-ci' ,
      NEW_Promo_Retail CHAR(1)  COLLATE 'en-ci' ,
      NEW_Promo_GPpctg CHAR(1)  COLLATE 'en-ci' ,
      NEW_Passthrough CHAR(1)  COLLATE 'en-ci' ,
      compet_code VARCHAR(6)  COLLATE 'en-ci' ,
      price_chk_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      comp_reg_mult DECIMAL(2,0),
      com_reg_price DECIMAL(7,2),
      REG_CPI CHAR(1)  COLLATE 'en-ci' ,
      COMP_AD_MULT DECIMAL(2,0),
      COMP_AD_PRICE DECIMAL(7,2),
      Comments CHAR(1)  COLLATE 'en-ci' ,
      Modified_flag CHAR(1)  COLLATE 'en-ci' ,
      ROG_and_CIG VARCHAR(15)  COLLATE 'en-ci' ,
      Allowance_Counts CHAR(50)  COLLATE 'en-ci' ,
      Report_Date DATE  comment '{"FORMAT":"YY/MM/DD"}');
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

## Step 48:
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6
(       division_id
,       promo_no_allowance
,       allow_no_promo
,       missing_allowance
,       cost_change
,       ad_plan
,       less_10_Promo
,       less_10_allowance
,       greater_100_pass_through
,       t_09_Retail
,       lead_item
,       Dominant_Price_Area
,       OOB
,       sskvi
,       group_cd
,       group_nm
,       SMIC
,       SMIC_name
,       rog_cd
,       price_area_id
,       PA_name
,       pricing_role
,       OOB_gap_id
,       DIV_PROMO_GRP_CD
,       loc_common_retail_cd
,       vendor_name
,       VEND_NBR
,       VEND_SUB_ACCT_NBR
,       cost_area_id
,       Manuf
,       upc_id
,       corp_item_cd
,       item_description
,       DST
,       FACILITY
,       dst_stat
,       rtl_stat
,       buyer_nm
,       vend_conv_fctr
,       t_pack_retail_qty
,       size_dsc
,       Row_Offset
,       UPC_13_Wk_Avg_Sales
,       UPC_13_Wk_Avg_Qty
,       UPC_13_Wk_Avg_RTL
,       t_Rank
,       pct_ACV_Stores
,       CPC_13_Wk_Avg_Sales
,       CPC_13_Wk_Avg_Qty
,       CPC_13_Wk_Avg_RTL
,       PND_Cost_Change_VND
,       PND_VEN_Date_Effective
,       New_Recc_Reg_Retail
,       Vendor_Unit_Cost
,       Unit_Item_Billing_Cost
,       Prev_Retail_Price_Fctr
,       Previous_Retail_Price
,       Prev_Retail_Effective_Date
,       Pending_EDLP_Mult
,       Pending_EDLP_Retail
,       Pending_EDLP_Chg_Date
,       Pending_GP_pctg
,       Reg_Retail_Price_Fctr
,       Reg_Retail
,       t_price_Per
,       t_Unit
,       Reg_GP_pctg
,       case_allow_count
,       case_allow_amt
,       Case_Allow_amt_per_Unit
,       Case_Start_Date
,       Case_End_Date
,       S2S_Allow_count
,       S2S_Allow_amt
,       S2S_Allow_amt_per_Unit
,       S2S_Start_Date
,       S2S_End_Date
,       Scan_Allow_count
,       Scan_Allow_amt
,       Scan_Start_Date
,       Scan_End_Date
,       Redem_Scan_Allow_count
,       Redem_Allow_amt
,       Redem_Start_Date
,       Redem_End_Date
,       Total_Allow_Unit
,       Allowance_pctg
,       Net_Cost_with_Allow
,       Promo_Multiple
,       Promo_Price
,       Coupon_Method
,       Min_Purch
,       Limit_Per_Txn
,       Promo_GP_pctg
,       Net_Promo_Price
,       Price_Per
,       t2_Unit
,       Markdown_pctg
,       Mark_down
,       Promo_Start
,       Promo_End
,       Pass_Through
,       NEW_Multiple
,       NEW_Retail
,       NEW_EDLP_GP_pctg
,       NEW_Promo_Multiple
,       NEW_Promo_Retail
,       NEW_Promo_GPpctg
,       NEW_Passthrough
,       compet_code
,       price_chk_date
,       comp_reg_mult
,       com_reg_price
,       REG_CPI
,       COMP_AD_MULT
,       COMP_AD_PRICE
,       Comments
,       Modified_flag
,       ROG_and_CIG
,       Allowance_Counts
,       Report_Date
)
SELECT  t.division_id
,       TRIM(' ') ::CHAR(1) AS promo_no_allowance
,       TRIM(' ') ::CHAR(1) AS allow_no_promo
,       TRIM(' ') ::CHAR(1) AS missing_allowance
,       TRIM(' ') ::CHAR(1) AS cost_change
,       adplan_flag ::CHAR(1) AS ad_plan
,       TRIM(' ') ::CHAR(1) AS less_10_Promo
,       TRIM(' ') ::CHAR(1) AS less_10_allowance
,       TRIM(' ') ::CHAR(1) AS greater_100_pass_through
,       TRIM(' ') ::CHAR(1) AS t_09_Retail
,       TRIM(' ') ::CHAR(1) AS lead_item
,       CASE 
            WHEN t.rog_cd = 'SEAS' AND price_area_id = 36 THEN 'Y'     -- SEAS
            WHEN t.rog_cd = 'SEAG' AND price_area_id = 2  THEN 'Y'     -- SEAG
            WHEN t.rog_cd = 'ACME' AND price_area_id = 87    THEN 'Y' -- ACME
            WHEN t.rog_cd = 'SDEN' AND price_area_id = 71    THEN 'Y' -- SDEN
            WHEN t.rog_cd = 'ADEN' AND price_area_id = 1 THEN 'Y'
            WHEN t.rog_cd = 'AIMT' AND price_area_id = 6 THEN 'Y'
            WHEN t.rog_cd = 'AJWL' AND price_area_id = 1 THEN 'Y'
            WHEN t.rog_cd = 'SHAW' AND price_area_id = 47 THEN 'Y'
            WHEN t.rog_cd = 'SNCA' AND price_area_id = 11 THEN 'Y'
            WHEN t.rog_cd = 'SPRT' AND price_area_id = 22 THEN 'Y'
            WHEN t.rog_cd = 'APOR' AND price_area_id = 4 THEN 'Y'
            WHEN t.rog_cd = 'SSEA' AND price_area_id = 47 THEN 'Y'
            WHEN t.rog_cd = 'SSPK' AND price_area_id = 60 THEN 'Y'
            WHEN t.rog_cd = 'SACG' AND price_area_id = 8 THEN 'Y'
            WHEN t.rog_cd = 'ASHA' AND price_area_id = 4 THEN 'Y'
            WHEN t.rog_cd = 'AVMT' AND price_area_id = 32 THEN 'Y'
            WHEN t.rog_cd = 'VSOC' AND price_area_id = 1 THEN 'Y'
            WHEN t.rog_cd = 'ASOC' AND price_area_id = 1 THEN 'Y'
            WHEN t.rog_cd = 'PSOC' AND price_area_id = 89 THEN 'Y'
            WHEN t.rog_cd = 'RHOU' AND price_area_id = 33 THEN 'Y'
            WHEN t.rog_cd = 'RDAL' AND price_area_id = 80 THEN 'Y'
            WHEN t.rog_cd = 'ADAL' AND price_area_id = 4 THEN 'Y'
            WHEN t.rog_cd = 'AHOU' AND price_area_id = 1 THEN 'Y'
            WHEN t.rog_cd = 'ALAS' AND price_area_id = 71 THEN 'Y'
            WHEN t.rog_cd = 'APHO' AND price_area_id = 41 THEN 'Y'
            WHEN t.rog_cd = 'SPHO' AND price_area_id = 1 THEN 'Y'
            WHEN t.rog_cd = 'SPHX' AND price_area_id = 1 THEN 'Y'
            WHEN t.rog_cd = 'VLAS' AND price_area_id = 62 THEN 'Y'
            WHEN t.rog_cd = 'SWMA' AND price_area_id = 53 THEN 'Y' -- add this on 9.28
            WHEN t.rog_cd = 'AKBA' AND price_area_id = 1 THEN 'Y'  -- add this on 9.28
            WHEN t.rog_cd = 'SHGN' AND price_area_id = 53 THEN 'Y'
            ELSE ' '
        END ::CHAR(1) AS Dominant_Price_Area                      -- MAYANK : Changed rog_cd to t.rog_cd for Lead Item issue
,       TRIM(' ') ::CHAR(1) AS OOB
,       TRIM(' ') ::CHAR(1) AS sskvi
,       translate(to_char(t.group_id , '00'), ' .+-', '')  group_cd
,       grp.SMIC_GROUP_DSC AS GROUP_NM
,       translate(to_char(t.category_id , '0000'), ' .+-', '') ::CHAR(4) AS SMIC
,       cat.smic_category_dsc AS SMIC_name
,       t.rog_cd                                                  -- MAYANK : Changed rog_cd to t.rog_cd for Lead Item issue.
,       price_area_id
,       TRIM(' ') ::CHAR(1) AS PA_name
,       snstv_tier_cd AS pricing_role
,       ((translate(to_char(t.group_id , '00'), ' .+-', '') ::CHAR(2)) || '-' || TRIM(translate(to_char(item_type_cd , '000'), ' .+-', '') ::CHAR(3))) ::CHAR(20) AS OOB_gap_id  -- change "|" to "-"
,       div_promo_grp_cd  -- CIG
,       loc_common_retail_cd  -- COMMON_CD
,       COALESCE(vend_nm, '') AS vendor_name
,       vend_nbr
,       vend_sub_acct_nbr
,       cost_area_id
,       SUBSTR(translate(to_char(t.upc_id , '000000000000'), ' .+-', '') ::CHAR(12), 3, 5) ::CHAR(5) AS Manuf
,       t.upc_id
,       corp_item_cd
,       cic_dsc AS item_description
,       dst_cntr AS DST
,       whse_cd AS FACILITY
,       status_dst AS dst_stat
,       retail_status_cd AS rtl_stat
,       buyer_nm
,       vend_conv_fctr
,       pack_retail_qty ::DECIMAL(10,0) AS t_pack_retail_qty
,       size_dsc
,       TRIM(' ') ::CHAR(1) AS Row_Offset
,       TRIM(' ') ::CHAR(1) AS UPC_13_Wk_Avg_Sales
,       TRIM(' ') ::CHAR(1) AS UPC_13_Wk_Avg_Qty
,       TRIM(' ') ::CHAR(1) AS UPC_13_Wk_Avg_RTL
,       TRIM(' ') ::CHAR(1) AS t_Rank
,       TRIM(' ') ::CHAR(1) AS pct_ACV_Stores
,       TRIM(' ') ::CHAR(1) AS CPC_13_Wk_Avg_Sales
,       TRIM(' ') ::CHAR(1) AS CPC_13_Wk_Avg_Qty
,       TRIM(' ') ::CHAR(1) AS CPC_13_Wk_Avg_RTL
,       pending_cost / NULLIF(vend_conv_fctr,'0') / NULLIF(pack_retail_qty,'0') AS PND_Cost_Change_VND
,       pending_cost_date AS PND_VEN_Date_Effective
,       TRIM(' ') ::CHAR(1) AS New_Recc_Reg_Retail
,       (cic_vend_cst / NULLIF(vend_conv_fctr,'0') / NULLIF(pack_retail_qty,'0')) ::DECIMAL(10,3) AS Vendor_Unit_Cost
,       (cic_ib_cst / NULLIF(pack_retail_qty,'0')) ::DECIMAL(10,3) AS Unit_Item_Billing_Cost
,       hist_reg_rtl_prc_fctr AS Prev_Retail_Price_Fctr
,       hist_reg_rtl_prc AS Previous_Retail_Price
,       hist_last_eff_dt AS Prev_Retail_Effective_Date
,       prtl_reg_rtl_prc_fctr AS Pending_EDLP_Mult
,       prtl_reg_rtl_prc AS Pending_EDLP_Retail
,       prtl_first_eff_dt AS Pending_EDLP_Chg_Date
,       TRIM(' ') ::CHAR(1) AS Pending_GP_pctg
,       reg_rtl_prc_fctr AS Reg_Retail_Price_Fctr
,       reg_rtl_prc ::DECIMAL(10,2) AS Reg_Retail
-- MAYANK : added itm.up_measure
--Akash : converted '/' to '*' between up_measure and up_mult_factor
,       (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0') / NULLIF(itm.up_measure,'0') * NULLIF(up_mult_factor,'0')) ::DECIMAL(10,3) AS t_price_Per -- missing SSITMPOS.up_measure
--, Decimal(Round(promo_prc / promo_prc_fctr /
--  ITM.UP_MEASURE * up_mult_factor, 2), 10, 3) AS "Price_Per..."
,       CASE
            WHEN reg_rtl_prc IS NOT NULL THEN up_label_unit
            ELSE ''
        END AS t_Unit
,       TRIM(' ') ::CHAR(1) AS Reg_GP_pctg
,       COUNT(CASE WHEN ALW_TYP = 'C' THEN ALLOW_AMT END) AS case_allow_count
,       Sum (CASE  WHEN ALW_TYP = 'C' THEN ALLOW_AMT ELSE 0 END) AS case_allow_amt
,       Sum (CASE  WHEN ALW_TYP = 'C' THEN ALLOW_AMT / NULLIF(pack_retail_qty,'0') /
            NULLIF(VEND_CONV_FCTR,'0') ELSE 0 END) AS Case_Allow_amt_per_Unit
,       Max (CASE  WHEN ALW_TYP = 'C' THEN ARRIVAL_FROM_DATE
            ELSE NULL END) AS Case_Start_Date
,       Min (CASE  WHEN ALW_TYP = 'C' THEN ARRIVAL_TO_DATE
            ELSE NULL END) AS Case_End_Date
,       COUNT(CASE WHEN ALW_TYP = 'S' THEN ALLOW_AMT END) AS S2S_Allow_count
,       Sum (CASE  WHEN ALW_TYP = 'S' THEN ALLOW_AMT ELSE 0 END) AS S2S_Allow_amt
,       Sum (CASE  WHEN ALW_TYP = 'S' THEN ALLOW_AMT / NULLIF(pack_retail_qty,'0')
            ELSE 0 END) AS S2S_Allow_amt_per_Unit
,       Max (CASE  WHEN ALW_TYP = 'S' THEN ARRIVAL_FROM_DATE
            ELSE NULL END) AS S2S_Start_Date
,       Min (CASE  WHEN ALW_TYP = 'S' THEN ARRIVAL_TO_DATE
            ELSE NULL END) AS S2S_End_Date
,       COUNT(CASE WHEN ALW_TYP = 'T' THEN ALLOW_AMT END) AS Scan_Allow_count
,       Sum (CASE  WHEN ALW_TYP = 'T' THEN ALLOW_AMT ELSE 0 END) AS Scan_Allow_amt
,       Max (CASE  WHEN ALW_TYP = 'T' THEN ARRIVAL_FROM_DATE
            ELSE NULL END) AS Scan_Start_Date
,       Min (CASE  WHEN ALW_TYP = 'T' THEN ARRIVAL_TO_DATE
            ELSE NULL END) AS Scan_End_Date
,       COUNT(CASE WHEN ALW_TYP = 'R' THEN ALLOW_AMT END) AS Redem_Scan_Allow_count
,       Sum (CASE  WHEN ALW_TYP = 'R' THEN ALLOW_AMT ELSE 0 END) AS Redem_Allow_amt
,       Max (CASE  WHEN ALW_TYP = 'R' THEN ARRIVAL_FROM_DATE
            ELSE NULL END) AS Redem_Start_Date
,       Min (CASE  WHEN ALW_TYP = 'R' THEN ARRIVAL_TO_DATE
            ELSE NULL END) AS Redem_End_Date
,       TRIM(' ') ::CHAR(1) AS Total_Allow_Unit
,       TRIM(' ') ::CHAR(1) AS Allowance_pctg
,       TRIM(' ') ::CHAR(1) AS Net_Cost_with_Allow
,       CASE WHEN promo_prc_fctr = 0 THEN 1 ELSE promo_prc_fctr END AS Promo_Multiple
,       CASE WHEN promo_prc_fctr = 0 AND coupon_amt > 0 THEN coupon_amt
            ELSE promo_prc + 0.0001 END AS Promo_Price
,       promo_method_cd AS Coupon_Method
,       promo_min_purch_qty AS Min_Purch
,       promo_lim_qty AS Limit_Per_Txn
,       TRIM(' ') ::CHAR(1) AS Promo_GP_pctg
,       CASE
            WHEN promo_method_cd = 'NE' AND coupon_amt > 0 THEN coupon_amt
            WHEN promo_method_cd = 'NW' AND coupon_amt > 0 THEN coupon_amt
            WHEN promo_method_cd = 'NE' AND promo_min_purch_qty > 1 AND coupon_amt = 0 THEN
                 (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                 ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) / NULLIF(promo_min_purch_qty,'0') )
            WHEN promo_method_cd = 'NW' AND promo_min_purch_qty > 1 AND coupon_amt = 0 THEN
                 (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                 ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) / NULLIF(promo_min_purch_qty,'0'))
            WHEN promo_method_cd = 'CE' AND promo_min_purch_qty = 1 THEN
                 (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) - coupon_amt
            WHEN promo_method_cd = 'CW' AND promo_min_purch_qty = 1 THEN
                 (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) - coupon_amt
            WHEN promo_method_cd = 'CE' AND promo_min_purch_qty > 1 THEN
                 (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                 ( coupon_amt / NULLIF(promo_min_purch_qty,'0'))
            WHEN promo_method_cd = 'CW' AND promo_min_purch_qty > 1 THEN
                 (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                 ( coupon_amt / NULLIF(promo_min_purch_qty,'0'))
            WHEN promo_method_cd = 'PE' THEN (reg_rtl_prc/NULLIF(reg_rtl_prc_fctr,'0'))-
                 ( ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0'))*coupon_amt)/100)
            WHEN promo_method_cd = 'PW' THEN (reg_rtl_prc/NULLIF(reg_rtl_prc_fctr,'0'))-
                 ( ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0'))*coupon_amt )/100)
            ELSE 0
        END ::DECIMAL(10,2) AS Net_Promo_Price -- (DECIMAL(10,2)) AS Net_Promo_Price
,       (CASE
                WHEN promo_method_cd ='NE' AND coupon_amt > 0 THEN coupon_amt
                WHEN promo_method_cd ='NW' AND coupon_amt > 0 THEN coupon_amt
                WHEN promo_method_cd ='NE' AND promo_min_purch_qty>1 AND coupon_amt=0 THEN
                   (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                   ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) /
                   NULLIF(promo_min_purch_qty,'0') )
                WHEN promo_method_cd ='NW' AND promo_min_purch_qty>1 AND coupon_amt=0 THEN
                   (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                 ((reg_rtl_prc/NULLIF(reg_rtl_prc_fctr,'0'))/ NULLIF(promo_min_purch_qty,'0'))
                WHEN promo_method_cd ='CE' AND promo_min_purch_qty = 1 THEN
                   (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0'))-coupon_amt
                WHEN promo_method_cd ='CW' AND promo_min_purch_qty = 1 THEN
                   (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0'))-coupon_amt
                WHEN promo_method_cd ='CE' AND promo_min_purch_qty > 1 THEN
                   (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                   ( coupon_amt / NULLIF(promo_min_purch_qty,'0'))
                WHEN promo_method_cd ='CW' AND promo_min_purch_qty > 1 THEN
                   (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                   ( coupon_amt / NULLIF(promo_min_purch_qty,'0'))
                WHEN promo_method_cd ='PE' THEN
                   (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                   ( ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) *
                   coupon_amt ) / 100 )
                WHEN promo_method_cd ='PW' THEN
                   (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                   ( ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) *
                   coupon_amt ) / 100 )
                ELSE 0
        END --/ (CASE WHEN promo_prc_fctr = 0 THEN 1 ELSE promo_prc_fctr END) --AKASH : commented this line as suggested by Steve
        / NULLIF(itm.up_measure,'0') * up_mult_factor ) ::DECIMAL(10,3) AS Price_Per -- (DECIMAL(10,3)) AS Price_Per
--                        / ITM.UP_MEASURE * up_mult_factor AS "Price_Per...." -- MAYANK : added itm.up_measure
,       CASE
            WHEN (
                CASE
                    WHEN promo_method_cd = 'NE' AND coupon_amt > 0 THEN coupon_amt
                    WHEN promo_method_cd = 'NW' AND coupon_amt > 0 THEN coupon_amt
                    WHEN promo_method_cd = 'NE' AND promo_min_purch_qty > 1 AND coupon_amt = 0 THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                        ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) / NULLIF(promo_min_purch_qty,'0') )
                    WHEN promo_method_cd = 'NW' AND promo_min_purch_qty > 1 AND coupon_amt = 0 THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                        ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) / NULLIF(promo_min_purch_qty,'0') )
                    WHEN promo_method_cd = 'CE' AND promo_min_purch_qty = 1 THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) - coupon_amt
                    WHEN promo_method_cd = 'CW' AND promo_min_purch_qty = 1 THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) - coupon_amt
                    WHEN promo_method_cd = 'CE' AND promo_min_purch_qty > 1 THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                        ( coupon_amt / NULLIF(promo_min_purch_qty,'0') )
                    WHEN promo_method_cd = 'CW' AND promo_min_purch_qty > 1 THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                        ( coupon_amt / NULLIF(promo_min_purch_qty,'0') )
                    WHEN promo_method_cd = 'PE' THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                        (((reg_rtl_prc/NULLIF(reg_rtl_prc_fctr,'0'))*coupon_amt )/ 100)
                    WHEN promo_method_cd = 'PW' THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                        (((reg_rtl_prc/NULLIF(reg_rtl_prc_fctr,'0'))*coupon_amt)/100)
                    ELSE 0
                END) <> 0 THEN up_label_unit
            ELSE NULL --TRIM(' ')
        END AS t2_Unit
,       TRIM(' ') ::CHAR(1) AS Markdown_pctg
,       CASE
            WHEN NOT promo_method_cd IS NULL THEN (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                (
                CASE
                    WHEN promo_method_cd = 'NE' AND coupon_amt > 0 THEN coupon_amt
                    WHEN promo_method_cd = 'NW' AND coupon_amt > 0 THEN coupon_amt
                    WHEN promo_method_cd = 'NE' AND promo_min_purch_qty > 1 AND coupon_amt = 0 THEN
                        ( reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                        ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) /NULLIF(promo_min_purch_qty,'0') )
                    WHEN promo_method_cd = 'NW' AND promo_min_purch_qty > 1 AND coupon_amt = 0 THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) - ( (reg_rtl_prc /
                        NULLIF(reg_rtl_prc_fctr,'0')) / NULLIF(promo_min_purch_qty,'0'))
                    WHEN promo_method_cd = 'CE' AND promo_min_purch_qty = 1 THEN
                        ( reg_rtl_prc /NULLIF(reg_rtl_prc_fctr,'0')) - coupon_amt
                    WHEN promo_method_cd = 'CW' AND promo_min_purch_qty = 1 THEN
                        ( reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) - coupon_amt
                    WHEN promo_method_cd = 'CE' AND promo_min_purch_qty > 1 THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) - ( coupon_amt /NULLIF(promo_min_purch_qty,'0'))
                    WHEN promo_method_cd = 'CW' AND promo_min_purch_qty > 1 THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) - ( coupon_amt / NULLIF(promo_min_purch_qty,'0'))
                    WHEN promo_method_cd = 'PE' THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                        ( ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) * coupon_amt ) / 100 )
                    WHEN promo_method_cd = 'PW' THEN
                        (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) -
                        ( ( (reg_rtl_prc / NULLIF(reg_rtl_prc_fctr,'0')) * coupon_amt ) / 100 )
                    ELSE 0
                END)
            ELSE NULL
        END ::DECIMAL(10,2) AS Mark_down
,       first_eff_dt AS Promo_Start
,       last_eff_dt AS Promo_End
,       TRIM(' ') ::CHAR(1) AS Pass_Through
,       TRIM(' ') ::CHAR(1) AS NEW_Multiple
,       TRIM(' ') ::CHAR(1) AS NEW_Retail
,       TRIM(' ') ::CHAR(1) AS NEW_EDLP_GP_pctg
,       TRIM(' ') ::CHAR(1) AS NEW_Promo_Multiple
,       TRIM(' ') ::CHAR(1) AS NEW_Promo_Retail
,       TRIM(' ') ::CHAR(1) AS NEW_Promo_GPpctg
,       TRIM(' ') ::CHAR(1) AS NEW_Passthrough
,       cmp.COMPET_CODE AS compet_code  --TRIM(TB2.COMPET_CODE) AS "COMPET_CODE"
,       cmp.PRICE_CHK_DATE AS price_chk_date --TB2.PRICE_CHK_DATE AS "DATE"
,       cmp.CMP_PRICE_FCTR AS comp_reg_mult -- TB2.CMP_PRICE_FCTR AS "COMP_REG_MULT"
,       cmp.CMP_PRICE AS com_reg_price -- (TB2.CMP_PRICE) AS "COMP_REG_PRICE"
,       TRIM(' ') ::CHAR(1) AS REG_CPI
,       cmp.CMP_SHELF_PRC_FCTR AS COMP_AD_MULT --TB2.CMP_SHELF_PRC_FCTR AS "COMP_AD_MULT"
,       cmp.CMP_SHELF_PRICE AS COMP_AD_PRICE --(TB2.CMP_SHELF_PRICE) AS "COMP_AD_PRICE"
,       TRIM(' ') ::CHAR(1) AS Comments
,       TRIM(' ') ::CHAR(1) AS Modified_flag
,       CASE
            WHEN t.rog_cd = 'SEAS' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-SEAS' || '-'
            WHEN t.rog_cd = 'SEAG' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-SEAG' || '-'
            WHEN t.rog_cd = 'ACME' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-ACME' || '-'
            WHEN t.rog_cd = 'AKAB' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-AKAB' || '-' -- added this logic on 9.28
            WHEN t.rog_cd = 'SWMA' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '3-SWMA' || '-' -- added this logic on 9.28
            WHEN t.rog_cd = 'SDEN' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-SDEN' || '-'
            WHEN t.rog_cd = 'ADEN' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-ADEN' || '-'
            WHEN t.rog_cd = 'AIMT' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-AIMT' || '-'
            WHEN t.rog_cd = 'AJWL' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-AJWL' || '-'
            WHEN t.rog_cd = 'SNCA' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-SNCA' || '-'
            WHEN t.rog_cd = 'SHAW' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-SHAW' || '-'
            WHEN t.rog_cd = 'SPRT' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-SPRT' || '-'
            WHEN t.rog_cd = 'APOR' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-APOR' || '-'
            WHEN t.rog_cd = 'SSEA' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-SSEA' || '-'
            WHEN t.rog_cd = 'SSPK' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-SSPK' || '-'
            WHEN t.rog_cd = 'SACG' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '3-SACG' || '-'
            WHEN t.rog_cd = 'ASHA' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-ASHA' || '-'
            WHEN t.rog_cd = 'AVMT' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-AVMT' || '-'
            WHEN t.rog_cd = 'VSOC' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-VSOC' || '-'
            WHEN t.rog_cd = 'ASOC' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-ASOC' || '-'
            WHEN t.rog_cd = 'PSOC' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '3-PSOC' || '-'
            WHEN t.rog_cd = 'RDAL' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-RDAL' || '-'
            WHEN t.rog_cd = 'ADAL' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-ADAL' || '-'
            WHEN t.rog_cd = 'RHOU' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '3-RHOU' || '-'
            WHEN t.rog_cd = 'AHOU' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '4-AHOU' || '-'
            WHEN t.rog_cd = 'SPHO' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '1-SPHO' || '-'
            WHEN t.rog_cd = 'APHO' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '2-APHO' || '-'
            WHEN t.rog_cd = 'ALAS' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '3-ALAS' || '-'
            WHEN t.rog_cd = 'VLAS' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '4-VLAS' || '-'
            WHEN t.rog_cd = 'SPHX' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '5-SPHX' || '-'
            WHEN t.rog_cd = 'TEST' AND DIV_PROMO_GRP_CD IS NOT NULL THEN '6-TEST' || '-'
            ELSE TRIM(' ')  --MAYANK : changed rog_cd to t.rog_cd for Lead Item
        END || COALESCE(TRIM(CAST(DIV_PROMO_GRP_CD AS CHAR(11))), TRIM(' ')) ::VARCHAR(15) AS ROG_and_CIG
,       TRIM(COUNT(CASE WHEN ALW_TYP = 'C' THEN ALLOW_AMT END)) || '-' ||
        TRIM(COUNT(CASE WHEN ALW_TYP = 'S' THEN ALLOW_AMT END)) || '-' ||
        TRIM(COUNT(CASE WHEN ALW_TYP = 'T' THEN ALLOW_AMT END)) || '-' ||
        TRIM(COUNT(CASE WHEN ALW_TYP = 'R' THEN ALLOW_AMT END)) ::CHAR(50) AS Allowance_Counts
-- ,       DATE '2021-07-01' AS Report_Date  --Max(CURRENT DATE+(13-Dayofweek(CURRENT DATE+2 DAY))DAY) AS "Report Date"
-- Akash: 27/03/2019 : Changed Report_Date with below logic
,       (CASE
            WHEN (t.division_id = '33' or t.division_id = '34') THEN (current_date + """+str(east_gap)+""")::DATE
            ELSE (current_date + """+str(reg_gap)+""")::DATE
        END) AS Report_Date
        
FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_5 t
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.SMIC_Group grp
ON      grp.SMIC_GROUP_CD = t.group_id
AND     grp.DW_CURRENT_VERSION_IND = TRUE 
AND     grp.DW_LOGICAL_DELETE_IND = FALSE
INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.SMIC_Category cat
ON      cat.smic_group_cd * 100 + cat.smic_category_cd = t.category_id
AND     cat.DW_CURRENT_VERSION_IND = TRUE 
AND     cat.DW_LOGICAL_DELETE_IND = FALSE
 
-- MAYANK : Adding join for Price Per column    
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.lu_upc_rog_clone itm    
ON      itm.corporation_id = t.corporation_id   
AND     itm.rog_cd = t.rog_cd   
AND     itm.upc_id = t.upc_id
 
LEFT    OUTER JOIN (
        SELECT  CORPORATE_UPC_NBR AS UPC_ID
        ,       COMPETITOR_FACILITY_CD     AS COMPET_FACILITY
        ,       COMPETITOR_ITEM_PRICE_FCTR    AS CMP_PRICE_FCTR
        ,       COMPETITOR_ITEM_PRICE_AMT     AS CMP_PRICE
        ,       COMPETITOR_SHELF_PRICE_FCTR    AS CMP_SHELF_PRC_FCTR
        ,       COMPETITOR_SHELF_PRICE_AMT      AS CMP_SHELF_PRICE
        ,       COMPETITOR_PRICE_CHECK_TS::DATE        AS PRICE_CHK_DATE
        ,        COMPETITOR_CD               AS COMPET_CODE
       
        FROM    EDM_VIEWS_PRD.DW_VIEWS.Competitor_Item_Price
                WHERE DW_CURRENT_VERSION_IND = TRUE 
                AND DW_LOGICAL_DELETE_IND = FALSE
        GROUP   BY 1,2,3,4,5,6,7,8
        ) cmp
ON      cmp.upc_id = t.upc_id
AND     cmp.compet_facility = (
        CASE WHEN t.ROG_CD = 'SEAG' THEN '7319'
            WHEN t.ROG_CD = 'SEAS' AND PRICE_AREA_ID = 4 THEN '1806'
            WHEN t.ROG_CD = 'SEAS' AND PRICE_AREA_ID <> 4 THEN '7319'
 
            WHEN t.ROG_CD = 'ACME' AND PRICE_AREA_ID IN (2,3,5) THEN '9503'
            WHEN t.ROG_CD = 'ACME' AND PRICE_AREA_ID IN (4) THEN '9504'
            WHEN t.ROG_CD = 'ACME' AND PRICE_AREA_ID IN (6, 87) THEN '9587'
            WHEN t.ROG_CD = 'ACME' AND PRICE_AREA_ID IN (7, 9) THEN '9509'
            WHEN t.ROG_CD = 'ACME' AND PRICE_AREA_ID IN (8) THEN '9508'
            WHEN t.ROG_CD = 'ACME' AND PRICE_AREA_ID IN (88, 89) THEN '9589'
 
            WHEN t.ROG_CD = 'ADEN' AND PRICE_AREA_ID IN (1) THEN '2039'
            WHEN t.ROG_CD = 'ADEN' AND PRICE_AREA_ID IN (2, 3) THEN '2000'
            WHEN t.ROG_CD = 'SDEN' AND PRICE_AREA_ID IN (12) THEN '2650'
            WHEN t.ROG_CD = 'SDEN' AND PRICE_AREA_ID IN (13, 41, 51, 55, 61, 70, 71, 95) THEN '2000'
 
            WHEN t.ROG_CD = 'AIMT' AND PRICE_AREA_ID IN (1, 2, 5) THEN '9119'
            WHEN t.ROG_CD = 'AIMT' AND PRICE_AREA_ID IN (4) THEN '9112'
            WHEN t.ROG_CD = 'AIMT' AND PRICE_AREA_ID IN (6) THEN '9111'
            WHEN t.ROG_CD = 'AIMT' AND PRICE_AREA_ID IN (8) THEN '9124'
            WHEN t.ROG_CD = 'AIMT' AND PRICE_AREA_ID IN (14) THEN '9128'
 
            WHEN t.ROG_CD = 'AJWL' THEN '8002'
 
            WHEN t.ROG_CD = 'SHAW' AND PRICE_AREA_ID IN (47) THEN '5601'
            WHEN t.ROG_CD = 'SNCA' AND PRICE_AREA_ID IN (1) THEN '4194'
            WHEN t.ROG_CD = 'SNCA' AND PRICE_AREA_ID IN (11) THEN '4115'
            WHEN t.ROG_CD = 'SNCA' AND PRICE_AREA_ID IN (17) THEN '4128'
            WHEN t.ROG_CD = 'SNCA' AND PRICE_AREA_ID IN (18) THEN '4295'
 
            WHEN t.ROG_CD = 'SPRT' AND PRICE_AREA_ID IN (1, 61) THEN '9502'
            WHEN t.ROG_CD = 'SPRT' AND PRICE_AREA_ID NOT IN (1, 61) THEN '3500'
            WHEN t.ROG_CD = 'APOR' THEN '3500'
 
            WHEN t.ROG_CD = 'SSEA' AND PRICE_AREA_ID IN (47,73, 42,53) THEN '6185'
            WHEN t.ROG_CD = 'SSEA' AND PRICE_AREA_ID IN (33,43) THEN '6120'
            WHEN t.ROG_CD = 'SSEA' AND PRICE_AREA_ID IN (34) THEN '6134'
            WHEN t.ROG_CD = 'SSPK' AND PRICE_AREA_ID IN (60,62,63,65) THEN '6160'
            WHEN t.ROG_CD = 'SSPK' AND PRICE_AREA_ID IN (71) THEN '6161'
            WHEN t.ROG_CD = 'SSPK' AND PRICE_AREA_ID IN (72) THEN '6673'
            WHEN t.ROG_CD = 'SACG' AND PRICE_AREA_ID IN (8,10,16) THEN '6108'
 
            WHEN t.ROG_CD = 'ASHA' AND PRICE_AREA_ID IN (3,4,8,10) THEN '3304'
            WHEN t.ROG_CD = 'ASHA' AND PRICE_AREA_ID IN (5) THEN '3305'
            WHEN t.ROG_CD = 'ASHA' AND PRICE_AREA_ID IN (6,16,22,23) THEN '3301'
            WHEN t.ROG_CD = 'ASHA' AND PRICE_AREA_ID IN (54,61) THEN '3302'
            WHEN t.ROG_CD = 'AVMT' AND PRICE_AREA_ID IN (32,35) THEN '3303'
            WHEN t.ROG_CD = 'AVMT' AND PRICE_AREA_ID IN (58) THEN '3302'
 
            WHEN t.ROG_CD = 'ASOC' AND PRICE_AREA_ID IN (1,2,4) THEN '9307'
            WHEN t.ROG_CD = 'ASOC' AND PRICE_AREA_ID IN (3,10) THEN '9106'
            WHEN t.ROG_CD = 'VSOC' AND PRICE_AREA_ID IN (1,2,4) THEN '9307'
            WHEN t.ROG_CD = 'VSOC' AND PRICE_AREA_ID IN (3) THEN '9106'
            WHEN t.ROG_CD = 'PSOC' AND PRICE_AREA_ID IN (88,89) THEN '9307'
 
            WHEN t.ROG_CD = 'RHOU' THEN '1720'
            WHEN t.ROG_CD = 'RDAL' THEN '1724'
            WHEN t.ROG_CD = 'ADAL' AND PRICE_AREA_ID NOT IN (6) THEN '1789'
            WHEN t.ROG_CD = 'ADAL' AND PRICE_AREA_ID IN (6) THEN '1791'
            WHEN t.ROG_CD = 'AHOU' THEN '1804'
 
            WHEN t.ROG_CD = 'ALAS' AND PRICE_AREA_ID IN (71,72) THEN '9401'
            WHEN t.ROG_CD = 'APHO' AND PRICE_AREA_ID IN (31) THEN '9931'
            WHEN t.ROG_CD = 'APHO' AND PRICE_AREA_ID IN (41) THEN '2749'
            WHEN t.ROG_CD = 'SPHO' AND PRICE_AREA_ID IN (1) THEN '2749'
        END
        )
GROUP   BY cmp.compet_facility
,       itm.up_measure
,       division_id
,       promo_no_allowance
,       allow_no_promo
,       missing_allowance
,       cost_change
,       ad_plan
,       less_10_Promo
,       less_10_allowance
,       greater_100_pass_through
,       t_09_Retail
,       lead_item
,       Dominant_Price_Area
,       OOB
,       sskvi
,       group_cd
,       group_nm
,       SMIC
,       SMIC_name
,       t.rog_cd      --MAYANK : changed rog_cd to t.rog_cd
,       price_area_id
,       PA_name
,       pricing_role
,       OOB_gap_id
,       div_promo_grp_cd  -- CIG
,       loc_common_retail_cd
,       vendor_name
,       vend_nbr
,       vend_sub_acct_nbr
,       cost_area_id
,       Manuf
,       t.upc_id
,       corp_item_cd
,       item_description
,       DST
,       FACILITY
,       dst_stat
,       rtl_stat
,       buyer_nm
,       vend_conv_fctr
,       pack_retail_qty
,       size_dsc
,       Row_Offset
,       UPC_13_Wk_Avg_Sales
,       UPC_13_Wk_Avg_Qty
,       UPC_13_Wk_Avg_RTL
,       t_Rank
,       pct_ACV_Stores
,       CPC_13_Wk_Avg_Sales
,       CPC_13_Wk_Avg_Qty
,       CPC_13_Wk_Avg_RTL
,       PND_Cost_Change_VND
--COALESCE (PENDING_VENDOR_COST.COST_VEND, PENDING_WSD_COST.COST_VEND) /
--  VEND_CONV_FCTR / PACK_WHSE AS "PND Cost Change VND"
,       PND_VEN_Date_Effective
,       New_Recc_Reg_Retail
,       Vendor_Unit_Cost
--, Decimal(TB2.COST_VEND/PACK_WHSE/VEND_CONV_FCTR,10,3) AS "Vendor Unit Cost"
,       Unit_Item_Billing_Cost
--, Decimal(TB2.COST_IB / PACK_WHSE, 10, 3) AS "Unit Item Billing Cost"
,       Prev_Retail_Price_Fctr
,       Previous_Retail_Price
,       Prev_Retail_Effective_Date
,       Pending_EDLP_Mult
,       Pending_EDLP_Retail
,       Pending_EDLP_Chg_Date
,       Pending_GP_pctg
,       Reg_Retail_Price_Fctr
,       Reg_Retail
,       t_price_Per -- missing SSITMPOS.up_measure
--, Decimal(Round(promo_prc / promo_prc_fctr /
--  ITM.UP_MEASURE * up_mult_factor, 2), 10, 3) AS "Price_Per..."
,       t_Unit
--, CASE
--    WHEN TB2.PRICE IS NOT NULL THEN up_label_unit
--    ELSE TRIM(' ')
--  END AS "...Unit"
,       Reg_GP_pctg
,       Total_Allow_Unit
,       Allowance_pctg
,       Net_Cost_with_Allow
,       Promo_Multiple
,       Promo_Price
,       Coupon_Method
,       Min_Purch
,       Limit_Per_Txn
,       Promo_GP_pctg
,       Net_Promo_Price
,       Price_Per
,       t2_Unit
,       Markdown_pctg
,       Mark_down
,       Promo_Start
,       Promo_End
,       Pass_Through
,       NEW_Multiple
,       NEW_Retail
,       NEW_EDLP_GP_pctg
,       NEW_Promo_Multiple
,       NEW_Promo_Retail
,       NEW_Promo_GPpctg
,       NEW_Passthrough
,       compet_code  --TRIM(TB2.COMPET_CODE) AS "COMPET_CODE"
,       price_chk_date --TB2.PRICE_CHK_DATE AS "DATE"
,       comp_reg_mult -- TB2.CMP_PRICE_FCTR AS "COMP_REG_MULT"
,       com_reg_price -- (TB2.CMP_PRICE) AS "COMP_REG_PRICE"
,       REG_CPI
,       COMP_AD_MULT --TB2.CMP_SHELF_PRC_FCTR AS "COMP_AD_MULT"
,       COMP_AD_PRICE --(TB2.CMP_SHELF_PRICE) AS "COMP_AD_PRICE"
,       Comments
,       Modified_flag
,       ROG_and_CIG
,       Report_Date;
-- EXPR_FORMAT - Convert expression FORMAT/CAST_AS_FORMAT to TO_CHAR/TO_DATE
-- FUN_CAST_OPTR - Reformat casting
-- FUN_DATE_CAST - Reformat STRING-to-DATE casting
-- NULLIFZERO: Translated NullifZero to nullif function
```

## Step 49:
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales;
-- DROP_TBL: Add if exists in drop table
```

## Step 50:
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales 
     (
      division_id INTEGER,
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      upc_id DECIMAL(14,0),
      rog_upc VARCHAR(25)  COLLATE 'en-ci' ,
      upc_dsc VARCHAR(40)  COLLATE 'en-ci' ,
      department_id VARCHAR(11)  COLLATE 'en-ci' ,
      department_name VARCHAR(30)  COLLATE 'en-ci' ,
      group_id VARCHAR(6)  COLLATE 'en-ci' ,
      group_nm VARCHAR(46)  COLLATE 'en-ci' ,
      category_id VARCHAR(6)  COLLATE 'en-ci' ,
      category_nm VARCHAR(46)  COLLATE 'en-ci' ,
      cpc VARCHAR(11)  COLLATE 'en-ci' ,
      div_promo_grp_cd INTEGER,
      avg_net_sales_13_wk DECIMAL(18,2),
      rank_by_rog_and_cpc VARCHAR(18)  COLLATE 'en-ci' ,
      avg_item_qty_13_wk DECIMAL(18,3),
      num_stores_selling INTEGER,
      num_stores_in_rog INTEGER);
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

## Step 51:
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales
(       division_id
,       rog_cd
,       upc_id
,       ROG_UPC
,       upc_dsc
,       department_id
,       department_name
,       group_id
,       group_nm
,       category_id
,       category_nm
,       cpc
,       div_promo_grp_cd
,       avg_net_sales_13_wk
,       rank_by_rog_and_cpc
,       avg_item_qty_13_wk
,       num_stores_selling
,       num_stores_in_rog
)
SELECT  ft.division_id
,       str.rog_id
,       upc.upc_nbr
,       str.rog_id || '-' || TRIM(cast(upc.upc_nbr AS BIGINT)) AS rog_upc
,       upc.ITEM_DSC
,       TRIM(upc.RETAIL_DEPARTMENT_ID) AS department_id
,       TRIM(upc.department_nm) AS department_name
,       TRIM(upc.smic_group_id)      AS group_id
,       TRIM(upc.SMIC_GROUP_DSC)      AS group_nm
,       TRIM(upc.SMIC_CATEGORY_ID)   AS category_id
,       TRIM(upc.SMIC_CATEGORY_DSC)   AS category_nm
,       cic.COMMON_PROMOTION_CD  AS cpc
,       urx.common_item_group_cd
,       SUM(agp.net_amt) /13    AS avg_net_sales_13_wk
,       CASE
            WHEN cic.COMMON_PROMOTION_CD <> NULL
                THEN str.rog_id || '_' || RIGHT('00' || TRIM(ROW_NUMBER() OVER(
                    PARTITION BY str.rog_id,cic.COMMON_PROMOTION_CD
                    ORDER BY avg_net_sales_13_wk DESC)),3)
            ELSE NULL
        END AS rank_by_rog_and_cpc
,       SUM(agp.item_qty)/13    AS avg_item_qty_13_wk
,       COUNT (DISTINCT FT.FACILITY_NBR) AS num_stores_selling
,       tb1.total_stores            AS num_stores_in_rog
FROM    EDM_VIEWS_PRD.DW_VIEWS.store_upc_agp agp
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE AS STR
        ON  STR.FACILITY_INTEGRATION_ID = AGP.FACILITY_INTEGRATION_ID
        AND STR.DW_LOGICAL_DELETE_IND = FALSE
        AND STR.DW_CURRENT_VERSION_IND = TRUE
        AND STR.CORPORATION_ID = 1
 
 
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.FACILITY AS FT
        ON STR.FACILITY_INTEGRATION_ID = FT.FACILITY_INTEGRATION_ID
        AND FT.DW_LOGICAL_DELETE_IND = FALSE
        AND FT.DW_CURRENT_VERSION_IND = TRUE
INNER JOIN (
        SELECT  rog_id
        ,       upc_nbr
        ,       common_item_group_cd
        FROM    (
                SELECT  CUR.rog_id
                ,       upc_nbr
                ,       COALESCE(CIGI.common_item_group_cd,0)::NUMBER as common_item_group_cd
                ,       COUNT(1) AS cnt
                FROM    EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_ROG_REFERENCE as cur
          
                INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP AS ROG
                ON ROG.ROG_ID = CUR.ROG_ID 
                AND ROG.DW_LOGICAL_DELETE_IND = FALSE AND ROG.DW_CURRENT_VERSION_IND = TRUE  
          
                LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.Common_Item_Group_Item AS CIGI
                on CIGI.Corporate_item_integration_id = cur.Corporate_item_integration_id
                AND ROG.DIVISION_ID = CIGI.DIVISION_ID
                AND CIGI.DW_LOGICAL_DELETE_IND = FALSE
                AND CIGI.DW_CURRENT_VERSION_IND = TRUE
               
                WHERE   CUR.DW_LOGICAL_DELETE_IND = FALSE
                AND CUR.DW_CURRENT_VERSION_IND = TRUE
                -- MAYANK : 20190625 : To bring out the sales for UPCs with blank CIGs
                -- AND     div_promo_grp_cd > 0
                -- MAYANK : 20190527 : To stop UPCs with following status
                AND    Cur.status_cd NOT IN ('X','D')
                GROUP   BY 1,2,3
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY rog_id, upc_NBR ORDER BY TO_NUMBER(common_item_group_cd) DESC, cnt DESC) = 1
        ) urx
ON      urx.rog_id = str.rog_id
AND     urx.upc_nbr = agp.upc_nbr
INNER   JOIN    (
        SELECT  str.rog_id
        ,       COUNT(DISTINCT STR.FACILITY_NBR) AS total_stores
 
        FROM     EDM_VIEWS_PRD.DW_VIEWS.FACILITY as FT
 
        INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE AS STR
        ON STR.FACILITY_INTEGRATION_ID = FT.FACILITY_INTEGRATION_ID
        AND STR.DW_LOGICAL_DELETE_IND = FALSE
        AND STR.DW_CURRENT_VERSION_IND = TRUE
       
       JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_RETAIL_STORE D1
        ON   D1.FACILITY_INTEGRATION_ID = FT.FACILITY_INTEGRATION_ID
        AND  D1.DW_LOGICAL_DELETE_IND = FALSE
        AND  D1.CORPORATION_ID = '001'
        AND  D1.FACILITY_STATUS_CD    = 'OPEN'
        AND  D1.FACILITY_SUB_TYPE_CD <> 'X'
        AND  D1.FACILITY_SUB_TYPE_CD <> 'N/A'
        AND  D1.DISTRICT_TYPE_CD     <> 'N'
        AND  D1.CONVERSION_IND        = TRUE

        WHERE   STR.corporation_id = 1
        AND     FT.close_dt > CURRENT_DATE
        AND     FT.DW_LOGICAL_DELETE_IND = FALSE
        AND     FT.DW_CURRENT_VERSION_IND = TRUE
        AND     FT.FACILITY_NBR NOT IN (38,47,339,630,1227,1509,1708,4025,4041,4602  -- div 30 exclusions
                , 1132,1615,2911,2919 -- div 05 exclusions
                , 9835 -- div 35 exclusions
                )
       
        AND     STR.FACILITY_NBR NOT IN (
          SELECT RT.FACILITY_NBR FROM  EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE RT
         
          INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.FACILITY FT
          ON FT.FACILITY_NBR = RT.FACILITY_NBR
          AND FT.DW_LOGICAL_DELETE_IND = FALSE
          AND FT.DW_CURRENT_VERSION_IND = TRUE
         
          INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.DISTRICT DT
          ON DT.DISTRICT_CD = RT.DISTRICT_CD
          AND DT.DW_LOGICAL_DELETE_IND = FALSE
          AND DT.DW_CURRENT_VERSION_IND = TRUE
         
          WHERE FT.division_id = 27 AND DT.district_id = 39 AND RT.DW_LOGICAL_DELETE_IND = FALSE AND  RT.DW_CURRENT_VERSION_IND = TRUE)  -- div 27 exclusions
        GROUP   BY STR.rog_id
        ) AS tb1
ON      str.rog_id = tb1.rog_id
INNER   JOIN    EDM_VIEWS_PRD.DW_VIEWS.D1_UPC upc
ON      upc.upc_nbr = agp.upc_nbr
AND     upc.corporation_id = 1
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM AS CIC
ON CIC.CORPORATE_ITEM_INTEGRATION_ID = UPC.CORPORATE_ITEM_INTEGRATION_ID
AND CIC.DW_LOGICAL_DELETE_IND = FALSE
AND CIC.DW_CURRENT_VERSION_IND = TRUE
WHERE   -- MAYANK : 20190627 : Including all the SMICs as suggested by Steve
--      upc.SMIC_CATEGORY_ID NOT IN (1350, 1390,1995,3290,7301,7305,7306,7310,7315,
--      7320,7325,7330,7335,7340,7345,7401,7402,7405,
--      7410,7415,7416,7420,7425,7430,7445,7450,7470,
--      7480,7485,7490)
--      AND    
        agp.transaction_dt BETWEEN CURRENT_DATE - 93 AND CURRENT_DATE - 3
AND     NOT (agp.net_amt <= 0 and agp.item_qty <= 0)
AND     FT.close_dt > CURRENT_DATE
AND     upc.smic_group_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,36,37,38,39,40,42,43,44,45,46,47,48,
49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,84,85,86,88,89,91,92,94,95,96,97)
AND     ft.facility_nbr NOT IN (38,47,339,630,1227,1509,1708,4025,4041,4602
        , 1132,1615,2911,2919
        , 9835
        )
AND     ft.facility_nbr NOT IN (
  SELECT RT.FACILITY_NBR FROM  EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE RT
         
          INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.FACILITY FT
          ON FT.FACILITY_NBR = RT.FACILITY_NBR
          AND FT.DW_LOGICAL_DELETE_IND = FALSE
          AND FT.DW_CURRENT_VERSION_IND = TRUE
         
          INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.DISTRICT DT
          ON DT.DISTRICT_CD = RT.DISTRICT_CD
          AND DT.DW_LOGICAL_DELETE_IND = FALSE
          AND DT.DW_CURRENT_VERSION_IND = TRUE
         
          WHERE FT.division_id = 27 AND DT.district_id = 39 AND RT.DW_LOGICAL_DELETE_IND = FALSE AND  RT.DW_CURRENT_VERSION_IND = TRUE
 
)
GROUP   BY ft.division_id
,       STR.rog_id
,       upc.upc_nbr
,       upc.ITEM_DSC
,       upc.RETAIL_DEPARTMENT_ID
,       upc.department_nm
,       upc.smic_group_id
,       upc.SMIC_GROUP_DSC
,       upc.SMIC_CATEGORY_ID
,       upc.SMIC_CATEGORY_DSC
,       cic.COMMON_PROMOTION_CD
,       urx.common_item_group_cd
,       tb1.total_stores;
-- COL_ALIAS_IN_EXPR - Review column alias reference
-- FUN_DATE_CAST - Reformat STRING-to-DATE casting
-- FUN_TD_LEFT_RIGHT - replace function LEFT and RIGHT with TD_LEFT and TD_RIGHT
```

## Step 52:
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases;
-- DROP_TBL: Add if exists in drop table
```

## Step 53:
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases 
     (
      division_id INTEGER,
      rpt_group VARCHAR(100)  COLLATE 'en-ci' ,
      rog_cd CHAR(4)  COLLATE 'en-ci' ,
      upc_id DECIMAL(14,0),
      rog_upc VARCHAR(25)  COLLATE 'en-ci' ,
      upc_dsc VARCHAR(40)  COLLATE 'en-ci' ,
      department_id VARCHAR(11)  COLLATE 'en-ci' ,
      department_name VARCHAR(30)  COLLATE 'en-ci' ,
      group_id VARCHAR(6)  COLLATE 'en-ci' ,
      group_nm VARCHAR(46)  COLLATE 'en-ci' ,
      category_id VARCHAR(6)  COLLATE 'en-ci' ,
      category_nm VARCHAR(46)  COLLATE 'en-ci' ,
      cpc VARCHAR(11)  COLLATE 'en-ci' ,
      div_promo_grp_cd INTEGER,
      rank_by_rog_and_cpc VARCHAR(18)  COLLATE 'en-ci' ,
      avg_net_sales_13_wk DECIMAL(18,2),
      avg_item_qty_13_wk DECIMAL(18,3),
      num_stores_selling INTEGER,
      num_stores_in_rog INTEGER);
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

## Step 54:
```sql
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
(       division_id
,       rpt_group
,       rog_cd
,       upc_id
,       ROG_UPC
,       upc_dsc
,       department_id
,       department_name
,       group_id
,       group_nm
,       category_id
,       category_nm
,       cpc
,       div_promo_grp_cd
,       rank_by_rog_and_cpc
,       avg_net_sales_13_wk
,       avg_item_qty_13_wk
,       num_stores_selling
,       num_stores_in_rog
)
SELECT dt.division_id
,        dt.rpt_group
,       dt.rog_id
,       dt.upc_nbr
,       dt.rog_upc
,       dt.item_dsc
,       dt.department_id
,       dt.department_name
,       dt.group_id
,       dt.group_nm
,       dt.category_id
,       dt.category_nm
,       dt.cpc
,       dt.common_item_group_cd
,       dt.rank_by_rog_and_cpc
,       dt.avg_net_sales_13_wk
,       dt.avg_item_qty_13_wk
,       dt.num_stores_selling
,       tb1.total_stores AS num_stores_in_rog
FROM    (
        SELECT  ft.division_id
        ,       CASE
                    WHEN ft.facility_nbr = 199 THEN 'S30_02'
                    WHEN ft.facility_nbr IN (339, 1509, 1708) THEN 'S30_03'
                    WHEN ft.facility_nbr IN (1911,1912,2089,2105,2203,2210,2212,2214,2215,2216,2217,2224,2225,2226,2228,2229,2231,2233,2235,2412,2739,2803,2813,3005,3237) THEN 'S29_02'
                END AS rpt_group
        ,       str.rog_id
        ,       upc.upc_nbr
        ,       str.rog_id || '-' || TRIM(cast(upc.upc_nbr AS BIGINT)) AS rog_upc
        ,       upc.ITEM_DSC
        ,       TRIM(upc.RETAIL_DEPARTMENT_ID) AS department_id
        ,       TRIM(upc.department_nm) AS department_name
        ,       TRIM(upc.smic_group_id)      AS group_id
        ,       TRIM(upc.SMIC_GROUP_DSC)      AS group_nm
        ,       TRIM(upc.SMIC_CATEGORY_ID)   AS category_id
        ,       TRIM(upc.SMIC_CATEGORY_DSC)   AS category_nm
        ,       cic.COMMON_PROMOTION_CD  AS cpc
        ,       urx.common_item_group_cd
        ,       SUM(agp.net_amt) /13    AS avg_net_sales_13_wk
        ,       CASE
                    WHEN cic.COMMON_PROMOTION_CD <> NULL
                        THEN str.rog_id || '_' || RIGHT('00' || TRIM(ROW_NUMBER() OVER(
                            PARTITION BY str.rog_id,cic.COMMON_PROMOTION_CD
                            ORDER BY avg_net_sales_13_wk DESC)),3)
                    ELSE NULL
                END AS rank_by_rog_and_cpc
        ,       SUM(agp.item_qty)/13    AS avg_item_qty_13_wk
        ,       COUNT (DISTINCT ft.facility_nbr) AS num_stores_selling
  
        FROM EDM_VIEWS_PRD.DW_VIEWS.STORE_UPC_AGP AS AGP
        
       INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE AS STR
        ON  STR.FACILITY_INTEGRATION_ID = AGP.FACILITY_INTEGRATION_ID
        AND STR.DW_LOGICAL_DELETE_IND = FALSE 
        AND STR.DW_CURRENT_VERSION_IND = TRUE
        AND STR.CORPORATION_ID = 1
  
  
        INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.FACILITY AS FT 
        ON STR.FACILITY_INTEGRATION_ID = FT.FACILITY_INTEGRATION_ID
        AND FT.DW_LOGICAL_DELETE_IND = FALSE
        AND FT.DW_CURRENT_VERSION_IND = TRUE
  
        INNER JOIN (
        SELECT  rog_id
        ,       upc_nbr
        ,       common_item_group_cd
        FROM    (
                SELECT  CUR.rog_id
                ,       upc_nbr
                ,       COALESCE(CIGI.common_item_group_cd,0)::NUMBER as common_item_group_cd
                ,       COUNT(1) AS cnt
                FROM    EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_ROG_REFERENCE as cur
          
                INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP AS ROG
                ON ROG.ROG_ID = CUR.ROG_ID 
                AND ROG.DW_LOGICAL_DELETE_IND = FALSE AND ROG.DW_CURRENT_VERSION_IND = TRUE  
          
                LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.Common_Item_Group_Item AS CIGI
                on CIGI.Corporate_item_integration_id = cur.Corporate_item_integration_id
                AND ROG.DIVISION_ID = CIGI.DIVISION_ID
                AND CIGI.DW_LOGICAL_DELETE_IND = FALSE
                AND CIGI.DW_CURRENT_VERSION_IND = TRUE
               
                WHERE   CUR.DW_LOGICAL_DELETE_IND = FALSE
                AND CUR.DW_CURRENT_VERSION_IND = TRUE
                -- MAYANK : 20190625 : To bring out the sales for UPCs with blank CIGs
                -- AND     div_promo_grp_cd > 0
                -- MAYANK : 20190527 : To stop UPCs with following status
                AND    Cur.status_cd NOT IN ('X','D')
                GROUP   BY 1,2,3
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY rog_id, upc_NBR ORDER BY TO_NUMBER(common_item_group_cd) DESC, cnt DESC) = 1
        ) urx
ON      urx.rog_id = str.rog_id
AND     urx.upc_nbr = agp.upc_nbr
  
        INNER   JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_UPC upc
        ON      upc.upc_nbr = agp.upc_nbr
        AND     upc.corporation_id = 1
        INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM AS CIC
        ON CIC.CORPORATE_ITEM_INTEGRATION_ID = UPC.CORPORATE_ITEM_INTEGRATION_ID
        AND CIC.DW_LOGICAL_DELETE_IND = FALSE
        AND CIC.DW_CURRENT_VERSION_IND = TRUE
        
        WHERE   -- MAYANK : 20190627 : Including all the SMICs as suggested by Steve
        --         upc.SMIC_CATEGORY_ID NOT IN (1350, 1390,1995,3290,7301,7305,7306,7310,7315,
        --         7320,7325,7330,7335,7340,7345,7401,7402,7405,
        --         7410,7415,7416,7420,7425,7430,7445,7450,7470,
        --         7480,7485,7490)
        -- AND     
                agp.transaction_dt BETWEEN CURRENT_DATE - 93 AND CURRENT_DATE - 3
        AND     NOT (agp.net_amt <= 0 and agp.item_qty <= 0)
        AND     ft.close_dt > CURRENT_DATE
        AND     upc.smic_group_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,36,37,38,39,40,42,43,44,45,46,47,48,
49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,84,85,86,88,89,91,92,94,95,96,97)
        AND     ft.facility_nbr IN (199 -- S30_02
                , 339, 1509, 1708 -- 'S30_03'
                , 1911,1912,2089,2105,2203,2210,2212,2214,2215,2216,2217,2224,2225,2226,2228,2229,2231,2233,2235,2412,2739,2803,2813,3005,3237 -- 'S29_02'
                )
        GROUP   BY ft.division_id
        ,        rpt_group
        ,       str.rog_id
        ,       upc.upc_nbr
        ,       upc.ITEM_DSC
        ,       upc.RETAIL_DEPARTMENT_ID
        ,       upc.department_nm
        ,       upc.smic_group_id
        ,       upc.SMIC_GROUP_DSC
        ,       upc.SMIC_CATEGORY_ID
        ,       upc.SMIC_CATEGORY_DSC
        ,       cic.COMMON_PROMOTION_CD
        ,       urx. common_item_group_cd
        ) dt
INNER   JOIN    (
        SELECT  str.rog_id
        ,       CASE
                    WHEN FT.FACILITY_NBR = 199 THEN 'S30_02'
                    WHEN FT.FACILITY_NBR IN (339, 1509, 1708) THEN 'S30_03'
                    WHEN FT.FACILITY_NBR IN (1911,1912,2089,2105,2203,2210,2212,2214,2215,2216,2217,2224,2225,2226,2228,2229,2231,2233,2235,2412,2739,2803,2813,3005,3237) THEN 'S29_02'
                END AS rpt_group
        ,       COUNT(DISTINCT FT.FACILITY_NBR) AS total_stores
        FROM     EDM_VIEWS_PRD.DW_VIEWS.FACILITY as FT
        INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE AS STR
        ON STR.FACILITY_INTEGRATION_ID = FT.FACILITY_INTEGRATION_ID
        AND STR.DW_LOGICAL_DELETE_IND = FALSE 
        AND STR.DW_CURRENT_VERSION_IND = TRUE
        
        JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_RETAIL_STORE D1
          ON   D1.FACILITY_INTEGRATION_ID = FT.FACILITY_INTEGRATION_ID
          AND  D1.DW_LOGICAL_DELETE_IND = FALSE
          AND  D1.CORPORATION_ID = '001'
          AND  D1.FACILITY_STATUS_CD    = 'OPEN'
          AND  D1.FACILITY_SUB_TYPE_CD <> 'X'
          AND  D1.FACILITY_SUB_TYPE_CD <> 'N/A'
          AND  D1.DISTRICT_TYPE_CD     <> 'N'
          AND  D1.CONVERSION_IND        = TRUE


        WHERE   STR.corporation_id = 1
        AND     FT.close_dt > CURRENT_DATE
        AND FT.DW_LOGICAL_DELETE_IND = FALSE
        AND FT.DW_CURRENT_VERSION_IND = TRUE
        AND     FT.FACILITY_NBR IN (199, 339, 1509, 1708  -- div 30 special reports
                ,1911,1912,2089,2105,2203,2210,2212,2214,2215,2216,2217,2224,2225,2226,2228,2229,2231,2233,2235,2412,2739,2803,2813,3005,3237 -- div 29 special reports
                )
        GROUP   BY str.rog_id
        ,       rpt_group
        ) AS tb1
ON      dt.rog_id = tb1.rog_id
AND     dt.rpt_group = tb1.rpt_group;
-- COL_ALIAS_IN_EXPR - Review column alias reference
-- FUN_DATE_CAST - Reformat STRING-to-DATE casting
-- FUN_TD_LEFT_RIGHT - replace function LEFT and RIGHT with TD_LEFT and TD_RIGHT
```

## Step 55:
```sql
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_final;
-- DROP_TBL: Add if exists in drop table
```

## Step 56:
```sql
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_final 
     (
      division_id INTEGER,
      rpt_group VARCHAR(100)  COLLATE 'en-ci' ,
      promo_no_allowance VARCHAR(1)  COLLATE 'en-ci' ,
      allow_no_promo VARCHAR(1)  COLLATE 'en-ci' ,
      missing_allowance VARCHAR(1)  COLLATE 'en-ci' ,
      cost_change VARCHAR(1)  COLLATE 'en-ci' ,
      ad_plan CHAR(1)  COLLATE 'en-ci' ,
      less_10_Promo VARCHAR(1)  COLLATE 'en-ci' ,
      less_10_allowance VARCHAR(1)  COLLATE 'en-ci' ,
      greater_100_pass_through VARCHAR(1)  COLLATE 'en-ci' ,
      t_09_Retail CHAR(1)  COLLATE 'en-ci' ,
      lead_item VARCHAR(1)  COLLATE 'en-ci' ,
      Dominant_Price_Area CHAR(1)  COLLATE 'en-ci' ,
      t_OOB VARCHAR(1)  COLLATE 'en-ci' ,
      sskvi CHAR(1)  COLLATE 'en-ci' ,
      group_cd SMALLINT,
      group_nm CHAR(46)  COLLATE 'en-ci' ,
      SMIC CHAR(4)  COLLATE 'en-ci' ,
      SMIC_name CHAR(46)  COLLATE 'en-ci' ,
      rog VARCHAR(10)  COLLATE 'en-ci' ,
      price_area_id INTEGER,
      PA_name CHAR(1)  COLLATE 'en-ci' ,
      pricing_role CHAR(1)  COLLATE 'en-ci' ,
      OOB_gap_id CHAR(20)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      loc_common_retail_cd DECIMAL(5,0),
      vendor_name VARCHAR(40)  COLLATE 'en-ci' ,
      vend_nbr CHAR(6)  COLLATE 'en-ci' ,
      vend_sub_acct_nbr CHAR(3)  COLLATE 'en-ci' ,
      cost_area_id SMALLINT,
      Manuf CHAR(5)  COLLATE 'en-ci' ,
      upc_id DECIMAL(13,0),
      corp_item_cd DECIMAL(8,0),
      item_description VARCHAR(40)  COLLATE 'en-ci' ,
      DST CHAR(4)  COLLATE 'en-ci' ,
      FACILITY CHAR(4)  COLLATE 'en-ci' ,
      dst_stat CHAR(1)  COLLATE 'en-ci' ,
      rtl_stat CHAR(1)  COLLATE 'en-ci' ,
      buyer_nm CHAR(20)  COLLATE 'en-ci' ,
      vend_conv_fctr SMALLINT,
      t_pack_retail_qty DECIMAL(10,0),
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      Row_Offset CHAR(1)  COLLATE 'en-ci' ,
      UPC_13_Wk_Avg_Sales DECIMAL(18,2),
      UPC_13_Wk_Avg_Qty DECIMAL(18,3),
      UPC_13_Wk_Avg_RTL DECIMAL(18,3),
      T_RANK_BY_ROG_AND_CPC INTEGER,
      pct_ACV_Stores DECIMAL(5,2),
      t_CPC_13_Wk_Avg_Sales DECIMAL(18,2),
      t_CPC_13_Wk_Avg_Qty DECIMAL(18,3),
      t_CPC_13_Wk_Avg_RTL DECIMAL(18,3),
      PND_Cost_Change_VND DECIMAL(15,4),
      PND_VEN_Date_Effective DATE  comment '{"FORMAT":"YY/MM/DD"}',
      New_Recc_Reg_Retail DECIMAL(18,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      Vendor_Unit_Cost DECIMAL(10,3),
      Unit_Item_Billing_Cost DECIMAL(10,3),
      Prev_Retail_Price_Fctr DECIMAL(2,0),
      Previous_Retail_Price DECIMAL(7,2),
      Prev_Retail_Effective_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Pending_EDLP_Mult DECIMAL(2,0),
      Pending_EDLP_Retail DECIMAL(7,2),
      Pending_EDLP_Chg_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Pending_GP_pctg DECIMAL(15,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      Reg_Retail_Price_Fctr DECIMAL(2,0),
      Reg_Retail DECIMAL(10,2),
      t_price_Per DECIMAL(10,3),
      t_Unit VARCHAR(5)  COLLATE 'en-ci' ,
      Reg_GP_pctg DECIMAL(15,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      case_allow_count INTEGER,
      case_allow_amt DECIMAL(15,4),
      Case_Allow_amt_per_Unit DECIMAL(15,4),
      Case_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Case_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      S2S_Allow_count INTEGER,
      S2S_Allow_amt DECIMAL(15,4),
      S2S_Allow_amt_per_Unit DECIMAL(15,4),
      S2S_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      S2S_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Scan_Allow_count INTEGER,
      Scan_Allow_amt DECIMAL(15,4),
      Scan_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Scan_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Redem_Scan_Allow_count INTEGER,
      Redem_Allow_amt DECIMAL(15,4),
      Redem_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Redem_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Total_Allow_Unit DECIMAL(15,4),
      t_Allowance_pctg DECIMAL(15,3),
      Net_Cost_with_Allow DECIMAL(18,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      Promo_Multiple DECIMAL(3,0),
      Promo_Price DECIMAL(10,4),
      Coupon_Method CHAR(2)  COLLATE 'en-ci' ,
      Min_Purch DECIMAL(2,0),
      Limit_Per_Txn SMALLINT,
      Promo_GP_pctg DECIMAL(15,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      Net_Promo_Price DECIMAL(10,2),
      Price_Per DECIMAL(10,3),
      t2_Unit VARCHAR(5)  COLLATE 'en-ci' ,
      t_markdown_pctg DECIMAL(15,4),
      Mark_down DECIMAL(10,2),
      Promo_Start DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Promo_End DATE  comment '{"FORMAT":"YY/MM/DD"}',
      pass_through DECIMAL(15,2),
      NEW_Multiple CHAR(1)  COLLATE 'en-ci' ,
      NEW_Retail CHAR(1)  COLLATE 'en-ci' ,
      NEW_EDLP_GP_pctg CHAR(1)  COLLATE 'en-ci' ,
      NEW_Promo_Multiple CHAR(1)  COLLATE 'en-ci' ,
      NEW_Promo_Retail CHAR(1)  COLLATE 'en-ci' ,
      NEW_Promo_GPpctg CHAR(1)  COLLATE 'en-ci' ,
      NEW_Passthrough CHAR(1)  COLLATE 'en-ci' ,
      compet_code VARCHAR(6)  COLLATE 'en-ci' ,
      price_chk_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      comp_reg_mult DECIMAL(2,0),
      com_reg_price DECIMAL(7,2),
      REG_CPI DECIMAL(18,3),  --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      COMP_AD_MULT DECIMAL(2,0),
      COMP_AD_PRICE DECIMAL(7,2),
      Comments CHAR(1)  COLLATE 'en-ci' ,
      Modified_flag CHAR(1)  COLLATE 'en-ci' ,
      ROG_and_CIG VARCHAR(15)  COLLATE 'en-ci' ,
      Allowance_Counts CHAR(50)  COLLATE 'en-ci' ,
      Report_Date DATE  comment '{"FORMAT":"YY/MM/DD"}');
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
```

## Step 57:
```sql
sql_step55 = """
DROP TABLE if exists EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_final;
-- DROP_TBL: Add if exists in drop table"""
 
sql_step56 = """
CREATE /*SET*/ or replace TABLE EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_final 
     (
      division_id INTEGER,
      rpt_group VARCHAR(100)  COLLATE 'en-ci' ,
      promo_no_allowance VARCHAR(1)  COLLATE 'en-ci' ,
      allow_no_promo VARCHAR(1)  COLLATE 'en-ci' ,
      missing_allowance VARCHAR(1)  COLLATE 'en-ci' ,
      cost_change VARCHAR(1)  COLLATE 'en-ci' ,
      ad_plan CHAR(1)  COLLATE 'en-ci' ,
      less_10_Promo VARCHAR(1)  COLLATE 'en-ci' ,
      less_10_allowance VARCHAR(1)  COLLATE 'en-ci' ,
      greater_100_pass_through VARCHAR(1)  COLLATE 'en-ci' ,
      t_09_Retail CHAR(1)  COLLATE 'en-ci' ,
      lead_item VARCHAR(1)  COLLATE 'en-ci' ,
      Dominant_Price_Area CHAR(1)  COLLATE 'en-ci' ,
      t_OOB VARCHAR(1)  COLLATE 'en-ci' ,
      sskvi CHAR(1)  COLLATE 'en-ci' ,
      group_cd SMALLINT,
      group_nm CHAR(46)  COLLATE 'en-ci' ,
      SMIC CHAR(4)  COLLATE 'en-ci' ,
      SMIC_name CHAR(46)  COLLATE 'en-ci' ,
      rog VARCHAR(10)  COLLATE 'en-ci' ,
      price_area_id INTEGER,
      PA_name CHAR(1)  COLLATE 'en-ci' ,
      pricing_role CHAR(1)  COLLATE 'en-ci' ,
      OOB_gap_id CHAR(20)  COLLATE 'en-ci' ,
      DIV_PROMO_GRP_CD INTEGER,
      loc_common_retail_cd DECIMAL(5,0),
      vendor_name VARCHAR(40)  COLLATE 'en-ci' ,
      vend_nbr CHAR(6)  COLLATE 'en-ci' ,
      vend_sub_acct_nbr CHAR(3)  COLLATE 'en-ci' ,
      cost_area_id SMALLINT,
      Manuf CHAR(5)  COLLATE 'en-ci' ,
      upc_id DECIMAL(13,0),
      corp_item_cd DECIMAL(8,0),
      item_description VARCHAR(40)  COLLATE 'en-ci' ,
      DST CHAR(4)  COLLATE 'en-ci' ,
      FACILITY CHAR(4)  COLLATE 'en-ci' ,
      dst_stat CHAR(1)  COLLATE 'en-ci' ,
      rtl_stat CHAR(1)  COLLATE 'en-ci' ,
      buyer_nm CHAR(20)  COLLATE 'en-ci' ,
      vend_conv_fctr SMALLINT,
      t_pack_retail_qty DECIMAL(10,0),
      size_dsc CHAR(30)  COLLATE 'en-ci' ,
      Row_Offset CHAR(1)  COLLATE 'en-ci' ,
      UPC_13_Wk_Avg_Sales DECIMAL(18,2),
      UPC_13_Wk_Avg_Qty DECIMAL(18,3),
      UPC_13_Wk_Avg_RTL DECIMAL(18,3),
      T_RANK_BY_ROG_AND_CPC INTEGER,
      pct_ACV_Stores DECIMAL(5,2),
      t_CPC_13_Wk_Avg_Sales DECIMAL(18,2),
      t_CPC_13_Wk_Avg_Qty DECIMAL(18,3),
      t_CPC_13_Wk_Avg_RTL DECIMAL(18,3),
      PND_Cost_Change_VND DECIMAL(15,4),
      PND_VEN_Date_Effective DATE  comment '{"FORMAT":"YY/MM/DD"}',
      New_Recc_Reg_Retail DECIMAL(18,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      Vendor_Unit_Cost DECIMAL(10,3),
      Unit_Item_Billing_Cost DECIMAL(10,3),
      Prev_Retail_Price_Fctr DECIMAL(2,0),
      Previous_Retail_Price DECIMAL(7,2),
      Prev_Retail_Effective_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Pending_EDLP_Mult DECIMAL(2,0),
      Pending_EDLP_Retail DECIMAL(7,2),
      Pending_EDLP_Chg_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Pending_GP_pctg DECIMAL(15,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      Reg_Retail_Price_Fctr DECIMAL(2,0),
      Reg_Retail DECIMAL(10,2),
      t_price_Per DECIMAL(10,3),
      t_Unit VARCHAR(5)  COLLATE 'en-ci' ,
      Reg_GP_pctg DECIMAL(15,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      case_allow_count INTEGER,
      case_allow_amt DECIMAL(15,4),
      Case_Allow_amt_per_Unit DECIMAL(15,4),
      Case_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Case_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      S2S_Allow_count INTEGER,
      S2S_Allow_amt DECIMAL(15,4),
      S2S_Allow_amt_per_Unit DECIMAL(15,4),
      S2S_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      S2S_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Scan_Allow_count INTEGER,
      Scan_Allow_amt DECIMAL(15,4),
      Scan_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Scan_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Redem_Scan_Allow_count INTEGER,
      Redem_Allow_amt DECIMAL(15,4),
      Redem_Start_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Redem_End_Date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Total_Allow_Unit DECIMAL(15,4),
      t_Allowance_pctg DECIMAL(15,3),
      Net_Cost_with_Allow DECIMAL(18,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      Promo_Multiple DECIMAL(3,0),
      Promo_Price DECIMAL(10,4),
      Coupon_Method CHAR(2)  COLLATE 'en-ci' ,
      Min_Purch DECIMAL(2,0),
      Limit_Per_Txn SMALLINT,
      Promo_GP_pctg DECIMAL(15,3), --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      Net_Promo_Price DECIMAL(10,2),
      Price_Per DECIMAL(10,3),
      t2_Unit VARCHAR(5)  COLLATE 'en-ci' ,
      t_markdown_pctg DECIMAL(15,4),
      Mark_down DECIMAL(10,2),
      Promo_Start DATE  comment '{"FORMAT":"YY/MM/DD"}',
      Promo_End DATE  comment '{"FORMAT":"YY/MM/DD"}',
      pass_through DECIMAL(15,2),
      NEW_Multiple CHAR(1)  COLLATE 'en-ci' ,
      NEW_Retail CHAR(1)  COLLATE 'en-ci' ,
      NEW_EDLP_GP_pctg CHAR(1)  COLLATE 'en-ci' ,
      NEW_Promo_Multiple CHAR(1)  COLLATE 'en-ci' ,
      NEW_Promo_Retail CHAR(1)  COLLATE 'en-ci' ,
      NEW_Promo_GPpctg CHAR(1)  COLLATE 'en-ci' ,
      NEW_Passthrough CHAR(1)  COLLATE 'en-ci' ,
      compet_code VARCHAR(6)  COLLATE 'en-ci' ,
      price_chk_date DATE  comment '{"FORMAT":"YY/MM/DD"}',
      comp_reg_mult DECIMAL(2,0),
      com_reg_price DECIMAL(7,2),
      REG_CPI DECIMAL(18,3),  --CHARACTER SET UNICODE NOT CASESPECIFIC, MAYANK 20190430:changed column type from char(1)->decimal(18,3) for flattening the formulas
      COMP_AD_MULT DECIMAL(2,0),
      COMP_AD_PRICE DECIMAL(7,2),
      Comments CHAR(1)  COLLATE 'en-ci' ,
      Modified_flag CHAR(1)  COLLATE 'en-ci' ,
      ROG_and_CIG VARCHAR(15)  COLLATE 'en-ci' ,
      Allowance_Counts CHAR(50)  COLLATE 'en-ci' ,
      Report_Date DATE  comment '{"FORMAT":"YY/MM/DD"}');
-- CHAR_SET - Remove CHARACTER SET create table option
-- COLUMN_ATTRIBUTES - Remove not casespecific and replace with collate en-ci
-- COLUMN_ATTRIBUTES - move format into comment
-- CREATE_SET_OPTION - Remove SET create table option, please review in insert statement!
-- CREATE_TABLE_OPTION - remove create table options
-- TABLE_INDEX - Remove table index options
 
"""
 
sql_step57 = """
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_final
(       division_id
,       rpt_group
,       promo_no_allowance
,       allow_no_promo
,       missing_allowance
,       cost_change
,       ad_plan
,       t_markdown_pctg
,       less_10_Promo
,       t_Allowance_pctg
,       less_10_allowance
,       greater_100_pass_through
,       t_09_Retail
,       lead_item
,       Dominant_Price_Area
,       t_OOB
,       sskvi
,       group_cd
,       group_nm
,       SMIC
,       SMIC_name
,       rog
,       price_area_id
,       PA_name
,       pricing_role
,       OOB_gap_id
,       DIV_PROMO_GRP_CD
,       loc_common_retail_cd
,       vendor_name
,       vend_nbr
,       vend_sub_acct_nbr
,       cost_area_id
,       Manuf
,       upc_id
,       corp_item_cd
,       item_description
,       DST
,       FACILITY
,       dst_stat
,       rtl_stat
,       buyer_nm
,       vend_conv_fctr
,       t_pack_retail_qty
,       size_dsc
,       Row_Offset
,       UPC_13_Wk_Avg_Sales
,       UPC_13_Wk_Avg_Qty
,       UPC_13_Wk_Avg_RTL
,       T_RANK_BY_ROG_AND_CPC
,       pct_ACV_Stores
,       t_CPC_13_Wk_Avg_Sales
,       t_CPC_13_Wk_Avg_Qty
,       t_CPC_13_Wk_Avg_RTL
,       PND_Cost_Change_VND
,       PND_VEN_Date_Effective
,       New_Recc_Reg_Retail
,       Vendor_Unit_Cost
,       Unit_Item_Billing_Cost
,       Prev_Retail_Price_Fctr
,       Previous_Retail_Price
,       Prev_Retail_Effective_Date
,       Pending_EDLP_Mult
,       Pending_EDLP_Retail
,       Pending_EDLP_Chg_Date
,       Pending_GP_pctg
,       Reg_Retail_Price_Fctr
,       Reg_Retail
,       t_price_Per
,       t_Unit
,       Reg_GP_pctg
,       case_allow_count
,       case_allow_amt
,       Case_Allow_amt_per_Unit
,       Case_Start_Date
,       Case_End_Date
,       S2S_Allow_count
,       S2S_Allow_amt
,       S2S_Allow_amt_per_Unit
,       S2S_Start_Date
,       S2S_End_Date
,       Scan_Allow_count
,       Scan_Allow_amt
,       Scan_Start_Date
,       Scan_End_Date
,       Redem_Scan_Allow_count
,       Redem_Allow_amt
,       Redem_Start_Date
,       Redem_End_Date
,       Total_Allow_Unit
,       Net_Cost_with_Allow
,       Promo_Multiple
,       Promo_Price
,       Coupon_Method
,       Min_Purch
,       Limit_Per_Txn
,       Promo_GP_pctg
,       Net_Promo_Price
,       Price_Per
,       t2_Unit
,       Mark_down
,       Promo_Start
,       Promo_End
,       pass_through
,       NEW_Multiple
,       NEW_Retail
,       NEW_EDLP_GP_pctg
,       NEW_Promo_Multiple
,       NEW_Promo_Retail
,       NEW_Promo_GPpctg
,       NEW_Passthrough
,       compet_code
,       price_chk_date
,       comp_reg_mult
,       com_reg_price
,       REG_CPI
,       COMP_AD_MULT
,       COMP_AD_PRICE
,       Comments
,       Modified_flag
,       ROG_and_CIG
,       Allowance_Counts
,       Report_Date
)
SELECT  dsd.division_id
,       CASE
            WHEN dsd.division_id = 34 AND dsd.price_area_id IN (87, 3)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^ACME PAs 87 & 03'
            WHEN dsd.division_id = 33 AND dsd.price_area_id IN (4, 10, 23, 61)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^ASHA PAs 04,23,61 & 10'
            WHEN dsd.division_id = 32 AND dsd.price_area_id IN (1)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^AJWL PA 01'
            WHEN dsd.division_id = 30 AND dsd.price_area_id IN (2,4,6,8)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^AIMT PAs 02, 04, 06, 08 ONLY'
            WHEN dsd.division_id = 5 AND dsd.price_area_id IN (2,41, 51, 71)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^SDEN PAs 02, 41, 51, 71'
            WHEN dsd.division_id = 25 AND dsd.price_area_id IN (11,17,18,47)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^SHAW 47, SNCA 11, 17, 18'
            WHEN dsd.division_id = 29 AND dsd.price_area_id IN (1,3)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^VSOC 01 & 03, ASOCs 01 & 03'
            WHEN dsd.division_id = 19 AND dsd.price_area_id IN (1,69,3,75)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^SPRT 01, 69, 75 & APOR 03'
            WHEN dsd.division_id = 17 AND dsd.price_area_id IN (1,31,41,71)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^SPHO 01, APHO 31 & 41, ALAS 71'
            WHEN dsd.division_id = 35 AND dsd.price_area_id IN (40, 49, 53) AND dsd.group_cd > 1
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^(ALL GROUPS) - SEAS PAs 40, 49, 53'
-- MAYANK : 20190614 : changed the PA 33 to 73
            WHEN dsd.division_id = 27 AND dsd.group_cd IN (96,97) AND dsd.price_area_id IN (8,47,73,60)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^Beth & Randy' || '^SSEA 47 & 73, SSPK 60, & SACG 08'
            WHEN dsd.division_id = 27 AND dsd.group_cd IN (96,97) AND dsd.price_area_id NOT IN (8,47,73,60)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^Beth & Randy'
            WHEN dsd.division_id = 27 AND dsd.price_area_id IN (8,47,73,60)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^SSEA 47 & 73, SSPK 60, & SACG 08'
            WHEN dsd.division_id = 20 AND dsd.price_area_id IN (1,4,33,83)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^RDAL 83, ADAL 04, RHOU 33, AHOU 01'
-- MAYANK : 20190614 : adding the conditions for Haggen Master Files
            WHEN dsd.division_id = 24 AND dsd.price_area_id IN (53)
                THEN (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2)) || '^SHGN PA 53'
            ELSE    (translate(to_char(dsd.group_cd , '00'), ' .+-', '') ::CHAR(2))
        END AS rpt_group
,       CASE WHEN COALESCE(dsd.Net_Promo_Price, 0) > 0 AND (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0)) = 0 THEN 'Y' ELSE ' ' END AS promo_no_allowance
,       CASE WHEN COALESCE(dsd.S2S_Allow_amt, 0) + COALESCE(dsd.Scan_Allow_amt, 0) > 0 AND COALESCE(dsd.Net_Promo_Price, 0) = 0 THEN 'Y' ELSE ' ' END AS allow_no_promo
,       CASE WHEN malw.rog_and_cig IS NULL THEN ' ' ELSE 'Y' END AS missing_allowance
,       CASE WHEN dsd.PND_Cost_Change_VND IS NULL THEN ' ' ELSE 'Y' END AS cost_change
,       dsd.ad_plan
,       dsd.Mark_down / NULLIF(dsd.Reg_Retail,'0') / NULLIF(dsd.Reg_Retail_Price_Fctr,'0') AS t_markdown_pctg -- dsd.Markdown_pctg
,       CASE WHEN t_markdown_pctg > 0 AND t_markdown_pctg < 0.1 THEN 'Y' ELSE ' ' END AS less_10_Promo  -- dsd.less_10_Promo
,       ((COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0))/ NULLIF(dsd.Vendor_Unit_Cost,'0')) ::DECIMAL(15,3) AS t_Allowance_pctg -- dsd.Allowance_pctg    -- COLUMN CH
,       CASE WHEN t_Allowance_pctg > 0 AND t_Allowance_pctg < 0.1 THEN 'Y' ELSE ' ' END AS less_10_allowance  -- dsd.less_10_allowance
,       CASE WHEN (dsd.mark_down / NULLIF(COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0),'0')) /*(DECIMAL(5,2))*/ > 1 THEN 'Y' ELSE ' ' END AS greater_100_pass_through  --dsd.greater_100_pass_through
 
--,       dsd.t_09_Retail MAYANK : 20190426 : flattening the formulas in excel
,       CASE WHEN (round(dsd.Reg_Retail - floor(dsd.Reg_Retail),2) = 0.09) THEN 'Y' ELSE ' ' END AS t_09_Retail 
,       -- MAYANK : marking blank CIGS as Lead Item (28/03/2019)
        CASE 
            WHEN litm.division_id IS NOT NULL THEN 'Y'  
            WHEN dsd.div_promo_grp_cd = 0 THEN 'Y'
            --WHEN litm.division_id IS NULL THEN ' '
            ELSE ' '            
        END AS lead_item -- dsd.lead_item
,       dsd.Dominant_Price_Area
,       CASE WHEN dsd.manuf IN ('21130', '79893', '58200', '11535', '41303', '41130') THEN 'Y' ELSE ' ' END AS t_OOB  -- dsd.OOB
,       dsd.sskvi
,       dsd.group_cd
,       dsd.group_nm
,       dsd.SMIC
,       dsd.SMIC_name
,       CASE
            WHEN dsd.rog_cd = 'SEAS' THEN '1-SEAS'
            WHEN dsd.rog_cd = 'SEAG' THEN '2-SEAG'
            WHEN dsd.rog_cd = 'ACME' THEN '1-ACME'
            WHEN dsd.rog_cd = 'SDEN' THEN '1-SDEN'
            WHEN dsd.rog_cd = 'ADEN' THEN '2-ADEN'
            WHEN dsd.rog_cd = 'AIMT' THEN '1-AIMT'
            WHEN dsd.rog_cd = 'AJWL' THEN '1-AJWL'
            WHEN dsd.rog_cd = 'SNCA' THEN '1-SNCA'
            WHEN dsd.rog_cd = 'SHAW' THEN '2-SHAW'
            WHEN dsd.rog_cd = 'SPRT' THEN '1-SPRT'
            WHEN dsd.rog_cd = 'APOR' THEN '2-APOR'
            WHEN dsd.rog_cd = 'SSEA' THEN '1-SSEA'
            WHEN dsd.rog_cd = 'SSPK' THEN '2-SSPK'
            WHEN dsd.rog_cd = 'SACG' THEN '3-SACG'
            WHEN dsd.rog_cd = 'ASHA' THEN '1-ASHA'
            WHEN dsd.rog_cd = 'AVMT' THEN '2-AVMT'
            WHEN dsd.rog_cd = 'VSOC' THEN '1-VSOC'
            WHEN dsd.rog_cd = 'ASOC' THEN '2-ASOC'
            WHEN dsd.rog_cd = 'PSOC' THEN '3-PSOC'
            WHEN dsd.rog_cd = 'RDAL' THEN '1-RDAL'
            WHEN dsd.rog_cd = 'ADAL' THEN '2-ADAL'
            WHEN dsd.rog_cd = 'RHOU' THEN '3-RHOU'
            WHEN dsd.rog_cd = 'AHOU' THEN '4-AHOU'
            WHEN dsd.rog_cd = 'SPHO' THEN '1-SPHO'
            WHEN dsd.rog_cd = 'APHO' THEN '2-APHO'
            WHEN dsd.rog_cd = 'ALAS' THEN '3-ALAS'
            WHEN dsd.rog_cd = 'VLAS' THEN '4-VLAS'
            WHEN dsd.rog_cd = 'SPHX' THEN '5-SPHX'
            WHEN dsd.rog_cd = 'SHGN' THEN '1-SHGN'
            WHEN dsd.rog_cd = 'TEST' THEN '6-TEST'
            ELSE dsd.rog_cd
        END ::VARCHAR(10) AS rog
,       dsd.price_area_id               -- COLUMN S
,       dsd.PA_name                     -- COLUMN T
,       dsd.pricing_role
,       dsd.OOB_gap_id
,       dsd.DIV_PROMO_GRP_CD            -- COLUMN W CIG
,       dsd.loc_common_retail_cd        -- COLUMN X
,       dsd.vendor_name
,       dsd.vend_nbr
,       dsd.vend_sub_acct_nbr
,       dsd.cost_area_id
,       dsd.Manuf
,       dsd.upc_id
,       dsd.corp_item_cd
,       dsd.item_description
,       dsd.DST
,       dsd.FACILITY
,       dsd.dst_stat
,       dsd.rtl_stat
,       dsd.buyer_nm
,       dsd.vend_conv_fctr
,       dsd.t_pack_retail_qty
,       dsd.size_dsc
,       dsd.Row_Offset
,       sua.AVG_NET_SALES_13_WK     AS UPC_13_Wk_Avg_Sales           -- COLUMN: AP
,       sua.AVG_ITEM_QTY_13_WK      AS UPC_13_Wk_Avg_Qty
,       sua.AVG_NET_SALES_13_WK / NULLIF(sua.AVG_ITEM_QTY_13_WK,'0') AS UPC_13_Wk_Avg_RTL  -- dsd.UPC_13_Wk_Avg_RTL
,       CASE WHEN litm.division_id IS NULL THEN NULL ELSE 1 END ::INTEGER AS T_RANK_BY_ROG_AND_CPC
,       (sua.NUM_STORES_SELLING * 1.00) / (sua.NUM_STORES_IN_ROG * 1.00)  ::DECIMAL(5,2) AS pct_ACV_Stores
-- ,       COALESCE(cig.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK) AS t_CPC_13_Wk_Avg_Sales -- dsd.CPC_13_Wk_Avg_Sales   -- COLUMN: AU
-- ,       COALESCE(cig.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK) AS t_CPC_13_Wk_Avg_Qty  -- dsd.CPC_13_Wk_Avg_Qty
--      MAYANK : 20190625 : CIG sales cases for blank and non-blank CIGs
,       CASE 
            WHEN dsd.div_promo_grp_cd = 0 THEN COALESCE(cig_upc.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK)
            ELSE COALESCE(cig.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK)
        END AS t_CPC_13_Wk_Avg_Sales -- dsd.CPC_13_Wk_Avg_Sales   -- COLUMN: AU
,       CASE
            WHEN dsd.div_promo_grp_cd = 0 THEN COALESCE(cig_upc.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK)
            ELSE COALESCE(cig.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK)
        END AS t_CPC_13_Wk_Avg_Qty  -- dsd.t_CPC_13_Wk_Avg_Qty
,       t_CPC_13_Wk_Avg_Sales / NULLIF(t_CPC_13_Wk_Avg_Qty,'0')  AS t_CPC_13_Wk_Avg_RTL  --dsd.CPC_13_Wk_Avg_RTL
,       dsd.PND_Cost_Change_VND
,       dsd.PND_VEN_Date_Effective
--,       dsd.New_Recc_Reg_Retail MAYANK : 20190426 : flattening the formulas in excel
,       ((dsd.PND_Cost_Change_VND/NULLIF(dsd.Vendor_Unit_Cost,'0'))*dsd.Reg_Retail) ::DECIMAL(18,3) as New_Recc_Reg_Retail
,       dsd.Vendor_Unit_Cost
,       dsd.Unit_Item_Billing_Cost
,       dsd.Prev_Retail_Price_Fctr
,       dsd.Previous_Retail_Price
,       dsd.Prev_Retail_Effective_Date
,       dsd.Pending_EDLP_Mult
,       dsd.Pending_EDLP_Retail
,       dsd.Pending_EDLP_Chg_Date
--,       dsd.Pending_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,       (((dsd.Pending_EDLP_Retail/NULLIF(dsd.Pending_EDLP_Mult,'0'))-dsd.Unit_Item_Billing_Cost)/NULLIF(dsd.Pending_EDLP_Retail/NULLIF(dsd.Pending_EDLP_Mult,'0'),'0')) ::DECIMAL(15,3) as Pending_GP_pctg
,       dsd.Reg_Retail_Price_Fctr
,       dsd.Reg_Retail
,       dsd.t_price_Per
,       dsd.t_Unit
--,       dsd.Reg_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,       (((dsd.Reg_Retail/NULLIF(dsd.Reg_Retail_Price_Fctr,'0'))-dsd.Unit_Item_Billing_Cost)/NULLIF(dsd.Reg_Retail/NULLIF(dsd.Reg_Retail_Price_Fctr,'0'),'0')) ::DECIMAL(15,3) as Reg_GP_pctg
,       dsd.case_allow_count
,       dsd.case_allow_amt            AS case_allow_amt
,       dsd.Case_Allow_amt_per_Unit   AS Case_Allow_amt_per_Unit
,       dsd.Case_Start_Date
,       dsd.Case_End_Date
,       dsd.S2S_Allow_count
,       dsd.S2S_Allow_amt             AS S2S_Allow_amt
,       dsd.S2S_Allow_amt_per_Unit    AS S2S_Allow_amt_per_Unit
,       dsd.S2S_Start_Date
,       dsd.S2S_End_Date
,       dsd.Scan_Allow_count
,       dsd.Scan_Allow_amt            AS Scan_Allow_amt
,       dsd.Scan_Start_Date
,       dsd.Scan_End_Date
,       dsd.Redem_Scan_Allow_count
,       dsd.Redem_Allow_amt           AS Redem_Allow_amt
,       dsd.Redem_Start_Date
,       dsd.Redem_End_Date
,       COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0) AS Total_Allow_Unit
--,       dsd.Net_Cost_with_Allow MAYANK : 20190426 : flattening the formulas in excel
,       (dsd.Vendor_Unit_Cost - (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0))) ::DECIMAL(18,3) as Net_Cost_with_Allow
,       dsd.Promo_Multiple
,       dsd.Promo_Price
,       dsd.Coupon_Method
,       dsd.Min_Purch
,       dsd.Limit_Per_Txn
--,       dsd.Promo_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,       ((dsd.Net_Promo_Price - (dsd.Vendor_Unit_Cost - (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0))))/NULLIF(dsd.Net_Promo_Price,'0')) ::DECIMAL(15,3) as Promo_GP_pctg
,       dsd.Net_Promo_Price
,       dsd.Price_Per
,       dsd.t2_Unit
,       dsd.Mark_down
,       dsd.Promo_Start
,       dsd.Promo_End
,       (dsd.Mark_down/ NULLIF(COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0),'0')) ::DECIMAL(15,2) AS pass_through  -- dsd.Pass_Through  -- COLUMN CW
,       dsd.NEW_Multiple
,       dsd.NEW_Retail
,       dsd.NEW_EDLP_GP_pctg
,       dsd.NEW_Promo_Multiple
,       dsd.NEW_Promo_Retail
,       dsd.NEW_Promo_GPpctg
,       dsd.NEW_Passthrough
,       dsd.compet_code
,       dsd.price_chk_date
,       dsd.comp_reg_mult
,       dsd.com_reg_price
--,       dsd.REG_CPI MAYANK : 20190426 : flattening the formulas in excel
,       CASE WHEN com_reg_price>0 THEN 
        (dsd.Reg_Retail/GREATEST(dsd.Reg_Retail_Price_Fctr,1) - dsd.com_reg_price/GREATEST(dsd.comp_reg_mult,1))/NULLIF(dsd.Reg_Retail/ GREATEST(dsd.Reg_Retail_Price_Fctr,1),'0')
        ELSE NULL
        END ::DECIMAL(18,3) AS REG_CPI
,       dsd.COMP_AD_MULT
,       dsd.COMP_AD_PRICE
,       dsd.Comments
,       dsd.Modified_flag
,       dsd.ROG_and_CIG
,       dsd.Allowance_Counts
,       dsd.Report_Date
FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6 dsd
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales sua
ON      sua.rog_cd = dsd.rog_cd
AND     sua.upc_id = dsd.upc_id
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       upc_id
        ,       SUM(AVG_NET_SALES_13_WK) AS upc_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS upc_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales
        GROUP   BY 1,2
        ) upc
ON      upc.division_id = dsd.division_id
AND     upc.upc_id      = dsd.upc_id
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       DIV_PROMO_GRP_CD
        ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS cig_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales
        GROUP   BY 1,2
        ) cig
ON      cig.division_id = dsd.division_id
AND     cig.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
--      MAYANK : 20190625 : creating this table for matching the upc and cig sales for blank cigs.
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       upc_id
        ,       DIV_PROMO_GRP_CD
        ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS cig_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales
        GROUP   BY 1,2,3
        ) cig_upc
ON      cig_upc.division_id = dsd.division_id
AND     cig_upc.upc_id = dsd.upc_id
AND     cig_upc.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       group_id          -- MAYANK : 20190519 : Added group if for correcting the Lead Items in a group also in partition criteria
        ,       div_promo_grp_cd
        ,       upc_id
        FROM    (
                SELECT  division_id
                ,       group_id      -- MAYANK : 20190519 : Added group if for correcting the Lead Items in a group also in partition criteria
                ,       div_promo_grp_cd
                ,       upc_id
                ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
                
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales
WHERE   div_promo_grp_cd > 0
                
                GROUP   BY 1,2,3,4
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY division_id, group_id, div_promo_grp_cd ORDER by cig_sum_AVG_NET_SALES_13_WK DESC, upc_id ASC) = 1
        ) litm
ON      litm.division_id = dsd.division_id
AND     litm.group_id = dsd.group_cd      -- MAYANK : 20190519 : Added group if for correcting the Lead Items in a group also in partition criteria
AND     litm.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
AND     litm.upc_id = dsd.upc_id
LEFT    OUTER JOIN (
        SELECT  rog_and_cig
        ,       COUNT(DISTINCT allowance_counts) AS cnt
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6
        GROUP   BY rog_and_cig
        HAVING  cnt > 1
        ) malw
ON      malw.rog_and_cig = dsd.rog_and_cig;
-- EXPR_FORMAT - Convert expression FORMAT/CAST_AS_FORMAT to TO_CHAR/TO_DATE
-- FUN_CAST_OPTR - Reformat casting
-- NULLIFZERO: Translated NullifZero to nullif function
-- SEL_BODY_CLAUSES_ORDER - change the order of all the clauses of a select statement
-- MAYANK(20190425) : commented below lines to obtain group 1 reports
-- WHERE   dsd.group_cd > 1
-- OR      (dsd.division_id = 35 AND group_cd = 1)
```

## Step 58
```sql
-- Division 30, PA 19 special report
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_final
(       division_id
,       rpt_group
,       promo_no_allowance
,       allow_no_promo
,       missing_allowance
,       cost_change
,       ad_plan
,       t_markdown_pctg
,       less_10_Promo
,       t_Allowance_pctg
,       less_10_allowance
,       greater_100_pass_through
,       t_09_Retail
,       lead_item
,       Dominant_Price_Area
,       t_OOB
,       sskvi
,       group_cd
,       group_nm
,       SMIC
,       SMIC_name
,       rog
,       price_area_id
,       PA_name
,       pricing_role
,       OOB_gap_id
,       DIV_PROMO_GRP_CD
,       loc_common_retail_cd
,       vendor_name
,       vend_nbr
,       vend_sub_acct_nbr
,       cost_area_id
,       Manuf
,       upc_id
,       corp_item_cd
,       item_description
,       DST
,       FACILITY
,       dst_stat
,       rtl_stat
,       buyer_nm
,       vend_conv_fctr
,       t_pack_retail_qty
,       size_dsc
,       Row_Offset
,       UPC_13_Wk_Avg_Sales
,       UPC_13_Wk_Avg_Qty
,       UPC_13_Wk_Avg_RTL
,       T_RANK_BY_ROG_AND_CPC
,       pct_ACV_Stores
,       t_CPC_13_Wk_Avg_Sales
,       t_CPC_13_Wk_Avg_Qty
,       t_CPC_13_Wk_Avg_RTL
,       PND_Cost_Change_VND
,       PND_VEN_Date_Effective
,       New_Recc_Reg_Retail
,       Vendor_Unit_Cost
,       Unit_Item_Billing_Cost
,       Prev_Retail_Price_Fctr
,       Previous_Retail_Price
,       Prev_Retail_Effective_Date
,       Pending_EDLP_Mult
,       Pending_EDLP_Retail
,       Pending_EDLP_Chg_Date
,       Pending_GP_pctg
,       Reg_Retail_Price_Fctr
,       Reg_Retail
,       t_price_Per
,       t_Unit
,       Reg_GP_pctg
,       case_allow_count
,       case_allow_amt
,       Case_Allow_amt_per_Unit
,       Case_Start_Date
,       Case_End_Date
,       S2S_Allow_count
,       S2S_Allow_amt
,       S2S_Allow_amt_per_Unit
,       S2S_Start_Date
,       S2S_End_Date
,       Scan_Allow_count
,       Scan_Allow_amt
,       Scan_Start_Date
,       Scan_End_Date
,       Redem_Scan_Allow_count
,       Redem_Allow_amt
,       Redem_Start_Date
,       Redem_End_Date
,       Total_Allow_Unit
,       Net_Cost_with_Allow
,       Promo_Multiple
,       Promo_Price
,       Coupon_Method
,       Min_Purch
,       Limit_Per_Txn
,       Promo_GP_pctg
,       Net_Promo_Price
,       Price_Per
,       t2_Unit
,       Mark_down
,       Promo_Start
,       Promo_End
,       pass_through
,       NEW_Multiple
,       NEW_Retail
,       NEW_EDLP_GP_pctg
,       NEW_Promo_Multiple
,       NEW_Promo_Retail
,       NEW_Promo_GPpctg
,       NEW_Passthrough
,       compet_code
,       price_chk_date
,       comp_reg_mult
,       com_reg_price
,       REG_CPI
,       COMP_AD_MULT
,       COMP_AD_PRICE
,       Comments
,       Modified_flag
,       ROG_and_CIG
,       Allowance_Counts
,       Report_Date
)
SELECT  dsd.division_id
,       'AIMT PA 19 ONLY' AS rpt_group
,       CASE WHEN COALESCE(dsd.Net_Promo_Price, 0) > 0 AND (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0)) = 0 THEN 'Y' ELSE ' ' END AS promo_no_allowance
,       CASE WHEN COALESCE(dsd.S2S_Allow_amt, 0) + COALESCE(dsd.Scan_Allow_amt, 0) > 0 AND COALESCE(dsd.Net_Promo_Price, 0) = 0 THEN 'Y' ELSE ' ' END AS allow_no_promo
,       CASE WHEN malw.rog_and_cig IS NULL THEN ' ' ELSE 'Y' END AS missing_allowance
,       CASE WHEN dsd.PND_Cost_Change_VND IS NULL THEN ' ' ELSE 'Y' END AS cost_change
,       dsd.ad_plan
,       dsd.Mark_down / NULLIF(dsd.reg_retail,'0') / NULLIF(dsd.Reg_Retail_Price_Fctr,'0') AS t_markdown_pctg -- dsd.Markdown_pctg
,       CASE WHEN t_markdown_pctg > 0 AND t_markdown_pctg < 0.1 THEN 'Y' ELSE ' ' END AS less_10_Promo  -- dsd.less_10_Promo
,       ((COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0)) / NULLIF(dsd.Vendor_Unit_Cost,'0')) ::DECIMAL(15,3) AS t_Allowance_pctg -- dsd.Allowance_pctg    -- COLUMN CH
,       CASE WHEN t_Allowance_pctg > 0 AND t_Allowance_pctg < 0.1 THEN 'Y' ELSE ' ' END AS less_10_allowance  -- dsd.less_10_allowance
,       CASE WHEN (dsd.mark_down / NULLIF(COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0),'0')) ::DECIMAL(5,2) > 1 THEN 'Y' ELSE ' ' END AS greater_100_pass_through  --dsd.greater_100_pass_through
 
--,       dsd.t_09_Retail MAYANK : 20190426 : flattening the formulas in excel
,            CASE WHEN (round(dsd.Reg_Retail - floor(dsd.Reg_Retail),2) = 0.09) THEN 'Y' ELSE ' ' END AS t_09_Retail
,       -- MAYANK : marking blank CIGS as Lead Item (28/03/2019)
        CASE 
            WHEN litm.division_id IS NOT NULL THEN 'Y'  
            WHEN dsd.div_promo_grp_cd = 0 THEN 'Y'
            --WHEN litm.division_id IS NULL THEN ' '
            ELSE ' '            
        END AS lead_item -- dsd.lead_item
,       dsd.Dominant_Price_Area
,       CASE WHEN dsd.manuf IN ('21130', '79893', '58200', '11535', '41303', '41130') THEN 'Y' ELSE ' ' END AS t_OOB  -- dsd.OOB
,       dsd.sskvi
,       dsd.group_cd
,       dsd.group_nm
,       dsd.SMIC
,       dsd.SMIC_name
,       CASE
            WHEN dsd.rog_cd = 'SEAS' THEN '1-SEAS'
            WHEN dsd.rog_cd = 'SEAG' THEN '2-SEAG'
            WHEN dsd.rog_cd = 'ACME' THEN '1-ACME'
            WHEN dsd.rog_cd = 'SDEN' THEN '1-SDEN'
            WHEN dsd.rog_cd = 'ADEN' THEN '2-ADEN'
            WHEN dsd.rog_cd = 'AIMT' THEN '1-AIMT'
            WHEN dsd.rog_cd = 'AJWL' THEN '1-AJWL'
            WHEN dsd.rog_cd = 'SNCA' THEN '1-SNCA'
            WHEN dsd.rog_cd = 'SHAW' THEN '2-SHAW'
            WHEN dsd.rog_cd = 'SPRT' THEN '1-SPRT'
            WHEN dsd.rog_cd = 'APOR' THEN '2-APOR'
            WHEN dsd.rog_cd = 'SSEA' THEN '1-SSEA'
            WHEN dsd.rog_cd = 'SSPK' THEN '2-SSPK'
            WHEN dsd.rog_cd = 'SACG' THEN '3-SACG'
            WHEN dsd.rog_cd = 'ASHA' THEN '1-ASHA'
            WHEN dsd.rog_cd = 'AVMT' THEN '2-AVMT'
            WHEN dsd.rog_cd = 'VSOC' THEN '1-VSOC'
            WHEN dsd.rog_cd = 'ASOC' THEN '2-ASOC'
            WHEN dsd.rog_cd = 'PSOC' THEN '3-PSOC'
            WHEN dsd.rog_cd = 'RDAL' THEN '1-RDAL'
            WHEN dsd.rog_cd = 'ADAL' THEN '2-ADAL'
            WHEN dsd.rog_cd = 'RHOU' THEN '3-RHOU'
            WHEN dsd.rog_cd = 'AHOU' THEN '4-AHOU'
            WHEN dsd.rog_cd = 'SPHO' THEN '1-SPHO'
            WHEN dsd.rog_cd = 'APHO' THEN '2-APHO'
            WHEN dsd.rog_cd = 'ALAS' THEN '3-ALAS'
            WHEN dsd.rog_cd = 'VLAS' THEN '4-VLAS'
            WHEN dsd.rog_cd = 'SPHX' THEN '5-SPHX'
            WHEN dsd.rog_cd = 'TEST' THEN '6-TEST'
            ELSE dsd.rog_cd
        END ::VARCHAR(10) AS rog
,       dsd.price_area_id               -- COLUMN S
,       dsd.PA_name                     -- COLUMN T
,       dsd.pricing_role
,       dsd.OOB_gap_id
,       dsd.DIV_PROMO_GRP_CD            -- COLUMN W CIG
,       dsd.loc_common_retail_cd        -- COLUMN X
,       dsd.vendor_name
,       dsd.vend_nbr
,       dsd.vend_sub_acct_nbr
,       dsd.cost_area_id
,       dsd.Manuf
,       dsd.upc_id
,       dsd.corp_item_cd
,       dsd.item_description
,       dsd.DST
,       dsd.FACILITY
,       dsd.dst_stat
,       dsd.rtl_stat
,       dsd.buyer_nm
,       dsd.vend_conv_fctr
,       dsd.t_pack_retail_qty
,       dsd.size_dsc
,       dsd.Row_Offset
,       sua.AVG_NET_SALES_13_WK     AS UPC_13_Wk_Avg_Sales           -- COLUMN: AP
,       sua.AVG_ITEM_QTY_13_WK      AS UPC_13_Wk_Avg_Qty
,       sua.AVG_NET_SALES_13_WK / NULLIF(sua.AVG_ITEM_QTY_13_WK,'0') AS UPC_13_Wk_Avg_RTL  -- dsd.UPC_13_Wk_Avg_RTL
,       CASE WHEN litm.division_id IS NULL THEN NULL ELSE 1 END ::INTEGER AS T_RANK_BY_ROG_AND_CPC
,       (sua.NUM_STORES_SELLING * 1.00) / (sua.NUM_STORES_IN_ROG * 1.00) * 100 ::DECIMAL(5,2) AS pct_ACV_Stores
--,       COALESCE(cig.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK) AS t_CPC_13_Wk_Avg_Sales -- dsd.CPC_13_Wk_Avg_Sales   -- COLUMN: AU
,        CASE 
            WHEN dsd.div_promo_grp_cd = 0 THEN COALESCE(cig_upc.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK)
            ELSE COALESCE(cig.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK)
        END AS t_CPC_13_Wk_Avg_Sales -- dsd.CPC_13_Wk_Avg_Sales   -- COLUMN: AU
--,       COALESCE(cig.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK) AS t_CPC_13_Wk_Avg_Qty  -- dsd.CPC_13_Wk_Avg_Qty
,       CASE
            WHEN dsd.div_promo_grp_cd = 0 THEN COALESCE(cig_upc.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK)
            ELSE COALESCE(cig.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK)
        END AS t_CPC_13_Wk_Avg_Qty  -- dsd.t_CPC_13_Wk_Avg_Qty
,       t_CPC_13_Wk_Avg_Sales / NULLIF(t_CPC_13_Wk_Avg_Qty,'0')  AS t_CPC_13_Wk_Avg_RTL  --dsd.CPC_13_Wk_Avg_RTL
,       dsd.PND_Cost_Change_VND
,       dsd.PND_VEN_Date_Effective
--,       dsd.New_Recc_Reg_Retail MAYANK : 20190426 : flattening the formulas in excel
,            (dsd.PND_Cost_Change_VND/NULLIF(dsd.Vendor_Unit_Cost,'0'))*dsd.Reg_Retail ::DECIMAL(18,3) as New_Recc_Reg_Retail
,       dsd.Vendor_Unit_Cost
,       dsd.Unit_Item_Billing_Cost
,       dsd.Prev_Retail_Price_Fctr
,       dsd.Previous_Retail_Price
,       dsd.Prev_Retail_Effective_Date
,       dsd.Pending_EDLP_Mult
,       dsd.Pending_EDLP_Retail
,       dsd.Pending_EDLP_Chg_Date
--,       dsd.Pending_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,            ((dsd.Pending_EDLP_Retail/NULLIF(dsd.Pending_EDLP_Mult,'0'))-dsd.Unit_Item_Billing_Cost)/NULLIF((dsd.Pending_EDLP_Retail/NULLIF(dsd.Pending_EDLP_Mult,'0')),'0') ::DECIMAL(15,3) as Pending_GP_pctg
,       dsd.Reg_Retail_Price_Fctr
,       dsd.Reg_Retail
,       dsd.t_price_Per
,       dsd.t_Unit
--,       dsd.Reg_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,            ((dsd.Reg_Retail/NULLIF(dsd.Reg_Retail_Price_Fctr,'0'))-dsd.Unit_Item_Billing_Cost)/NULLIF((dsd.Reg_Retail/NULLIF(dsd.Reg_Retail_Price_Fctr,'0')),'0') ::DECIMAL(15,3) as Reg_GP_pctg
,       dsd.case_allow_count
,       dsd.case_allow_amt            AS case_allow_amt
,       dsd.Case_Allow_amt_per_Unit   AS Case_Allow_amt_per_Unit
,       dsd.Case_Start_Date
,       dsd.Case_End_Date
,       dsd.S2S_Allow_count
,       dsd.S2S_Allow_amt             AS S2S_Allow_amt
,       dsd.S2S_Allow_amt_per_Unit    AS S2S_Allow_amt_per_Unit
,       dsd.S2S_Start_Date
,       dsd.S2S_End_Date
,       dsd.Scan_Allow_count
,       dsd.Scan_Allow_amt            AS Scan_Allow_amt
,       dsd.Scan_Start_Date
,       dsd.Scan_End_Date
,       dsd.Redem_Scan_Allow_count
,       dsd.Redem_Allow_amt           AS Redem_Allow_amt
,       dsd.Redem_Start_Date
,       dsd.Redem_End_Date
,       COALESCE(dsd.Case_Allow_amt_per_Unit, 0) +  COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) +  COALESCE(dsd.Scan_Allow_amt, 0) AS Total_Allow_Unit
--,       dsd.Net_Cost_with_Allow MAYANK : 20190426 : flattening the formulas in excel
,            dsd.Vendor_Unit_Cost - (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0)) ::DECIMAL(18,3) as Net_Cost_with_Allow
,       dsd.Promo_Multiple
,       dsd.Promo_Price
,       dsd.Coupon_Method
,       dsd.Min_Purch
,       dsd.Limit_Per_Txn
--,       dsd.Promo_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,            ((dsd.Net_Promo_Price - (dsd.Vendor_Unit_Cost - (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0))))/NULLIF(dsd.Net_Promo_Price,'0')) ::DECIMAL(15,3) as Promo_GP_pctg
,       dsd.Net_Promo_Price
,       dsd.Price_Per
,       dsd.t2_Unit
,       dsd.Mark_down
,       dsd.Promo_Start
,       dsd.Promo_End
,       (dsd.mark_down / NULLIF(COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0),'0')) ::DECIMAL(5,2) AS pass_through  -- dsd.Pass_Through  -- COLUMN CW
,       dsd.NEW_Multiple
,       dsd.NEW_Retail
,       dsd.NEW_EDLP_GP_pctg
,       dsd.NEW_Promo_Multiple
,       dsd.NEW_Promo_Retail
,       dsd.NEW_Promo_GPpctg
,       dsd.NEW_Passthrough
,       dsd.compet_code
,       dsd.price_chk_date
,       dsd.comp_reg_mult
,       dsd.com_reg_price
--,       dsd.REG_CPI MAYANK : 20190426 : flattening the formulas in excel
    ,        CASE WHEN com_reg_price>0 THEN 
            (dsd.Reg_Retail/GREATEST(dsd.Reg_Retail_Price_Fctr,1) - dsd.com_reg_price/GREATEST(dsd.comp_reg_mult,1))/NULLIF(dsd.Reg_Retail/ GREATEST(dsd.Reg_Retail_Price_Fctr,1),'0')
            ELSE NULL
            END ::DECIMAL(18,3) AS REG_CPI
,       dsd.COMP_AD_MULT
,       dsd.COMP_AD_PRICE
,       dsd.Comments
,       dsd.Modified_flag
,       dsd.ROG_and_CIG
,       dsd.Allowance_Counts
,       dsd.Report_Date
FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6 dsd
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases sua
ON      sua.rog_cd = dsd.rog_cd
AND     sua.upc_id = dsd.upc_id
AND     sua.rpt_group = 'S30_02'
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       upc_id
        ,       SUM(AVG_NET_SALES_13_WK) AS upc_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS upc_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
        WHERE   rpt_group = 'S30_02'
        GROUP   BY 1,2
        ) upc
ON      upc.division_id = dsd.division_id
AND     upc.upc_id      = dsd.upc_id
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       DIV_PROMO_GRP_CD
        ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS cig_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
        WHERE   rpt_group = 'S30_02'
        GROUP   BY 1,2
        ) cig
ON      cig.division_id = dsd.division_id
AND     cig.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
--      MAYANK : 20190625 : creating this table for matching the upc and cig sales for blank cigs.
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       upc_id
        ,       DIV_PROMO_GRP_CD
        ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS cig_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
        GROUP   BY 1,2,3
        ) cig_upc
ON      cig_upc.division_id = dsd.division_id
AND     cig_upc.upc_id = dsd.upc_id
AND     cig_upc.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       div_promo_grp_cd
        ,       upc_id
        FROM    (
                SELECT  division_id
                ,       div_promo_grp_cd
                ,       upc_id
                ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
                WHERE   div_promo_grp_cd > 0
                AND     rpt_group = 'S30_02'
                GROUP   BY 1,2,3
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY division_id, div_promo_grp_cd ORDER by cig_sum_AVG_NET_SALES_13_WK DESC, upc_id ASC) = 1
        ) litm
ON      litm.division_id = dsd.division_id
AND     litm.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
AND     litm.upc_id = dsd.upc_id
LEFT    OUTER JOIN (
        SELECT  rog_and_cig
        ,       COUNT(DISTINCT allowance_counts) AS cnt
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6
        GROUP   BY rog_and_cig
        HAVING  cnt > 1
        ) malw
ON      malw.rog_and_cig = dsd.rog_and_cig
WHERE   dsd.division_id = 30
and     dsd.price_area_id = 19;
-- FUN_CAST_OPTR - Reformat casting
-- NULLIFZERO: Translated NullifZero to nullif function
-- MAYANK(20190425) : commented below lines to obtain group 1 reports
-- AND     dsd.group_cd > 1;
```

## Step 59
```sql
sql_step59 = """
-- Division 30, PA 13 & 14 special report
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_final
(       division_id
,       rpt_group
,       promo_no_allowance
,       allow_no_promo
,       missing_allowance
,       cost_change
,       ad_plan
,       t_markdown_pctg
,       less_10_Promo
,       t_Allowance_pctg
,       less_10_allowance
,       greater_100_pass_through
,       t_09_Retail
,       lead_item
,       Dominant_Price_Area
,       t_OOB
,       sskvi
,       group_cd
,       group_nm
,       SMIC
,       SMIC_name
,       rog
,       price_area_id
,       PA_name
,       pricing_role
,       OOB_gap_id
,       DIV_PROMO_GRP_CD
,       loc_common_retail_cd
,       vendor_name
,       vend_nbr
,       vend_sub_acct_nbr
,       cost_area_id
,       Manuf
,       upc_id
,       corp_item_cd
,       item_description
,       DST
,       FACILITY
,       dst_stat
,       rtl_stat
,       buyer_nm
,       vend_conv_fctr
,       t_pack_retail_qty
,       size_dsc
,       Row_Offset
,       UPC_13_Wk_Avg_Sales
,       UPC_13_Wk_Avg_Qty
,       UPC_13_Wk_Avg_RTL
,       T_RANK_BY_ROG_AND_CPC
,       pct_ACV_Stores
,       t_CPC_13_Wk_Avg_Sales
,       t_CPC_13_Wk_Avg_Qty
,       t_CPC_13_Wk_Avg_RTL
,       PND_Cost_Change_VND
,       PND_VEN_Date_Effective
,       New_Recc_Reg_Retail
,       Vendor_Unit_Cost
,       Unit_Item_Billing_Cost
,       Prev_Retail_Price_Fctr
,       Previous_Retail_Price
,       Prev_Retail_Effective_Date
,       Pending_EDLP_Mult
,       Pending_EDLP_Retail
,       Pending_EDLP_Chg_Date
,       Pending_GP_pctg
,       Reg_Retail_Price_Fctr
,       Reg_Retail
,       t_price_Per
,       t_Unit
,       Reg_GP_pctg
,       case_allow_count
,       case_allow_amt
,       Case_Allow_amt_per_Unit
,       Case_Start_Date
,       Case_End_Date
,       S2S_Allow_count
,       S2S_Allow_amt
,       S2S_Allow_amt_per_Unit
,       S2S_Start_Date
,       S2S_End_Date
,       Scan_Allow_count
,       Scan_Allow_amt
,       Scan_Start_Date
,       Scan_End_Date
,       Redem_Scan_Allow_count
,       Redem_Allow_amt
,       Redem_Start_Date
,       Redem_End_Date
,       Total_Allow_Unit
,       Net_Cost_with_Allow
,       Promo_Multiple
,       Promo_Price
,       Coupon_Method
,       Min_Purch
,       Limit_Per_Txn
,       Promo_GP_pctg
,       Net_Promo_Price
,       Price_Per
,       t2_Unit
,       Mark_down
,       Promo_Start
,       Promo_End
,       pass_through
,       NEW_Multiple
,       NEW_Retail
,       NEW_EDLP_GP_pctg
,       NEW_Promo_Multiple
,       NEW_Promo_Retail
,       NEW_Promo_GPpctg
,       NEW_Passthrough
,       compet_code
,       price_chk_date
,       comp_reg_mult
,       com_reg_price
,       REG_CPI
,       COMP_AD_MULT
,       COMP_AD_PRICE
,       Comments
,       Modified_flag
,       ROG_and_CIG
,       Allowance_Counts
,       Report_Date
)
SELECT  dsd.division_id
,       'AIMT PAs 13 & 14 ONLY' AS rpt_group
,       CASE WHEN COALESCE(dsd.Net_Promo_Price, 0) > 0 AND (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0)) = 0 THEN 'Y' ELSE ' ' END AS promo_no_allowance
,       CASE WHEN COALESCE(dsd.S2S_Allow_amt, 0) + COALESCE(dsd.Scan_Allow_amt, 0) > 0 AND COALESCE(dsd.Net_Promo_Price, 0) = 0 THEN 'Y' ELSE ' ' END AS allow_no_promo
,       CASE WHEN malw.rog_and_cig IS NULL THEN ' ' ELSE 'Y' END AS missing_allowance
,       CASE WHEN dsd.PND_Cost_Change_VND IS NULL THEN ' ' ELSE 'Y' END AS cost_change
,       dsd.ad_plan
,       dsd.Mark_down / NULLIF(dsd.reg_retail,'0') / NULLIF(dsd.Reg_Retail_Price_Fctr,'0') AS t_markdown_pctg -- dsd.Markdown_pctg
,       CASE WHEN t_markdown_pctg > 0 AND t_markdown_pctg < 0.1 THEN 'Y' ELSE ' ' END AS less_10_Promo  -- dsd.less_10_Promo
,       ((COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0)) / NULLIF(dsd.Vendor_Unit_Cost,'0')) ::DECIMAL(15,3) AS t_Allowance_pctg -- dsd.Allowance_pctg    -- COLUMN CH
,       CASE WHEN t_Allowance_pctg > 0 AND t_Allowance_pctg < 0.1 THEN 'Y' ELSE ' ' END AS less_10_allowance  -- dsd.less_10_allowance
,       CASE WHEN (dsd.mark_down / NULLIF(COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0),'0')) ::DECIMAL(5,2) > 1 THEN 'Y' ELSE ' ' END AS greater_100_pass_through  --dsd.greater_100_pass_through
 
--,       dsd.t_09_Retail MAYANK : 20190426 : flattening the formulas in excel
,            CASE WHEN (round(dsd.Reg_Retail - floor(dsd.Reg_Retail),2) = 0.09) THEN 'Y' ELSE ' ' END AS t_09_Retail 
,       -- MAYANK : marking blank CIGS as Lead Item (28/03/2019)
        CASE 
            WHEN litm.division_id IS NOT NULL THEN 'Y'  
            WHEN dsd.div_promo_grp_cd = 0 THEN 'Y'
            --WHEN litm.division_id IS NULL THEN ' '
            ELSE ' '            
        END AS lead_item -- dsd.lead_item
,       dsd.Dominant_Price_Area
,       CASE WHEN dsd.manuf IN ('21130', '79893', '58200', '11535', '41303', '41130') THEN 'Y' ELSE ' ' END AS t_OOB  -- dsd.OOB
,       dsd.sskvi
,       dsd.group_cd
,       dsd.group_nm
,       dsd.SMIC
,       dsd.SMIC_name
,       CASE
            WHEN dsd.rog_cd = 'SEAS' THEN '1-SEAS'
            WHEN dsd.rog_cd = 'SEAG' THEN '2-SEAG'
            WHEN dsd.rog_cd = 'ACME' THEN '1-ACME'
            WHEN dsd.rog_cd = 'SDEN' THEN '1-SDEN'
            WHEN dsd.rog_cd = 'ADEN' THEN '2-ADEN'
            WHEN dsd.rog_cd = 'AIMT' THEN '1-AIMT'
            WHEN dsd.rog_cd = 'AJWL' THEN '1-AJWL'
            WHEN dsd.rog_cd = 'SNCA' THEN '1-SNCA'
            WHEN dsd.rog_cd = 'SHAW' THEN '2-SHAW'
            WHEN dsd.rog_cd = 'SPRT' THEN '1-SPRT'
            WHEN dsd.rog_cd = 'APOR' THEN '2-APOR'
            WHEN dsd.rog_cd = 'SSEA' THEN '1-SSEA'
            WHEN dsd.rog_cd = 'SSPK' THEN '2-SSPK'
            WHEN dsd.rog_cd = 'SACG' THEN '3-SACG'
            WHEN dsd.rog_cd = 'ASHA' THEN '1-ASHA'
            WHEN dsd.rog_cd = 'AVMT' THEN '2-AVMT'
            WHEN dsd.rog_cd = 'VSOC' THEN '1-VSOC'
            WHEN dsd.rog_cd = 'ASOC' THEN '2-ASOC'
            WHEN dsd.rog_cd = 'PSOC' THEN '3-PSOC'
            WHEN dsd.rog_cd = 'RDAL' THEN '1-RDAL'
            WHEN dsd.rog_cd = 'ADAL' THEN '2-ADAL'
            WHEN dsd.rog_cd = 'RHOU' THEN '3-RHOU'
            WHEN dsd.rog_cd = 'AHOU' THEN '4-AHOU'
            WHEN dsd.rog_cd = 'SPHO' THEN '1-SPHO'
            WHEN dsd.rog_cd = 'APHO' THEN '2-APHO'
            WHEN dsd.rog_cd = 'ALAS' THEN '3-ALAS'
            WHEN dsd.rog_cd = 'VLAS' THEN '4-VLAS'
            WHEN dsd.rog_cd = 'SPHX' THEN '5-SPHX'
            WHEN dsd.rog_cd = 'TEST' THEN '6-TEST'
            ELSE dsd.rog_cd
        END ::VARCHAR(10) AS rog
,       dsd.price_area_id               -- COLUMN S
,       dsd.PA_name                     -- COLUMN T
,       dsd.pricing_role
,       dsd.OOB_gap_id
,       dsd.DIV_PROMO_GRP_CD            -- COLUMN W CIG
,       dsd.loc_common_retail_cd        -- COLUMN X
,       dsd.vendor_name
,       dsd.vend_nbr
,       dsd.vend_sub_acct_nbr
,       dsd.cost_area_id
,       dsd.Manuf
,       dsd.upc_id
,       dsd.corp_item_cd
,       dsd.item_description
,       dsd.DST
,       dsd.FACILITY
,       dsd.dst_stat
,       dsd.rtl_stat
,       dsd.buyer_nm
,       dsd.vend_conv_fctr
,       dsd.t_pack_retail_qty
,       dsd.size_dsc
,       dsd.Row_Offset
,       sua.AVG_NET_SALES_13_WK     AS UPC_13_Wk_Avg_Sales           -- COLUMN: AP
,       sua.AVG_ITEM_QTY_13_WK      AS UPC_13_Wk_Avg_Qty
,       sua.AVG_NET_SALES_13_WK / NULLIF(sua.AVG_ITEM_QTY_13_WK,'0') AS UPC_13_Wk_Avg_RTL  -- dsd.UPC_13_Wk_Avg_RTL
,       CASE WHEN litm.division_id IS NULL THEN NULL ELSE 1 END ::INTEGER AS T_RANK_BY_ROG_AND_CPC
,       (sua.NUM_STORES_SELLING * 1.00) / (sua.NUM_STORES_IN_ROG * 1.00) * 100 ::DECIMAL(5,2) AS pct_ACV_Stores
--,       COALESCE(cig.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK) AS t_CPC_13_Wk_Avg_Sales -- dsd.CPC_13_Wk_Avg_Sales   -- COLUMN: AU
,        CASE 
            WHEN dsd.div_promo_grp_cd = 0 THEN COALESCE(cig_upc.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK)
            ELSE COALESCE(cig.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK)
        END AS t_CPC_13_Wk_Avg_Sales -- dsd.CPC_13_Wk_Avg_Sales   -- COLUMN: AU
--,       COALESCE(cig.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK) AS t_CPC_13_Wk_Avg_Qty  -- dsd.CPC_13_Wk_Avg_Qty
,       CASE
            WHEN dsd.div_promo_grp_cd = 0 THEN COALESCE(cig_upc.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK)
            ELSE COALESCE(cig.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK)
        END AS t_CPC_13_Wk_Avg_Qty  -- dsd.t_CPC_13_Wk_Avg_Qty
,       t_CPC_13_Wk_Avg_Sales / NULLIF(t_CPC_13_Wk_Avg_Qty,'0')  AS t_CPC_13_Wk_Avg_RTL  --dsd.CPC_13_Wk_Avg_RTL
,       dsd.PND_Cost_Change_VND
,       dsd.PND_VEN_Date_Effective
--,       dsd.New_Recc_Reg_Retail MAYANK : 20190426 : flattening the formulas in excel
,            (dsd.PND_Cost_Change_VND/NULLIF(dsd.Vendor_Unit_Cost,'0'))*dsd.Reg_Retail ::DECIMAL(18,3) as New_Recc_Reg_Retail
,       dsd.Vendor_Unit_Cost
,       dsd.Unit_Item_Billing_Cost
,       dsd.Prev_Retail_Price_Fctr
,       dsd.Previous_Retail_Price
,       dsd.Prev_Retail_Effective_Date
,       dsd.Pending_EDLP_Mult
,       dsd.Pending_EDLP_Retail
,       dsd.Pending_EDLP_Chg_Date
--,       dsd.Pending_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,            ((dsd.Pending_EDLP_Retail/NULLIF(dsd.Pending_EDLP_Mult,'0'))-dsd.Unit_Item_Billing_Cost)/NULLIF((dsd.Pending_EDLP_Retail/NULLIF(dsd.Pending_EDLP_Mult,'0')),'0') ::DECIMAL(15,3) as Pending_GP_pctg
,       dsd.Reg_Retail_Price_Fctr
,       dsd.Reg_Retail
,       dsd.t_price_Per
,       dsd.t_Unit
--,       dsd.Reg_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,            ((dsd.Reg_Retail/NULLIF(dsd.Reg_Retail_Price_Fctr,'0'))-dsd.Unit_Item_Billing_Cost)/NULLIF((dsd.Reg_Retail/NULLIF(dsd.Reg_Retail_Price_Fctr,'0')),'0') ::DECIMAL(15,3) as Reg_GP_pctg
,       dsd.case_allow_count
,       dsd.case_allow_amt            AS case_allow_amt
,       dsd.Case_Allow_amt_per_Unit   AS Case_Allow_amt_per_Unit
,       dsd.Case_Start_Date
,       dsd.Case_End_Date
,       dsd.S2S_Allow_count
,       dsd.S2S_Allow_amt             AS S2S_Allow_amt
,       dsd.S2S_Allow_amt_per_Unit    AS S2S_Allow_amt_per_Unit
,       dsd.S2S_Start_Date
,       dsd.S2S_End_Date
,       dsd.Scan_Allow_count
,       dsd.Scan_Allow_amt            AS Scan_Allow_amt
,       dsd.Scan_Start_Date
,       dsd.Scan_End_Date
,       dsd.Redem_Scan_Allow_count
,       dsd.Redem_Allow_amt           AS Redem_Allow_amt
,       dsd.Redem_Start_Date
,       dsd.Redem_End_Date
,       COALESCE(dsd.Case_Allow_amt_per_Unit, 0) +  COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) +  COALESCE(dsd.Scan_Allow_amt, 0) AS Total_Allow_Unit
--,       dsd.Net_Cost_with_Allow MAYANK : 20190426 : flattening the formulas in excel
,        (dsd.Vendor_Unit_Cost - (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0))) ::DECIMAL(18,3) as Net_Cost_with_Allow
,       dsd.Promo_Multiple
,       dsd.Promo_Price
,       dsd.Coupon_Method
,       dsd.Min_Purch
,       dsd.Limit_Per_Txn
--,       dsd.Promo_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,        ((dsd.Net_Promo_Price - (dsd.Vendor_Unit_Cost - (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0))))/NULLIF(dsd.Net_Promo_Price,'0')) ::DECIMAL(15,3) as Promo_GP_pctg
,       dsd.Net_Promo_Price
,       dsd.Price_Per
,       dsd.t2_Unit
,       dsd.Mark_down
,       dsd.Promo_Start
,       dsd.Promo_End
,       (dsd.mark_down / NULLIF(COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0),'0')) ::DECIMAL(5,2) AS pass_through  -- dsd.Pass_Through  -- COLUMN CW
,       dsd.NEW_Multiple
,       dsd.NEW_Retail
,       dsd.NEW_EDLP_GP_pctg
,       dsd.NEW_Promo_Multiple
,       dsd.NEW_Promo_Retail
,       dsd.NEW_Promo_GPpctg
,       dsd.NEW_Passthrough
,       dsd.compet_code
,       dsd.price_chk_date
,       dsd.comp_reg_mult
,       dsd.com_reg_price
--,       dsd.REG_CPI MAYANK : 20190426 : flattening the formulas in excel
,        CASE WHEN com_reg_price>0 THEN 
        (dsd.Reg_Retail/GREATEST(dsd.Reg_Retail_Price_Fctr,1) - dsd.com_reg_price/GREATEST(dsd.comp_reg_mult,1))/NULLIF(dsd.Reg_Retail/ GREATEST(dsd.Reg_Retail_Price_Fctr,1),'0')
        ELSE NULL
        END ::DECIMAL(18,3) AS REG_CPI
,       dsd.COMP_AD_MULT
,       dsd.COMP_AD_PRICE
,       dsd.Comments
,       dsd.Modified_flag
,       dsd.ROG_and_CIG
,       dsd.Allowance_Counts
,       dsd.Report_Date
FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6 dsd
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases sua
ON      sua.rog_cd = dsd.rog_cd
AND     sua.upc_id = dsd.upc_id
AND     sua.rpt_group = 'S30_03'
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       upc_id
        ,       SUM(AVG_NET_SALES_13_WK) AS upc_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS upc_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
        WHERE   rpt_group = 'S30_03'
        GROUP   BY 1,2
        ) upc
ON      upc.division_id = dsd.division_id
AND     upc.upc_id      = dsd.upc_id
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       DIV_PROMO_GRP_CD
        ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS cig_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
        WHERE   rpt_group = 'S30_03'
        GROUP   BY 1,2
        ) cig
ON      cig.division_id = dsd.division_id
AND     cig.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
--      MAYANK : 20190625 : creating this table for matching the upc and cig sales for blank cigs.
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       upc_id
        ,       DIV_PROMO_GRP_CD
        ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS cig_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
        GROUP   BY 1,2,3
        ) cig_upc
ON      cig_upc.division_id = dsd.division_id
AND     cig_upc.upc_id = dsd.upc_id
AND     cig_upc.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       div_promo_grp_cd
        ,       upc_id
        FROM    (
                SELECT  division_id
                ,       div_promo_grp_cd
                ,       upc_id
                ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
                WHERE   div_promo_grp_cd > 0
                AND     rpt_group = 'S30_03'
                GROUP   BY 1,2,3
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY division_id, div_promo_grp_cd ORDER by cig_sum_AVG_NET_SALES_13_WK DESC, upc_id ASC) = 1
        ) litm
ON      litm.division_id = dsd.division_id
AND     litm.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
AND     litm.upc_id = dsd.upc_id
LEFT    OUTER JOIN (
        SELECT  rog_and_cig
        ,       COUNT(DISTINCT allowance_counts) AS cnt
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6
        GROUP   BY rog_and_cig
        HAVING  cnt > 1
        ) malw
ON      malw.rog_and_cig = dsd.rog_and_cig
WHERE   dsd.division_id = 30
and     dsd.price_area_id IN (13, 14);
-- FUN_CAST_OPTR - Reformat casting
-- NULLIFZERO: Translated NullifZero to nullif function
-- MAYANK(20190425) : commented below lines to obtain group 1 reports
-- AND     dsd.group_cd > 1;
```

## Step 60:
```sql
sql_step60 = """
-- Division 29, PA 88 & 89 special report
INSERT
INTO    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_final
(       division_id
,       rpt_group
,       promo_no_allowance
,       allow_no_promo
,       missing_allowance
,       cost_change
,       ad_plan
,       t_markdown_pctg
,       less_10_Promo
,       t_Allowance_pctg
,       less_10_allowance
,       greater_100_pass_through
,       t_09_Retail
,       lead_item
,       Dominant_Price_Area
,       t_OOB
,       sskvi
,       group_cd
,       group_nm
,       SMIC
,       SMIC_name
,       rog
,       price_area_id
,       PA_name
,       pricing_role
,       OOB_gap_id
,       DIV_PROMO_GRP_CD
,       loc_common_retail_cd
,       vendor_name
,       vend_nbr
,       vend_sub_acct_nbr
,       cost_area_id
,       Manuf
,       upc_id
,       corp_item_cd
,       item_description
,       DST
,       FACILITY
,       dst_stat
,       rtl_stat
,       buyer_nm
,       vend_conv_fctr
,       t_pack_retail_qty
,       size_dsc
,       Row_Offset
,       UPC_13_Wk_Avg_Sales
,       UPC_13_Wk_Avg_Qty
,       UPC_13_Wk_Avg_RTL
,       T_RANK_BY_ROG_AND_CPC
,       pct_ACV_Stores
,       t_CPC_13_Wk_Avg_Sales
,       t_CPC_13_Wk_Avg_Qty
,       t_CPC_13_Wk_Avg_RTL
,       PND_Cost_Change_VND
,       PND_VEN_Date_Effective
,       New_Recc_Reg_Retail
,       Vendor_Unit_Cost
,       Unit_Item_Billing_Cost
,       Prev_Retail_Price_Fctr
,       Previous_Retail_Price
,       Prev_Retail_Effective_Date
,       Pending_EDLP_Mult
,       Pending_EDLP_Retail
,       Pending_EDLP_Chg_Date
,       Pending_GP_pctg
,       Reg_Retail_Price_Fctr
,       Reg_Retail
,       t_price_Per
,       t_Unit
,       Reg_GP_pctg
,       case_allow_count
,       case_allow_amt
,       Case_Allow_amt_per_Unit
,       Case_Start_Date
,       Case_End_Date
,       S2S_Allow_count
,       S2S_Allow_amt
,       S2S_Allow_amt_per_Unit
,       S2S_Start_Date
,       S2S_End_Date
,       Scan_Allow_count
,       Scan_Allow_amt
,       Scan_Start_Date
,       Scan_End_Date
,       Redem_Scan_Allow_count
,       Redem_Allow_amt
,       Redem_Start_Date
,       Redem_End_Date
,       Total_Allow_Unit
,       Net_Cost_with_Allow
,       Promo_Multiple
,       Promo_Price
,       Coupon_Method
,       Min_Purch
,       Limit_Per_Txn
,       Promo_GP_pctg
,       Net_Promo_Price
,       Price_Per
,       t2_Unit
,       Mark_down
,       Promo_Start
,       Promo_End
,       pass_through
,       NEW_Multiple
,       NEW_Retail
,       NEW_EDLP_GP_pctg
,       NEW_Promo_Multiple
,       NEW_Promo_Retail
,       NEW_Promo_GPpctg
,       NEW_Passthrough
,       compet_code
,       price_chk_date
,       comp_reg_mult
,       com_reg_price
,       REG_CPI
,       COMP_AD_MULT
,       COMP_AD_PRICE
,       Comments
,       Modified_flag
,       ROG_and_CIG
,       Allowance_Counts
,       Report_Date
)
SELECT  dsd.division_id
,       'PSOC 88, 89' AS rpt_group
,       CASE WHEN COALESCE(dsd.Net_Promo_Price, 0) > 0 AND (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0)) = 0 THEN 'Y' ELSE ' ' END AS promo_no_allowance
,       CASE WHEN COALESCE(dsd.S2S_Allow_amt, 0) + COALESCE(dsd.Scan_Allow_amt, 0) > 0 AND COALESCE(dsd.Net_Promo_Price, 0) = 0 THEN 'Y' ELSE ' ' END AS allow_no_promo
,       CASE WHEN malw.rog_and_cig IS NULL THEN ' ' ELSE 'Y' END AS missing_allowance
,       CASE WHEN dsd.PND_Cost_Change_VND IS NULL THEN ' ' ELSE 'Y' END AS cost_change
,       dsd.ad_plan
,       dsd.Mark_down / NULLIF(dsd.reg_retail,'0') / NULLIF(dsd.Reg_Retail_Price_Fctr,'0') AS t_markdown_pctg -- dsd.Markdown_pctg
,       CASE WHEN t_markdown_pctg > 0 AND t_markdown_pctg < 0.1 THEN 'Y' ELSE ' ' END AS less_10_Promo  -- dsd.less_10_Promo
,       (((COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0))) / NULLIF(dsd.Vendor_Unit_Cost,'0')) ::DECIMAL(15,3) AS t_Allowance_pctg -- dsd.Allowance_pctg    -- COLUMN CH
,       CASE WHEN t_Allowance_pctg > 0 AND t_Allowance_pctg < 0.1 THEN 'Y' ELSE ' ' END AS less_10_allowance  -- dsd.less_10_allowance
,       CASE WHEN (dsd.mark_down / NULLIF(COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0),'0')) ::DECIMAL(5,2) > 1 THEN 'Y' ELSE ' ' END AS greater_100_pass_through  --dsd.greater_100_pass_through
 
--,       dsd.t_09_Retail MAYANK : 20190426 : flattening the formulas in excel
,       CASE WHEN (round(dsd.Reg_Retail - floor(dsd.Reg_Retail),2) = 0.09) THEN 'Y' ELSE ' ' END AS t_09_Retail 
,       -- MAYANK : marking blank CIGS as Lead Item (28/03/2019)
        CASE 
            WHEN litm.division_id IS NOT NULL THEN 'Y'  
            WHEN dsd.div_promo_grp_cd = 0 THEN 'Y'
            --WHEN litm.division_id IS NULL THEN ' '
            ELSE ' '            
        END AS lead_item -- dsd.lead_item
,       dsd.Dominant_Price_Area
,       CASE WHEN dsd.manuf IN ('21130', '79893', '58200', '11535', '41303', '41130') THEN 'Y' ELSE ' ' END AS t_OOB  -- dsd.OOB
,       dsd.sskvi
,       dsd.group_cd
,       dsd.group_nm
,       dsd.SMIC
,       dsd.SMIC_name
,       CASE
            WHEN dsd.rog_cd = 'SEAS' THEN '1-SEAS'
            WHEN dsd.rog_cd = 'SEAG' THEN '2-SEAG'
            WHEN dsd.rog_cd = 'ACME' THEN '1-ACME'
            WHEN dsd.rog_cd = 'SDEN' THEN '1-SDEN'
            WHEN dsd.rog_cd = 'ADEN' THEN '2-ADEN'
            WHEN dsd.rog_cd = 'AIMT' THEN '1-AIMT'
            WHEN dsd.rog_cd = 'AJWL' THEN '1-AJWL'
            WHEN dsd.rog_cd = 'SNCA' THEN '1-SNCA'
            WHEN dsd.rog_cd = 'SHAW' THEN '2-SHAW'
            WHEN dsd.rog_cd = 'SPRT' THEN '1-SPRT'
            WHEN dsd.rog_cd = 'APOR' THEN '2-APOR'
            WHEN dsd.rog_cd = 'SSEA' THEN '1-SSEA'
            WHEN dsd.rog_cd = 'SSPK' THEN '2-SSPK'
            WHEN dsd.rog_cd = 'SACG' THEN '3-SACG'
            WHEN dsd.rog_cd = 'ASHA' THEN '1-ASHA'
            WHEN dsd.rog_cd = 'AVMT' THEN '2-AVMT'
            WHEN dsd.rog_cd = 'VSOC' THEN '1-VSOC'
            WHEN dsd.rog_cd = 'ASOC' THEN '2-ASOC'
            WHEN dsd.rog_cd = 'PSOC' THEN '3-PSOC'
            WHEN dsd.rog_cd = 'RDAL' THEN '1-RDAL'
            WHEN dsd.rog_cd = 'ADAL' THEN '2-ADAL'
            WHEN dsd.rog_cd = 'RHOU' THEN '3-RHOU'
            WHEN dsd.rog_cd = 'AHOU' THEN '4-AHOU'
            WHEN dsd.rog_cd = 'SPHO' THEN '1-SPHO'
            WHEN dsd.rog_cd = 'APHO' THEN '2-APHO'
            WHEN dsd.rog_cd = 'ALAS' THEN '3-ALAS'
            WHEN dsd.rog_cd = 'VLAS' THEN '4-VLAS'
            WHEN dsd.rog_cd = 'SPHX' THEN '5-SPHX'
            WHEN dsd.rog_cd = 'TEST' THEN '6-TEST'
            ELSE dsd.rog_cd
        END ::VARCHAR(10) AS rog
,       dsd.price_area_id               -- COLUMN S
,       dsd.PA_name                     -- COLUMN T
,       dsd.pricing_role
,       dsd.OOB_gap_id
,       dsd.DIV_PROMO_GRP_CD            -- COLUMN W CIG
,       dsd.loc_common_retail_cd        -- COLUMN X
,       dsd.vendor_name
,       dsd.vend_nbr
,       dsd.vend_sub_acct_nbr
,       dsd.cost_area_id
,       dsd.Manuf
,       dsd.upc_id
,       dsd.corp_item_cd
,       dsd.item_description
,       dsd.DST
,       dsd.FACILITY
,       dsd.dst_stat
,       dsd.rtl_stat
,       dsd.buyer_nm
,       dsd.vend_conv_fctr
,       dsd.t_pack_retail_qty
,       dsd.size_dsc
,       dsd.Row_Offset
,       sua.AVG_NET_SALES_13_WK     AS UPC_13_Wk_Avg_Sales           -- COLUMN: AP
,       sua.AVG_ITEM_QTY_13_WK      AS UPC_13_Wk_Avg_Qty
,       sua.AVG_NET_SALES_13_WK / NULLIF(sua.AVG_ITEM_QTY_13_WK,'0') AS UPC_13_Wk_Avg_RTL  -- dsd.UPC_13_Wk_Avg_RTL
,       CASE WHEN litm.division_id IS NULL THEN NULL ELSE 1 END ::INTEGER AS T_RANK_BY_ROG_AND_CPC
,       (sua.NUM_STORES_SELLING * 1.00) / (sua.NUM_STORES_IN_ROG * 1.00) * 100 ::DECIMAL(5,2) AS pct_ACV_Stores
--,       COALESCE(cig.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK) AS t_CPC_13_Wk_Avg_Sales -- dsd.CPC_13_Wk_Avg_Sales   -- COLUMN: AU
,        CASE 
            WHEN dsd.div_promo_grp_cd = 0 THEN COALESCE(cig_upc.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK)
            ELSE COALESCE(cig.cig_sum_AVG_NET_SALES_13_WK, upc.upc_sum_AVG_NET_SALES_13_WK)
        END AS t_CPC_13_Wk_Avg_Sales -- dsd.CPC_13_Wk_Avg_Sales   -- COLUMN: AU
--,       COALESCE(cig.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK) AS t_CPC_13_Wk_Avg_Qty  -- dsd.CPC_13_Wk_Avg_Qty
,       CASE
            WHEN dsd.div_promo_grp_cd = 0 THEN COALESCE(cig_upc.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK)
            ELSE COALESCE(cig.cig_sum_AVG_ITEM_QTY_13_WK, upc.upc_sum_AVG_ITEM_QTY_13_WK)
        END AS t_CPC_13_Wk_Avg_Qty  -- dsd.t_CPC_13_Wk_Avg_Qty
,       t_CPC_13_Wk_Avg_Sales / NULLIF(t_CPC_13_Wk_Avg_Qty,'0')  AS t_CPC_13_Wk_Avg_RTL  --dsd.CPC_13_Wk_Avg_RTL
,       dsd.PND_Cost_Change_VND
,       dsd.PND_VEN_Date_Effective
--,       dsd.New_Recc_Reg_Retail MAYANK : 20190426 : flattening the formulas in excel
,       (dsd.PND_Cost_Change_VND/NULLIF(dsd.Vendor_Unit_Cost,'0'))*dsd.Reg_Retail ::DECIMAL(18,3) as New_Recc_Reg_Retail
,       dsd.Vendor_Unit_Cost
,       dsd.Unit_Item_Billing_Cost
,       dsd.Prev_Retail_Price_Fctr
,       dsd.Previous_Retail_Price
,       dsd.Prev_Retail_Effective_Date
,       dsd.Pending_EDLP_Mult
,       dsd.Pending_EDLP_Retail
,       dsd.Pending_EDLP_Chg_Date
--,       dsd.Pending_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,       ((dsd.Pending_EDLP_Retail/NULLIF(dsd.Pending_EDLP_Mult,'0'))-dsd.Unit_Item_Billing_Cost)/NULLIF(dsd.Pending_EDLP_Retail/NULLIF(dsd.Pending_EDLP_Mult,'0'),'0') ::DECIMAL(15,3) as Pending_GP_pctg
,       dsd.Reg_Retail_Price_Fctr
,       dsd.Reg_Retail
,       dsd.t_price_Per
,       dsd.t_Unit
--,       dsd.Reg_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,       ((dsd.Reg_Retail/NULLIF(dsd.Reg_Retail_Price_Fctr,'0'))-dsd.Unit_Item_Billing_Cost)/NULLIF(dsd.Reg_Retail/NULLIF(dsd.Reg_Retail_Price_Fctr,'0'),'0') ::DECIMAL(15,3) as Reg_GP_pctg
,       dsd.case_allow_count
,       dsd.case_allow_amt            AS case_allow_amt
,       dsd.Case_Allow_amt_per_Unit   AS Case_Allow_amt_per_Unit
,       dsd.Case_Start_Date
,       dsd.Case_End_Date
,       dsd.S2S_Allow_count
,       dsd.S2S_Allow_amt             AS S2S_Allow_amt
,       dsd.S2S_Allow_amt_per_Unit    AS S2S_Allow_amt_per_Unit
,       dsd.S2S_Start_Date
,       dsd.S2S_End_Date
,       dsd.Scan_Allow_count
,       dsd.Scan_Allow_amt            AS Scan_Allow_amt
,       dsd.Scan_Start_Date
,       dsd.Scan_End_Date
,       dsd.Redem_Scan_Allow_count
,       dsd.Redem_Allow_amt           AS Redem_Allow_amt
,       dsd.Redem_Start_Date
,       dsd.Redem_End_Date
,       COALESCE(dsd.Case_Allow_amt_per_Unit, 0) +  COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) +  COALESCE(dsd.Scan_Allow_amt, 0) AS Total_Allow_Unit
--,       dsd.Net_Cost_with_Allow MAYANK : 20190426 : flattening the formulas in excel
,       (dsd.Vendor_Unit_Cost - (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0))) ::DECIMAL(18,3) as Net_Cost_with_Allow
,       dsd.Promo_Multiple
,       dsd.Promo_Price
,       dsd.Coupon_Method
,       dsd.Min_Purch
,       dsd.Limit_Per_Txn
--,       dsd.Promo_GP_pctg MAYANK : 20190426 : flattening the formulas in excel
,       ((dsd.Net_Promo_Price - (dsd.Vendor_Unit_Cost - (COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0))))/NULLIF(dsd.Net_Promo_Price,'0')) ::DECIMAL(15,3) as Promo_GP_pctg
,       dsd.Net_Promo_Price
,       dsd.Price_Per
,       dsd.t2_Unit
,       dsd.Mark_down
,       dsd.Promo_Start
,       dsd.Promo_End
,       (dsd.mark_down / NULLIF(COALESCE(dsd.Case_Allow_amt_per_Unit, 0) + COALESCE(dsd.S2S_Allow_amt_per_Unit, 0) + COALESCE(dsd.Scan_Allow_amt, 0),'0')) ::DECIMAL(5,2) AS pass_through  -- dsd.Pass_Through  -- COLUMN CW
,       dsd.NEW_Multiple
,       dsd.NEW_Retail
,       dsd.NEW_EDLP_GP_pctg
,       dsd.NEW_Promo_Multiple
,       dsd.NEW_Promo_Retail
,       dsd.NEW_Promo_GPpctg
,       dsd.NEW_Passthrough
,       dsd.compet_code
,       dsd.price_chk_date
,       dsd.comp_reg_mult
,       dsd.com_reg_price
--,       dsd.REG_CPI MAYANK : 20190426 : flattening the formulas in excel
,       CASE WHEN com_reg_price>0 THEN 
        (dsd.Reg_Retail/GREATEST(dsd.Reg_Retail_Price_Fctr,1) - dsd.com_reg_price/GREATEST(dsd.comp_reg_mult,1))/NULLIF(dsd.Reg_Retail/ GREATEST(dsd.Reg_Retail_Price_Fctr,1),'0')
        ELSE NULL
        END ::DECIMAL(18,3) AS REG_CPI
,       dsd.COMP_AD_MULT
,       dsd.COMP_AD_PRICE
,       dsd.Comments
,       dsd.Modified_flag
,       dsd.ROG_and_CIG
,       dsd.Allowance_Counts
,       dsd.Report_Date
FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6 dsd
LEFT    OUTER JOIN EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases sua
ON      sua.rog_cd = dsd.rog_cd
AND     sua.upc_id = dsd.upc_id
AND     sua.rpt_group = 'S29_02'
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       upc_id
        ,       SUM(AVG_NET_SALES_13_WK) AS upc_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS upc_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
        WHERE   rpt_group = 'S29_02'
        GROUP   BY 1,2
        ) upc
ON      upc.division_id = dsd.division_id
AND     upc.upc_id      = dsd.upc_id
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       DIV_PROMO_GRP_CD
        ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS cig_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
        WHERE   rpt_group = 'S29_02'
        GROUP   BY 1,2
        ) cig
ON      cig.division_id = dsd.division_id
AND     cig.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
--      MAYANK : 20190625 : creating this table for matching the upc and cig sales for blank cigs.
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       upc_id
        ,       DIV_PROMO_GRP_CD
        ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
        ,       SUM(AVG_ITEM_QTY_13_WK)  AS cig_sum_AVG_ITEM_QTY_13_WK
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
        GROUP   BY 1,2,3
        ) cig_upc
ON      cig_upc.division_id = dsd.division_id
AND     cig_upc.upc_id = dsd.upc_id
AND     cig_upc.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
LEFT    OUTER JOIN (
        SELECT  division_id
        ,       div_promo_grp_cd
        ,       upc_id
        FROM    (
                SELECT  division_id
                ,       div_promo_grp_cd
                ,       upc_id
                ,       SUM(AVG_NET_SALES_13_WK) AS cig_sum_AVG_NET_SALES_13_WK
                FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_sua_sales_sp_cases
                WHERE   div_promo_grp_cd > 0
                AND     rpt_group = 'S29_02'
                GROUP   BY 1,2,3
                ) dt
        QUALIFY ROW_NUMBER() OVER (PARTITION BY division_id, div_promo_grp_cd ORDER by cig_sum_AVG_NET_SALES_13_WK DESC, upc_id ASC) = 1
        ) litm
ON      litm.division_id = dsd.division_id
AND     litm.DIV_PROMO_GRP_CD = dsd.DIV_PROMO_GRP_CD
AND     litm.upc_id = dsd.upc_id
LEFT    OUTER JOIN (
        SELECT  rog_and_cig
        ,       COUNT(DISTINCT allowance_counts) AS cnt
        FROM    EDM_SANDBOX_PRD.MERCHAPPS.t_pe_dsd_whse_item_attr_6
        GROUP   BY rog_and_cig
        HAVING  cnt > 1
        ) malw
ON      malw.rog_and_cig = dsd.rog_and_cig
WHERE   dsd.division_id = 29
and     dsd.price_area_id IN (88, 89);
```

## Step 61:
```sql
update EDM_SANDBOX_PRD.MERCHAPPS.T_PE_DSD_WHSE_FINAL set rog = '3-AKBA' where rog = 'AKBA' 
```

## Step 62:
```sql
update EDM_SANDBOX_PRD.MERCHAPPS.T_PE_DSD_WHSE_FINAL set rog = '2-SWMA' where rog = 'SWMA' 
```