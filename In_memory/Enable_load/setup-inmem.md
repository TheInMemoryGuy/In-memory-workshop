# Enable In-Memory

Oracle Database In-Memory option comes preinstalled with the Oracle Database 12c and does not require additional software installation or recompilation of existing database software. This is because the In-Memory option has been seamlessly integrated into the core of the Oracle Database software as a new component of the Shared Global Area (SGA), so when the Oracle Database is installed, Oracle Database In-Memory gets installed with it.
However, the IM column store is not enabled by default, but can be easily enabled via a few steps, as outlined in this lesson.
It is important to remember that after the In-Memory option is enabled at the instance level, you also have to specifically enable objects so they would be considered for In-Memory column store population.



##  Step 1: Logging In and Enabling In-Memory

1.  All scripts for this lab are stored in the labs/inmemory folder and are run as the oracle user.  Let's navigate there now.  We recommend you type the commands to get a feel for working with In-Memory. But we will also allow you to copy the commands via the COPY button.

    ````
    <copy>
    sudo su - oracle
    cd ~/labs/inmemory
    ls
    </copy>
    ````

2. The In-Memory Area is a static pool within the SGA that holds the column format data (also referred to as the In-Memory Column Store). The size of the In-Memory Area is controlled by the initialization parameter INMEMORY\_SIZE (default is 0, i.e. disabled).
As the IM column store is a static pool, any changes to the INMEMORY\_SIZE parameter will not take effect until the database instance is restarted.
Note :  The default install of database usually set a parameter MEMORY_TARGET which manages both SGA (System Global Area) and PGA(Process Global Area). In the earlier version, AMM was not supported and we had to set SGA and PGA exclusively.


    ````

    <copy>sqlplus / as sysdba ;
    show pdbs </copy>
    ````
    You will observe that you are connect to CDB (Container Database) with one PDB (pluggable Database) preinstalled.


    Next, note the memory settings and enable In-Memory.
    ````
    <copy>
    show parameter inmemory_size
    show parameter sga_target
    show parameter pga_aggregate_target
    show parameter memory_target
    </copy>
    ````


3.  Enter the commands to enable In-Memory.  The database will need to be restarted for the changes to take effect.

    ````
    <copy>
    alter system set inmemory_size=2G scope=spfile;
    shutdown immediate;
    startup;
    </copy>
    ````


4.  Now let's take a look at the parameters.

    ````
    <copy>
    show sga;
    show parameter inmemory;
    show parameter keep;
    exit
    </copy>
    ````

## Step 2: Enabling In-Memory

The Oracle environment is already set up so sqlplus can be invoked directly from the shell environment. Since the lab is being run in a pdb called orclpdb you must supply this alias when connecting to the ssb account.

1.  Login to the pdb as the SSB user.  
    ````
    <copy>
    sqlplus ssb/Ora_DB4U@localhost:1521/orclpdb
    set pages 9999
    set lines 200
    </copy>
    ````


2.  The In-Memory area is sub-divided into two pools:  a 1MB pool used to store actual column formatted data populated into memory and a 64K pool to store metadata about the objects populated into the IM columns store.  V$INMEMORY_AREA shows the total IM column store.  The COLUMN command in these scripts identifies the column you want to format and the model you want to use.  

    ````
    <copy>
    column alloc_bytes format 999,999,999,999;
    column used_bytes format 999,999,999,999;
    column populate_status format a15;
    --QUERY

    select * from v$inmemory_area;
    </copy>
    ````


3.  
To check if the IM column store is populated with object, run the query below.

    ````
    <copy>
    column name format a30
    column owner format a20
    --QUERY

    select v.owner, v.segment_name name, v.populate_status status from v$im_segments v;
    </copy>
    ````


4.  To add objects to the IM column store the inmemory attribute needs to be set.  This tells the Oracle DB these tables should be populated into the IM column store.
Objects are populated into the IM column store either in a prioritized list immediately after the database is opened or
after they are scanned (queried) for the first time. The order in which objects are populated is controlled by the
keyword PRIORITY, which has five levels. The default PRIORITY is NONE, which means an object is populated only after it is scanned for the first time. All objects at a given priority level must be fully populated before the population of any objects at a lower priority level can commence. However, the population order can be superseded if an object without a PRIORITY is scanned, triggering its population into IM column store.

To enable an existing table for the IM column store, you would use the ALTER table DDL with the INMEMORY clause and the PRIORITY parameter. The syntax for this statement is:
	ALTER TABLE … INMEMORY PRIORITY [NONE|LOW|MEDIUM|HIGH|CRITICAL]

