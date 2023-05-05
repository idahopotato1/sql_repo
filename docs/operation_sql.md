# Operation SQL

## Store Distress by Quarter Period Department Item
<!-- === "Hourly Sales" -->

~~~ sql
SELECT
// EDW VERSIONS THAT WERE CONVERTED ARE IN COMMENTS//
PD.FISCAL_PERIOD_NBR AS PERIOD,            --LU.PERIOD_ID as PERIOD,
PD.FISCAL_QUARTER_NBR AS QUARTER,            --LU.QUARTER_ID as QUARTER,
STR.DISTRICT_CD AS DISTRICT,            --district_cd AS DISTRICT,
STR.RETAIL_STORE_FACILITY_NBR AS STORE,            --a.store_id AS STORE,
ROG.DEPARTMENT_ID ||' '||ROG.SMIC_GROUP_DSC AS DEPARTMENT,            --b.department_id ||' '|| b.department_nm as DEPARTMENT,
ROG.CATEGORY_ID||' '||ROG.SMIC_CATEGORY_DSC AS CATEGORY,            --b.CATEGORY_ID ||' '|| B.CATEGORY_NM AS CATEGORY,
ROG.CLASS_ID||' '||ROG.SMIC_CLASS_DSC AS CLASS,            --B.CLASS_ID ||' '|| B.CLASS_NM AS CLASS,
A.UPC_ID||' '||CI.ITEM_DSC||' - CIC '||CI.CORPORATE_ITEM_CD AS DESCRIPTION,            --B.UPC_ID ||' '|| B.UPC_DSC ||' - CIC '||B.CORP_ITEM_CD AS DESCRIPTION,
SUM(NET_AMT) AS DISTRESS_TY            --sum(net_amt) as DISTRESS_TY
            
FROM EDM_VIEWS_PRD.DW_EDW_VIEWS.ITEM_DISTRESS_AGG A            --from dw_dss.item_distress_agg a
JOIN DW_VIEWS.RETAIL_ORDER_GROUP_UPC_EXTENDED ROG            --JOIN dw_dss.lu_upc b
ON A.UPC_ID = ROG.UPC_NBR            --ON a.upc_id = b.upc_id

    JOIN DW_VIEWS.CORPORATE_ITEM CI
    ON ROG.COMMON_RETAIL_CD = CI.COMMON_RETAIL_CD
    AND CI.CORPORATE_ITEM_CD = ROG.PREFERRED_CORPORATE_ITEM_CD 

JOIN DW_VIEWS.D1_RETAIL_STORE STR            --JOIN dw_dss.lu_store c
ON A.STORE_ID = STR.RETAIL_STORE_FACILITY_NBR            --ON a.store_id = c.store_id
AND ROG.ROG_ID = STR.ROG_ID             --AND b.corporation_id = c.corporation_id
            
JOIN DW_VIEWS.CALENDAR CAL            --JOIN DW_DSS.LU_DAY_MERGE LU
ON A.DISTRESS_DT = CAL.CALENDAR_DT            --ON a.DISTRESS_DT = LU.D_DATE

JOIN DW_VIEWS.FISCAL_PERIOD PD
ON CAL.FISCAL_PERIOD_NBR = PD.FISCAL_PERIOD_NBR
AND CAL.FISCAL_YEAR_NBR = PD.FISCAL_YEAR_NBR 
            
WHERE STR.CORPORATION_ID = '001'            --where b.corporation_id =1
AND ROG.DEPARTMENT_ID IN (306,309)            --and department_id in (306,309)
AND STR.DIVISION_ID = '32'             --and division_id = 32
AND STR.DIVISION_NM LIKE '%JEWEL%'
AND CAL.CALENDAR_DT BETWEEN '04/01/2022' and CURRENT_DATE-1            --and d_date between '04/01/2022' and current_date-1
AND ROG.DW_CURRENT_VERSION_IND = TRUE
AND STR.DW_LOGICAL_DELETE_IND = FALSE
AND PD.DW_CURRENT_VERSION_IND = TRUE
AND PD.DW_LOGICAL_DELETE_IND = FALSE
AND CAL.DW_CURRENT_VERSION_IND = TRUE
AND CAL.DW_LOGICAL_DELETE_IND = FALSE
AND CI.DW_CURRENT_VERSION_IND = TRUE
AND CI.DW_LOGICAL_DELETE_IND = FALSE
group by 1,2,3,4,5,6,7,8
order by 1,3,4,8;
~~~

### Far PS Queries 
~~~ sql
STEP 1 - 
create or replace table EDM_BIZOPS_PRD.MERCHAPPS.STORE_PS_ITEM_STORE_PLANOGRAM_T as
 SELECT  z.store_upc store_upc
