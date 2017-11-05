# TWAP LME SQL processing code (Greater Caribbean version, actually!)
Emilio Mayorga, University of Washington
- http://staff.washington.edu/emiliom/
- http://apl.uw.edu/people/profile.php?last_name=Mayorga&first_name=Emilio
- https://github.com/emiliom/

This code is from PostgreSQL 9.3. Other basic (but Global NEWS 2 "generic") source tables used in the views are not documented here.

*11/5/2017. Work in progress, specially in terms of documentation! *

## 1. Table lme.caribt33_stn30basin
Description: Mapping of STN30 NEWS2 basins to TR33 Greater Caribbean (sub)regions (Subregions_TR33.shp), originally performed in QGIS based on attribute queries guided and validated by visual inspection. 2017-6-4

```sql
CREATE TABLE lme.caribt33_stn30basin
(
  basinid integer NOT NULL,
  tr33_nbr integer NOT NULL,
  tr33_label character varying,
  completed boolean NOT NULL DEFAULT true,
  CONSTRAINT caribt33_stn30basin_pkey PRIMARY KEY (basinid)
)
WITH (OIDS=FALSE);
```

## 2. View: newstwap.v_r00_caribt33agg

```sql
CREATE OR REPLACE VIEW newstwap.v_r00_caribt33agg AS 
 SELECT lmestn.tr33_nbr,
    lmestn.tr33_label,
    count(lmestn.basinid) AS stnbasin_cnt,
    sum(stn30t.area) AS stnbasin_area,
    sum(scsi.qact) AS qact,
    0.000001::double precision * sum(sceo.ld_din) AS ld_din,
    0.000001::double precision * sum(sceo.ld_don) AS ld_don,
    0.000001::double precision * sum(sceo.ld_pn) AS ld_pn,
    0.000001::double precision * sum(sceo.ld_dip) AS ld_dip,
    0.000001::double precision * sum(sceo.ld_dop) AS ld_dop,
    0.000001::double precision * sum(sceo.ld_pp) AS ld_pp,
    0.000001::double precision * sum(sceo.ld_dsi) AS ld_dsi,
    0.000001::double precision * (sum(sceo.ld_din) + sum(sceo.ld_don) + sum(sceo.ld_pn)) AS ld_tn,
    0.000001::double precision * (sum(sceo.ld_dip) + sum(sceo.ld_dop) + sum(sceo.ld_pp)) AS ld_tp,
    0.000000001::double precision * sum(scoo.ys1pnt_din * stn30t.area) AS ls1pnt_din,
    0.000000001::double precision * sum(scoo.ys1pnt_dip * stn30t.area) AS ls1pnt_dip,
    0.000000001::double precision * sum(scoo.ys1pnt_don * stn30t.area) AS ls1pnt_don,
    0.000000001::double precision * sum(scoo.ys1pnt_dop * stn30t.area) AS ls1pnt_dop,
    0.000000001::double precision * sum(scoo.ys1dif_ant_din * stn30t.area) AS ls1dif_ant_din,
    0.000000001::double precision * sum(scoo.ys1dif_ant_dip * stn30t.area) AS ls1dif_ant_dip,
    0.000000001::double precision * sum(scoo.ys1dif_ant_don * stn30t.area) AS ls1dif_ant_don,
    0.000000001::double precision * sum(scoo.ys1dif_ant_dop * stn30t.area) AS ls1dif_ant_dop,
    0.000000001::double precision * sum(scoo.ys1dif_nat_din * stn30t.area) AS ls1dif_nat_din,
    0.000000001::double precision * sum(scoo.ys1dif_nat_dip * stn30t.area) AS ls1dif_nat_dip,
    0.000000001::double precision * sum(scoo.ys1dif_nat_don * stn30t.area) AS ls1dif_nat_don,
    0.000000001::double precision * sum(scoo.ys1dif_nat_dop * stn30t.area) AS ls1dif_nat_dop
   FROM lme.caribt33_stn30basin lmestn
     JOIN news2.stn30v6ng_tropperc stn30t ON lmestn.basinid = stn30t.basinid
     JOIN news2.r00_news2sceninputs scsi ON lmestn.basinid = scsi.basinid
     JOIN news2.r00_news2exportsoutput sceo ON lmestn.basinid = sceo.basinid
     JOIN news2.r00_news2otheroutput scoo ON lmestn.basinid = scoo.basinid
  GROUP BY lmestn.tr33_nbr, lmestn.tr33_label
  ORDER BY lmestn.tr33_nbr;
```
  
