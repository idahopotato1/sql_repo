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