,        z.store_id store_id
,        z.rog_id
,        z.section_CD as DEPT_SECTION_ID
,        z.upc_nbr upc_nbr 
,        z.corporate_item_cd corporate_item_cd
,        z.item_dsc
,        z.size_qty
,        z.size_uom_cd
,        z.Fixture_Id
,        z.HORIZONTAL_CNT HORIZONTAL_CNT
,        z.VERTICAL_CNT VERTICAL_CNT
--                  ,        z.bay_shelf_nbr
,        z.STORE_SCHEMATIC_EFFECTIVE_START_DT 
,        z.STORE_SCHEMATIC_EFFECTIVE_END_DT 
--,        z.large_pet_supply_3
,        z.large_pet_supply_20
,        z.spice_pack_ind
--                  ,        max(z.bay_shelf_nbr) OVER (PARTITION BY z.fxtr_cd) AS max_bay_shelf_nbr
FROM
(SELECT a.Retail_store_facility_nbr || '-' || u.upc_nbr AS store_upc
,               a.Retail_store_facility_nbr as store_id
,               a.rog_id
,               u.section_CD
,               u.CONSUMER_SELLING_CD
,               u.upc_nbr
,               u.corporate_item_cd
,               u.item_dsc
,               c.size_qty
,               c.size_uom_cd
,               psi.Fixture_Id
--                  ,               pfs.shelf_y_coord_nbr
,               psi.HORIZONTAL_CNT
,               psi.VERTICAL_CNT
,               psi.STORE_SCHEMATIC_EFFECTIVE_START_DT 
,               psi.STORE_SCHEMATIC_EFFECTIVE_END_DT 
,               CASE WHEN u.SMIC_category_id = 525 OR u.SMIC_class_id = 53015 THEN 'Y' ELSE 'N' END AS spice_pack_ind
--,               CASE WHEN c.Corporate_Item_Integration_Id IS NOT NULL THEN 'Y' ELSE 'N' END AS large_pet_supply_3
,               CASE WHEN c2.Corporate_Item_Integration_Id IS NOT NULL THEN 'Y' ELSE 'N' END AS large_pet_supply_20
--                  ,               dense_rank() OVER (PARTITION BY pfs.fxtr_cd ORDER BY pfs.shelf_y_coord_nbr) AS bay_shelf_nbr                                                                                                                                                                                               

FROM
	(SELECT * FROM  
	(SELECT F.Facility_NBR
,           u.upc_nbr
,           psi.PLANOGRAM_ID
,           psf.Fixture_Id
,           psi.HORIZONTAL_CNT
,           psi.VERTICAL_CNT
,           psf.STORE_SCHEMATIC_EFFECTIVE_START_DT 
,           psf.STORE_SCHEMATIC_EFFECTIVE_END_DT
FROM EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM_SLOT_ITEM psi
JOIN EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM_STORE_FIXTURE psf
ON psf.PLANOGRAM_ID = psi.PLANOGRAM_ID
JOIN EDM_VIEWS_PRD.DW_VIEWS.FACILITY F
ON psf.Facility_Integration_id = F.Facility_Integration_id
JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_UPC u
ON u.CONSUMER_SELLING_CD = psi.CONSUMER_SELLING_CD and u.CORPORATE_ITEM_CD > 0
WHERE 1=1
AND current_date BETWEEN psf.STORE_SCHEMATIC_EFFECTIVE_START_DT - 14 AND psf.STORE_SCHEMATIC_EFFECTIVE_END_DT - 14
AND psi.CONSUMER_SELLING_CD <> 0
AND psi.CONSUMER_SELLING_CD IS NOT NULL
and F.dw_logical_delete_ind = FALSE
and F.dw_current_version_ind = True
and psi.dw_logical_delete_ind = FALSE
and psi.dw_current_version_ind = True
and psf.dw_logical_delete_ind = FALSE
and psf.dw_current_version_ind = True
and F.Facility_type_cd ='RT'
AND F.corporation_id ='001'
--AND u.corporation_id ='001'
GROUP BY 1,2,3,4,5,6,7,8
UNION ALL
SELECT F.FACILITY_NBR
,               psi.upc_NBR
,               psi.PLANOGRAM_ID
,               psf.Fixture_Id
,               psi.HORIZONTAL_CNT
,               psi.VERTICAL_CNT
,              psf.STORE_SCHEMATIC_EFFECTIVE_START_DT
,              psf.STORE_SCHEMATIC_EFFECTIVE_END_DT
FROM EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM_SLOT_ITEM psi
JOIN EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM_STORE_FIXTURE psf
ON psf.PLANOGRAM_ID = psi.PLANOGRAM_ID
JOIN EDM_VIEWS_PRD.DW_VIEWS.FACILITY F
ON psf.Facility_Integration_id = F.Facility_Integration_id
WHERE 1=1
AND current_date BETWEEN psf.STORE_SCHEMATIC_EFFECTIVE_START_DT - 14 AND psf.STORE_SCHEMATIC_EFFECTIVE_END_DT - 14
AND (psi.CONSUMER_SELLING_CD = 0 OR psi.CONSUMER_SELLING_CD IS NULL)
and F.dw_logical_delete_ind = FALSE
and F.dw_current_version_ind = True
and psi.dw_logical_delete_ind = FALSE
and psi.dw_current_version_ind = True
and psf.dw_logical_delete_ind = FALSE
and psf.dw_current_version_ind = True
and F.Facility_type_cd ='RT'
AND F.corporation_id ='001'
GROUP BY 1,2,3,4,5,6,7,8) csc
GROUP BY 1,2,3,4,5,6,7,8) psi
	 JOIN EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM p
	   ON p.PLANOGRAM_ID = psi.PLANOGRAM_ID
		--AND p.Fixture_Id = psf.Fixture_Id                                                                                               
--        JOIN (
--        SELECT p.store_id
--    ,          p.pog_id
--    ,          p.pog_fxtr_start_dt
--    ,   p.pog_fxtr_end_dt
--       FROM EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM_STORE_FIXTURE p
--       WHERE current_date() BETWEEN p.pog_fxtr_start_dt - 14 AND p.pog_fxtr_end_dt - 14
--      GROUP BY 1,2,3,4
--      ) psf
--      ON psf.PLANOGRAM_ID = psi.PLANOGRAM_ID                                                                                               
--        JOIN EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM_fxtr_shelf pfs
--        ON pfs.fxtr_cd = p.fxtr_cd
--        AND pfs.fxtr_shelf_nbr = psi.fxtr_shelf_nbr                                                                                              
	 LEFT JOIN
	(SELECT F.FACILITY_NBR || '-' || psi.upc_NBR AS store_upc
,               F.FACILITY_NBR
,               psi.upc_NBR                                                                                                              
	 FROM EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM_STORE_FIXTURE psf
	 JOIN EDM_VIEWS_PRD.DW_VIEWS.FACILITY F
	 ON psf.Facility_Integration_id = F.Facility_Integration_id	 
	 JOIN EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM_SLOT_ITEM psi
	 ON psi.PLANOGRAM_ID = psf.PLANOGRAM_ID                                                                                    
	 JOIN EDM_VIEWS_PRD.DW_VIEWS.PLANOGRAM pog
	 ON pog.PLANOGRAM_ID = psf.PLANOGRAM_ID                                                                                                               
	 JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_UPC u
	 ON u.upc_nbr = psi.upc_NBR                                                                                                             
	 WHERE 1=1
	 AND current_date BETWEEN psf.STORE_SCHEMATIC_EFFECTIVE_START_DT - 14 AND psf.STORE_SCHEMATIC_EFFECTIVE_END_DT - 14
     and u.SAFEWAY_UPC_IND <> FALSE 
	 and F.dw_logical_delete_ind = FALSE
	 and F.dw_current_version_ind = True
	 and F.Facility_type_cd ='RT'
	 AND F.corporation_id ='001'
     and psi.dw_logical_delete_ind = FALSE
    and psi.dw_current_version_ind = True
    and psf.dw_logical_delete_ind = FALSE
    and psf.dw_current_version_ind = True
	 AND pog.Stocking_Section_Nbr IN (101, 103, 107, 140, 141, 142, 143, 959, 962, 990, 991)
   --AND u.dept_section_id = 318                                                                                      
	GROUP BY 1,2,3
		) ss
	  ON ss.store_upc = psi.FACILITY_NBR || '-' || psi.upc_NBR                                                                                                                                                                               
	--	JOIN EDM_VIEWS_PRD.DW_EDW_VIEWS.lu_stock_sect st
	--	ON st.stock_sect_nbr = p.Stocking_Section_Nbr
		JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_UPC u
		ON u.upc_nbr = psi.upc_NBR                     
	--USE FOR KEHE
	--JOIN DW_PRD.DW_DSS.BDR_PURCHASE_ITEM b                                
	--ON b.upc_id = u.upc_nbr                                                                                              
	  JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_Retail_store a
	  ON a.Retail_store_facility_nbr = psi.FACILITY_NBR                                                                                
	  LEFT JOIN
	 (SELECT c.Corporate_Item_Integration_Id
  ,          c.Size_Qty
  ,          c.size_uom_cd                                                                                
   FROM EDM_VIEWS_PRD.DW_VIEWS.Corporate_Item c                                                                                         
	JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_UPC u
	ON u.Corporate_Item_Integration_Id = c.Corporate_Item_Integration_Id
	WHERE c.CORPORATE_ITEM_CD > 0
       and c.dw_logical_delete_ind = FALSE
      and u.SAFEWAY_UPC_IND <> FALSE 
    and c.dw_current_version_ind = True
--	AND (((c.size_qty >= 48 AND c.size_uom_cd = 'OZ') OR (c.size_qty >= 3 AND c.size_uom_cd = 'LB')) AND ((c.size_qty < 320 AND c.size_uom_cd = 'OZ') OR (c.size_qty < 20 AND c.size_uom_cd = 'LB')))
--	AND u.smic_category_id IN (3207,3202,3210)
--	AND c.corporation_id = '001'
--	AND u.smic_group_id = 32
	GROUP BY 1,2,3
) c
ON u.Corporate_Item_Integration_Id = c.Corporate_Item_Integration_Id
LEFT JOIN
(SELECT c.Corporate_Item_Integration_Id
	FROM EDM_VIEWS_PRD.DW_VIEWS.Corporate_Item c
	JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_UPC u
	ON u.Corporate_Item_Integration_Id = c.Corporate_Item_Integration_Id
	WHERE c.CORPORATE_ITEM_CD  > 0
     and u.SAFEWAY_UPC_IND <> FALSE 
     and c.dw_logical_delete_ind = FALSE
    and c.dw_current_version_ind = True
	AND ((c.size_qty >= 320 AND c.size_uom_cd = 'OZ') OR (c.size_qty >= 20 AND c.size_uom_cd = 'LB'))
	AND u.SMIC_category_id IN (3207,3202,3210)
	AND c.corporation_id = '001'
	AND u.SMIC_group_id = 32
	GROUP BY 1
) c2
ON u.Corporate_Item_Integration_Id = c2.Corporate_Item_Integration_Id
WHERE 1=1
 and p.dw_logical_delete_ind = FALSE
and p.dw_current_version_ind = True
AND psi.upc_NBR > 0
-- AND ss.store_upc IS NULL
AND u.CORPORATE_ITEM_CD  > 0
 and u.SAFEWAY_UPC_IND <> FALSE 
--AND NOT a.parent_op_area_cd IN ()
--AND a.store_id in (0767)
--AND a.parent_op_area_cd IN (33)
--AND u.department_id IN (317)
--AND U.corp_item_cd IN (65150011)
--AND u.department_id IN (317,314,323,324,325)
--AND u.upc_nbr IN (2113045724)
--AND u.category_id IN (8855,8856,8857,8858,8859,8863)
--AND a.rog_id = 23
--USE FOR KEHE (check JOIN above)
--AND b.vend_nbr = '006446'                                                                                          
GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17
) z            
GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16;

