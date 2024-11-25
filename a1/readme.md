## 0 环境搭建

## 1 数据库建表及数据导入

### 1.1 创建关系表

#### 1.1.1 订单表 `ORDERS`

终端显示：

```shell
omm=# CREATE TABLE ORDERS ( O_ORDERKEY    INTEGER NOT NULL,
omm(# O_CUSTKEY    INTEGER NOT NULL,
omm(# O_ORDERSTATUS    CHAR(1) NOT NULL,
omm(# O_TOTALPRICE    DECIMAL(15,2) NOT NULL,
omm(# O_ORDERDATE    DATE NOT NULL,
omm(# O_ORDERPRIORITY    CHAR(15) NOT NULL,
omm(# O_CLERK    CHAR(15) NOT NULL,
omm(# O_SHIPPRIORITY    INTEGER NOT NULL,
omm(# O_COMMENT    VARCHAR(79) NOT NULL);
CREATE TABLE
```

末尾显示 `CREATE TABLE` 表示建表成功。

#### 1.1.2 区域表 `REGION`

```shell
omm=# CREATE TABLE REGION ( R_REGIONKEY    INTEGER NOT NULL,
omm(# R_NAME    CHAR(25) NOT NULL,
omm(# R_COMMENT    VARCHAR(152));
CREATE TABLE
```

#### 1.1.3 国家表 `NATION`

```shell
omm=# CREATE TABLE NATION ( N_NATIONKEY    INTEGER NOT NULL,
omm(# N_NAME    CHAR(25) NOT NULL,
omm(# N_REGIONKEY    INTEGER NOT NULL,
omm(# N_COMMENT    VARCHAR(152));
CREATE TABLE
```

#### 1.1.4 供应商表 `SUPPLIER`

```shell
omm=# CREATE TABLE SUPPLIER ( S_SUPPKEY    INTEGER NOT NULL,
omm(# S_NAME    CHAR(25) NOT NULL,
omm(# S_ADDRESS    VARCHAR(40) NOT NULL,
omm(# S_NATIONKEY    INTEGER NOT NULL,
omm(# S_PHONE    CHAR(15) NOT NULL,
omm(# S_ACCTBAL    DECIMAL(15,2) NOT NULL,
omm(# S_COMMENT    VARCHAR(101) NOT NULL);
CREATE TABLE
```

#### 1.1.5 零部件表 `PART`

```shell
omm=# CREATE TABLE PART ( P_PARTKEY    INTEGER NOT NULL,
omm(# P_NAME    VARCHAR(55) NOT NULL,
omm(# P_MFGR    CHAR(25) NOT NULL,
omm(# P_BRAND    CHAR(10) NOT NULL,
omm(# P_TYPE    VARCHAR(25) NOT NULL,
omm(# P_SIZE    INTEGER NOT NULL,
omm(# P_CONTAINER    CHAR(10) NOT NULL,
omm(# P_RETAILPRICE    DECIMAL(15,2) NOT NULL,
omm(# P_COMMENT    VARCHAR(23) NOT NULL);
CREATE TABLE
```

#### 1.3.6 零部件供应表 `PARTSUPP`

```shell
omm=# CREATE TABLE PARTSUPP ( PS_PARTKEY    INTEGER NOT NULL,
omm(# PS_SUPPKEY    INTEGER NOT NULL,
omm(# PS_AVAILQTY    INTEGER NOT NULL,
omm(# PS_SUPPLYCOST    DECIMAL(15,2) NOT NULL,
omm(# PS_COMMENT    VARCHAR(199) NOT NULL);
CREATE TABLE
```

#### 1.3.7 客户表 `CUSTOMER`

```shell
omm=# CREATE TABLE CUSTOMER ( C_CUSTKEY    INTEGER NOT NULL,
omm(# C_NAME    VARCHAR(25) NOT NULL,
omm(# C_ADDRESS    VARCHAR(40) NOT NULL,
omm(# C_NATIONKEY    INTEGER NOT NULL,
omm(# C_PHONE    CHAR(15) NOT NULL,
omm(# C_ACCTBAL    DECIMAL(15,2) NOT NULL,
omm(# C_MKTSEGMENT    CHAR(10) NOT NULL,
omm(# C_COMMENT    VARCHAR(117) NOT NULL);
CREATE TABLE
```