## 3. View: newstwap.v_r00_caribt33_indic

``` sql
CREATE OR REPLACE VIEW newstwap.v_r00_caribt33_indic AS 
 WITH w AS (
         SELECT v_r00_caribt33agg.tr33_nbr,
            ((106 * 12)::numeric * (1.0 / 365.0))::double precision * (v_r00_caribt33agg.ld_tn / (14 * 16)::double precision - v_r00_caribt33agg.ld_dsi / (28 * 20)::double precision) * (1000000000::numeric::double precision / v_r00_caribt33agg.stnbasin_area) AS icep_n,
            ((106 * 12)::numeric * (1.0 / 365.0))::double precision * (v_r00_caribt33agg.ld_tp / 31::double precision - v_r00_caribt33agg.ld_dsi / (28 * 20)::double precision) * (1000000000::numeric::double precision / v_r00_caribt33agg.stnbasin_area) AS icep_p,
                CASE
                    WHEN ((31 / 14)::double precision * (v_r00_caribt33agg.ld_tn / v_r00_caribt33agg.ld_tp)) < 16::double precision THEN ((106 * 12)::numeric * (1.0 / 365.0))::double precision * (v_r00_caribt33agg.ld_tn / (14 * 16)::double precision - v_r00_caribt33agg.ld_dsi / (28 * 20)::double precision) * (1000000000::numeric::double precision / v_r00_caribt33agg.stnbasin_area)
                    ELSE ((106 * 12)::numeric * (1.0 / 365.0))::double precision * (v_r00_caribt33agg.ld_tp / 31::double precision - v_r00_caribt33agg.ld_dsi / (28 * 20)::double precision) * (1000000000::numeric::double precision / v_r00_caribt33agg.stnbasin_area)
                END AS icep
           FROM newstwap.v_r00_caribt33agg
        )
 SELECT tla.tr33_nbr,
    tla.tr33_label,
    tla.stnbasin_cnt,
    tla.stnbasin_area,
    tla.qact,
    tla.ld_din,
    tla.ld_don,
    tla.ld_pn,
    tla.ld_dip,
    tla.ld_dop,
    tla.ld_pp,
    tla.ld_dsi,
    tla.ld_tn,
    tla.ld_tp,
    w.icep_n,
    w.icep_p,
    w.icep,
    tla.ls1pnt_din,
    tla.ls1pnt_dip,
    tla.ls1pnt_don,
    tla.ls1pnt_dop,
    tla.ls1dif_ant_din,
    tla.ls1dif_ant_dip,
    tla.ls1dif_ant_don,
    tla.ls1dif_ant_dop,
    tla.ls1dif_nat_din,
    tla.ls1dif_nat_dip,
    tla.ls1dif_nat_don,
    tla.ls1dif_nat_dop,
        CASE
            WHEN tla.ld_din <= 0.10::double precision THEN 1
            WHEN tla.ld_din <= 0.25::double precision THEN 2
            WHEN tla.ld_din <= 0.60::double precision THEN 3
            WHEN tla.ld_din <= 1.00::double precision THEN 4
            WHEN tla.ld_din > 1.00::double precision THEN 5
            ELSE NULL::integer
        END AS ld_din_cat,
        CASE
            WHEN w.icep <= (-5)::double precision THEN 1
            WHEN w.icep <= (-1)::double precision THEN 2
            WHEN w.icep <= 1::double precision THEN 3
            WHEN w.icep <= 5::double precision THEN 4
            WHEN w.icep > 5::double precision THEN 5
            ELSE NULL::integer
        END AS icep_cat
   FROM newstwap.v_r00_caribt33agg tla
     JOIN w ON tla.tr33_nbr = w.tr33_nbr
  ORDER BY tla.tr33_nbr;
```