STEP 2 -
create or replace table EDM_BIZOPS_PRD.MERCHAPPS.STORE_PS_SALES_BY_ITEM_STORE_T (
store_upc,
store_id ,
upc_nbr ,
avg_week_item_qty) as 
SELECT z.store_id || '-' || z.upc_nbr AS store_upc  
   ,        z.store_id store_id
   ,        z.upc_nbr upc_nbr
   ,        div0(z.item_qty, w.weeks_dif) AS avg_week_item_qty       
    FROM
            (
        SELECT rs.facility_nbr as store_id
                ,s.FACILITY_INTEGRATION_ID
        ,        s.upc_NBR
        ,        sum(s.ITEM_QTY) AS item_qty
        FROM EDM_VIEWS_PRD.DW_VIEWS.SALES_AGGREGATE_DY s
        join EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE rs
         ON rs.FACILITY_INTEGRATION_ID = s.FACILITY_INTEGRATION_ID
        JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_RETAIL_STORE a
        ON a.FACILITY_INTEGRATION_ID = s.FACILITY_INTEGRATION_ID and a.division_id <> 'N/A'
         AND rs.FACILITY_INTEGRATION_ID=a.FACILITY_INTEGRATION_ID 
        JOIN EDM_VIEWS_PRD.DW_VIEWS.CALENDAR D
        ON D.Calendar_dt = s.TRANSACTION_DT
        JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_upc u
        ON u.upc_nbr = s.upc_nbr and u.CORPORATE_ITEM_CD > 0
        AND u.corporation_id = '001'
        WHERE D.DW_LOGICAL_DELETE_IND = FALSE 
              AND D.DW_CURRENT_VERSION_IND = TRUE
              --and s.DW_LOGICAL_DELETE_IND = FALSE 
              AND s.DW_CURRENT_VERSION_IND = TRUE
              and u.SAFEWAY_UPC_IND <> FALSE 
              and rs.DW_LOGICAL_DELETE_IND = FALSE 
              AND rs.DW_CURRENT_VERSION_IND = TRUE
          --AND A.DW_LOGICAL_DELETE_IND = FALSE
         AND D.Calendar_dt BETWEEN current_date - 365 AND current_date - 1
        --AND NOT a.parent_op_area_cd IN ()
        --AND a.store_id in (0767)
        --AND a.parent_op_area_cd IN (33)
        --AND u.department_id IN (317)
        --AND U.corp_item_cd IN (65150011)
        --AND u.department_id IN (317,314,323,324,325)
        --AND u.upc_id IN (2113045724)
        --AND u.category_id IN (8855,8856,8857,8858,8859,8863)
        --AND a.rog_id = 23                                                   
         GROUP BY 1,2,3
        ) z      
        JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_RETAIL_STORE a
         ON a.FACILITY_INTEGRATION_ID = z.FACILITY_INTEGRATION_ID
         and a.division_id <> 'N/A'
        LEFT JOIN
        (                
            SELECT rs.FACILITY_nbr as store_id
            ,               s.upc_nbr
            ,               round((current_date - 7 - min(s.TRANSACTION_DT)) / 7,0) AS weeks_dif
            FROM EDM_VIEWS_PRD.DW_VIEWS.SALES_AGGREGATE_DY s
            join EDM_VIEWS_PRD.DW_VIEWS.RETAIL_STORE rs
             ON rs.FACILITY_INTEGRATION_ID = s.FACILITY_INTEGRATION_ID

            WHERE 1=1
           -- and s.DW_LOGICAL_DELETE_IND = FALSE 
              AND s.DW_CURRENT_VERSION_IND = TRUE
                           and rs.DW_LOGICAL_DELETE_IND = FALSE 
              AND rs.DW_CURRENT_VERSION_IND = TRUE
            AND s.TRANSACTION_DT BETWEEN current_date - 365 and current_date - 1
            GROUP BY 1,2
) w
ON w.store_id || '-' || w.upc_nbr = z.store_id || '-' || z.upc_nbr
            WHERE 1=1
            
        --AND z.sum_item_qty / 52 < 1
            AND current_date - a.Open_Dt >= 364
            GROUP BY 1,2,3,4;