#### 1.3.8 订单明细表 `LINEITEM`

```shell
omm=# CREATE TABLE LINEITEM ( L_ORDERKEY    INTEGER NOT NULL,
omm(# L_PARTKEY    INTEGER NOT NULL,
omm(# L_SUPPKEY    INTEGER NOT NULL,
omm(# L_LINENUMBER    INTEGER NOT NULL,
omm(# L_QUANTITY    DECIMAL(15,2) NOT NULL,
omm(# L_EXTENDEDPRICE    DECIMAL(15,2) NOT NULL,
omm(# L_DISCOUNT    DECIMAL(15,2) NOT NULL,
omm(# L_TAX    DECIMAL(15,2) NOT NULL,
omm(# L_RETURNFLAG    CHAR(1) NOT NULL,
omm(# L_LINESTATUS    CHAR(1) NOT NULL,
omm(# L_SHIPDATE    DATE NOT NULL,
omm(# L_COMMITDATE    DATE NOT NULL,
omm(# L_RECEIPTDATE    DATE NOT NULL,
omm(# L_SHIPINSTRUCT    CHAR(25) NOT NULL,
omm(# L_SHIPMODE    CHAR(10) NOT NULL,
omm(# L_COMMENT    VARCHAR(44) NOT NULL);
CREATE TABLE
```

通过 `\d+` 命令可看到创建完的 8 个表：

![image-20241125171935924](assets/image-20241125171935924.png)

### 1.2 数据导入

#### 1.2.1 上传文件

将 tpc-h 的 txt 格式数据文件用 SCP 上传到 openGauss 所在的容器上，首先在容器 `omm` 用户下创建目录 `tpc-h/data` 并授予全部权限：

```shell
[omm@cfde74516988 ~]$ pwd   
/home/omm
[omm@cfde74516988 ~]$ mkdir tpc-h   
[omm@cfde74516988 ~]$ cd tpc-h
[omm@cfde74516988 tpc-h]$ mkdir data
```

```shell
[root@cfde74516988 ~]# cd /home/omm
[root@cfde74516988 omm]# chmod 777 tpc-h
[root@cfde74516988 omm]# cd tpc-h
[root@cfde74516988 tpc-h]# chmod 777 data
```

然后退出 omm 用户回到 root，将 txt 格式的数据文件移到该文件夹里（第一次远程连接需要设置密钥）：

![image-20241125182123117](assets/image-20241125182123117.png)

最后授予文件全部权限：

```shell
[root@cfde74516988 ~]# cd /home/omm/tpc-h/data
[root@cfde74516988 data]# chmod 777 *
[root@cfde74516988 data]# ls -l
total 208992
-rwxrwxrwx 1 root root   4835158 Nov 25 10:19 customer.txt
-rwxrwxrwx 1 root root 155651209 Nov 25 10:19 lineitem.txt
-rwxrwxrwx 1 root root      2356 Nov 25 10:19 nation.txt
-rwxrwxrwx 1 root root  25118002 Nov 25 10:19 orders.txt
-rwxrwxrwx 1 root root  23143960 Nov 25 10:19 partsupp.txt
-rwxrwxrwx 1 root root   4982186 Nov 25 10:19 part.txt
-rwxrwxrwx 1 root root       199 Nov 25 10:19 region.txt
-rwxrwxrwx 1 root root    250820 Nov 25 10:19 supplier.txt
```

#### 1.2.2 导入数据

以 omm 用户登录数据库主节点，并连接数据库，然后使用 `\copy` 命令导入数据：

