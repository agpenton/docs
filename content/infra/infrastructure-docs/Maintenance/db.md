---
title: "Data Consistency / DB Maintenance"
date: 2022-07-16T16:05:38+02:00
draft: false
---

Extracting products without attributes. It might take up to 30m to complete. Separate database `juniqe_data_inconsistency` is used to hold results.

```
CREATE TABLE catalog_product_entity_without_attributes;

SELECT "name" AS "attribute", cpe.entity_id AS "product_entity_id" FROM juniqe.catalog_product_entity cpe LEFT JOIN juniqe.catalog_product_entity_varchar cpev ON ( cpe.entity_id = cpev.entity_id )
 WHERE
 ( cpev.attribute_id in (SELECT ea.attribute_id FROM juniqe.eav_attribute ea WHERE ( ea.attribute_code = "name" ) ) ) AND
 ( cpev.value_id IS NULL )
 GROUP BY cpe.entity_id
UNION

SELECT "status" AS "attribute", cpe.entity_id as "product_entity_id" FROM
 juniqe.catalog_product_entity cpe LEFT JOIN
 juniqe.`catalog_product_entity_int` cpei ON ( cpe.entity_id = cpei.entity_id )
WHERE 
 ( cpei.attribute_id IN (SELECT ea.attribute_id FROM juniqe.eav_attribute ea WHERE ( ea.attribute_code = "status" ) ) ) AND
 ( cpei.value IS NULL )
GROUP BY cpe.entity_id UNION 


SELECT "url_key" AS "attribute", cpe.entity_id AS "product_entity_id" FROM
juniqe.catalog_product_entity cpe LEFT JOIN
 juniqe.catalog_product_entity_varchar cpev ON ( cpe.entity_id = cpev.entity_id )
 where
 ( cpev.attribute_id in (select ea.attribute_id from juniqe.eav_attribute ea where ( ea.attribute_code = "url_key" ) ) ) and
 ( cpev.value_id IS NULL )
group by cpe.entity_id union

 select "description" as "attribute", cpe.entity_id as "product_entity_id" from
juniqe.catalog_product_entity cpe LEFT JOIN
 juniqe.catalog_product_entity_text cpet ON ( cpe.entity_id = cpet.entity_id )
 where
 ( cpet.attribute_id in (select ea.attribute_id from juniqe.eav_attribute ea where ( ea.attribute_code = "description" ) ) ) and
 ( cpet.value_id IS NULL )
group by cpe.entity_id union
select "short_description" as "attribute", cpe.entity_id as "product_entity_id" from
juniqe.catalog_product_entity cpe LEFT JOIN
 juniqe.catalog_product_entity_text cpet ON ( cpe.entity_id = cpet.entity_id )
 where
  ( cpet.attribute_id in (select ea.attribute_id from  juniqe.eav_attribute ea where ( ea.attribute_code = "short_description" )  ) ) and
 ( cpet.value_id IS NULL )
group by cpe.entity_id union

select "ready_to_ship_to" as "attribute", cpe.entity_id as "product_entity_id" from
 juniqe.catalog_product_entity cpe LEFT JOIN
 juniqe.`catalog_product_entity_int` cpei ON ( cpe.entity_id = cpei.entity_id )
where
  ( cpei.attribute_id in (select ea.attribute_id from  juniqe.eav_attribute ea where ( ea.attribute_code = "ready_to_ship_to" )  ) ) and
 ( cpei.value IS NULL )
 group by cpe.entity_id union

 select "ready_to_ship_from" as "attribute", cpe.entity_id as "product_entity_id" from
 juniqe.catalog_product_entity cpe LEFT JOIN 
 juniqe.`catalog_product_entity_int` cpei ON ( cpe.entity_id = cpei.entity_id )
where
  ( cpei.attribute_id in (select ea.attribute_id from  juniqe.eav_attribute ea where ( ea.attribute_code = "ready_to_ship_from"  ) ) ) and
 ( cpei.value IS NULL )
group by cpe.entity_id;
```