STEP 3 - 
create or replace table EDM_BIZOPS_PRD.MERCHAPPS.STORE_PS_ITEM_SALES_COMBINED_T as
SELECT zz.store_id as store_id
,             zz.rog_id
,             zz.DEPT_SECTION_ID as dept_section_id
,             zz.upc_NBR as upc_NBR
,             zz.Item_dsc
,             zz.Fixture_Id
,             zz.HORIZONTAL_CNT
,             zz.VERTICAL_CNT
--              ,             zz.bay_shelf_nbr
,             zz.STORE_SCHEMATIC_EFFECTIVE_START_DT
,             zz.STORE_SCHEMATIC_EFFECTIVE_END_DT
--,             zz.large_pet_supply_3
,             zz.large_pet_supply_20
,             zz.spice_pack_ind
,             case when s.avg_week_item_qty is null then 0 else s.avg_week_item_qty end as avg_week_item_qty
,             CASE 
                   WHEN     s.avg_week_item_qty is null then 'Y'
                   WHEN     s.avg_week_item_qty < 1 THEN 'Y' ELSE 'N' END AS slow_moving_ind
--              ,             CASE WHEN zz.bay_shelf_nbr = 1 AND zz.max_bay_shelf_nbr <> 1 THEN 'Y' ELSE 'N' END AS low_shelf_ind
--              ,             CASE WHEN zz.bay_shelf_nbr = zz.max_bay_shelf_nbr AND zz.bay_shelf_nbr <> 1 THEN 'Y' ELSE 'N' END AS hi_shelf_ind    
,             CASE 
--   WHEN zz.Item_dsc LIKE 'BANQ%MEGA%' THEN 1
--   WHEN zz.Item_dsc LIKE 'BANQ%' THEN 2
--   WHEN zz.Item_dsc LIKE 'MICHELINA%BOWL%' THEN 3
--   WHEN zz.Item_dsc LIKE 'MICHELINA%' THEN 4
--   WHEN CAST(zz.corporate_item_cd AS VARCHAR(9)) LIKE '4805%' THEN 5
--   WHEN (zz.Item_dsc LIKE 'DANNON%') AND (((zz.HORIZONTAL_CNT * zz.VERTICAL_CNT) + zz.HORIZONTAL_CNT)>10) THEN 6
--   WHEN (zz.Item_dsc LIKE 'EL MONT%') AND (zz.size_qty BETWEEN 4 AND 5) AND (zz.size_uom_cd LIKE 'OZ') THEN 7
--   WHEN (zz.Item_dsc LIKE 'JOSE OLE%') AND (zz.size_qty BETWEEN 4 AND 5) AND (zz.size_uom_cd LIKE 'OZ') THEN 8
--   WHEN (zz.Item_dsc LIKE 'REDS%') AND (zz.size_qty BETWEEN 4 AND 5) AND (zz.size_uom_cd LIKE 'OZ') THEN 9
--   WHEN (zz.Item_dsc LIKE 'TINAS%') AND (zz.size_qty BETWEEN 4 AND 5) AND (zz.size_uom_cd LIKE 'OZ') THEN 10
--   WHEN CAST(zz.corporate_item_cd AS VARCHAR(9)) LIKE '4310%'  AND (((zz.HORIZONTAL_CNT * zz.VERTICAL_CNT) + zz.HORIZONTAL_CNT) > 8) THEN 11
--   WHEN (CAST(zz.corporate_item_cd AS VARCHAR(9)) LIKE '47%')
--AND NOT
--(zz.Item_dsc LIKE 'GREEN GIANT%'
--OR zz.Item_dsc LIKE 'GG%'
--OR zz.Item_dsc LIKE 'BIRDS EYE%'
--OR zz.Item_dsc LIKE 'BE %'
--OR zz.Item_dsc LIKE 'BESF %'
--OR zz.Item_dsc LIKE 'PICTSWEET%'
--OR zz.Item_dsc LIKE 'PICST%'
--OR zz.Item_dsc LIKE 'PCTS%'
--OR zz.Item_dsc LIKE 'PIC %'
--OR zz.Item_dsc LIKE '%POT%'
--OR zz.Item_dsc LIKE '%TOT%'
--OR zz.Item_dsc LIKE '%HASH%'
--OR zz.Item_dsc LIKE '%FRIES%'
--OR zz.Item_dsc LIKE 'ORE IDA%')
--AND (zz.VERTICAL_CNT::integer BETWEEN 1 AND 2) THEN 12                                
	WHEN (try_to_number(zz.DEPT_SECTION_ID) = 325) AND (zz.size_qty BETWEEN 0.75 AND 1) AND (zz.size_uom_cd LIKE 'OZ') THEN 13
	--WHEN (zz.Item_dsc LIKE '%ROLLS%') AND (((zz.HORIZONTAL_CNT * zz.VERTICAL_CNT) + zz.HORIZONTAL_CNT) >= 32) THEN 14
	--WHEN (zz.Item_dsc LIKE '%HUNGRY MAN%' OR zz.Item_dsc LIKE 'HM%') AND (zz.VERTICAL_CNT > 1) THEN 15
	   ELSE NULL END AS special_item_cd

FROM

--######################################## 
EDM_BIZOPS_PRD.MERCHAPPS.store_ps_item_store_planogram_T zz
--######################################## 


//JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_RETAIL_STORE a
//ON a.retail_store_facility_nbr = zz.store_id and a.division_id <> 'N/A' and zz.DEPT_SECTION_ID <> 'N/A' and zz.rog_id <> 'N/A' and zz.upc_NBR <> 'N/A' and zz.Item_dsc <> 'N/A' and  zz.store_id <> 'N/A'
//JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_UPC u
//ON u.upc_NBR = zz.upc_NBR and u.CORPORATE_ITEM_CD > 0 
LEFT JOIN

--########################################         
 (select * from EDM_BIZOPS_PRD.MERCHAPPS.store_ps_sales_by_item_store_t where AVG_WEEK_ITEM_QTY > 0 and UPC_NBR >0 and STORE_UPC <>'N/A' and STORE_ID <>'N/A')s
--######################################## 

ON zz.store_upc = s.store_upc

-- where s.store_upc <> 'N/A' and s.store_id <> 'N/A' and s.avg_week_item_qty  <> 'N/A'


GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15

