# How to migrate SSC data from MySQL to SQL Server
This is a work in progress.

## MySQL Connector/NET
1. Download and install [MySQL Connector/NET](https://dev.mysql.com/downloads/connector/net/).
(**Note:** It’s also possible to perform the migration using the [MySQL ODBC drivers](https://dev.mysql.com/downloads/connector/odbc/), but it has some quirks that can end up wasting your time.)

2. Assuming you have the latest version of SSMS (version 20.x as of this writing), open the file `C:\Program Files (x86)\Microsoft SQL Server Management Studio 20\Common7\IDE\CommonExtensions\Microsoft\SSIS\160\ProviderDescriptors\ProviderDescriptors.xml` in your favorite text editor and create a couple of empty lines after line 197.

3. Copy the following snippet and paste it at the end of `ProviderDescriptors.xml` right before `</dtm:ProviderDescriptors>`:
   ```xml
       <dtm:ProviderDescriptor SourceType="MySql.Data.MySqlClient.MySqlConnection">
   
           <dtm:SchemaNames
               TablesSchemaName="Tables"
               ColumnsSchemaName="Columns"
               ViewsSchemaName="Views"
           />
           
           <dtm:TableSchemaAttributes
               TableCatalogColumnName=""
               TableSchemaColumnName=""
               TableNameColumnName="TABLE_NAME"
               TableTypeColumnName="TABLE_TYPE"
               TableDescriptor="BASE TABLE"
               ViewDescriptor="VIEW"
               SynonymDescriptor ="SYNONYM"
               NumberOfTableRestrictions="200"
           />
   
           <dtm:ColumnSchemaAttributes
               NameColumnName = "COLUMN_NAME"
               OrdinalPositionColumnName="ORDINAL_POSITION"
               DataTypeColumnName = "DATA_TYPE"
               MaximumLengthColumnName = "CHARACTER_MAXIMUM_LENGTH"
               NumericPrecisionColumnName = "NUMERIC_PRECISION"
               NumericScaleColumnName = "NUMERIC_SCALE"
               NullableColumnName="IS_NULLABLE"
               DateTimePrecisionColumnName="DATETIME_PRECISION"
               NumberOfColumnRestrictions="200"
           />
   
           <dtm:Literals
               PrefixQualifier=""
               SuffixQualifier=""
               CatalogSeparator="."
               SchemaSeparator="."
           />
       </dtm:ProviderDescriptor>
   ```

## Create and prepare the SSC schema on SQL Server
1. Using SSMS, create a new SSC database in SQL Server with case-sensitive collation (e.g., `SQL_Latin1_General_CP1_CS_AS`). Also make sure to enable “Is Read Committed Snapshot” and “Allow Snapshot Isolation”.
   ```tsql
   CREATE DATABASE ssc
   COLLATE SQL_Latin1_General_CP1_CS_AS;

   ALTER DATABASE ssc
   SET ALLOW_SNAPSHOT_ISOLATION ON;

   ALTER DATABASE ssc
   SET READ_COMMITTED_SNAPSHOT ON;

   ALTER DATABASE ssc
   SET AUTO_UPDATE_STATISTICS_ASYNC ON;

   ALTER DATABASE ssc
   SET RECOVERY FULL;
   ```
2. Open the `create-tables.sql` script for SQL Server in SSMS corresponding to the same version of the SSC schema in the existing MySQL database. (For example, if the SSC schema version for the existing MySQL database is 21.1.2, open the `create-tables.sql` script for SSC 21.1.2 under `<Fortify_21.1.2_Server_WAR_Tomcat>\sql\sqlserver`).

3. Run the script on the new database to generate the schema.

4. Disable all constraints on the new database:
   ```tsql
   -- Disable all constraints for database
   EXEC sp_MSforeachtable "ALTER TABLE ? NOCHECK CONSTRAINT all";
   ```
   **Note:** This is only temporary. We will re-enable constraints at the end of the data migration.

5. Run the following query on the new database to find all non-empty tables:
   ```tsql
   SELECT 'Table Name'=convert(char(25),t.TABLE_NAME), 'Total Record Count'=max(i.rows)
   FROM sysindexes i, INFORMATION_SCHEMA.TABLES t
   WHERE t.TABLE_NAME = object_name(i.id)
   AND t.TABLE_TYPE = 'BASE TABLE'
   GROUP BY t.TABLE_SCHEMA, t.TABLE_NAME
   HAVING max(i.rows)>0
   ORDER BY 'Total Record Count' DESC;
   ```
6. Truncate all tables in the result from the previous step. If one of the tables is `bugfieldtemplategroup`, you’ll need to first drop the `FK_BugfieldTemplateGroupId` constraint and recreate it after the truncate command. For example:
   ```tsql
   TRUNCATE TABLE DATABASECHANGELOG;
   TRUNCATE TABLE configproperty;
   TRUNCATE TABLE bugfieldtemplate;
   TRUNCATE TABLE applicationstate;
   
   ALTER TABLE dbo.bugfieldtemplate
   DROP CONSTRAINT FK_BugfieldTemplateGroupId;
   
   TRUNCATE TABLE bugfieldtemplategroup;

   ALTER TABLE dbo.bugfieldtemplate
   ADD CONSTRAINT FK_BugfieldTemplateGroupId
   FOREIGN KEY (bugfieldTemplateGroup_id) REFERENCES dbo.bugfieldtemplategroup (id)
   ON DELETE CASCADE;
   ```
## Prepare the MySQL database
1. On the existing SSC database in MySQL, run the following SQL commands to change the data type of a couple of columns:
   ```sql
   ALTER TABLE f360global MODIFY privateKey VARBINARY(3000);
   ALTER TABLE f360global MODIFY publicKey VARBINARY(3000);
   ```
   If you don’t change the data type from `BLOB` to `VARBINARY`, the Import and Export Wizard will throw an error about incompatible data types between source and target columns. If you wish, you can change it back to `BLOB` after the migration is complete.

2. Add `ANSI_QUOTES` to the `sql_mode` system variable. You can do this directly in MySQL Workbench. Here’s a screenshot for reference:
   (add screenshot)
   If you don’t do this, the migration will fail.

## Migrate the data from MySQL to SQL Server
1. In SSMS, right-click on the new (and empty) SSC database and select **Tasks > Import Data**. The SQL Server Import and Export Wizard will open.

2. In the “Choose a data Source” section, select “.Net Framework Data Provider for MySQL” from the “Data source” drop-down list.

3. Fill in the necessary fields. Usually, you only need to fill out the following:
   - **Database:** `ssc`
   - **Server:** `localhost`
   - **Password:** `*****`
   - **User ID:** `root`
  
   If MySQL is located on a remote machine and encryption is required, you’ll need to fill out the necessary fields. Click Next.

4. (Need to update this step) In the “Choose a Destination” section, select “SQL Server Native Client 11.0” from the “Destination” drop-down list. Enter the appropriate server name and database name. (Make sure it’s the same database that you created in the “Create and prepare the SSC schema on SQL Server” section above.) Click Next.

5. In the “Specify Table Copy or Query” step, make sure “Copy data from one or more tables or views” is selected and click Next.

6. In the “Select Source Tables and Views” section, select everything by clicking the checkbox to the left of the “Source” column. While everything is selected and highlighted, click the “Edit Mappings” button and select the option “Enable identity insert”. Here’s a screenshot for reference:
   > (Include screenshot)

   Click OK.

7. Back in the “Select Source Tables and Views” section, look for the databasechangelog table. In SQL Server, that table is in all upper-case. Under the Destination column, select `[dbo].[DATABASECHANGELOG]`. Here’s a screenshot for reference:
   > (Include screenshot)

8. Before you click Next, highlight all tables again and click the “Edit Mappings” button. Make sure “Enable identity insert” is selected.

9. Click Next until you reach the end. Then click the Finish button to start the migration process.

10. Once the migration has successfully completed, enable all constraints on the SSC database in SQL Server:
    ```tsql
    -- Enable all constraints for database
    EXEC sp_MSforeachtable "ALTER TABLE ? WITH CHECK CHECK CONSTRAINT all";
    ```

11. In the MySQL database, undo the change to the data type of the two `VARBINARY` columns:
    ```tsql
    ALTER TABLE f360global MODIFY privateKey BLOB;
    ALTER TABLE f360global MODIFY publicKey BLOB;
    ```
12. Remove `ANSI_QUOTES` from the `sql_mode` system variable.
