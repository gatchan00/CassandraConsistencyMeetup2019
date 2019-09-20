# Cassandra Consistency Meetup 09-2019

Pequeña guía con los comandos ejecutados por si queréis reproducir en casa esto.

## Requisitos

Instalar Cassandra Cluster Manager: https://github.com/riptano/ccm

## Creación del clúster con CCM (página 19)

Ejecutamos las siguientes instrucciones. En ellas creamos un clúster de 3 nodos, con virtual nodes activado (3 nodos virtuales por nodo),
 y arrancamos el nodo 1. Si el **status** no funciona, dadle algo de tiempo a que termine de arrancar.

```
  ccm create arkham -v 3.10
	ccm populate -n 3 --vnodes
	ccm status
	ccm updateconf "num_tokens: 3"
	ccm node1 start
	ccm node1 nodetool status
```

Ahor nos conectaoms a dicho nodo:

```
	ccm node1 cqlsh
```

Y creamos un pequeño keyspace (acordaros de usar el tabulador) e insertamos algún dato:

```
	CREATE KEYSPACE test1 WITH replication = {'class': 'SimpleStrategy' , 'replication_factor': 1 };
	use test1;
	create table ejemplo1(col1 int, col2 text, primary key (col1));
	insert into ejemplo1(col1,col2) values (1,'uno');
	select * from ejemplo1;

	exit;
```

## Añadimos un nodo al clúster (página 20)

Revisamos el estado del clúster
```
	ccm node1 nodetool status
```

Añadimos el nuevo nodo (el nodo 2) y revisamos el estado del clúster:

```
	ccm node2 start
	ccm node1 nodetool status 
```

repetimos para el nodo 3

```
  ccm node3 start
	ccm node1 nodetool status 
```

## Distribución de datos (página 21-24)

Para entender la distribución de datos, vamos a añadir más datos, y consultar los tokens de dichas **partition keys**.

Nos conectamos al nodo 1:

```
ccm node1 cqlsh
```

Vamos al keyspace **test1**, insertamos datos, y consultamos sus tokens:

```
	use test1;
	insert into ejemplo1(col1,col2) values (2,'dos');
	insert into ejemplo1(col1,col2) values (3,'tres');
	insert into ejemplo1(col1,col2) values (4,'cuatro');
	select col1,token(col1) from ejemplo1;
  exit;
```

Consultamos la distribución de dotekns usando el **nodetool ring**

```
	ccm node1 nodetool ring  
```

Y contrastamos con dónde está cada dato

```
  ccm node1 nodetool getendpoints test1 ejemplo1 1
  ccm node1 nodetool getendpoints test1 ejemplo1 2
	ccm node1 nodetool getendpoints test1 ejemplo1 3
	ccm node1 nodetool getendpoints test1 ejemplo1 4
```

Incluso podemos consultar datos que no están:

```
	 ccm node1 nodetool getendpoints test1 ejemplo1 5
```

## Mostramos la replicación:

Nos conectamos al nodo 1
```
  ccm node1 cqlsh
```

Creamos un nuevo **keyspace**

```
	CREATE KEYSPACE test2 WITH replication = {'class': 'SimpleStrategy' , 'replication_factor': 3 };
	use test2;
	create table ejemplo1(col1 int, col2 text, primary key (col1));
	insert into ejemplo1(col1,col2) values (1,'uno');
	insert into ejemplo1(col1,col2) values (2,'dos');
	insert into ejemplo1(col1,col2) values (3,'tres');
	insert into ejemplo1(col1,col2) values (4,'cuatro');
	select col1,token(col1) from ejemplo1;
  exit;
```

ahora voy a tirar un nodo, para ello miro dónde está cada dato

```
	 ccm node1 nodetool getendpoints test1 ejemplo1 1
	 ccm node1 nodetool getendpoints test1 ejemplo1 2
	 ccm node1 nodetool getendpoints test1 ejemplo1 3
	 ccm node1 nodetool getendpoints test1 ejemplo1 4
```

Cojo uno que pierda info del test1  (n este ejemplo key =3 )

```
	ccm node2 stop
	./cqlsh
	use test1;
	 select * from ejemplo1 where col1=3;   --> no host available
	use test2;
	 select * from ejemplo1 where col1=3;
```