STEP 4 -
create or replace table EDM_BIZOPS_PRD.MERCHAPPS.STORE_PS_CUSTOM_PS_BY_REQUEST_T as
SELECT
  a.STORE_ID as store_id,
  a.division_id as parent_op_area_cd,
  u.DEPT_SECTION_ID as dept_section_id,
  u.CATEGORY_ID as category_id,
  '2020-01-01' AS start_date,
  '2020-01-01' AS end_date,
  u.CSC_ID as csc_id,
  u.CORP_ITEM_CD as corp_item_cd,
  u.upc_id as upc_id,
  u.UPC_DSC as upc_dsc,
  ff.ps_value
FROM
  (select * from 
(
    SELECT
      a.STORE_ID as STORE_ID,
      f.upc_id,
      f.ps_value
    FROM
      EDM_BIZOPS_PRD.SUPPLY_CHAIN.FAR_PS_CUSTOM_VALUES_MERGE_FINAL f
      JOIN EDM_BIZOPS_PRD.MERCHAPPS.LU_STORE a ON try_to_number(a.DIVISION_ID) = f.parent_op_area_cd and a.division_id >0 and f.parent_op_area_cd >0
    WHERE f.store_id IS NULL 
      AND current_date BETWEEN f.first_eff_dt AND f.last_eff_dt
    GROUP BY 1,2,3
    UNION ALL
    SELECT
      f.store_id,
      f.upc_id,
      f.ps_value
    FROM
      EDM_BIZOPS_PRD.SUPPLY_CHAIN.FAR_PS_CUSTOM_VALUES_MERGE_FINAL f
    WHERE f.store_id IS NOT NULL AND current_date BETWEEN f.first_eff_dt AND f.last_eff_dt and f.store_id >0
    GROUP BY 1,2,3) where STORE_ID > 0 and upc_id > 0 and (ps_value=0 or ps_value>0)
  ) ff
  JOIN EDM_BIZOPS_PRD.MERCHAPPS.ZHU_VERSION_LU_UPC u ON u.upc_id = ff.upc_id and u.CORP_ITEM_CD > 0
  JOIN EDM_BIZOPS_PRD.MERCHAPPS.LU_STORE a ON a.STORE_ID = ff.STORE_ID and a.division_id <> 'N/A'
WHERE 1 = 1
  --This is the third spot for filtering
  --AND a.Retail_Store_Facility_Nbr in ('0767')
  --AND a.Parent_Operating_Area_Cd IN (33)
  --AND u.retail_department_id IN (317)
  --AND U.corporate_item_cd IN (65150011)
  --AND u.retail_department_id IN (317,314,323,324,325)
  --AND u.upc_nbr IN (2113045724)                                                                         
  -- End filter area
GROUP BY  1,2,3,4,5,6,7,8,9,10,11

STEP 5 -
create or replace table EDM_BIZOPS_PRD.MERCHAPPS.STORE_PS_CHECKSTAND_CANDY_T as 
SELECT 
  a.Retail_Store_Facility_Nbr store_id, 
  a.Parent_Operating_Area_Cd  as parent_op_area_cd, 
  u.section_cd as dept_section_id, 
  u.smic_category_id  as category_id, 
  to_date('2020-01-01') AS start_date_dummy, 
  to_date('2020-02-01') AS end_date_dummy, 
  u.CONSUMER_SELLING_CD as csc_id, 
  u.corporate_item_cd as corp_item_cd, 
  z.upc_nbr as upc_id, 
  u.item_dsc as upc_dsc, 
  z.total_ps 
