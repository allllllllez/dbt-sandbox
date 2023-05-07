# Building a Kimball dimensional model with dbt <!-- omit in toc -->
[Building a Kimball dimensional model with dbt _ dbt Developer Blog](https://docs.getdbt.com/blog/kimball-dimensional-model) をやるよ。

# 目次 <!-- omit in toc -->
- [解説](#解説)
    - [はじめに](#はじめに)
    - [Dimentional modeling とは](#dimentional-modeling-とは)
    - [準備](#準備)
    - [Part 1: Setup dbt project and database](#part-1-setup-dbt-project-and-database)
        - [Step 1: Before you get started](#step-1-before-you-get-started)
        - [Step 2: Clone the repository](#step-2-clone-the-repository)
        - [Step 3: Install dbt database adaptors](#step-3-install-dbt-database-adaptors)
        - [Step 4: Setup dbt profile](#step-4-setup-dbt-profile)
        - [Step 5: Install dbt dependencies](#step-5-install-dbt-dependencies)
        - [Step 6: Seed your database](#step-6-seed-your-database)
        - [Step 7: Examine the database source schema](#step-7-examine-the-database-source-schema)
        - [Step 8: Query the tables](#step-8-query-the-tables)
    - [Part 2: Identify the business process](#part-2-identify-the-business-process)
    - [Part 3: Identify the fact and dimension tables](#part-3-identify-the-fact-and-dimension-tables)
    - [Fact tables](#fact-tables)
    - [Dimension tables](#dimension-tables)
    - [Part 4: Create the dimension tables](#part-4-create-the-dimension-tables)
        - [Step 1: Create model files](#step-1-create-model-files)
        - [Step 2: Fetch data from the upstream tables](#step-2-fetch-data-from-the-upstream-tables)
        - [Step 3: Perform the joins](#step-3-perform-the-joins)
        - [Step 4: Create the surrogate key](#step-4-create-the-surrogate-key)
        - [Step 5: Select dimension table columns](#step-5-select-dimension-table-columns)
        - [Step 6: Choose a materialization type](#step-6-choose-a-materialization-type)
        - [Step 7: Create model documentation and tests](#step-7-create-model-documentation-and-tests)
        - [Step 8: Build dbt models](#step-8-build-dbt-models)
    - [Part 5: Create the fact table](#part-5-create-the-fact-table)
        - [Step 1: Create model files](#step-1-create-model-files-1)
        - [Step 2: Fetch data from the upstream tables](#step-2-fetch-data-from-the-upstream-tables-1)
        - [Step 3: Perform joins](#step-3-perform-joins)
        - [Step 4: Create the surrogate key](#step-4-create-the-surrogate-key-1)
        - [Step 5: Select fact table columns](#step-5-select-fact-table-columns)
        - [Step 6: Create foreign surrogate keys](#step-6-create-foreign-surrogate-keys)
        - [Step 7: Choose a materialization type](#step-7-choose-a-materialization-type)
        - [Step 8: Create model documentation and tests](#step-8-create-model-documentation-and-tests)
        - [Step 9: Build dbt models](#step-9-build-dbt-models)
    - [Part 6: Document the dimensional model relationships](#part-6-document-the-dimensional-model-relationships)
    - [Part 7: Consume dimensional model](#part-7-consume-dimensional-model)

# 解説
## はじめに
Dimentional modeling はデータモデリング手法（※）の一つで、分析用に最も広く採用されている手法です。
にもかかわらず、世の中に dbt を使って dimentional modeling を行うための資料が足りていないので、このチュートリアルでは dimentional modeling の決定版ガイドを提供します。

（※）その他のデータモデリング手法には、Data Vault (DV)、Third Normal Form (3NF)、One Big Table (OBT) などがあります。[元記事より拝借](https://docs.getdbt.com/img/blog/2023-04-18-building-a-kimball-dimensional-model-with-dbt/data-modelling.png)：

<img src="./img/data-modelling.png" width=600 title="Data modeling techniques on a normalization vs denormalization scale">

このチュートリアルを終了すると、次のことができるようになります。

- dimentional modeling の概念を理解する
- モック dbt プロジェクトとデータベースをセットアップする
- モデル化するビジネス プロセスを特定する
- ファクト テーブルとディメンション テーブルを特定する
- ディメンション テーブルを作成する
- ファクト テーブルを作成する
- dimentional modeling のリレーションをドキュメント化する
- dimentional modeling を使用する

## Dimentional modeling とは
Dimentional modeling は、1996年にRalph Kimball氏が著書「The Data Warehouse Toolkit」で紹介した手法である。

Dimentional modeling の目的は、raw データを、ビジネスを表現するファクトテーブルとディメンションテーブルに変換することである。

ディメンショナルモデリングのメリットを挙げる：

- 分析用のデータモデルがよりシンプルになる： 分析用にdimentional model を使用する際、ファクトテーブルとディメンションテーブル間の結合は、サロゲートキーを使用することで簡単に行うことができ、複雑な結合を行う必要がない
- Don’t repeat yourself[^1]：ディメンションは、他のファクトテーブルで簡単に再利用でき、労力とコード・ロジックの重複を避けることができます。再利用可能なディメンジョンは、[コンフォームド・ディメンジョン](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/conformed-dimension/)と呼ばれます。
- データ検索の高速化： Dimentional model に対して実行される分析クエリは、結合や集約などのデータ変換がすでに適用されているため、3NFモデルよりも大幅に高速です。
- 実際のビジネスプロセスとの密接な整合性：ビジネスプロセスとメトリクスは、dimentional model の一部としてモデル化され、計算されます。これにより、モデル化されたデータが容易に利用できるようになります。

[^1]: DRYとは、"Don't Repeat Yourself "の略で、ソフトウェア開発の原則の1つです。この原則に従うと、繰り返しのパターンや重複するコードやロジックを減らし、モジュール化された参照可能なコードにすることを目指すことになります。

さて、dimentional modeling の大まかな概念と利点を理解したところで、実際にdbtを使用して最初の dimentional model を作成してみましょう。

## 準備
## Part 1: Setup dbt project and database
これらを準備します：
- [DuckDB](https://duckdb.org/docs/installation/index) または [PostgreSQL](https://www.postgresql.org/download/) 
- Python 3.8 以上
- dbt 1.3.0 以上
- SQLの基本的な理解
- dbtの基本的な理解

はい、作りましょう。

```
$ docker compose up -d --build
：（中略）
[+] Running 2/2
 - Container kimball-dimensional-model-dev_postgres-1  Started                                          11.3s
 - Container kimball-dimensional-model-dev_dbt-1       Started                                          11.4s

you@DESKTOP-PFHGAJG MINGW64 /c/git/private-kajiya-dbt-sandbox/kimball-dimensional-model (main)
$ docker compose ps
NAME                                       IMAGE                             COMMAND                  SERVICE             CREATED             STATUS              PORTS
kimball-dimensional-model-dev_dbt-1        ghcr.io/dbt-labs/dbt-core:1.5.0   "bash"                   dev_dbt             14 seconds ago      Up 12 seconds
kimball-dimensional-model-dev_postgres-1   postgres                          "docker-entrypoint.s…"   dev_postgres        24 seconds ago      Up 12 seconds       0.0.0.0:5430->5432/tcp

$ docker compose exec dev_dbt bash
root@docker-desktop:/usr/app/dbt#
```



```
postgres=# create database adventureworks;
CREATE DATABASE
postgres=# \c adventureworks;
psql (13.10 (Debian 13.10-0+deb11u1), server 15.2 (Debian 15.2-1.pgdg110+1))
WARNING: psql major version 13, server major version 15.
         Some psql features might not work.
You are now connected to database "adventureworks" as user "root".
```

### Step 1: Before you get started
### Step 2: Clone the repository
### Step 3: Install dbt database adaptors
### Step 4: Setup dbt profile
### Step 5: Install dbt dependencies
### Step 6: Seed your database
### Step 7: Examine the database source schema
### Step 8: Query the tables
## Part 2: Identify the business process
## Part 3: Identify the fact and dimension tables 
## Fact tables 
## Dimension tables
## Part 4: Create the dimension tables
### Step 1: Create model files
### Step 2: Fetch data from the upstream tables
### Step 3: Perform the joins
### Step 4: Create the surrogate key
### Step 5: Select dimension table columns
### Step 6: Choose a materialization type
### Step 7: Create model documentation and tests
### Step 8: Build dbt models
## Part 5: Create the fact table
### Step 1: Create model files
### Step 2: Fetch data from the upstream tables
### Step 3: Perform joins
### Step 4: Create the surrogate key
### Step 5: Select fact table columns
### Step 6: Create foreign surrogate keys
### Step 7: Choose a materialization type
### Step 8: Create model documentation and tests
### Step 9: Build dbt models
## Part 6: Document the dimensional model relationships
## Part 7: Consume dimensional model
Learning resources
