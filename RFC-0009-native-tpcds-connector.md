# **RFC-0009 for Presto**

## Presto - Native TPC-DS connector

Proposers

* Pratik Joseph Dabre
* Pramod Satya

## [Related Issues]

Related issues: https://github.com/prestodb/presto/issues/22361

Related PRs: https://github.com/prestodb/presto/pull/23067

## Summary

A native TPC-DS connector capable of generating in-memory data on the fly is proposed.

## Background

- TPC-DS Benchmarks:
The TPC-DS benchmark is a well-known benchmark suite developed by the Transaction Processing Performance Council (TPC) for evaluating the performance of database systems handling complex queries and data processing workloads typical of data warehouses.
1. Purpose: The TPC-DS benchmark tests a system's ability to handle a variety of SQL queries, including ad-hoc, reporting, OLAP (Online Analytical Processing) and data mining workloads. It focuses on measuring query performance, throughput and response times. It includes 99 pre-defined queries of varying complexity, including joins, aggregations, sub-queries and sorts which are representative of real-world analytics and reporting queries in DSS(Decision Support Systems) environments.
2. TPC-DS Schema: The TPC-DS schema is designed to mimic a real-wold retail sales business and consists of 24 tables. The key elements of the schema are:
   - Fact Tables: These tables store large amounts of transactional data.
   - Dimension Tables: These tables store information about dimensions such as time, products, customers and locations.
3. DSDGen: To generate data for the TPC-DS benchmark, the dsdgen program is used. It is a tool specifically built to generate large volumes of synthetic data that matches the structure and distributions defined by the TPC-DS specification. It generates realistic datasets at different scale factors, which allows the simulation of different database sizes.

- Currently , Presto does not have a native implementation of the TPC-DS connector. This RFC proposes the addition of a new TPC-DS connector which will allow the generation of synthetic data on the fly confirming to the specifications provided by the TPC org. The new connector can be used as a Presto - Native catalog.

### [Optional] Goals

1. Add a TPC-DS connector to generate TPC-DS data in Presto native. 
2. Write end-to-end tests in Presto native with TPC-DS tables. 

### [Optional] Non-goals

## Proposed Implementation

The Presto - Native TPC-DS connector will be a wrapper for the generator distributed (dsdgen) by the TPC organization from C. This means we need our implementation to have the exact same behavior as the C implementation. DuckDB already has a TPC-DS connector of their own and they have wrapped the C files into C++ files, we are going to use these C++ files in our implementation.

In the C++ implementation, there are two types of tables: source tables and target tables used for generation. Source table files are prefixed with "s_", while target table files are prefixed with "w_". For instance, there may be files like "s_call_center.c" and "w_call_center.c". Currently, our focus is solely on implementing functionalities for the target tables (w_ tables).

In the target table files prefixed with “w_”, there are some helper functions(need to be implemented by us) precisely called as “append_row_start“ and “append_row_end“ which help in the row generation. Depending on the schema of the table, there will be “append_ “ functions depending on the data type to be appended.