FROM 
 
 (
   SELECT 
   facility_integration_id, 
   upc_nbr,
   sum(total_ps) as total_ps
   from
   
  (
    (SELECT 
      p.facility_integration_id, 
      p.upc_nbr, 
      p.ps_per_checkstand, 
      r.register_count, 
      sum(p.ps_per_checkstand * r.register_count) AS total_ps
      
    FROM 
      (
        SELECT 
          zz.facility_integration_id, 
          zz.upc_nbr, 
          CASE WHEN round(zz.avg_day_item_qty) > 12 THEN 12 WHEN round(zz.avg_day_item_qty) < 3 THEN 3 ELSE round(zz.avg_day_item_qty) END AS ps_per_checkstand 
        FROM 
          (
            SELECT 
              s.facility_integration_id, 
              s.upc_nbr, 
              sum(s.item_qty) / 364 / count(DISTINCT s.facility_integration_id) AS avg_day_item_qty 
            FROM edm_views_prd.dw_views.sales_aggregate_dy s 
              JOIN (
                SELECT 
                  psf.facility_integration_id, 
                  psi.UPC_NBR, 
                  c.ship_unit_pack_qty as pack_whse_qty 
                FROM 
                  edm_views_prd.dw_views.PLANOGRAM_STORE_FIXTURE psf 
                  JOIN edm_views_prd.dw_views.Planogram pog ON pog.Planogram_Id = psf.PLANOGRAM_ID 
                  AND pog.dw_logical_delete_ind = FALSE 
                  AND pog.dw_current_version_ind = True 
                  JOIN edm_views_prd.dw_views.PLANOGRAM_SLOT_ITEM psi ON psi.PLANOGRAM_ID = pog.Planogram_Id 
                  AND psi.dw_logical_delete_ind = FALSE 
                  AND psi.dw_current_version_ind = True 
                  JOIN edm_views_prd.dw_views.d1_upc u ON u.upc_nbr = psi.UPC_NBR and u.CORPORATE_ITEM_CD > 0
                  JOIN edm_views_prd.dw_views.corporate_item c ON c.CORPORATE_ITEM_INTEGRATION_ID = u.CORPORATE_ITEM_INTEGRATION_ID 
                  AND c.corporation_id = 1 
                  AND c.DW_CURRENT_VERSION_IND = TRUE 
                  AND c.DW_LOGICAL_DELETE_IND = FALSE 
                WHERE current_date BETWEEN psf.STORE_SCHEMATIC_EFFECTIVE_START_DT - 14 
                  AND psf.STORE_SCHEMATIC_EFFECTIVE_END_DT - 14 
                  AND pog.Stocking_Section_Nbr in (101, 103, 107, 140, 141, 142, 143, 959, 962, 990, 991) 
                  AND psf.DW_CURRENT_VERSION_IND = TRUE 
                  AND psf.DW_LOGICAL_DELETE_IND = FALSE 
                GROUP BY 1,2,3
              ) z ON s.facility_integration_id = z.facility_integration_id 
              AND s.upc_nbr = z.upc_nbr 
            WHERE s.transaction_dt BETWEEN current_date - 365 AND current_date - 1 
            GROUP BY 1,2
          ) zz
      ) p 
      
      JOIN (
        SELECT 
          z.facility_integration_id, 
          count(z.register_nbr) AS register_count 
        FROM 
          (
            SELECT 
              a.facility_integration_id, 
              th.register_nbr, 
              sum(th.net_amt) AS sum_net_amt 
            FROM 
              edm_views_prd.dw_edw_views.txn_hdr th 
              JOIN edm_views_prd.dw_views.D1_RETAIL_STORE a ON try_to_number(a.Retail_Store_Facility_Nbr) = th.store_id and a.division_id <> 'N/A'
            WHERE 
              th.txn_dte >= current_date - 21 
              AND ((th.register_nbr BETWEEN 1 AND 15) OR (th.register_nbr BETWEEN 116 AND 125)) 
            GROUP BY 1,2
          ) z 
        WHERE z.sum_net_amt > 100 
        GROUP BY 1
      ) r ON r.facility_integration_id = p.facility_integration_id 
      
    GROUP BY 1,2,3,4)
  
  
  union all
  
   
    (SELECT 
      p.facility_integration_id, 
      p.upc_nbr, 
      p.ps_per_checkstand as ps_per_checkstand_2, 
      r.register_count as register_count_2, 
      sum(p.ps_per_checkstand * r.register_count) AS total_ps
      
    FROM 
      (
        SELECT 
          zz.facility_integration_id, 
          zz.upc_nbr, 
          CASE WHEN round(zz.avg_day_item_qty) > 12 THEN 12 WHEN round(zz.avg_day_item_qty) < 3 THEN 3 ELSE round(zz.avg_day_item_qty) END AS ps_per_checkstand 
        FROM 
          (
            SELECT 
              s.facility_integration_id, 
              s.upc_nbr, 
              sum(s.item_qty) / 364 / count(DISTINCT s.facility_integration_id) AS avg_day_item_qty 
            FROM edm_views_prd.dw_views.sales_aggregate_dy s 
              JOIN (
                SELECT 
                  psf.facility_integration_id, 
                  psi.UPC_NBR, 
                  c.ship_unit_pack_qty as pack_whse_qty 
                FROM 
                  edm_views_prd.dw_views.PLANOGRAM_STORE_FIXTURE psf 
                  JOIN edm_views_prd.dw_views.Planogram pog ON pog.Planogram_Id = psf.PLANOGRAM_ID 
                  AND pog.dw_logical_delete_ind = FALSE 
                  AND pog.dw_current_version_ind = True 
                  JOIN edm_views_prd.dw_views.PLANOGRAM_SLOT_ITEM psi ON psi.PLANOGRAM_ID = pog.Planogram_Id 
                  AND psi.dw_logical_delete_ind = FALSE 
                  AND psi.dw_current_version_ind = True 
                  JOIN edm_views_prd.dw_views.d1_upc u ON u.upc_nbr = psi.UPC_NBR and u.CORPORATE_ITEM_CD > 0
                  JOIN edm_views_prd.dw_views.corporate_item c ON c.CORPORATE_ITEM_INTEGRATION_ID = u.CORPORATE_ITEM_INTEGRATION_ID 
                  AND c.corporation_id = 1 
                  AND c.DW_CURRENT_VERSION_IND = TRUE 
                  AND c.DW_LOGICAL_DELETE_IND = FALSE 
                WHERE current_date BETWEEN psf.STORE_SCHEMATIC_EFFECTIVE_START_DT - 14 
                  AND psf.STORE_SCHEMATIC_EFFECTIVE_END_DT - 14 
                  AND pog.Stocking_Section_Nbr in (9235,9234,9233,9232,9231,9230,9229,9228,9227,9226,9225,9224,9223,9222,9221,9220,9219,9218,9217,9216,9215,9300,9299,9298,9297,9296,9295,9294,9293,9292,9291,9290,9289,9288,9287,9286,9285,9284,9283,9282,9281,9280
) 
                  AND psf.DW_CURRENT_VERSION_IND = TRUE 
                  AND psf.DW_LOGICAL_DELETE_IND = FALSE 
                GROUP BY 1,2,3
              ) z ON s.facility_integration_id = z.facility_integration_id 
              AND s.upc_nbr = z.upc_nbr 
            WHERE s.transaction_dt BETWEEN current_date - 365 AND current_date - 1 
            GROUP BY 1,2
          ) zz
      ) p 
      
      JOIN (
        SELECT 
          z.facility_integration_id, 
          count(z.register_nbr) AS register_count 
        FROM 
          (
            SELECT 
              a.facility_integration_id, 
              th.register_nbr, 
              sum(th.net_amt) AS sum_net_amt 
            FROM 
              edm_views_prd.dw_edw_views.txn_hdr th 
              JOIN edm_views_prd.dw_views.D1_RETAIL_STORE a ON try_to_number(a.Retail_Store_Facility_Nbr) = th.store_id and a.division_id <> 'N/A'
            WHERE 
              th.txn_dte >= current_date - 21 
              AND ((th.register_nbr in (16,17,18,19,20,49,50,51,52,53,54,93,94,95,96,97,98,106,107,108,109,135,136,137,138,139,175,176,177,178,179,180,181,182))) 
            GROUP BY 1,2
          ) z 
        WHERE z.sum_net_amt > 100 
        GROUP BY 1
      ) r ON r.facility_integration_id = p.facility_integration_id 
      
    GROUP BY 1,2,3,4))
  
  group by 1,2)z

  JOIN edm_views_prd.dw_views.d1_upc u ON u.upc_nbr = z.upc_nbr 
  JOIN EDM_SANDBOX_PRD.MERCHAPPS.D1_RETAIL_STORE a ON a.facility_integration_id = z.facility_integration_id and a.division_id <> 'N/A'

