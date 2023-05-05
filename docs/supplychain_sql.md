# Supply Chain SQL

## Store Inventory

~~~ sql
select 
        corporate_item_cd,
        retail_item_dsc,
        distribution_center_id,
        rog_id,
        txn_dte,
        sum(onhand_qty) as onhand_qty,
        sum(oos_day_flg) as oos_day_flg
        from
        (SELECT  DISTINCT 
        C.CORPORATE_ITEM_CD,
        C.RETAIL_ITEM_DSC, 
        SOC.WAREHOUSE_ID,
        lpad(rs.FACILITY_NBR,4,'0') AS STORE_ID, 
        rs.DIVISION_ID AS STORE_DIVISION,
        SOC.DISTRIBUTION_CENTER_ID,
        SOC.ROG_ID,
        ii.ONHAND_QTY,
        ii.TXN_DTE,
        ii.WEEK_ID,
        case when ONHAND_QTY <= 0 then 1 else 0 end OOS_DAY_FLG
        FROM    (
            SELECT
                  CL.CALENDAR_DT AS TXN_DTE, 
                  CL.FISCAL_YEAR_NBR||CL.FISCAL_WEEK_NBR AS WEEK_ID,
                  FACILITY_INTEGRATION_ID,
                  CORPORATE_ITEM_INTEGRATION_ID,
                  ONHAND_QTY
            FROM   (
                    SELECT  FACILITY_INTEGRATION_ID
                            ,CORPORATE_ITEM_INTEGRATION_ID
                            ,TXN_DTE
                            ,ITEM_QTY AS ONHAND_QTY
                            ,lead(TXN_DTE,1,CURRENT_DATE + 1) over(PARTITION BY FACILITY_INTEGRATION_ID, CORPORATE_ITEM_INTEGRATION_ID ORDER BY TXN_DTE asc) as lead_txn_dte
                    FROM    (
                                SELECT  II.FACILITY_INTEGRATION_ID, II.CORPORATE_ITEM_INTEGRATION_ID, DATE(II.TRANSACTION_TS) TXN_DTE, II.ITEM_QTY
                                FROM    EDM_VIEWS_PRD.DW_VIEWS.INVENTORY_ITEM II
                                WHERE   DW_CURRENT_VERSION_IND = TRUE
                                AND     DW_LOGICAL_DELETE_IND = FALSE
                                AND     INVENTORY_TYPE_CD = 'PP'
                                AND     DATE(TRANSACTION_TS) >=  current_date - 7 
                                QUALIFY ROW_NUMBER() 
                                        OVER(PARTITION BY II.FACILITY_INTEGRATION_ID, II.CORPORATE_ITEM_INTEGRATION_ID, DATE(TRANSACTION_TS) ORDER BY TRANSACTION_TS DESC) = 1 
                              )
                    ) inv
            join    EDM_VIEWS_PRD.DW_VIEWS.CALENDAR cl 
              on    cl.CALENDAR_DT >= inv.txn_dte 
             AND    cl.CALENDAR_DT < lead_txn_dte 
            WHERE   CL.CALENDAR_DT =  current_date - 1
            AND     CL.DW_CURRENT_VERSION_IND = TRUE
        ) II
    JOIN   (
        select *
        from EDM_VIEWS_PRD.DW_VIEWS.STORE_ORDER_CATALOG soc 
        WHERE   SOC.DW_CURRENT_VERSION_IND = TRUE
        AND     SOC.DW_LOGICAL_DELETE_IND = FALSE
        AND     SOC.PERPETUAL_INVENTORY_IND = 'Y'
        QUALIFY row_number() over (partition by store_facility_integration_id, corporate_item_integration_id order by dw_first_effective_dt desc) = 1
        ) SOC   ON SOC.store_facility_integration_id = II.FACILITY_INTEGRATION_ID 
                AND SOC.corporate_item_integration_id = II.corporate_item_integration_id
                AND II.TXN_DTE between AUTHORIZATION_FIRST_EFFECTIVE_TS AND AUTHORIZATION_LAST_EFFECTIVE_TS
    JOIN    EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM C ON C.CORPORATE_ITEM_INTEGRATION_ID = SOC.CORPORATE_ITEM_INTEGRATION_ID
    JOIN EDM_VIEWS_PRD.DW_VIEWS.SMIC_GROUP SG ON C.SMIC_GROUP_CD = SG.SMIC_GROUP_CD
    JOIN EDM_VIEWS_PRD.DW_VIEWS.SMIC_CATEGORY SC ON C.SMIC_GROUP_CD = SC.SMIC_GROUP_CD  and C.SMIC_CATEGORY_CD = SC.SMIC_CATEGORY_CD
    JOIN EDM_VIEWS_PRD.DW_VIEWS.SMIC_CLASS SCL ON C.SMIC_GROUP_CD = SCL.SMIC_GROUP_CD and C.SMIC_CATEGORY_CD = SCL.SMIC_CATEGORY_CD
    and C.SMIC_CLASS_CD=SCL.SMIC_CLASS_CD
    JOIN    EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE RS ON RS.FACILITY_INTEGRATION_ID = SOC.STORE_FACILITY_INTEGRATION_ID
    JOIN    EDM_VIEWS_PRD.DW_VIEWS.FACILITY_ADDRESS FA ON RS.FACILITY_INTEGRATION_ID = FA.FACILITY_INTEGRATION_ID
    WHERE   C.DW_CURRENT_VERSION_IND = TRUE
    AND     SG.DW_CURRENT_VERSION_IND = TRUE
    AND     SC.DW_CURRENT_VERSION_IND = TRUE
    AND     SCL.DW_CURRENT_VERSION_IND = TRUE
    AND     RS.DW_CURRENT_VERSION_IND = TRUE
    AND     RS.DW_LOGICAL_DELETE_IND = FALSE
    AND     FA.DW_CURRENT_VERSION_IND = TRUE
    AND     FA.DW_LOGICAL_DELETE_IND = FALSE) 
    group by 1,2,3,4,5
~~~

## Warehouse Inventory Activities
~~~ sql
select
      cic.corporate_item_cd
    , cic.ship_unit_pack_qty
    , winv.warehouse_id
    , wh.distribution_center_id as warehouse_nm
    , sum(transaction_qty) as sum_transaction_qty
    , sum(available_qty) as sum_available_qty
    , sum(allocated_qty) as sum_allocated_qty
    , sum(blocked_qty) as sum_blocked_qty
    , sum(inbound_transit_qty) as sum_inbound_transit_qty
    , sum(day_inbound_transit_qty) as sum_day_inbound_transit_qty
    , sum(sum_order_qty) as sum_on_order_qty
    , sum(total_onhand_qty) as sum_total_onhand_qty
    from edm_views_prd.dw_views.inventory_balance winv
    join edm_views_prd.dw_views.corporate_item cic
    on cic.corporate_item_integration_id = winv.corporate_item_integration_id 
    and current_date between cic.dw_first_effective_dt and cic.dw_last_effective_dt 
    and cic.corporation_id = 1
   -- join edm_views_prd.dw_views.supply_chain_item sci
   -- on sci.corporate_item_integration_id = winv.corporate_item_integration_id
   -- and current_date between sci.dw_first_effective_dt and sci.dw_last_effective_dt
    join edm_views_prd.dw_views.warehouse wh
    on wh.warehouse_id = winv.warehouse_id and wh.corporation_id=1 and current_date between wh.dw_first_effective_dt and wh.dw_last_effective_dt
    join
    ( select
    warehouse_id
    , corporate_item_integration_id
    , max(transaction_ts) as max_ts

    from edm_views_prd.dw_views.inventory_balance
    where
    dw_current_version_ind = true
    and dw_logical_delete_ind = false
    group by
    warehouse_id
    , corporate_item_integration_id
    ) mxts
    on mxts.warehouse_id = winv.warehouse_id
    and mxts.corporate_item_integration_id = winv.corporate_item_integration_id
    and mxts.max_ts = winv.transaction_ts  and cic.corporate_item_cd in (11101812)
    and winv.dw_current_version_ind = true and winv.dw_logical_delete_ind = false 
    
    left join (select distribution_center_id, corporate_item_cd, max(sum_order_qty)  as sum_order_qty from "EDM_BIZOPS_PRD"."MERCHAPPS"."zhu_warehouse_future_po" 
    where promised_delivery_ts between current_date and current_date + 7 group by 1,2) po 
    on po.distribution_center_id = wh.distribution_center_id and po.corporate_item_cd = cic.corporate_item_cd 
    
    group by 1, 2, 3, 4