Se muestra como un modificar la consistencia a nivel de sesión, y como **consistency all** falla con el col1=3 (página 29):

```
	consistency all
	select * from ejemplo1 where col1=3;
	consistency one
	select * from ejemplo1 where col1=3;
  exit;
```	

Para finalizar, levantamos el nodo 2:

```
  ccm node2 start
```

## Creamos un **keyspace** con replication factor 2 y lo poblamos:

Antes, hay que conectarse usando cqlsh:
```
ccm node1 cqlsh:
```

```
	CREATE KEYSPACE test3 WITH replication = {'class': 'SimpleStrategy' , 'replication_factor': 2};
	use test3;
	create table ejemplo1(col1 int, col2 text, primary key (col1));
	insert into ejemplo1(col1,col2) values (1,'uno');
	insert into ejemplo1(col1,col2) values (2,'uno');
	insert into ejemplo1(col1,col2) values (3,'uno');
	insert into ejemplo1(col1,col2) values (4,'uno');
	select col1,token(col1) from ejemplo1;
  exit;
```

Ahora buscamos un punto nuevo a meter, por ejemplo el 5 y vemos dónde debería acabar:

```
	 ccm node1 nodetool getendpoints test3 ejemplo1 5
```

Salen 2 nodos, **tiro uno** y **me conecto al otro** (En el ejemplo salieron el 3 y el 1, tiro el 3 y me conecto al 1) 

```
	ccm node3 stop
	ccm node1 cqlsh
	use test3;
	insert into ejemplo1(col1,col2) values (5,'uno');
	select * from ejemplo1; 
  exit;
 ```
  
Hemos visto que se muestra el 5, ahora tiramos el nodo 1 y levantamos el 3 (o los correspondientes si os dio otros nodos el punto anterior):

```	
  ccm node1 stop
	ccm node3 start
```

Ahora me conecto a un nodo activo con cqlsh y repito la query:
```
	ccm node2 cqlsh
	use test3;
 	select * from ejemplo1; 
  exit;
```

**OH NO, el 5 ha desaparecido!!!**

## Zombie data (Braiinss):

Paramos el clúster

```
  ccm stop
```

Retiramos el **hinted handoff** (Ojo, si habéis cambiado el nombre del clúster la ruta cambia):

Para ello editamos los siguientes ficheros:

```
 vi .ccm/arkham/node1/conf/cassandra.yaml
 vi .ccm/arkham/node2/conf/cassandra.yaml
 vi .ccm/arkham/node3/conf/cassandra.yaml
```

Y buscamos **hinted_handoff_enabled**, sustituyéndolo por **true**:

```
hinted_handoff_enabled: false
```

Levantamos el clúster

```
ccm start
```

Modificamos el **gc_grace_seconds** para ponerlo a 0 en la tabla **ejemplo1** del keyspace *test2**:

```
ccm node1 cqlsh
```

```
use test2;

ALTER TABLE ejemplo1 
WITH gc_grace_seconds = 0;
```


Ahora paro un nodo (el 3 en este caso)

```
 ccm node3 stop
```


Me conecto a nodo 1:

```
  ccm node1 cqlsh
```

Y borro un dato:

```
use test2;
select * from ejemplo1;
 delete from ejemplo1 where col1=1;
select * from ejemplo1;
exit;
```

Forzamos un flush:

```
 ccm node1 nodetool flush
 ccm node2 nodetool flush
```

Ahora forzamos una compactación:

```
 ccm node1 compact
 ccm node2 compact
```

Levantamos el nodo caído y nos conectamos a él:

```
ccm node3 start
ccm node3 cqlsh
```

Y vemos qué pasa con las siguientes querys

```
select * from ejemplo1 where col1=1;  
select * from ejemplo1; 
```

NOTA: es posible que según el nodo al que te conectes las queries den valores distintos. Vamos a forzar un **read_repair* para que el dato resucite seguro:


```
ALTER TABLE ejemplo1 
WITH dclocal_read_repair_chance = 1;
```

Y ahora volvemos a lanzar las querys:

```
select * from ejemplo1 where col1=1;  
select * from ejemplo1; 
```

**Y eso es todo!!**




 

	