## 4. View: newstwap.v_r00_caribt33

```sql
CREATE OR REPLACE VIEW newstwap.v_r00_caribt33 AS 
 SELECT v_r00_caribt33_indic.tr33_nbr,
    v_r00_caribt33_indic.tr33_label,
    v_r00_caribt33_indic.stnbasin_cnt,
    v_r00_caribt33_indic.stnbasin_area,
    v_r00_caribt33_indic.qact,
    v_r00_caribt33_indic.ld_din,
    v_r00_caribt33_indic.ld_don,
    v_r00_caribt33_indic.ld_pn,
    v_r00_caribt33_indic.ld_dip,
    v_r00_caribt33_indic.ld_dop,
    v_r00_caribt33_indic.ld_pp,
    v_r00_caribt33_indic.ld_dsi,
    v_r00_caribt33_indic.ld_tn,
    v_r00_caribt33_indic.ld_tp,
    v_r00_caribt33_indic.icep_n,
    v_r00_caribt33_indic.icep_p,
    v_r00_caribt33_indic.icep,
    v_r00_caribt33_indic.ls1pnt_din,
    v_r00_caribt33_indic.ls1pnt_dip,
    v_r00_caribt33_indic.ls1pnt_don,
    v_r00_caribt33_indic.ls1pnt_dop,
    v_r00_caribt33_indic.ls1dif_ant_din,
    v_r00_caribt33_indic.ls1dif_ant_dip,
    v_r00_caribt33_indic.ls1dif_ant_don,
    v_r00_caribt33_indic.ls1dif_ant_dop,
    v_r00_caribt33_indic.ls1dif_nat_din,
    v_r00_caribt33_indic.ls1dif_nat_dip,
    v_r00_caribt33_indic.ls1dif_nat_don,
    v_r00_caribt33_indic.ls1dif_nat_dop,
    v_r00_caribt33_indic.ld_din_cat,
    v_r00_caribt33_indic.icep_cat,
        CASE
            WHEN v_r00_caribt33_indic.ld_din_cat = 1 AND v_r00_caribt33_indic.icep_cat >= 1 AND v_r00_caribt33_indic.icep_cat <= 5 THEN 1
            WHEN v_r00_caribt33_indic.ld_din_cat = 2 AND v_r00_caribt33_indic.icep_cat >= 1 AND v_r00_caribt33_indic.icep_cat <= 5 THEN 2
            WHEN v_r00_caribt33_indic.ld_din_cat = 3 AND (v_r00_caribt33_indic.icep_cat = ANY (ARRAY[1, 2, 3])) THEN 3
            WHEN v_r00_caribt33_indic.ld_din_cat = 3 AND (v_r00_caribt33_indic.icep_cat = ANY (ARRAY[4, 5])) THEN 4
            WHEN v_r00_caribt33_indic.ld_din_cat = 4 AND v_r00_caribt33_indic.icep_cat >= 1 AND v_r00_caribt33_indic.icep_cat <= 4 THEN 4
            WHEN v_r00_caribt33_indic.ld_din_cat = 5 AND v_r00_caribt33_indic.icep_cat >= 1 AND v_r00_caribt33_indic.icep_cat <= 5 THEN 5
            ELSE NULL::integer
        END AS indic_cat_max
   FROM newstwap.v_r00_caribt33_indic
  ORDER BY v_r00_caribt33_indic.tr33_nbr;
```
