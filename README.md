# qwat-to-ghydraulics
Qwat to Epanet, the ghydraulics way

## Abstract

The approach chosen is to add virtual layers to Qgis project and then copy/paste required values in the corresponding ghydraulics layers and then to export the Epanet model

## Prerequisites

- A Qwat database with a working Qgis connection.(service='qwat')
- [Ghydraulics Qgis plugin](https://plugins.qgis.org/plugins/ghydraulic/) if using Qgis 2.X
- [Qwater](https://plugins.qgis.org/plugins/QWater/) if using Qgis 3.X *(actually not tested)*


## Usage

Open the provided project file, copy/paste the required sector in the corresponding Ghydraulics shapefiles and export to Epanet

## The virtual Layers

### nodes

```
((SELECT 
		node.id AS DC_ID,
		--ST_Z(node.geometry) AS ELEVATION,
		ROUND(ST_Z(node.geometry)::numeric,2) AS ELEVATION,
		null as DEMAND,
		null as PATTERN,
		ST_Force2D(node.geometry) AS geometry
		FROM qwat_od.node
		WHERE node.id IN(select fk_node_a from qwat_od.pipe WHERE pipe.fk_function IN  (4101,4105,4109,10001)  AND pipe.fk_status IN(101,102,103,1301,1306,1307) ) OR
					node.id IN(select fk_node_b from qwat_od.pipe WHERE pipe.fk_function IN  (4101,4105,4109,10001)  AND pipe.fk_status IN(101,102,103,1301,1306,1307))

))
```

### valves

```
((	SELECT 
		installation.id AS DC_ID,
		fk_pipe_in AS NODE1,
		fk_pipe_out AS NODE2,
		null AS DIAMETER,
				null as TYPE,
		null as SETTING,
		0.0 AS MINORLOSS,
		element.altitude as ELEVATION,

		status.code AS Etat_exploitation,
		LEFT(regexp_replace(element.remark, E'[\\n\\r\\f\\u000B\\u0085\\u2028\\u2029]+', ' ', 'g' ) , 80) AS Remarque,
		LEFT(pressurezone.name,30) AS Zone_pression,
		LEFT(COALESCE(installation.name, element.identification),40) AS Nom,
				CASE WHEN fk_pressurecontrol_type = 2801 THEN 
			'reducteur'
		ELSE 
			'coupe-pression'
		END AS Type_reducteur,
		CASE 
			WHEN GeometryType(element.geometry) != 'POINT' THEN 
				ST_Force2D(ST_Centroid(element.geometry))
			ELSE
				ST_Force2D(element.geometry) 
		END AS geometry

		
FROM qwat_od.vw_qwat_installation installation
     JOIN qwat_od.vw_node_element element ON installation.id = element.id
	 
		LEFT JOIN qwat_od.pressurezone pressurezone ON element.fk_pressurezone = pressurezone.id
		LEFT JOIN qwat_ch_fr_aquafri.status status ON element.fk_status = status.id
	 
		WHERE
		installation.installation_type = 'pressurecontrol'
		AND
		fk_pressurecontrol_type IN (2801,2802) -- réducteur, coupe-pression
))
```


### pipes

```
((	SELECT 
		pipe.id AS DC_ID,
		--CASE WHEN pipe._length3d > 1 THEN 
		--	pipe._length3d 
		--ELSE pipe._length2d
		--END AS LENGTH,
		pipe._length2d AS LENGTH,
		pipe.fk_node_a AS NODE1,
		pipe.fk_node_b AS NODE2,
		CASE WHEN pipe_material.diameter_internal > 1 then
			round(pipe_material.diameter_internal)::INTEGER 
		ELSE round(pipe_material.diameter_nominal)::INTEGER 
		END 	AS DIAMETER,
		0.1 AS ROUGHNESS,
		0.0 AS MINORLOSS,
		--CASE WHEN pipe.fk_status IN(101,102,103,1301,1306,1307) then
		CASE WHEN pipe._valve_closed = true OR pipe.fk_status = 1307 then
			'Closed'
		ELSE
			'Open'
		END AS STATUS,
		--pipe.fk_function,
		
			
		pipe_function.value_fr AS function,
		LEFT(pressurezone.name,30) AS Zone_pression,
		
		pipe_status.value_fr AS statut_conduite, 
		
		
		ST_Force2D(pipe.geometry) AS geometry

		
    FROM qwat_od.pipe
		LEFT JOIN qwat_vl.pipe_material pipe_material ON pipe.fk_material = pipe_material.id
		LEFT JOIN qwat_vl.pipe_function pipe_function ON pipe.fk_function = pipe_function.id
		LEFT JOIN qwat_od.pressurezone pressurezone ON pipe.fk_pressurezone = pressurezone.id
		LEFT JOIN qwat_vl.status pipe_status on pipe.fk_status = pipe_status.id
		WHERE pipe.fk_function IN (4101,4105)  AND pipe.fk_status IN(101,102,103,1301,1306,1307)
		--and pipe_material.diameter_internal > 0

))
```


### pumps

```
((  SELECT 
		'pump_'|| installation.id::varchar  AS DC_ID,
		element.altitude::DECIMAL AS ELEVATION, 
		null AS PROPERTIES,
		installation.rejected_flow::INTEGER AS Q_refoulement,
		installation.manometric_height::DECIMAL AS H_mano, 
		LEFT(installation.name,40) AS Nom,
		CASE 
			WHEN GeometryType(element.geometry) != 'POINT' THEN 
				ST_Force2D(ST_Centroid(element.geometry))
			ELSE
				ST_Force2D(element.geometry) 
		END AS geometry

		
FROM qwat_od.vw_qwat_installation installation
     JOIN qwat_od.vw_node_element element ON installation.id = element.id
	 
		WHERE
		installation.installation_type = 'pump'
	
))
```


### tanks
```
((SELECT 
		--'tank_'|| installation.id::varchar  AS DC_ID,
		installation.id AS DC_ID,
		--ROUND(ST_Z(geometry)::numeric,2) AS ELEVATION,
		CASE WHEN altitude_overflow > 1 then
			altitude_overflow::DECIMAL - 4 
		ELSE null
		END AS ELEVATION,
		--Elevation  = Niv_eau - 4 [m] sinon <Null> 
		--altitude_overflow::DECIMAL  AS INITIALLEV,
		2 AS INITIALLEV,
		0 AS MINIMUMLEV,
		2 AS MAXIMUMLEV,
		--Diameter = 2 x ( (V alim +V inc.)  / (3.14 x 4m))^0.5   [en m à arrondir au cm]
		round(2 * ((storage_supply+storage_fire)  / (3.14 * 4))^0.5)  AS DIAMETER,
		null AS MINIMUMVOL,
		null AS VOLUMECURV,
		altitude_overflow::DECIMAL  AS Niv_eau,
		storage_supply::INTEGER AS V_utilisation,
		storage_fire::INTEGER AS V_incendie,
		LEFT(installation.name,40) AS Nom,		
		CASE 
			WHEN GeometryType(element.geometry) != 'POINT' THEN 
				ST_Force2D(ST_Centroid(element.geometry))
			ELSE
				ST_Force2D(element.geometry) 
		END AS geometry

		
FROM qwat_od.vw_qwat_installation installation
     JOIN qwat_od.vw_node_element element ON installation.id = element.id
	 
		WHERE
		installation.installation_type = 'tank'
	
))
```