In-memory compression is specified using the keyword MEMCOMPRESS, a sub-clause of the INMEMORY attribute.
There are six levels, each of which provides a different level of compression and performance.

 ![](images/compress.png)

 5. Let us create a table and enable In-Memory.

 ````
 <copy>
  CREATE TABLE bonus
  (id number,
   emp_id number,
   bonus NUMBER
   year date) INMEMORY;
  </copy>
  ````

 6. Alter existing tables and enable In-Memory.

    ````
    <copy>
    ALTER TABLE lineorder INMEMORY PRIORITY CRITCAL MEMCOMPRESS FOR QUERY HIGH;
    ALTER TABLE part INMEMORY PRIORITY HIGH;
    ALTER TABLE customer INMEMORY PRIORITY MEFIUM;
    ALTER TABLE supplier INMEMORY PRIORITY LOW;
    ALTER TABLE date_dim INMEMORY ;
    </copy>
    ````
    By default, all of the columns in an object with the INMEMORY attribute will be populated into the IM column store.
However, it is possible to populate only a subset of columns if desired. For example, the following statement sets the
E.g Enabling the In-Memory attribute on the sales table but excluding the prod_id column
ALTER TABLE sales INMEMORY NO INMEMORY(prod_id)

Similarly, for a partitioned table, all of the table's partitions inherit the in-memory attribute but it is possible to
populate just a subset of the partitions or subpartitions.

7.  This looks at the USER_TABLES view and queries attributes of tables in the SSB schema.  

    ````
    <copy>
    set pages 999
    column table_name format a12
    column cache format a5;
    column buffer_pool format a11;
    column compression heading 'DISK|COMPRESSION' format a11;
    column compress_for format a12;
    column INMEMORY_PRIORITY heading 'INMEMORY|PRIORITY' format a10;
    column INMEMORY_DISTRIBUTE heading 'INMEMORY|DISTRIBUTE' format a12;
    column INMEMORY_COMPRESSION heading 'INMEMORY|COMPRESSION' format a14;
    --QUERY    

    SELECT table_name, cache, buffer_pool, compression, compress_for, inmemory,
        inmemory_priority, inmemory_distribute, inmemory_compression
    FROM   user_tables;
    </copy>
    ````


8.  Let's populate the store with some simple queries.

    ````
    <copy>
    SELECT /*+ full(d)  noparallel (d )*/ Count(*)   FROM   date_dim d;
    SELECT /*+ full(s)  noparallel (s )*/ Count(*)   FROM   supplier s;
    SELECT /*+ full(p)  noparallel (p )*/ Count(*)   FROM   part p;
    SELECT /*+ full(c)  noparallel (c )*/ Count(*)   FROM   customer c;
    SELECT /*+ full(lo) noparallel (lo )*/ Count(*) FROM   lineorder lo;
    </copy>
    ````
    By default, all of the columns in an object with the INMEMORY attribute will be populated into the IM column store.
However, it is possible to populate only a subset of columns if desired. For example, the following statement sets the
In-Memory attribute on the table SALES,
ALTER TABLE sales INMEMORY NO INMEMORY(prod_id);



9. Background processes are populating these segments into the IM column store.  To monitor this, you could query the V$IM_SEGMENTS.  Once the data population is complete, the BYTES_NOT_POPULATED should be 0 for each segment.  

    ````
    <copy>
    column name format a20
    column owner format a15
    column segment_name format a30
    column populate_status format a20
    column bytes_in_mem format 999,999,999,999,999
    column bytes_not_populated format 999,999,999,999,999
    --QUERY

    SELECT v.owner, v.segment_name name, v.populate_status status, v.bytes bytes_in_mem, v.bytes_not_populated
    FROM v$im_segments v;
    </copy>
    ````



10.  Now let's check the total space usage.

    ````
    <copy>
    column alloc_bytes format 999,999,999,999;
    column used_bytes      format 999,999,999,999;
    column populate_status format a15;
    select * from v$inmemory_area;
    exit
    </copy>
    ````


    In this Step you saw that the IM column store is configured by setting the initialization parameter INMEMORY_SIZE. The IM column store is a new static pool in the SGA, and once allocated it can be resized dynamically, but it is not managed by either of the automatic SGA memory features.

    You also had an opportunity to populate and view objects in the IM column store and to see how much memory they use. In this Lab we populated about 1471 MB of compressed data into the  IM column store, and the LINEORDER table is the largest of the tables populated with over 23 million rows.  Remember that the population speed depends on the CPU capacity of the system as the in-memory data compression is a CPU intensive operation. The more CPU and processes you allocate the faster the populations will occur.

11. 	Disabling a table
To disable a table for the IM column store, use the NO INMEMORY clause. Once a table is disabled, its information is purged from the data dictionary views, metadata from the IM column store is cleared, and its in-memory column representation invalidated.
````
<copy>
ALTER TABLE bonus NO INMEMORY ;
</copy>
````
12.	Check the in-memory attributes for table.
````
<copy>
SELECT INMEMORY, INMEMORY_PRIORITY, INMEMORY_COMPRESSION, INMEMORY_DISTRIBUTE, INMEMORY_DUPLICATE
FROM USER_TABLES
WHERE TABLE_NAME = 'BONUS';
</copy>
````