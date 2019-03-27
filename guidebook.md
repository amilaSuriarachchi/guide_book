# Section 0 : Lab Setup
## Section Goal

Before we get started building concurrent imports, we need to get our lab instance set up.

In this lab first you will create a Data Source, Transform Map and a Scheduled Import. Then run the Scheduled Import with and without concurrent import. 

## Clone the git repository to your machine

1. Run the following command

    `git clone https://github.com/amilaSuriarachchi/knowledge.git knowledge`

2. This downloads the lab content to your machine

## Import the lab set up to your machine

1. Goto **System Application -> Studio**

    ![](./images/section0/studio_initial.png)

2. Click on **Import From Source Control**
 
    ![](./images/section0/import_application.png)

3. Set the URL as `https://github.com/amilaSuriarachchi/knowledge.git`

4. Click **Import**

    ![](./images/section0/import_application_sucess.png)

5. Click **Select Application**

    ![](./images/section0/select_application.png)

6. Click **Sales**

    ![](./images/section0/sales.png)

# Section 1 : Set up **Data Source** and **Transform Map**

## Section Goal

**Data Sources** are used to specify from where to import data to ServiceNow. **Transform Maps** are used to define the mapping between the imported data and ServiceNow instance tables. In this section you will learn how to create a **Data Source** to import sales data and create a **Transform Map** to map the imported data to Sales table. Sales table is a predefined table for the application and loaded when importing application from the source control.

## Creating **Sales Data Source**

1. Go to **System Import Sets** -> **Administration** -> **Data Sources**

2. Click **New**

    ![](./images/section1/data_source_new.png)

3. Enter **Name** : Sales Data Source

4. Enter **Import set table label** : Sales Import Set

5. Click on **Manage Attachments** and choose `50000 Sales Records.csv` from the application cloned location.

6. Right click on top panel and click **Save**

    ![](./images/section1/data_source.png)

## Creating **Transform Map**

1. Click on **Test Load 20 Records**. This loads the first 20 records from the '50000 Sales Records.csv` to staging table *Sales Data Source[sn_sales_sales_import_set]*.

    ![](./images/section1/progress.png)

2. Click on **Create Transform Map**

    ![](./images/section1/transform_map_new.png)

3. Enter **Name** : Sales Transform Map

4. Set **Target table** : Sales[sn_sales_sales]

5. Clieck on **Auto Map Matching Fields**. This maps the staging table *Sales Data Source[sn_sales_sales_import_set]* fields to target table *Sales[sn_sales_sales]* fields.

## Creating a **Coalesce** field

1. On the *u_order_id* row set Coalesce to *true*

2. Click on  **Index Coalesce Fields**. Always it is important to index coalesce fileds to avoid performance issues.

3. Select **Do not notify me** and click **OK**

## Setting the date format to date fields

1. Click on *u_ship_date* row.

2. Set the **Date format** : *MM/dd/yyyy*. This is the date format used in csv file to specify dates. System uses this format to interpret the date and store it as a date filed in ServiceNow table.

3. Click on **Update**

4. Do the same to *u_order_date*

# Section 2 : Create a **Scheduled Import** and Import Data

## Section Goal

**Scheduled Imports** are used to periodically fetch data into ServiceNow instances. When importing data first it creates an import set and imports data into it. We can monitor the progress of the import through the import set. In this section we creates a schedlued import and execute it. Then monitor the performance through import set run statistics. We will see how we can detect perfomance bottle necks using **Slow Queries** and how to stop transformations at the middle if required.

## Creating **Scheduled Import**

1. Goto **System Import Sets** -> **Administration** -> **Scheduled Imports**

2. Click on **New**

    ![](./images/section2/scheduled_import_new.png)

3. Set **Name** : Sales Import

4. Select the **Data source** we created in Section 1.

5. Check **Execute pre-import** script and set the following **Pre script**

    ```javascript
    gs.info('Starting import with ' + import_set.number);
    ```

6. Check **Execute post-import script** and set the following **Post script**
    
    ```javascript
    gs.info('Finishing import with ' + import_set.number);
    ```

7. Right click on top panel and click **Save**

    ![](./images/section2/scheduled_import.png) 

## Running the **Scheduled Import**

1. Click on **Execute Now**. This starts the Schedule import as a back ground job.

2. Goto **System Import Sets** -> **Advanced** -> **Import Sets**

3. Click on *ISET0010002*. This is the import set created for the scheduled import. The **Import Set Rows** tab shows you all the import set rows and their state of processing. The **Import Set Runs** tab shows the statistics of curent import set processing.

    ![](./images/section2/import_set_view.png)

4. Observe the import set run progress slowly. We can diagnose such scenarios using **System Diagnostics**

## Diagnose import set performance 

1. Goto **System Diagnostics** -> **Stats** -> **Slow Scripts**. In this case we use **Slow Scripts**. But any of slow **Events**, **Mutex Locks**, **Queries**, **Scripts**, **Transactions** can be used in diagnosis.

2. Click on **Average execution time (ms)** to sort the scripts with time.

3. Locate the *sys_script:Process sales total* **Bussiness Rule**.

4. Click on the *sys_script:Process sales total* script.

5. Click on **Script source** preview.

    ![](./images/section2/slow_script.png) 

6. Click on **Open Record**

    ![](./images/section2/business_rule.png) 

7. Note that this business rule has a sleep which makes the transformation slow. We can deactivate this business rule at this point to make the transformation fast. But first lets see how to kill this process without doing that.


## Stop Transformation process

1. Goto **User Administration** -> **All Active Transactions**

    ![](./images/section2/active_transactions.png)

2. Right click on *System Trigger** transaction and select kill.