~~~

## Store Order Delivery 

~~~ sql
SELECT 
    SCHEDULE_GROUP_CD         AS DPATS_GROUP_CD,     
    RETAIL_STORE_FACILITY_NBR AS STORE_ID,
    DATA_BATCH_ID             AS DATA_BATCH_CD,
    EFFECTIVE_DT              AS SCHED_EFF_DT,
    CASE 
        WHEN SDET.Process_Week_Day_Nm = 'SUNDAY'    AND Process_Week_Day_Nbr < 8  THEN SCHED.Effective_Dt
        WHEN SDET.Process_Week_Day_Nm = 'SUNDAY'    AND Process_Week_Day_Nbr > 7  THEN DATEADD(day, 7,  SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'MONDAY'    AND Process_Week_Day_Nbr < 9  THEN DATEADD(day, 1,  SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'MONDAY'    AND Process_Week_Day_Nbr > 8  THEN DATEADD(day, 8,  SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'TUESDAY'   AND Process_Week_Day_Nbr < 10 THEN DATEADD(day, 2,  SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'TUESDAY'   AND Process_Week_Day_Nbr > 9  THEN DATEADD(day, 9,  SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'WEDNESDAY' AND Process_Week_Day_Nbr < 11 THEN DATEADD(day, 3,  SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'WEDNESDAY' AND Process_Week_Day_Nbr > 10 THEN DATEADD(day, 10, SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'THURSDAY'  AND Process_Week_Day_Nbr < 12 THEN DATEADD(day, 4,  SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'THURSDAY'  AND Process_Week_Day_Nbr > 11 THEN DATEADD(day, 11, SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'FRIDAY'    AND Process_Week_Day_Nbr < 13 THEN DATEADD(day, 5,  SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'FRIDAY'    AND Process_Week_Day_Nbr > 12 THEN DATEADD(day, 12, SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'SATURDAY'  AND Process_Week_Day_Nbr < 14 THEN DATEADD(day, 6,  SCHED.Effective_Dt)
        WHEN SDET.Process_Week_Day_Nm = 'SATURDAY'  AND Process_Week_Day_Nbr > 13 THEN DATEADD(day, 13, SCHED.Effective_Dt) 
    END AS ORDER_DT,
SCHEDULE_CUTOFF_TM AS POLL_TM,
DATEADD(day, SDET.Process_Week_Day_Nbr -1, SCHED.Effective_Dt)  AS PICK_DT,
DATEADD(day, SDET.Delivery_Week_Day_Nbr -1, SCHED.Effective_Dt) AS DELIVERY_DT,
DATEADD(day, SDET.Process_Week_Day_Nbr + SDET.Offset_Value_Week_Day_Nbr -1, SCHED.Effective_Dt) AS AVAIL_DT

FROM DW_VIEWS.SHIPMENT_SCHEDULE SCHED

JOIN DW_VIEWS.SHIPMENT_SCHEDULE_DETAIL SDET
  ON SCHED.SHIPMENT_SCHEDULE_INTEGRATION_ID = SDET.SHIPMENT_SCHEDULE_INTEGRATION_ID
  
JOIN DW_VIEWS.D1_RETAIL_STORE STR
  ON SCHED.DESTINATION_FACILITY_INTEGRATION_ID = STR.FACILITY_INTEGRATION_ID

WHERE SCHEDULE_GROUP_CD= '25'
--WHERE sched_eff_dt ='2022-04-17'

AND RETAIL_STORE_FACILITY_NBR IN ('1953')
AND EFFECTIVE_DT = '2022-04-24'

AND DATA_BATCH_ID = '0129'
AND DATA_BATCH_ID NOT IN ('5601','5602','5603','5604','5605','5606','5607','5608','5613',
5614','5615','5616','5651','8811','8812','8826','8832','8834','8841')

ORDER BY 1,6    -- STORE_ID, order_dt;
~~~

## Warehouse Shipment to Store 

~~~ sql
SELECT DISTINCT 
SCIH.DIVISION_ID, 
WHSE.WAREHOUSE_ID, --WHSE_ID
WHSE.WAREHOUSE_NM, --whs.whse_nm, 
SCIH.ORDER_NBR,
STORE.RETAIL_STORE_FACILITY_NBR,
DTE.FISCAL_YEAR_NBR || LPAD(DTE.FISCAL_PERIOD_NBR,2,0),
TO_CHAR(SUM(SCID.SHIPPED_QTY),'999,999,999.99') AS SHIPMENTS, 
SCIH.INVOICE_CREATE_DT

FROM DW_VIEWS.SUPPLY_CHAIN_INVOICE_HEADER SCIH

JOIN DW_VIEWS.SUPPLY_CHAIN_INVOICE_DETAIL SCID
  ON SCIH.SUPPLY_CHAIN_INVOICE_INTEGRATION_ID = SCID.SUPPLY_CHAIN_INVOICE_INTEGRATION_ID
 AND SCID.DW_CURRENT_VERSION_IND = TRUE
 
JOIN DW_VIEWS.D1_RETAIL_STORE STORE
  ON SCIH.RETAIL_STORE_FACILITY_INTEGRATION_ID = STORE.FACILITY_INTEGRATION_ID

JOIN DW_VIEWS.D0_FISCAL_DAY DTE
  ON DTE.CALENDAR_DT = SCIH.INVOICE_CREATE_DT

JOIN DW_VIEWS.DATA_BATCH WDR  --DSS.whse_databatch_rog wdr
  ON SCIH.WAREHOUSE_ID = WDR.WAREHOUSE_ID
  
JOIN DW_VIEWS.WAREHOUSE WHSE
  ON WHSE.WAREHOUSE_ID = WDR.WAREHOUSE_ID

WHERE 1 = 1
  AND SCIH.CORPORATION_ID = 1 --wsi.ship_corp_id = 1
  AND WDR.PRIMARY_IND = 'Y'
--AND wsi.ship_corp_id = wsi.dest_corp_id
--AND SCIH.INVOICE_CREATE_DT BETWEEN '04/06/2022' AND '04/28/2022' --CURRENT_DATE
AND SCIH.INVOICE_CREATE_DT BETWEEN '2022-04-06' AND '2022-04-28' --CURRENT_DATE
and STORE.RETAIL_STORE_FACILITY_NBR = '0169' --wsi.store_id in ('0169')
and WHSE.WAREHOUSE_ID = '5229'  --wsi.whse_cd= '5229'
AND SCIH.DW_CURRENT_VERSION_IND = TRUE

GROUP BY SCIH.DIVISION_ID,
SCIH.ORDER_NBR, --.wsi.whse_ordER_nbr,

WHSE.WAREHOUSE_ID, --WHSE_ID
WHSE.WAREHOUSE_NM, --whs.whse_nm, 
DTE.FISCAL_YEAR_NBR,  --dte.period_id
DTE.FISCAL_PERIOD_NBR,  --dte.period_id
SCIH.INVOICE_CREATE_DT
~~~

## Warehouse Order Qty Pack

~~~ sql
select warehouse_id, order_pack_qty from EDM_VIEWS_PRD.DW_VIEWS.SUPPLY_CHAIN_ITEM a join EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM b on a.corporate_item_integration_id = b.corporate_item_integration_id
 and current_date between a.dw_first_effective_dt and a.dw_last_effective_dt and current_date between b.dw_first_effective_dt and b.dw_last_effective_dt 
 and a.DW_CURRENT_VERSION_IND = 1 and b.DW_CURRENT_VERSION_IND = 1
 and corporate_item_cd = 73500114
~~~

## Volume plan for supply chain.

~~~ sql 
WITH CAL AS 
    (SELECT PROMOTION_WEEK_ID
    ,PROMOTION_WEEK_START_DT 
     FROM 
        (SELECT DISTINCT 
        PROMOTION_WEEK_ID
        ,DIVISION_ID
        ,PROMOTION_WEEK_START_DT 
        ,1 
        FROM EDM_VIEWS_PRD.DW_VIEWS.D1_PROMOTION_CALENDAR_WEEK
        WHERE PROMOTION_WEEK_END_DT < CURRENT_DATE()
        AND DIVISION_ID = '19')

    QUALIFY ROW_NUMBER() OVER (PARTITION BY DIVISION_ID,1 ORDER BY PROMOTION_WEEK_ID DESC) < 60)    

SELECT WEEK_ID,
A.CORPORATE_ITEM_CD AS CIC,
A.DIVISION_NM, 
--TRY_TO_NUMBER(WEEK1.DIVISION_ID) AS DIVISION_ID,
CASE
WHEN A.DIVISION_NM = 'PORTLAND' THEN 19
WHEN A.DIVISION_NM = 'SEATTLE' THEN 27
WHEN A.DIVISION_NM = 'NOR CALIFORNIA' THEN 25
WHEN A.DIVISION_NM = 'SO CALIFORNIA' THEN 29
WHEN A.DIVISION_NM = 'SOUTHWEST' THEN 17
WHEN A.DIVISION_NM = 'DENVER' THEN 5
WHEN A.DIVISION_NM = 'MID-ATLANTIC' THEN 34
WHEN A.DIVISION_NM = 'INTERMOUNTAIN' THEN 30
WHEN A.DIVISION_NM = 'JEWEL' THEN 32
WHEN A.DIVISION_NM = 'SOUTHERN' THEN 20
WHEN A.DIVISION_NM = 'SHAWS' THEN 33
END AS DIVISION_ID,    
A.ROG_ID AS ROG,
C.DISTRIBUTION_CENTER_ID AS DC,
E.UPC_NBR AS UPC,
ITM.SHIP_UNIT_PACK_QTY AS "Case size",
C.VENDOR_ID||C.VENDOR_SUB_ACCOUNT_ID||C.WIMS_SUB_VENDOR_NBR AS VEND_KEY,
WEEK2.PROMOTION_WEEK_ID AS "Time-shifted week ID",
WEEK2.PROMOTION_WEEK_START_DT AS "Calendar date",
WEEK2.PROMOTION_WEEK_START_DT + 364 AS CALENDAR_DT,    
ROUND((D.LEAD_TIME_DYS+7)/7) AS "Total lead time (weeks)",
SUM(PROMOTED_UNIT_NBR) AS "Promoted units",
SUM(NON_PROMO_UNIT_NBR) AS "Non-promoted units",
"Promoted units" + "Non-promoted units" AS "Total Units",
DIV0(("Promoted units" + "Non-promoted units"),ITM.SHIP_UNIT_PACK_QTY) AS "Total Cases" 


FROM EDM_VIEWS_PRD.DW_VIEWS.PROMO_PERF_RPT A
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM B
    ON A.CORPORATE_ITEM_CD = B.CORPORATE_ITEM_CD
    AND B.DW_CURRENT_VERSION_IND = 1
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_REFERENCE E
    ON E.CORPORATE_ITEM_INTEGRATION_ID = B.CORPORATE_ITEM_INTEGRATION_ID
    AND E.DW_CURRENT_VERSION_IND = 1
INNER JOIN (SELECT DISTINCT
            CORPORATE_ITEM_INTEGRATION_ID
            ,DISTRIBUTION_CENTER_ID
            ,WAREHOUSE_ID
            ,ROG_ID
            ,VENDOR_ID
            ,VENDOR_SUB_ACCOUNT_ID
            ,WIMS_SUB_VENDOR_NBR
            FROM EDM_VIEWS_PRD.DW_VIEWS.STORE_ORDER_CATALOG 
            WHERE DW_CURRENT_VERSION_IND = 1
            AND DISTRIBUTION_CENTER_ID IS NOT NULL
            AND WIMS_SUB_VENDOR_NBR IS NOT NULL) C
    ON A.ROG_ID = C.ROG_ID
    AND B.CORPORATE_ITEM_INTEGRATION_ID = C.CORPORATE_ITEM_INTEGRATION_ID
INNER JOIN (SELECT VENDOR_ID,VENDOR_SUB_ACCOUNT_ID,WIMS_SUB_VENDOR_NBR,LEAD_TIME_DYS,WAREHOUSE_ID,DISTRIBUTION_CENTER_ID 
            FROM  EDM_VIEWS_PRD.DW_VIEWS.VENDOR_WAREHOUSE 
            WHERE DISTRIBUTION_CENTER_ID <> ''
            QUALIFY ROW_NUMBER() OVER (PARTITION BY WAREHOUSE_ID,VENDOR_ID,VENDOR_SUB_ACCOUNT_ID,WIMS_SUB_VENDOR_NBR ORDER BY DW_LAST_EFFECTIVE_DT DESC) = 1) D
    ON D.DISTRIBUTION_CENTER_ID = C.DISTRIBUTION_CENTER_ID
    AND D.VENDOR_ID = C.VENDOR_ID
    AND D.VENDOR_SUB_ACCOUNT_ID = C.VENDOR_SUB_ACCOUNT_ID
    AND D.WIMS_SUB_VENDOR_NBR = C.WIMS_SUB_VENDOR_NBR
    AND D.WAREHOUSE_ID = C.WAREHOUSE_ID 
/*INNER JOIN CAL WEEK1
    ON TO_NUMBER(WEEK1.DIVISION_ID)::INTEGER = A.DIVISION_ID  
    AND WEEK1.PROMOTION_WEEK_ID = A.WEEK_ID*/
INNER JOIN CAL WEEK1
    --ON WEEK1.DIVISION_NM = A.DIVISION_NM
    ON WEEK1.PROMOTION_WEEK_ID = A.WEEK_ID 
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_PROMOTION_CALENDAR_WEEK WEEK2
    --ON WEEK2.DIVISION_ID = WEEK1.DIVISION_ID
    ON WEEK2.PROMOTION_WEEK_START_DT = WEEK1.PROMOTION_WEEK_START_DT - ROUND((D.LEAD_TIME_DYS+7)/7)*7
    AND WEEK2.DIVISION_ID = '19'
INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.SUPPLY_CHAIN_ITEM ITM
    ON ITM.WAREHOUSE_ID = D.WAREHOUSE_ID
    AND ITM.WAREHOUSE_ID = C.WAREHOUSE_ID
    AND ITM.CORPORATE_ITEM_INTEGRATION_ID = B.CORPORATE_ITEM_INTEGRATION_ID
    AND ITM.DW_CURRENT_VERSION_IND = 1

WHERE 1=1
--AND CATEGORY_ID IN (2501,2505,2510,2515,2520,2525,2530,2535,3001,3002,3003,3004)
--AND QUARTER_ID IN (20221,20222,20223,20224,20231) 
AND DEPARTMENT_ID IN (301,303,311,314,317,336)
/*AND A.CORPORATE_ITEM_CD IN (30020337,30020334,30020425,30040167,30040160,30040207,30040199,30040213,30040128,30040162,
                     30010374,30010376,25010054,25010057,25250643,25250644,30040053,30040087,30040089,30040083,
                     30040077,30040064,30020183,30020326,30020327,25250643,25250644)*/

GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13);
~~~

## All Warehouse Items 

~~~sql 
select
  distinct sci.division_id DIVISION,
  sci.distribution_center_id DST_CNTR,
  sci.warehouse_id FACILITY,
  ci.corporate_item_cd CORP_ITEM_CD,
  ci.Item_Dsc ITEM_DESCRIPTION,
  case
    when sci.warehouse_id like '20%' then sci.System_Specific_Item_Id
    else sci.branch_item_cd
  end WHS_ITEM,
  sci.Ship_Unit_Pack_Qty PACK_WHSE,
  sci.warehouse_item_status_cd STATUS_DST,
  ci.Size_Qty || ' ' || ci.Size_UOM_Cd SIZE,
  wc.ITEM_BILLING_COST_AMT IB_COST,
  wc.VENDOR_ID || '-' || vm.vendor_nm VENDOR,
  coalesce(ifc.SINGLE_SMOOTHED_VALUE_NBR, 0) as SSV1,
  cast(
    concat(
      lpad(ssc.smic_group_cd, 2, '0'),
      lpad(ssc.SMIC_Category_Cd, 2, '0'),
      lpad(ssc.SMIC_Class_Cd, 2, '0'),
      lpad(ssc.SMIC_Sub_Class_Cd, 2, '0')
    ) as int
  ) as CODE_KEY,
  m.team_mgr_cd || '-' || m.team_mgr_nm MANAGER,
  b.buyer_id || '-' || b.buyer_nm BUYER,
  case
    when sci.distribution_center_id = 'WSPK' then '32'
    else whse.OLD_DISTRIBUTION_CENTER_CD
  end as DC_ID,
  case
    when sci.warehouse_id = '3345' then '04'
    else whse.PHYSICAL_WAREHOUSE_GROUP_ID
  end as WH_ID,
  sc.Section_Cd || '-' || sc.Section_Nm RETAIL_SECTION,
  dept.department_id || '-' || dept.department_nm DEPARTMENT,
  '' as FORECAST,
  g.SMIC_GROUP_CD || '-' || g.SMIC_GROUP_DSC GROUP_CLASSIFICATION,
  cat.SMIC_Group_Cd || cat.smic_category_nm || '-' || cat.SMIC_Category_DSC SMIC_CLASSIFICATION
from
  EDM_VIEWS_PRD.DW_VIEWS.Supply_Chain_Item sci
  left outer join EDM_VIEWS_PRD.DW_VIEWS.corporate_item ci on ci.corporate_item_integration_id = sci.corporate_item_integration_id
  and ci.dw_current_version_ind = true
  and ci.dw_logical_delete_ind = false
  and sci.dw_current_version_ind = true
  and sci.dw_logical_delete_ind = false
  left outer join (
    select
      *
    from
      EDM_VIEWS_PRD.DW_VIEWS.vendor_warehouse_item_cost
    where
      dw_current_version_ind = true
      and dw_logical_delete_ind = false QUALIFY row_number() over (
        partition by WAREHOUSE_ID,
        CORPORATE_ITEM_INTEGRATION_ID
        order by
          source_creation_dt desc
      ) = 1
  ) wc on wc.division_id = sci.division_id
  and wc.warehouse_id = sci.warehouse_id
  and wc.corporate_item_integration_id = sci.corporate_item_integration_id
  left outer join EDM_VIEWS_PRD.DW_VIEWS.SMIC_Sub_Class ssc on ci.smic_group_cd = ssc.smic_group_cd
  and ci.smic_category_cd = ssc.smic_category_cd
  and ci.smic_class_cd = ssc.smic_class_cd
  and ci.smic_sub_class_cd = ssc.smic_sub_class_cd
  and ssc.dw_current_version_ind = true
  and ssc.dw_logical_delete_ind = false
  left outer join EDM_VIEWS_PRD.DW_VIEWS.smic_group g on g.SMIC_GROUP_CD = ci.SMIC_Group_Cd
  and g.dw_current_version_ind = true
  and g.dw_logical_delete_ind = false
  left outer join EDM_VIEWS_PRD.DW_VIEWS.smic_category cat on cat.SMIC_group_cd = ci.SMIC_Group_Cd
  and cat.SMIC_Category_nm = ci.SMIC_Category_Cd
  and cat.dw_current_version_ind = true
  and cat.dw_logical_delete_ind = false
  left outer join EDM_VIEWS_PRD.DW_VIEWS.warehouse whse on whse.division_id = sci.division_id
  and whse.WAREHOUSE_ID = sci.warehouse_id
  and whse.dw_current_version_ind = true
  and whse.dw_logical_delete_ind = false
  left outer join EDM_VIEWS_PRD.DW_VIEWS.buyer b on sci.buyer_id = b.buyer_id
  and b.dw_current_version_ind = true
  and b.dw_logical_delete_ind = false
  left outer join EDM_VIEWS_PRD.DW_BIZOPS_VIEWS.WMMGROPT m ON b.BUYER_MANAGER_CD = m.TEAM_MGR_CD
  left outer join EDM_VIEWS_PRD.DW_VIEWS.vendor_master vm on wc.VENDOR_ID = vm.vendor_id
  and vm.dw_current_version_ind = true
  and vm.dw_logical_delete_ind = false
  left outer join EDM_VIEWS_PRD.DW_VIEWS.Retail_Section r on r.Section_Cd = cat.Section_Cd
  and r.dw_current_version_ind = true
  and r.dw_logical_delete_ind = false
  left outer join EDM_VIEWS_PRD.DW_VIEWS.department dept on dept.department_id = r.Retail_Department_Id
  and dept.dw_current_version_ind = true
  and dept.dw_logical_delete_ind = false
  left outer join EDM_VIEWS_PRD.DW_VIEWS.Section sc on sc.section_cd = r.section_cd
  and sc.dw_current_version_ind = true
  and sc.dw_logical_delete_ind = false
  left outer join EDM_VIEWS_PRD.DW_VIEWS.Item_forecast ifc on wc.division_id = ifc.division_id
  and wc.warehouse_id = ifc.warehouse_id
  and wc.corporate_item_integration_id = ifc.corporate_item_integration_id
  and ifc.dw_current_version_ind = true
  and ifc.dw_logical_delete_ind = false
where
  1 = 1
  and sci.distribution_center_id like 'W%'
  and whse.WAREHOUSE_ID is not null
  and ci.corporate_item_cd is not null
  and b.BUYER_ID is not null
  and (
    vm.vendor_status_cd = 'A'
    or vm.vendor_status_cd is null
  )
  
~~~

### Fresh Order by Vendor 
~~~sql 
WITH STORE AS (
            SELECT FACILITY_INTEGRATION_ID, DIVISION_ID, ROG_ID, RETAIL_STORE_FACILITY_NBR AS STORE_ID
            FROM EDM_VIEWS_PRD.DW_VIEWS.D1_RETAIL_STORE 
            WHERE CORPORATION_ID = '001'
            AND DIVISION_ID IN ('05')    -----05,17,19,20,24,25,27,29,30,32,33,34
            AND FACILITY_STATUS_CD = 'OPEN'
            --AND RETAIL_STORE_FACILITY_NBR in ('1953')
            )

, DATE AS (
           SELECT FISCAL_WEEK_ID, FISCAL_WEEK_START_DT, FISCAL_WEEK_END_DT
           FROM EDM_VIEWS_PRD.DW_VIEWS.D0_FISCAL_WEEK
           --WHERE FISCAL_WEEK_END_DT BETWEEN CURRENT_DATE - 182 AND CURRENT_DATE    -----6months
           WHERE FISCAL_YEAR_NBR IN (2021)
           AND FISCAL_PERIOD_END_DT < CURRENT_DATE 
 )
			
, UPC_STR AS (	------pulls only UPC/Stores where the item has either SOLD or SHIPPED in the last 18 months 
	SELECT DISTINCT 
	  UPC.UPC_NBR
	, STR.FACILITY_INTEGRATION_ID
	, STR.ROG_ID
	, STR.DIVISION_ID
, STR.STORE_ID
	
FROM EDM_VIEWS_PRD.DW_VIEWS.D1_UPC UPC
		
	JOIN STORE STR
	ON 1=1
		
	JOIN DATE DTE
	ON 1=1
		
	LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.STORE_UPC_AGP AGP
	ON AGP.FACILITY_INTEGRATION_ID = STR.FACILITY_INTEGRATION_ID
	AND AGP.UPC_NBR = UPC.UPC_NBR
	AND AGP.TRANSACTION_DT >= FISCAL_WEEK_START_DT - 526
		
	LEFT JOIN EDM_VIEWS_PRD.DW_EDW_VIEWS.STORE_ITEM_SHIP_DAY SHP
	ON SHP.STORE_ID = STR.STORE_ID
	AND SHP.UPC_ID = UPC.UPC_NBR
	AND SHP.TXN_DT >= DTE.FISCAL_WEEK_START_DT - 526
		
	WHERE DW_LOGICAL_DELETE_IND = 'FALSE'
	AND SECTION_CD IN (
                    '306' --FOOD SERVICE                
                        ,	'309' --DELI                     
                        ,	'315' --FLORAL                   
                        ,	'316' --IN STORE BAKERY          
                        ,	'336' --BAKERY PACKAGED OUTSIDE  
                        ,	'329' --PRODUCE                  
                        ,	'330','331' --SEAFOOD            
                        ,	'333','334' --MEAT                
                        ,	'341','342' --JAMBA & CATERING  				
                            )				
		--AND  SECTION_CD = '309'     -----if filtering for a single department
		--and upc.upc_nbr = 81781801199     -----if filtering for a single UPC
	
	AND (AGP.UPC_NBR IS NOT NULL
	   OR SHP.UPC_ID IS NOT NULL)
	)

, ROG_UPC AS (	
	SELECT DISTINCT
	  US.UPC_NBR
	, US.ROG_ID
                    , US.DIVISION_ID
						
	FROM UPC_STR US

	JOIN DATE DTE
	ON 1=1

	JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_ROG_REFERENCE ROG
	ON ROG.ROG_ID = US.ROG_ID
	AND ROG.UPC_NBR = US.UPC_NBR
	AND ROG.DW_FIRST_EFFECTIVE_DT <= DTE.FISCAL_WEEK_END_DT
	AND ROG.DW_LAST_EFFECTIVE_DT >= DTE.FISCAL_WEEK_START_DT
	)    

, ROG_CIC AS (	 ------pulls by ROG the UPC/CIC, including the appropriate Primal CIC for cuts
	SELECT DISTINCT
	  US.ROG_ID
	, COALESCE(BMI.PRIMAL_CORPORATE_ITEM_INTEGRATION_ID, ROG.CORPORATE_ITEM_INTEGRATION_ID) AS CORPORATE_ITEM_INTEGRATION_ID
	, CASE WHEN BMI.PRIMAL_CORPORATE_ITEM_INTEGRATION_ID IS NOT NULL THEN 'W' ELSE PRODUCT_SOURCE_IND END AS PRODUCT_SOURCE_IND
	, ROG.CORPORATE_ITEM_INTEGRATION_ID AS CII_ID
			
	FROM ROG_UPC US

	JOIN DATE DTE
	ON 1=1

	JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_ROG_REFERENCE ROG
	ON ROG.ROG_ID = US.ROG_ID
	AND ROG.UPC_NBR = US.UPC_NBR
	AND ROG.DW_FIRST_EFFECTIVE_DT <= DTE.FISCAL_WEEK_END_DT
	AND ROG.DW_LAST_EFFECTIVE_DT >= DTE.FISCAL_WEEK_START_DT

	LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.BASE_MEAT_ITEM_YIELD BMI
	ON BMI.DIVISION_ID = US.DIVISION_ID
	AND BMI.BASE_CORPORATE_ITEM_INTEGRATION_ID = ROG.CORPORATE_ITEM_INTEGRATION_ID
				
	WHERE CASE WHEN BMI.PRIMAL_CORPORATE_ITEM_INTEGRATION_ID IS NOT NULL THEN 'W' ELSE PRODUCT_SOURCE_IND END IS NOT NULL
	)            

, DEFAULT_VENDOR AS (   
	SELECT  DISTINCT
	  STR.FACILITY_INTEGRATION_ID, RUS.ROG_ID
	, RUS.CORPORATE_ITEM_INTEGRATION_ID, RUS.CII_ID  
	, DTE.FISCAL_WEEK_ID				
	, AUT.VENDOR_ID, AUT.VENDOR_SUB_ACCOUNT_ID
	, 'AUT' AS SOURCE
                    , 0 AS RNK_DV

	FROM ROG_CIC RUS
			
            JOIN STORE STR
            ON STR.ROG_ID = RUS.ROG_ID

	JOIN DATE DTE
	ON 1=1

	JOIN EDM_VIEWS_PRD.DW_VIEWS.AUTHORIZED_BACKDOOR_RECEIVING_ITEM AUT
	ON AUT.CORPORATE_ITEM_INTEGRATION_ID = RUS.CORPORATE_ITEM_INTEGRATION_ID
	AND AUT.FACILITY_INTEGRATION_ID = STR.FACILITY_INTEGRATION_ID
	AND AUT.AUTHORIZED_ITEM_START_DT <= DTE.FISCAL_WEEK_END_DT
	AND AUT.AUTHORIZED_ITEM_END_DT >= DTE.FISCAL_WEEK_START_DT
	AND AUT.DW_LOGICAL_DELETE_IND = 'FALSE'
	AND AUT.DW_FIRST_EFFECTIVE_DT <= DTE.FISCAL_WEEK_END_DT		-----ensuring we get all the CIC/UPC relationships over time
	AND AUT.DW_LAST_EFFECTIVE_DT >= DTE.FISCAL_WEEK_START_DT

	UNION ALL

	SELECT  DISTINCT
	  STR.FACILITY_INTEGRATION_ID, RUS.ROG_ID
	, RUS.CORPORATE_ITEM_INTEGRATION_ID, RUS.CII_ID  
	, DTE.FISCAL_WEEK_ID				
, VI.VENDOR_ID, VI.VENDOR_SUB_ACCT_ID
	, 'WRG' AS SOURCE
                   , RANK() OVER(PARTITION BY RUS.ROG_ID, RUS.CORPORATE_ITEM_INTEGRATION_ID, RUS.CII_ID, DTE.FISCAL_WEEK_ID
                                              ORDER BY VI.DW_FIRST_EFFECTIVE_DT DESC) AS RNK_DV

	FROM ROG_CIC RUS
			
            JOIN STORE STR
            ON STR.ROG_ID = RUS.ROG_ID

	JOIN DATE DTE
	ON 1=1

	JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM C
	ON C.CORPORATE_ITEM_INTEGRATION_ID = RUS.CORPORATE_ITEM_INTEGRATION_ID
                   AND C.DW_CURRENT_VERSION_IND = 'TRUE'
	--AND C.DW_LOGICAL_DELETE_IND = 'FALSE'
	--AND C.DW_FIRST_EFFECTIVE_DT <= DTE.FISCAL_WEEK_END_DT	-----ensuring we get all the CIC/UPC relationships over time
	--AND C.DW_LAST_EFFECTIVE_DT >= DTE.FISCAL_WEEK_START_DT
							
	LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP_CORPORATE_ITEM ROG
	ON ROG.CORPORATE_ITEM_INTEGRATION_ID = RUS.CORPORATE_ITEM_INTEGRATION_ID
                    AND ROG.ROG_ID = RUS.ROG_ID
	AND (ROG.PRODUCT_SOURCE_IND = 'W'
	   OR (ROG.PRODUCT_SOURCE_IND = 'D' AND C.ITEM_USAGE_TYPE_CD = 'S'))
	--AND ROG.DW_LOGICAL_DELETE_IND = 'FALSE'
                    AND ROG.DW_CURRENT_VERSION_IND = 'TRUE'
	--AND ROG.DW_FIRST_EFFECTIVE_DT <= DTE.FISCAL_WEEK_END_DT	-----ensuring we get all the CIC/UPC relationships over time
	--AND ROG.DW_LAST_EFFECTIVE_DT >= DTE.FISCAL_WEEK_START_DT
							
	JOIN EDM_VIEWS_PRD.DW_VIEWS.VENDOR_ITEM VI			
   ----- ADDED 02/15/23 TO ACCOMODATE FOR ROWS WITH MISSING VENDORS, THAT NEVER SHIPPED, HAVEN'T SHIPPED YET, OR BEFORE THEY SHIPPED BUT WERE VALID
   ----DOES THIS NEED TO MOVE TO THE END AS A LEFT JOIN, NEED TO TEST TO MAKE SURE THIS IS NOT CAUSING ROWS TO DROP
	ON VI.CORPORATE_ITEM_INTEGRATION_ID = RUS.CORPORATE_ITEM_INTEGRATION_ID
	AND VI.WAREHOUSE_ID = ROG.WAREHOUSE_ID
	--AND VI.DW_FIRST_EFFECTIVE_DT <= DTE.FISCAL_WEEK_END_DT
	--AND VI.DW_LAST_EFFECTIVE_DT >= DTE.FISCAL_WEEK_START_DT
	AND VI.DW_LOGICAL_DELETE_IND = 'FALSE'        
                   AND VI.DW_CURRENT_VERSION_IND = 'TRUE'

            QUALIFY RNK_DV = 1
  )

SELECT FI.*
, COALESCE(VEN.VENDOR_OUTLET_NM, CASE WHEN FI.VENDOR_ID = '000000' THEN 'OWN BRANDS' ELSE  ' ' END) AS VENDOR_OUTLET_NM
FROM ( 
SELECT  DISTINCT
  STR.DIVISION_ID
, STR.ROG_ID
, STR.STORE_ID
, VLD.FACILITY_INTEGRATION_ID
, CUR.RETAIL_SECTION_CD
, CUR.UPC_NBR
--, RUS.CII_ID
, CIC.ITEM_USAGE_CD
, CIC.ITEM_USAGE_TYPE_CD
, CIC.CORPORATE_ITEM_CD
, CIC.ITEM_DSC
, DTE.FISCAL_WEEK_ID
, CUR.PRODUCT_SOURCE_IND
, COALESCE(FIN.SOURCE, VLD.SOURCE) AS SOURCEX

, CU.PREFERED_CORPORATE_ITEM_SEQ_NBR AS PREF_CIC
, CASE WHEN SOURCEX = 'BDR' THEN 1
            WHEN SOURCEX = 'WHS' THEN 2
            WHEN SOURCEX = 'AUT' THEN 3
            WHEN SOURCEX = 'WRG' THEN 4
       ELSE 9 END AS PREF_SRC

----this ranking puts DSD above WHSE if they ship in the same week to a given store using source and pref_cic and product_source_ind to rank
, RANK() OVER(PARTITION BY VLD.FACILITY_INTEGRATION_ID, CUR.UPC_NBR, DTE.FISCAL_WEEK_ID		
 	        ORDER BY PREF_SRC ASC, PREF_CIC ASC, CUR.PRODUCT_SOURCE_IND, RECV DESC) AS RNK
															
--------the LAG function allows you to pull the value from the prior row to propogate this value until it changes again (this fills in the NULLS or the default supplier
, COALESCE(FIN.VENDOR_ID, CASE WHEN FIN.VENDOR_ID IS NULL AND CIC.ITEM_USAGE_TYPE_CD = 'S' THEN '000000'
			   WHEN FIN.VENDOR_ID IS NULL THEN LAG(FIN.VENDOR_ID) IGNORE NULLS OVER (PARTITION BY VLD.FACILITY_INTEGRATION_ID, 
CUR.UPC_NBR ORDER BY DTE.FISCAL_WEEK_ID, RECV DESC)
			ELSE VLD.VENDOR_ID END, VLD.VENDOR_ID) AS VENDOR_ID

, COALESCE(FIN.VENDOR_SUB_ACCOUNT_ID, CASE WHEN FIN.VENDOR_SUB_ACCOUNT_ID IS NULL AND CIC.ITEM_USAGE_TYPE_CD = 'S' THEN '000'
				             WHEN FIN.VENDOR_SUB_ACCOUNT_ID IS NULL THEN LAG(FIN.VENDOR_SUB_ACCOUNT_ID) IGNORE NULLS 
OVER (PARTITION BY VLD.FACILITY_INTEGRATION_ID, CUR.UPC_NBR ORDER BY DTE.FISCAL_WEEK_ID, RECV DESC)
				ELSE VLD.VENDOR_SUB_ACCOUNT_ID END, VLD.VENDOR_SUB_ACCOUNT_ID) AS VENDOR_SUB_ACCOUNT_ID		

FROM UPC_STR STR

JOIN DATE DTE
ON 1=1

JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_ROG_REFERENCE CUR
ON CUR.ROG_ID = STR.ROG_ID
AND CUR.UPC_NBR = STR.UPC_NBR
AND CUR.DW_FIRST_EFFECTIVE_DT <= DTE.FISCAL_WEEK_END_DT
AND CUR.DW_LAST_EFFECTIVE_DT >= DTE.FISCAL_WEEK_START_DT

JOIN EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_REFERENCE CU	---pull this table in to get the pref cic seq nbr to use in ranking by week for dsd/whse/source/cic
ON CU.CORPORATE_ITEM_INTEGRATION_ID = CUR.CORPORATE_ITEM_INTEGRATION_ID
AND CU.UPC_NBR = CUR.UPC_NBR
AND CU.DW_FIRST_EFFECTIVE_DT <= DTE.FISCAL_WEEK_END_DT
AND CU.DW_LAST_EFFECTIVE_DT >= DTE.FISCAL_WEEK_START_DT

JOIN DEFAULT_VENDOR VLD
ON VLD.ROG_ID = STR.ROG_ID
AND VLD.FACILITY_INTEGRATION_ID = STR.FACILITY_INTEGRATION_ID
AND VLD.CII_ID = CUR.CORPORATE_ITEM_INTEGRATION_ID
AND VLD.FISCAL_WEEK_ID = DTE.FISCAL_WEEK_ID

JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_CORPORATE_ITEM CIC
ON CIC.CORPORATE_ITEM_INTEGRATION_ID = CUR.CORPORATE_ITEM_INTEGRATION_ID

LEFT JOIN ( 
SELECT -----pulling warehouse vendors by store based on Whse POs Received into warehouse and verifying that this same CIC shipped to the store (see EXISTS statement)   note:  might need to think about the date logic on the ship to store vs the whse po receive
	  'WHS' AS SOURCE
	, DIVISION_ID
	, ROG_ID
	, STORE_ID
	, FACILITY_INTEGRATION_ID
	--, RETAIL_SECTION_CD
	--, PRODUCT_SOURCE_IND
	--, UPC_NBR
	, CORPORATE_ITEM_INTEGRATION_ID
	, CII_ID
	, FISCAL_WEEK_ID
	, VENDOR_ID
	, VENDOR_SUB_ACCOUNT_ID
	, RECV

	FROM (
	SELECT 
	  TB1.*
	, RANK() OVER(PARTITION BY TB1.FACILITY_INTEGRATION_ID, CORPORATE_ITEM_INTEGRATION_ID, FISCAL_WEEK_ID
	ORDER BY RECV DESC, DTE DESC, PURCHASE_ORDER_NBR DESC) AS RNK -----when multiple vendors delivery on the same day same amount, added PO#

	FROM ( 
SELECT  DISTINCT
  ROG.DIVISION_ID
	, ROG.ROG_ID
	, STR.STORE_ID
	, STR.FACILITY_INTEGRATION_ID
	, HED.DISTRIBUTION_CENTER_ID
	, ROG.WAREHOUSE_ID
	, HED.VENDOR_ID
	, HED.VENDOR_SUB_ACCOUNT_ID
	, RUS.CII_ID
	, RUS.CORPORATE_ITEM_INTEGRATION_ID
	, DET.PURCHASE_ORDER_NBR
	, DTE.FISCAL_WEEK_ID

	, SUM(DET.RECEIVED_QTY) OVER(PARTITION BY ROG.WAREHOUSE_ID, RUS.CORPORATE_ITEM_INTEGRATION_ID, DTE.FISCAL_WEEK_ID, HED.VENDOR_ID, HED.VENDOR_SUB_ACCOUNT_ID)	AS RECV
	, MAX(HED.PURCHASE_ORDER_RECEIVE_DT) OVER(PARTITION BY  ROG.WAREHOUSE_ID, RUS.CORPORATE_ITEM_INTEGRATION_ID, DTE.FISCAL_WEEK_ID, HED.VENDOR_ID, HED.VENDOR_SUB_ACCOUNT_ID) 	AS DTE

	FROM DATE DTE
								
	JOIN ROG_CIC RUS
	ON RUS.PRODUCT_SOURCE_IND = 'W'
								
	JOIN STORE STR
	ON STR.ROG_ID = RUS.ROG_ID
	                  
	JOIN EDM_VIEWS_PRD.DW_VIEWS.RETAIL_ORDER_GROUP_CORPORATE_ITEM ROG
	ON ROG.CORPORATE_ITEM_INTEGRATION_ID = RUS.CORPORATE_ITEM_INTEGRATION_ID
	AND ROG.ROG_ID = RUS.ROG_ID
	AND ROG.DW_LOGICAL_DELETE_IND = 'FALSE'
	AND ROG.DW_CURRENT_VERSION_IND = 'TRUE'
	                  
	JOIN EDM_VIEWS_PRD.DW_VIEWS.WAREHOUSE_EXTENDED_RECEIVING_DETAIL DET
	ON DET.CORPORATE_ITEM_INTEGRATION_ID = ROG.CORPORATE_ITEM_INTEGRATION_ID
	AND DET.DIVISION_ID = ROG.ORDERING_DIVISION_ID
	AND DET.WAREHOUSE_ID = ROG.WAREHOUSE_ID
	AND DET.STATUS_CD = 'R'
	AND DET.RECEIVED_QTY > 0 
								
	JOIN EDM_VIEWS_PRD.DW_VIEWS.WAREHOUSE_EXTENDED_RECEIVING_HEADER HED
	ON  HED.CORPORATION_ID = DET.CORPORATION_ID
	AND HED.DIVISION_ID = DET.DIVISION_ID
	AND HED.WAREHOUSE_ID = DET.WAREHOUSE_ID
	AND HED.PURCHASE_ORDER_NBR = DET.PURCHASE_ORDER_NBR
	AND HED.PURCHASE_ORDER_RECEIVE_DT <= DTE.FISCAL_WEEK_END_DT
	AND HED.PURCHASE_ORDER_RECEIVE_DT >= DTE.FISCAL_WEEK_START_DT
								
	WHERE EXISTS (	------checking to see if the item has been shipped to the store, but need to adjust the dates to look beyond the range of the PO recv date
		SELECT 1
		FROM EDM_VIEWS_PRD.DW_VIEWS.SUPPLY_CHAIN_INVOICE_HEADER HED
		JOIN EDM_VIEWS_PRD.DW_VIEWS.SUPPLY_CHAIN_INVOICE_DETAIL DET
		ON DET.SUPPLY_CHAIN_INVOICE_INTEGRATION_ID = HED.SUPPLY_CHAIN_INVOICE_INTEGRATION_ID
		AND DET.CORPORATE_ITEM_INTEGRATION_ID = RUS.CORPORATE_ITEM_INTEGRATION_ID
		AND DET.SHIPPED_QTY <> 0
		AND DET.DW_CURRENT_VERSION_IND = 'TRUE'
		WHERE HED.REQUESTED_DELIVERY_DT <= DTE.FISCAL_WEEK_END_DT
----looking to see if the item has shipped to the store in the last year and a half
		AND HED.REQUESTED_DELIVERY_DT >= DTE.FISCAL_WEEK_START_DT - 526
		AND HED.RETAIL_STORE_FACILITY_INTEGRATION_ID = STR.FACILITY_INTEGRATION_ID
		AND HED.DW_CURRENT_VERSION_IND = 'TRUE'
		)
	) AS TB1
	) AS TB2

	WHERE RNK = 1	

	UNION ALL		   

SELECT  		-----pulling BDR vendors by store based on Invoices Received into store
	  'BDR' AS SOURCE
	, DIVISION_ID
	, ROG_ID
	, STORE_ID
	, FACILITY_INTEGRATION_ID
, UPC_NBR
 , UPC_NBR AS CII_ID
	, FISCAL_WEEK_ID
	, VENDOR_ID
	, VENDOR_SUB_ACCOUNT_ID
	, RECV

	FROM (
	SELECT 
	  TB1.*
----when multiple vendors delivery on the same day same amount (same ranking as warehouse without PO/invoice)
, RANK() OVER(PARTITION BY UPC_NBR, FISCAL_WEEK_ID ORDER BY RECV DESC, DTE DESC) AS RNK	

	FROM (											
	SELECT  DISTINCT
	  STR.DIVISION_ID
	, STR.ROG_ID
		, STR.STORE_ID
		, STR.FACILITY_INTEGRATION_ID
		, RUS.UPC_NBR
		, DTE.FISCAL_WEEK_ID
		, HED.VENDOR_ID
		, HED.VENDOR_SUB_ACCOUNT_ID
		, SUM(CASE WHEN DET.INVOICE_CD = 'D' AND FINAL_UNIT_OF_MEASURE_CD = 'C' THEN DET.FINAL_PACK_QTY * DET.FINAL_INVOICE_QTY
    	  		   WHEN DET.INVOICE_CD = 'D' AND FINAL_UNIT_OF_MEASURE_CD = 'E'  THEN DET.FINAL_INVOICE_QTY
			   WHEN DET.INVOICE_CD = 'D' AND FINAL_UNIT_OF_MEASURE_CD = 'L'   THEN DET.FINAL_INVOICE_QTY
			ELSE 0 END) OVER(PARTITION BY DET.UPC_NBR, DTE.FISCAL_WEEK_ID, HED.VENDOR_ID, HED.VENDOR_SUB_ACCOUNT_ID)        AS RECV
		, MAX(HED.RECEIVE_DT) OVER(PARTITION BY  DET.UPC_NBR, DTE.FISCAL_WEEK_ID, HED.VENDOR_ID, HED.VENDOR_SUB_ACCOUNT_ID)     AS DTE

		FROM STORE STR
		
JOIN ROG_UPC RUS
		ON RUS.ROG_ID = STR.ROG_ID
							
		JOIN DATE DTE
		ON 1=1

		JOIN EDM_VIEWS_PRD.DW_VIEWS.RECEIVE_DELIVERY_INVOICE_HEADER_DSD HED
		ON  HED.RECEIVE_DT <= DTE.FISCAL_WEEK_END_DT
		AND HED.RECEIVE_DT >= DTE.FISCAL_WEEK_START_DT
		AND HED.FACILITY_INTEGRATION_ID = STR.FACILITY_INTEGRATION_ID
		AND HED.INVOICE_CD = 'D'
		AND HED.INVOICE_VOID_IND = '0'
		AND HED.DW_CURRENT_VERSION_IND = 'TRUE'

		JOIN EDM_VIEWS_PRD.DW_VIEWS.RECEIVE_DELIVERY_INVOICE_DETAIL_DSD DET
		ON DET.FACILITY_INTEGRATION_ID = HED.FACILITY_INTEGRATION_ID
		AND DET.UPC_NBR = RUS.UPC_NBR
		AND DET.VENDOR_ID = HED.VENDOR_ID
		AND DET.BACKDOOR_VENDOR_SUB_ACCOUNT_ID = HED.BACKDOOR_VENDOR_SUB_ACCOUNT_ID
		AND DET.LOAD_DT = HED.LOAD_DT
		AND DET.INVOICE_CD = HED.INVOICE_CD
		AND DET.VENDOR_INVOICE_NBR = HED.VENDOR_INVOICE_NBR
		AND DET.ITEM_USAGE_TYPE_CD = 'R'
		AND DET.FINAL_RETAIL_PRICE_AMT <> 0
		AND DET.FINAL_PACK_QTY <> 0 
		AND DET.FINAL_INVOICE_QTY <> 0
		AND DET.DW_CURRENT_VERSION_IND = 'TRUE'
		AND DET.DW_LOGICAL_DELETE_IND = 'FALSE'
		AND DET.UPC_NBR IN (SELECT CUR.UPC_NBR
				  FROM EDM_VIEWS_PRD.DW_VIEWS.CORPORATE_ITEM_UPC_ROG_REFERENCE CUR
																			WHERE CUR.ROG_ID = STR.ROG_ID
				AND CUR.UPC_NBR = DET.UPC_NBR
				AND CUR.STATUS_CD <> 'D'
				AND CUR.DW_CURRENT_VERSION_IND = 'TRUE'
				AND CUR.DW_LOGICAL_DELETE_IND = 'FALSE' )

				WHERE 1=1
	) AS TB1
	) AS TB2
		
	WHERE RNK = 1	

UNION ALL

SELECT   		-----pulling SBT vendors by store based on actual authorization to SBT Vendor
	  'SBT' AS SOURCE
	, STR.DIVISION_ID
	, STR.ROG_ID
	, STR.STORE_ID
	, STR.FACILITY_INTEGRATION_ID
	, RUS.CORPORATE_ITEM_INTEGRATION_ID
	, CII_ID
	, DTE.FISCAL_WEEK_ID
	, AUT.VENDOR_ID
	, AUT.VENDOR_SUB_ACCOUNT_ID
	, 0 AS RECV

	FROM STORE STR

	JOIN ROG_CIC RUS
	ON RUS.ROG_ID = STR.ROG_ID
	
	JOIN DATE DTE
	ON 1=1

JOIN EDM_VIEWS_PRD.DW_VIEWS.AUTHORIZED_BACKDOOR_RECEIVING_ITEM AUT
	ON AUT.CORPORATE_ITEM_INTEGRATION_ID = RUS.CORPORATE_ITEM_INTEGRATION_ID
	AND AUT.FACILITY_INTEGRATION_ID = STR.FACILITY_INTEGRATION_ID
	AND AUT.AUTHORIZED_ITEM_START_DT <= DTE.FISCAL_WEEK_END_DT
	AND AUT.AUTHORIZED_ITEM_END_DT > DTE.FISCAL_WEEK_START_DT
	AND AUT.DW_CURRENT_VERSION_IND = 'TRUE'
	AND AUT.DW_LOGICAL_DELETE_IND = 'FALSE'

	JOIN (
	   --------the top union gets the SBT vendors that are authorized to ALL stores without any exceptions at the store level
		SELECT
		  STR.ROG_ID
		, STR.FACILITY_INTEGRATION_ID
		, STR.STORE_ID
		, SBT.VENDOR_ID
		, SBT.VENDOR_SUB_ACCOUNT_ID
		, SBT.PAYMENT_START_DT, SBT.PAYMENT_END_DT

		FROM EDM_VIEWS_PRD.DW_VIEWS.SCAN_BASED_TRADING_VENDOR SBT

		JOIN EDM_VIEWS_PRD.DW_VIEWS.VENDOR_COST_AREA_FACILITY CAF
		ON CAF.CORPORATION_ID = SBT.CORPORATION_ID
		AND CAF.DIVISION_ID = SBT.DIVISION_ID
		AND CAF.VENDOR_ID = SBT.VENDOR_ID
		AND CAF.VENDOR_SUB_ACCOUNT_ID = SBT.VENDOR_SUB_ACCOUNT_ID
		AND (CAF.ON_HOLD_IND <> 'H' OR CAF.ON_HOLD_IND IS NULL)
		AND CAF.DW_CURRENT_VERSION_IND = 'TRUE'

		JOIN STORE STR
		ON STR.FACILITY_INTEGRATION_ID = CAF.FACILITY_INTEGRATION_ID
			        
		WHERE SBT.PAYMENT_START_DT < CURRENT_DATE
		AND SBT.PAYMENT_END_DT > CURRENT_DATE
		AND SBT.DW_CURRENT_VERSION_IND = 'TRUE'
			  
		AND NOT EXISTS (SELECT 1
			            FROM EDM_VIEWS_PRD.DW_VIEWS.SCAN_BASED_TRADING_VENDOR_FACILITY SBF
			            WHERE  SBF.CORPORATION_ID = SBT.CORPORATION_ID
			            AND SBF.DIVISION_ID = SBT.DIVISION_ID
			            AND SBF.VENDOR_ID = SBT.VENDOR_ID
			            AND SBF.VENDOR_SUB_ACCOUNT_ID = SBT.VENDOR_SUB_ACCOUNT_ID
			            AND SBF.FACILITY_INTEGRATION_ID = CAF.FACILITY_INTEGRATION_ID
			            AND ((SBF.PAYMENT_CD = 'I' AND SBT.PAYMENT_END_DT > CURRENT_DATE)
			                OR (SBF.PAYMENT_CD = 'E' )	
			                OR (SBF.PRODUCT_ACTIVITY_CD = 'E' AND SBF.PRODUCT_ACTIVITY_END_DT > CURRENT_DATE)
			                   )            
			             AND SBF.DW_CURRENT_VERSION_IND = 'TRUE')

		GROUP BY 1, 2, 3, 4, 5, 6, 7

		UNION ALL
			--------the 2nd union gets the SBT vendors that store level exceptions
		SELECT 
		  STR.ROG_ID
		, STR.FACILITY_INTEGRATION_ID
		, STR.STORE_ID
		, SBT.VENDOR_ID
		, SBT.VENDOR_SUB_ACCOUNT_ID
		, CASE WHEN SBF.PAYMENT_CD = 'E' AND SBF.PAYMENT_END_DT <= CURRENT_DATE THEN SBF.PAYMENT_END_DT
            WHEN SBF.PAYMENT_CD = 'I' AND SBF.PAYMENT_END_DT >= CURRENT_DATE THEN SBF.PAYMENT_START_DT
		        ELSE SBF.PAYMENT_START_DT END AS PAYMENT_START_DT
		, CASE WHEN SBF.PAYMENT_CD = 'E' AND SBF.PAYMENT_END_DT <= CURRENT_DATE THEN SBT.PAYMENT_END_DT
		             WHEN SBF.PAYMENT_CD = 'I' AND SBF.PAYMENT_END_DT >= CURRENT_DATE THEN SBF.PAYMENT_END_DT
		       ELSE SBF.PAYMENT_END_DT END AS PAYMENT_END_DT

		FROM EDM_VIEWS_PRD.DW_VIEWS.SCAN_BASED_TRADING_VENDOR SBT

		JOIN EDM_VIEWS_PRD.DW_VIEWS.VENDOR_COST_AREA_FACILITY CAF
		ON CAF.CORPORATION_ID = SBT.CORPORATION_ID
		AND CAF.DIVISION_ID = SBT.DIVISION_ID
		AND CAF.VENDOR_ID = SBT.VENDOR_ID
		AND CAF.VENDOR_SUB_ACCOUNT_ID = SBT.VENDOR_SUB_ACCOUNT_ID
		AND (CAF.ON_HOLD_IND <> 'H' OR CAF.ON_HOLD_IND IS NULL)
		AND CAF.DW_CURRENT_VERSION_IND = 'TRUE'

		JOIN STORE STR
		ON STR.FACILITY_INTEGRATION_ID = CAF.FACILITY_INTEGRATION_ID
			        
		JOIN EDM_VIEWS_PRD.DW_VIEWS.SCAN_BASED_TRADING_VENDOR_FACILITY SBF
		ON  SBF.CORPORATION_ID = SBT.CORPORATION_ID
		AND SBF.DIVISION_ID = SBT.DIVISION_ID
		AND SBF.VENDOR_ID = SBT.VENDOR_ID
		AND SBF.VENDOR_SUB_ACCOUNT_ID = SBT.VENDOR_SUB_ACCOUNT_ID
		AND SBF.FACILITY_INTEGRATION_ID = CAF.FACILITY_INTEGRATION_ID
		AND ( (SBF.PAYMENT_CD = 'I' AND SBF.PAYMENT_START_DT <= CURRENT_DATE AND SBF.PAYMENT_END_DT >= CURRENT_DATE)
		    OR  (SBF.PAYMENT_CD = 'E'  AND SBF.PAYMENT_END_DT <= CURRENT_DATE)  )
		AND SBF.DW_CURRENT_VERSION_IND = 'TRUE'

		WHERE SBT.SALES_COLLECTION_START_DT < CURRENT_DATE
		AND SBT.SALES_COLLECTION_END_DT > CURRENT_DATE
		AND SBT.DW_CURRENT_VERSION_IND = 'TRUE'
				
		GROUP BY 1, 2, 3, 4, 5, 6, 7
		) AS DVS
		ON   DVS.PAYMENT_START_DT <= DTE.FISCAL_WEEK_END_DT
		AND DVS.PAYMENT_END_DT >= DTE.FISCAL_WEEK_START_DT
		AND DVS.FACILITY_INTEGRATION_ID = STR.FACILITY_INTEGRATION_ID
		AND DVS.VENDOR_ID = AUT.VENDOR_ID
		AND DVS.VENDOR_SUB_ACCOUNT_ID = AUT.VENDOR_SUB_ACCOUNT_ID

	WHERE 1=1

	GROUP BY 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
) AS FIN
ON FIN.FACILITY_INTEGRATION_ID = VLD.FACILITY_INTEGRATION_ID
AND FIN.FISCAL_WEEK_ID = DTE.FISCAL_WEEK_ID
AND (
(FIN.SOURCE = 'BDR' AND FIN.CII_ID = CUR.UPC_NBR )
             OR (FIN.SOURCE IN ('SBT','WHS') AND FIN.CII_ID = CUR.CORPORATE_ITEM_INTEGRATION_ID)
     )

QUALIFY RNK = 1
) AS FI

LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.VENDOR_OUTLET VEN
ON VEN.VENDOR_ID = FI.VENDOR_ID
AND VEN.VENDOR_SUB_ACCOUNT_ID = FI.VENDOR_SUB_ACCOUNT_ID
AND VEN.DW_CURRENT_VERSION_IND = 'TRUE'
AND VEN.DW_LOGICAL_DELETE_IND = 'FALSE'
);
~~~