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
にもかかわらず、世の中には dbt を使って dimentional modeling を行うための資料が足りていません。。。つらいね。。。というわけで、このチュートリアルで dimentional modeling の決定版ガイドを提供したいと思います。

（※）その他のデータモデリング手法には、Data Vault (DV)、Third Normal Form (3NF)、One Big Table (OBT) などがあります。[元記事より拝借](https://docs.getdbt.com/img/blog/2023-04-18-building-a-kimball-dimensional-model-with-dbt/data-modelling.png)：

<img src="./img/data-modelling.png" width=600 title="Data modeling techniques on a normalization vs denormalization scale">

このチュートリアルを完了すると、次のことができるようになります。

- dimentional modeling の概念を理解する
- モック dbt プロジェクトとデータベースをセットアップする
- モデル化するビジネス プロセスを特定する
- ファクト テーブルとディメンション テーブルを特定する
- ディメンション テーブルを作成する
- ファクト テーブルを作成する
- dimentional modeling のリレーションをドキュメント化する
- dimentional modeling を使用する

## Dimentional modeling とは
Dimentional modeling は、1996年にRalph Kimball氏が著書「The Data Warehouse Toolkit」で紹介した手法です。
Dimentional modeling の目的は、raw データを、ビジネスを表現するファクトテーブルとディメンションテーブルに変換することです。

ディメンショナルモデリングのメリットを挙げます：

- 分析用のデータモデルがよりシンプルになる： 分析用にdimentional model を使用する際、ファクトテーブルとディメンションテーブル間の結合は、サロゲートキーを使用することで簡単に行うことができ、複雑な結合を行う必要がない
- Don’t repeat yourself[^1]：ディメンションは、他のファクトテーブルで簡単に再利用でき、労力とコード・ロジックの重複を避けることができます。再利用可能なディメンジョンは、[コンフォームド・ディメンジョン](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/conformed-dimension/)と呼ばれます。
- データ検索の高速化： Dimentional model に対して実行される分析クエリは、結合や集約などのデータ変換がすでに適用されているため、3NFモデルよりも大幅に高速です。
- 実際のビジネスプロセスとの密接な整合性：ビジネスプロセスとメトリクスは、dimentional model の一部としてモデル化され、計算される。これにより、モデル化されたデータが容易に利用できるようになる

[^1]: DRYとは、"Don't Repeat Yourself "の略で、ソフトウェア開発の原則の1つです。この原則に従うと、繰り返しのパターンや重複するコードやロジックを減らし、モジュール化された参照可能なコードにすることを目指すことになります。

さて、dimentional modeling の大まかな概念と利点を理解したところで、実際に dbt を使用して最初の dimentional model を作成してみましょう。

## 準備
実行環境は docker で作成します。Docker Desktop version 23.0.5 で動作確認しています。

## Part 1: Setup dbt project and database
### Step 1: Before you get started

これらを準備します：

- [DuckDB](https://duckdb.org/docs/installation/index) または [PostgreSQL](https://www.postgresql.org/download/) 
- Python 3.8 以上
- dbt 1.3.0 以上
- SQLの基本的な理解
- dbtの基本的な理解

はい、作りましょう。[^1]

[^1]: ここでは PostgreSQL を使用します

```
$ docker compose up -d --build

[+] Running 2/2
 - Container kimball-dimensional-model-dev_postgres-1  Started                                           1.2s
 - Container kimball-dimensional-model-dev_dbt-1       Started                                           1.3s
```

dbt コマンドの確認と、 PostgreSQL への疎通確認をします。

```
$ docker compose exec dev_dbt dbt debug
18:06:40  Running with dbt=1.5.0
18:06:40  dbt version: 1.5.0
18:06:40  python version: 3.11.2
18:06:40  python path: /usr/local/bin/python
18:06:40  os info: Linux-5.10.16.3-microsoft-standard-WSL2-x86_64-with-glibc2.31
18:06:40  Using profiles.yml file at /root/.dbt/profiles.yml
18:06:40  Using dbt_project.yml file at /usr/app/dbt/dbt_project.yml
18:06:40  Configuration:
18:06:40    profiles.yml file [OK found and valid]
18:06:40    dbt_project.yml file [ERROR not found]
18:06:40  Required dependencies:
18:06:40   - git [OK found]

18:06:40  Connection:
18:06:40    host: localhost
18:06:40    port: 5430
18:06:40    user: postgres
18:06:40    database: adventureworks
18:06:40    schema: dbo
18:06:40    search_path: None
18:06:40    keepalives_idle: 0
18:06:40    sslmode: None
18:06:40    Connection test: [OK connection ok]

18:06:40  1 check failed:
18:06:40  Could not load dbt_project.yml
```

`dbt_project.yml` はまだ作っていないので、これでOKです。

### Step 2: Clone the repository

ソースコードがあるリポジトリをcloneしましょう。

```
git clone https://github.com/Data-Engineer-Camp/dbt-dimensional-modelling.git
cd dbt-dimensional-modelling/adventureworks
```

<details>
<summary>実行例</summary>

```
root@docker-desktop:/usr/app/dbt# git clone https://github.com/Data-Engineer-Camp/dbt-dimensional-modelling.git
Cloning into 'dbt-dimensional-modelling'...
remote: Enumerating objects: 361, done.
remote: Counting objects: 100% (80/80), done.
remote: Compressing objects: 100% (16/16), done.
remote: Total 361 (delta 66), reused 64 (delta 64), pack-reused 281
Receiving objects: 100% (361/361), 4.55 MiB | 3.75 MiB/s, done.
Resolving deltas: 100% (176/176), done.
root@docker-desktop:/usr/app/dbt# cd dbt-dimensional-modelling/adventureworks
```

</details>

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