```shell
omm=# copy region FROM '/home/omm/tpc-h/data/region.txt' with delimiter as '|';
COPY 5
omm=# copy nation FROM 'home/omm/tpc-h/data/nation.txt' with delimiter as '|';
ERROR:  could not open file "home/omm/tpc-h/data/nation.txt" for reading: No such file or directory
omm=# copy nation FROM '/home/omm/tpc-h/data/nation.txt' with delimiter as '|';
COPY 25
omm=# copy part FROM '/home/omm/tpc-h/data/part.txt' with delimiter as '|';
COPY 40000
omm=# copy supplier FROM '/home/omm/tpc-h/data/supplier.txt' with delimiter as '|';
COPY 2000
omm=# copy customer FROM '/home/omm/tpc-h/data/customer.txt' with delimiter as '|';
COPY 30000
omm=# copy lineitem FROM '/home/omm/tpc-h/data/lineitem.txt' with delimiter as '|'; 
COPY 1199969
omm=# copy partsupp FROM '/home/omm/tpc-h/data/partsupp.txt' with delimiter as '|';
COPY 160000
omm=# copy orders FROM '/home/omm/tpc-h/data/orders.txt' with delimiter as '|';
COPY 300000
```

通过 `\d+` 命令可看到表的大小有更新：

![image-20241125192423749](assets/image-20241125192423749.png)

导入后为关系表添加约束：

```shell
omm=# ALTER TABLE REGION
omm-# ADD PRIMARY KEY (R_REGIONKEY);
NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "region_pkey" for table "region"
ALTER TABLE

omm=# ALTER TABLE NATION
omm-# ADD PRIMARY KEY (N_NATIONKEY);
NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "nation_pkey" for table "nation"
ALTER TABLE

omm=# ALTER TABLE NATION 
omm-# ADD FOREIGN KEY (N_REGIONKEY) references REGION;
ALTER TABLE

omm=# ALTER TABLE PART
omm-# ADD PRIMARY KEY (P_PARTKEY);
NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "part_pkey" for table "part"
ALTER TABLE

omm=# ALTER TABLE SUPPLIER
omm-# ADD PRIMARY KEY (S_SUPPKEY);
NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "supplier_pkey" for table "supplier"
ALTER TABLE

omm=# ALTER TABLE SUPPLIER
omm-# ADD FOREIGN KEY (S_NATIONKEY) references NATION;
ALTER TABLE

omm=# ALTER TABLE PARTSUPP
omm-# ADD PRIMARY KEY (PS_PARTKEY,PS_SUPPKEY);
NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "partsupp_pkey" for table "partsupp"
ALTER TABLE

omm=# ALTER TABLE CUSTOMER
omm-# ADD PRIMARY KEY (C_CUSTKEY);
NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "customer_pkey" for table "customer"
ALTER TABLE

omm=# ALTER TABLE CUSTOMER
omm-# ADD FOREIGN KEY (C_NATIONKEY) references NATION;
ALTER TABLE

omm=# ALTER TABLE LINEITEM
omm-# ADD PRIMARY KEY (L_ORDERKEY,L_LINENUMBER);
NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "lineitem_pkey" for table "lineitem"
ALTER TABLE

omm=# ALTER TABLE ORDERS
omm-# ADD PRIMARY KEY (O_ORDERKEY);
NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "orders_pkey" for table "orders"
ALTER TABLE

omm=# ALTER TABLE PARTSUPP
omm-# ADD FOREIGN KEY (PS_SUPPKEY) references SUPPLIER;
ALTER TABLE

omm=# ALTER TABLE PARTSUPP
omm-# ADD FOREIGN KEY (PS_PARTKEY) references PART;
ALTER TABLE

omm=# ALTER TABLE ORDERS
omm-# ADD FOREIGN KEY (O_CUSTKEY) references CUSTOMER;
ALTER TABLE

omm=# ALTER TABLE LINEITEM
omm-# ADD FOREIGN KEY (L_ORDERKEY) references ORDERS;
ALTER TABLE

omm=# ALTER TABLE LINEITEM
omm-# ADD FOREIGN KEY (L_PARTKEY,L_SUPPKEY) references PARTSUPP;
ALTER TABLE
```

用 `select` 语句查看关系表可以显示刚刚导入进来的数据：

![image-20241125193718859](assets/image-20241125193718859.png)