STEP 6 -
create or replace table EDM_BIZOPS_PRD.MERCHAPPS.store_ps_final_step_2_t as 

 SELECT 
      zzz.store_id, 
      zzz.parent_op_area_cd, 
      zzz.dept_section_id, 
      zzz.category_id, 
      zzz.class_id, 
      to_date('2020-01-01') AS start_date_dummy, 
      to_date('2020-02-01') AS end_date_dummy, 
      zzz.csc_id, 
      zzz.corp_item_cd, 
      zzz.upc_id, 
      zzz.upc_dsc, 
      zzz.special_item_cd, 
      zzz.fast_ind, 
      CASE 
        WHEN zzz.dept_section_id = '317' AND zzz.parent_op_area_cd IN ('17', '19', '24', '27', '34') AND min(zzz.ps_value) < 4 THEN 4 
        WHEN zzz.dept_section_id = '317' AND zzz.class_id IN (440185, 440101, 471510, 473505, 470505, 470105, 473515, 471505, 449000, 470500, 470100, 471000, 480400, 440100, 440500, 473500, 471500, 470501, 473501, 470101, 473510, 471001) AND min(zzz.ps_value) < 4 THEN 4 
        
        WHEN zzz.category_id in (2160) and zzz.parent_op_area_cd in (34) THEN CEIL (min(zzz.ps_value) * 1.5)
        ELSE min(zzz.ps_value) END AS ps_value --****************************************** below is to get upc and op area
    FROM 
      (
        SELECT 
          ff.store_id, 
          a.Parent_Operating_Area_Cd as parent_op_area_cd, 
          ff.dept_section_id, 
          u.smic_category_id as category_id, 
          u.SMIC_CLASS_ID as class_id, 
          ff.pog_fxtr_start_dt, 
          ff.pog_fxtr_end_dt, 
          u.CONSUMER_SELLING_CD as csc_id, 
          u.corporate_item_cd as corp_item_cd, 
          ff.upc_id, 
          ff.upc_dsc, 
          ff.special_item_cd, 
          ff.fast_ind, 
          min(ff.ps_value) AS ps_value --****************************************** above is to get upc and op area 
        FROM 
          (
            SELECT 
              f.store_id, 
              f.dept_section_id, 
              f.upc_nbr as upc_id, 
              f.item_dsc as upc_dsc, 
              f.Fixture_Id as fxtr_cd, 
              f.HORIZONTAL_CNT as hrzntl_face_cnt, 
              f.VERTICAL_CNT as vrtcl_face_cnt, 
              f.STORE_SCHEMATIC_EFFECTIVE_START_DT as pog_fxtr_start_dt, 
              f.STORE_SCHEMATIC_EFFECTIVE_END_DT pog_fxtr_end_dt, 
      --        f.large_pet_supply_3, 
              f.large_pet_supply_20, 
              f.spice_pack_ind, 
              f.slow_moving_ind, 
              f.special_item_cd, 
              CASE WHEN (jk.upc_id IS NOT NULL) THEN f.HORIZONTAL_CNT --jewel kehe
          
				   WHEN f.special_item_cd IN (13) THEN 24 
				   WHEN f.slow_moving_ind = 'Y' and f.dept_section_id IN ('311', '312') THEN f.HORIZONTAL_CNT ----------APPLY ONLY TO f.dept_section_id IN (311,312)---------
				  -- WHEN f.large_pet_supply_20 = 'Y' THEN 2  removed per Kinzie on 2/7
				  -- WHEN f.large_pet_supply_3 = 'Y' THEN f.HORIZONTAL_CNT 
           
				   WHEN f.spice_pack_ind = 'Y' and c.size_uom_cd = 'OZ' and c.size_qty <= 3 THEN f.HORIZONTAL_CNT * 12 
                   WHEN f.spice_pack_ind = 'Y' and c.size_uom_cd = 'OZ' and c.size_qty > 3 THEN (f.HORIZONTAL_CNT * f.VERTICAL_CNT) + f.HORIZONTAL_CNT
           WHEN (u.smic_category_id = 3210 and a.Parent_Operating_Area_Cd = 34) THEN (f.HORIZONTAL_CNT * f.VERTICAL_CNT) + f.HORIZONTAL_CNT
				   WHEN (f.dept_section_id IN ('311', '312')) THEN f.HORIZONTAL_CNT ---****----APPLY TO a.Parent_Operating_Area_Cd IN (32), AND REMOVE SECTION 322 /changed to include all div 1/16/23 ---***--------
				  -- WHEN f.dept_section_id IN ('317') and a.Parent_Operating_Area_Cd IN ('30') and f.store_id not in ('0339','1509','1708','3197') THEN CEIL (((f.HORIZONTAL_CNT * f.VERTICAL_CNT) + f.HORIZONTAL_CNT) * 1.5)
                   WHEN f.rog_id = 'SACG' AND u.smic_category_id IN (8855, 8856, 8857, 8858, 8859, 8863) AND f.avg_week_item_qty >= 7 THEN ((f.HORIZONTAL_CNT * f.VERTICAL_CNT) + f.VERTICAL_CNT) * 2 
                   WHEN (u.smic_category_id = 3110) THEN f.HORIZONTAL_CNT 
                   WHEN (u.smic_category_id = 1005 AND c.size_uom_cd = 'LB' and c.size_qty > 5) THEN f.HORIZONTAL_CNT 
				   WHEN (u.SMIC_CLASS_ID = 80105 AND a.Parent_Operating_Area_Cd in (17, 32)) THEN f.HORIZONTAL_CNT --12pk can soda FOR Jewel/SW
				   WHEN (u.SMIC_SUB_SUB_CLASS_ID = 34100100 AND a.Parent_Operating_Area_Cd = '32') THEN f.HORIZONTAL_CNT --Firelogs FOR Jewel
				   WHEN (u.smic_category_id = 3430 AND a.Parent_Operating_Area_Cd = '32') THEN f.HORIZONTAL_CNT --bulk road salt FOR Jewel
           WHEN (u.smic_category_id = 6515 or u.smic_category_id = 6520) then f.HORIZONTAL_CNT * f.VERTICAL_CNT
           WHEN (u.smic_category_id = 1901 and c.size_uom_cd in ('CT', 'EA') and c.size_qty >= 32) THEN f.HORIZONTAL_CNT * f.VERTICAL_CNT
				   WHEN (u.smic_category_id = 3401 AND c.size_uom_cd = 'LB' AND a.Parent_Operating_Area_Cd in ('30', '32')) THEN f.HORIZONTAL_CNT --charcoal FOR Jewel
				   WHEN (u.SMIC_CLASS_ID = 329505  AND c.size_uom_cd = 'LB' AND a.Parent_Operating_Area_Cd IN ('25', '32')) THEN f.HORIZONTAL_CNT --bird seed FOR Jewel
                   WHEN (f.dept_section_id in ('303', '308') and a.parent_operating_area_cd = '27') THEN f.HORIZONTAL_CNT
                   WHEN (f.dept_section_id in ('327', '335') AND c.size_uom_cd = 'LB' and c.size_qty > 4) THEN f.HORIZONTAL_CNT  -- added on 1/16 to apply > 4lbs for charcoal / pet food / bird seed
				   WHEN (u.SMIC_SUB_SUB_CLASS_ID = 8152005 AND a.Parent_Operating_Area_Cd = '17') THEN f.HORIZONTAL_CNT --Seltzer FOR SW  
					ELSE (f.HORIZONTAL_CNT * f.VERTICAL_CNT) + f.HORIZONTAL_CNT 
			   END AS ps_value, 
              CASE WHEN f.rog_id = 'SACG' AND u.smic_category_id IN (8855, 8856, 8857, 8858, 8859, 8863) AND f.avg_week_item_qty >= 7 THEN 'Y' ELSE NULL 
			   END AS fast_ind 
            FROM 
              --######################################## 
              EDM_BIZOPS_PRD.MERCHAPPS.store_ps_item_sales_combined_t f -- this is the combination of item and sales to determine slow moving items
              --######################################## 
              JOIN EDM_SANDBOX_PRD.MERCHAPPS.D1_RETAIL_STORE a ON a.Retail_Store_Facility_Nbr = f.store_id 
              JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_UPC u ON u.upc_nbr = f.upc_nbr  and u.CORPORATE_ITEM_CD > 0
              LEFT JOIN EDM_VIEWS_PRD.DW_VIEWS.corporate_item c ON c.corporate_item_integration_id = u.corporate_item_integration_id 
              AND c.DW_CURRENT_VERSION_IND = TRUE 
              AND c.DW_LOGICAL_DELETE_IND = FALSE 
              LEFT JOIN (
                SELECT 
                  DTL.UPC_NBR as upc_id 
                FROM 
                  EDM_VIEWS_PRD.DW_VIEWS.RECEIVE_DELIVERY_INVOICE_HEADER_DSD HDR 
                  INNER JOIN EDM_VIEWS_PRD.DW_VIEWS.RECEIVE_DELIVERY_INVOICE_DETAIL_DSD DTL 
					ON HDR.FACILITY_INTEGRATION_ID = DTL.FACILITY_INTEGRATION_ID 
          JOIN 
          (select distinct RETAIL_UPC_NBR as upc_nbr, store_facility_integration_id from EDM_VIEWS_PRD.DW_VIEWS.STORE_ORDER_CATALOG a 
where  vendor_id = 6446 and a.dw_logical_delete_ind=False 
and a.dw_current_version_ind=True) catalog on catalog.store_facility_integration_id = DTL.FACILITY_INTEGRATION_ID and catalog.upc_nbr = DTL.UPC_NBR
					AND HDR.VENDOR_ID = DTL.VENDOR_ID 
					AND HDR.BACKDOOR_VENDOR_SUB_ACCOUNT_ID = DTL.BACKDOOR_VENDOR_SUB_ACCOUNT_ID 
					AND HDR.LOAD_DT = DTL.LOAD_DT --AND HDR.INVOICE_TYPE_CD = DTL.INVOICE_TYPE_CD 
					AND HDR.VENDOR_INVOICE_NBR = DTL.VENDOR_INVOICE_NBR 
                  INNER JOIN EDM_SANDBOX_PRD.MERCHAPPS.D1_RETAIL_STORE RETAIL_STORE 
					ON RETAIL_STORE.FACILITY_INTEGRATION_ID = HDR.FACILITY_INTEGRATION_ID 
					AND RETAIL_STORE.DW_LOGICAL_DELETE_IND = FALSE 
					--AND RETAIL_STORE.DW_CURRENT_VERSION_IND = TRUE 
                WHERE 1 = 1 
                  AND HDR.DW_LOGICAL_DELETE_IND = FALSE 
                  AND HDR.DW_CURRENT_VERSION_IND = TRUE 
                  AND DTL.DW_LOGICAL_DELETE_IND = FALSE 
                  AND DTL.DW_CURRENT_VERSION_IND = TRUE 
                 -- AND RETAIL_STORE.division_id = '32' 
                  AND HDR.RECEIVE_DT > current_Date - 364 
                  AND HDR.VENDOR_ID = '006446' 
                GROUP BY 1) jk 
			ON jk.upc_id = f.upc_nbr) ff 
		  --****************************************** below is to get upc and op area
          JOIN EDM_SANDBOX_PRD.MERCHAPPS.D1_RETAIL_STORE a 
			ON a.Retail_Store_Facility_Nbr = ff.store_id and a.division_id <> 'N/A'
          JOIN EDM_VIEWS_PRD.DW_VIEWS.D1_UPC u 
			ON u.upc_nbr = ff.upc_id 
			AND u.corporation_id = 1 
			AND u.corporate_item_cd > 0 
        GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13) zzz 
    GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13