3. Goto **System Import Sets** -> **Advanced** -> **Import Sets**

4. Click on *ISET0010002*.

5. Click on *Import Set Run*

6. Click on **Import Log** tab

7. Click on **Created** field to sort the messages with timestamp.

    ![](./images/section2/transform_history.png)

## Re Run **Scheduled Import**

First we need to deactivate the **Business Rule** we detected at the diagnosing. We can deactivate the business rule or deactivate the **Business rules** for whole transformation. Here we use the later approach.

1. Goto **System Import Sets** -> **Administration** -> **Transform Maps**

2. Select *Sales Transform Map*

3. Uncheck **Run business rules**

4. Click **Update**

5. Goto **System Import Sets** -> **Administration** -> **Scheduled Imports**

6. Select *Sales Import*

7. Click **Execute Now**

8. Goto **System Import Sets** -> **Advanced** -> **Import Sets**

9. Click on *ISET0010003*.

10. Refresh the import set view until import is finished. Import is finished around 2 min time

    ![](./images/section2/import_set_complete_view.png)

11. Goto **System Logs** -> **System Log** -> **Application Logs**

12. Locate following log statements

    *Finishing import with ISET0010003*  
    *Starting import with ISET0010003*

# Section 3 : Run the Scheduled Import Concurrently

## Section Goal

Import process mainly has two steps. First it imports data from external data source to a staging table through an import set. Then it inserts or updates the import set data, into ServiceNow tables (target tables) using transform maps. The later process invokes associated business rules of the target table and may take considerable time to execute.  
In the above example, we ran both these steps with the same scheduler thread. This process may take several days to complete when importing a huge data set (eg around 50 million records) to a target table with time consuming business rules. In order to improve performance in such a case, we can run the transformation part concurrently by enabling  **Concurrent Import**. In a concurrent import first, we divide the data into multiple import sets at the loading time. Then we push these import sets to **Import Set Job Queue**, so that **Import Set Transformer** jobs pick these import sets and process concurrently.  
In this section, first we see how to enable concurrent imports, then the concurrent import view to monitor the concurrent import progress and import set transformer jobs which process the import sets.

## Enabling concurrent import

1. Goto **System Import Sets** -> **Administration** -> **Scheduled Imports**

2. Check **Concurrent Import**. This is the only required step. We will add following two steps to see how to access import set details in a concurrent import.

3. Set the following as the **Pre script**
   ```javascript
    var msg = 'Concurrent import set number from pre script ' + import_set.number + ' ';
    var importSetGr = new GlideRecord('sys_import_set');
    importSetGr.addQuery('concurrent_import_set', import_set.sys_id);
    importSetGr.query();
    while (importSetGr.next())
	msg += importSetGr.number + ' ';
    gs.info(msg);
    ```

4. Set the following as the **Post script**

   ```javascript
    var msg = 'Concurrent import set number from post script ' + import_set.number + ' ';
    var importSetGr = new GlideRecord('sys_import_set');
    importSetGr.addQuery('concurrent_import_set', import_set.sys_id);
    importSetGr.query();
    while (importSetGr.next())
	msg += importSetGr.number + ' ';
    gs.info(msg);
    ```

5. Click **Execute Now**. This starts the concurrent import process.

6. Goto **System Import Sets** -> **Advanced** -> **Concurrent Import Sets**

7. Select the CISET0010001. This gives details about the **Import Sets**, **Concurrent Import Set Jobs** and **Transform Histories**

8. Click on **Transform Histories**. This view shows how much time it has spend on transforming each import set. 

    ![](./images/section3/concurrent_import_set_veiw.png)

9. Goto **System Logs** -> **System Log** -> **Application Logs**

10. Locate following logs

    *Concurrent import set number from post script CISET0010001 ISET0010007 ISET0010008*  
    *Concurrent import set number from pre script CISET0010001 ISET0010007 ISET0010008*

## Understanding how concurrent import works

1. **Concurrent Import** break the data into multiple import sets. To see the **Import Sets**, Goto **System Import Sets** -> **Advanced** -> **Concurrent Import Sets**.

2. Select the CISET0010001.

    ![](./images/section3/concurrent_import_set_view_import_set.png) 

3. After loading data into multiple **Import Sets**, it enqueue these **Import Sets** to **Concurrent Import Set Jobs** queue. Click on **Concurrent Import Set Jobs** to see the jobs in the queue.

    ![](./images/section3/concurrent_import_set_view_queue.png)

4. There are set of schedlued task periodically poll the above queue.

5. Goto **System Scheduler** -> **Scheduled Jobs**

6. Search Scheduled jobs with **Name** : Import Set Transformer. There are four **Import Set Transformer** jobs. Here we assume this is a single node cluster and for multinode cluster it will be it is 2 * (No of Active Nodes + 1 ).  

    ![](./images/section3/import_set_transformer_jobs.png)

7. Click on a **Import Set Transformer** job in red *Ready* state. These jobs are called parent jobs and have two such jobs by default. Note **System ID** of this task set to **ACTIVE NODES**. When the system starts up this creates a child task for each active node.

    ![](./images/section3/parent_job.png)

8. Click on a **Import Set Transformer** job in black *Ready* state. These jobs are called child jobs created from above parent jobs. These jobs has a **System ID** with an instance id. 

    ![](./images/section3/sectchild_job.png)

9. Click on **Job ID** preview. This shows the **Import Set Transformer Job**.

    ![](./images/section3/import_set_transformer_job.png)

10. This job referes to the real java implementation class. So the each scheduled task periodically invokes the implementation class and that polls the job queue. If an import set found to execute it will execute that.

    