A new TPC-DS config `tpcds.toggle-char-to-varchar` will be added to toggle the char columns to varchar, addressing the lack of support for the char data type in Presto - Native. This config allows the toggling of the char to varchar when required, ensuring consistency between Presto - Java and Presto - Native. At the schema level, all the columns which were of char type would be cast to a varchar type.
   - `SHOW COLUMNS` on `call_center` table when the `tpcds.toggle-char-to-varchar` is set to false.
     ```
     presto:sf1> show columns in call_center;

     Column    |   Type   | Extra | Comment
     -------------------+--------------+-------+---------
     cc_call_center_sk | bigint    |    |
     cc_call_center_id | char(16)   |    |
     cc_rec_start_date | date     |    |
     cc_rec_end_date  | date     |    |
     cc_closed_date_sk | integer   |    |
     cc_open_date_sk  | integer   |    |
     cc_name      | varchar(50) |    |
     cc_class     | varchar(50) |    |
     cc_employees   | integer   |    |
     cc_sq_ft     | integer   |    |
     cc_hours     | char(20)   |    |
     cc_manager    | varchar(40) |    |
     cc_mkt_id     | integer   |    |
     cc_mkt_class   | char(50)   |    |
     cc_mkt_desc    | varchar(100) |    |
     cc_market_manager | varchar(40) |    |
     cc_division    | integer   |    |
     cc_division_name | varchar(50) |    |
     cc_company    | integer   |    |
     cc_company_name  | char(50)   |    |
     cc_street_number | char(10)   |    |
     cc_street_name  | varchar(60) |    |
     cc_street_type  | char(15)   |    |
     cc_suite_number  | char(10)   |    |
     cc_city      | varchar(60) |    |
     cc_county     | varchar(30) |    |
     cc_state     | char(2)   |    |
     cc_zip      | char(10)   |    |
     cc_country    | varchar(20) |    |
     cc_gmt_offset   | decimal(5,2) |    |
     cc_tax_percentage | decimal(5,2) |    |
     (31 rows)
     ```

   - `SHOW COLUMNS` on `call_center `table when the `tpcds.toggle-char-to-varchar` is set to true.
   ```
     presto:sf1> show columns in call_center;
     Column    |   Type   | Extra | Comment
     -------------------+--------------+-------+---------
     cc_call_center_sk | bigint    |    |     
     cc_call_center_id | varchar(16) |    |     
     cc_rec_start_date | date     |    |     
     cc_rec_end_date  | date     |    |     
     cc_closed_date_sk | integer   |    |     
     cc_open_date_sk  | integer   |    |     
     cc_name      | varchar(50) |    |     
     cc_class     | varchar(50) |    |     
     cc_employees   | integer   |    |     
     cc_sq_ft     | integer   |    |     
     cc_hours     | varchar(20) |    |     
     cc_manager    | varchar(40) |    |     
     cc_mkt_id     | integer   |    |     
     cc_mkt_class   | varchar(50) |    |     
     cc_mkt_desc    | varchar(100) |    |     
     cc_market_manager | varchar(40) |    |     
     cc_division    | integer   |    |     
     cc_division_name | varchar(50) |    |     
     cc_company    | integer   |    |     
     cc_company_name  | varchar(50) |    |     
     cc_street_number | varchar(10) |    |     
     cc_street_name  | varchar(60) |    |     
     cc_street_type  | varchar(15) |    |     
     cc_suite_number  | varchar(10) |    |     
     cc_city      | varchar(60) |    |     
     cc_county     | varchar(30) |    |     
     cc_state     | varchar(2)  |    |     
     cc_zip      | varchar(10) |    |     
     cc_country    | varchar(20) |    |     
     cc_gmt_offset   | decimal(5,2) |    |     
     cc_tax_percentage | decimal(5,2) |    |     
     (31 rows)
     
   ```
- Generation of tables:
The TpcdsSplitManager in Presto partitions the data for a table into chunks known as splits, which will then be distributed among workers for processing. The number of splits processed by each worker is defined by the Tpcds connector config tpcds.splits-per-node. Each Presto split will be processed by the native worker using velox connector interfaces. First, the Presto TPC-DS split, column handle, and table layout will be converted to the corresponding velox ConnectorSplit, ColumnHandle, and ConnectorTableHandle respectively. The velox connector interfaces to add and process splits will be implemented for the TPC-DS connector. To generate TPC-DS data with this connector for instance, the connector interface DataSource, implemented as TpcdsDataSource, will be used. The addSplit API in TpcdsDataSource will consume a TpcdsSplit, and generate rows corresponding to the offsets in this split by calling the API next until all rows in this split are processed.

## Future Work
- Update data : The existing APIs will work for generating the update data using the Presto-Native TPC-DS connector, however it would require the addition of more Dsdgen codegen files (prefixed with s_*)). The generation of update data is still unsupported in Presto: https://github.com/prestodb/presto/issues/23135
## Test Plan

Native end-to-end tests are added in https://github.com/prestodb/presto/pull/23067. 
Future enhancements will include adding SpeedTest and ConnectorTest to the Velox repository.
