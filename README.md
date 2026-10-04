# Parameter-Driven Azure Bronze Data Ingestion Pipeline

A reusable and parameter-driven Azure Data Factory pipeline designed to incrementally ingest Spotify data from Azure SQL Database into an Azure Data Lake Storage Gen2 Bronze layer in Parquet format.

The project focuses on building a reusable cloud data ingestion framework that can process multiple source tables using the same pipeline logic.

---

## Project Overview

This project implements a Bronze-layer data ingestion pipeline using Azure Data Factory.

The source data consists of Spotify-related tables stored in Azure SQL Database. Instead of creating a separate pipeline for every table, the solution uses a parameterized configuration to dynamically determine:

- Source schema
- Source table
- CDC / watermark column
- Optional historical starting date

The pipeline then performs incremental ingestion and stores the extracted data in the Bronze layer of Azure Data Lake Storage Gen2.

The project intentionally focuses on the **Bronze ingestion layer** and does not include Silver or Gold transformations.

---

## Architecture

```text
                    Azure SQL Database
                           |
                           | Incremental SQL Query
                           v
                  Azure Data Factory
                           |
                  Parameterized Input
                           |
                           v
                       ForEach
                           |
             +-------------+-------------+
             |                           |
       CDC / Watermark              Table Configuration
             |                           |
             +-------------+-------------+
                           |
                           v
                    Copy Activity
                           |
                           v
              Azure Data Lake Storage Gen2
                           |
                        Bronze
                           |
        +------------------+------------------+
        |                  |                  |
     DimUser           DimTrack          DimArtist
        |                  |                  |
     DimDate          FactStream              |
        |                  |                  |
        +------------------+------------------+