STEP 7 -
create or replace table EDM_BIZOPS_PRD.MERCHAPPS.store_ps_logic_new_table as 
SELECT 
  CASE WHEN f.store_id IS NULL THEN p.store_id ELSE f.store_id END AS store_id, 
  CASE WHEN f.store_id IS NULL THEN p.parent_op_area_cd ELSE f.parent_op_area_cd END AS parent_op_area_cd, 
  CASE WHEN f.store_id IS NULL THEN p.dept_section_id ELSE f.dept_section_id END AS dept_section_id, 
  CASE WHEN f.store_id IS NULL THEN p.category_id ELSE f.category_id END AS category_id, 
  CASE WHEN f.store_id IS NULL THEN p.start_date_dummy ELSE f.start_date END AS start_date_dummy, 
  CASE WHEN f.store_id IS NULL THEN p.end_date_dummy ELSE f.end_date END AS end_date_dummy, 
  CASE WHEN f.store_id IS NULL THEN p.csc_id ELSE f.csc_id END AS csc_id, 
  CASE WHEN f.store_id IS NULL THEN p.corp_item_cd ELSE f.corp_item_cd END AS corp_item_cd, 
  CASE WHEN f.store_id IS NULL THEN p.upc_id ELSE f.upc_id END AS upc_id, 
  CASE WHEN f.store_id IS NULL THEN p.upc_dsc ELSE f.upc_dsc END AS upc_dsc, 
  CASE WHEN (p.store_id = 3493 and p.ps_value < 2) THEN 2
       WHEN (f.store_id IS NOT NULL or f.parent_op_area_cd IS NOT NULL ) THEN f.ps_value 
       WHEN candy.store_id IS NOT NULL THEN candy.total_ps
       WHEN (p.store_id = 3493 and p.ps_value < 2) THEN 2
    ELSE p.ps_value 
  END AS ps_value, 
  --CASE WHEN f.store_id IS NOT NULL THEN f.ps_value WHEN candy.store_id IS NOT NULL THEN candy.total_ps WHEN f.parent_op_area_cd IS NOT NULL THEN f.ps_value ELSE p.ps_value END AS ps_value, 
  CASE WHEN f.store_id IS NULL THEN NULL ELSE 'Y' END AS custom_ps_ind 
FROM 
  
--######################################## 
(select * from EDM_BIZOPS_PRD.MERCHAPPS.store_ps_final_step_2_t where parent_op_area_cd >0 and store_id <> 'N/A' and parent_op_area_cd <> 'N/A') p 
--######################################## 
  
FULL OUTER JOIN 
--######################################## 
(select * from EDM_BIZOPS_PRD.MERCHAPPS.store_ps_custom_ps_by_request_t where (ps_value >0 or ps_value=0) and store_id >0 ) f 
--########################################

ON f.store_id  || '-' || f.upc_id = p.store_id || '-' || p.upc_id and f.ps_value > 0 and  f.store_id > 0 and  f.upc_id > 0

LEFT JOIN 
--######################################## 
(select * from EDM_BIZOPS_PRD.MERCHAPPS.store_ps_checkstand_candy_t where (total_ps=0 or total_ps>0)) candy
--########################################  
        
ON p.store_id  || '-' || p.upc_id = candy.store_id || '-' || candy.upc_id and total_ps > 0

STEP 8 -
create or replace table EDM_BIZOPS_PRD.SUPPLY_CHAIN.store_ps_logic_new_table as 
select * from EDM_BIZOPS_PRD.MERCHAPPS.store_ps_logic_new_table
~~~
