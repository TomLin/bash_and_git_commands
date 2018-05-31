
** Reference

[Tianyi's script to export csv and upload to sftp](http://git.trivago.trv/projects/BI/repos/sem-bidding-wf/browse/co_semDailyProduction/wf_semDailyProduction/wf_biddingFeatures/wf_gclid_conversion_value/workflow.xml)

[Mariia's script to export csv and upload to sftp](http://git.trivago.trv/projects/BI/repos/bi-marketing/browse/marketing-wf/co_mdailyBP/wf_mdailyBP/wf_sftp_upload_adphorus_data/workflow.xml)

[Mariia's script to set up a workflow](http://git.trivago.trv/projects/BI/repos/bi-marketing/browse/marketing-wf/co_mdailyBP/wf_mdailyBP/wf_criteo_bp/workflow.xml)


1. When try to create a table on bi-marketing, we need to prepare all the script (table definiton and hive script) to Mariia to merge with the BI-marketing branch.

2. The table location should be `hdfs://nameservice1/user/marketing/files/<personal_defined_table_folder>` Like in this [link](http://git.trivago.trv/projects/BI/repos/bi-marketing/browse/marketing-wf/co_mdailyBP/wf_mdailyBP/wf_gclid_bp/table_gclid_bp_locale)

3. wrtie down the textfile table note for csv output (not finished)

4. software used to look up the files in remote sftp: WinSCP



** Learn how to export table as csv file
1. When create a table, it must be stored as `TEXTFILE`, instead of the usual `PARQUET`

```SQL
CREATE EXTERNAL TABLE sem.gclid_artest (
`google click id` string,
`conversion name` string,
`conversion time` string,
`conversion value` string,
`conversion currency` string
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n' 
STORED AS TEXTFILE
LOCATION 'hdfs://nameservice1/user/SEM/export/gclid_artest'; -- hdfs://nameservice1/user/XXX/XXX/table_folder
```

2. Remeber to specify all the columns as string forms. So that later we can union the column header to the table.

```SQL
-- Notice in this table, it doesn't have ymd partition. It is an one-time use table, meaning it will not store historical data, old data are overwrite.
SET hive.exec.dynamic.partition.mode=nonstrict;
SET mapreduce.map.java.opts = "-Xmx3700M";
SET mapreduce.reduce.java.opts = "-Xmx1700M";
SET hive.optimize.sort.dynamic.partition=true;
SET parquet.compression=snappy;
SET hive.auto.convert.join = false;

INSERT OVERWRITE TABLE ${HadoopSemDatabaseToWriteTo}.gclid_artest
SELECT
    *
FROM
	(
	SELECT
	    'google click id' AS `google click id`, -- use union all to append the column header to the table
	    'conversion name' AS `conversion name`,
	    'conversion time' AS `conversion time`,
	    'conversion value' AS `conversion value`,
	    'conversion currency' AS `conversion currency`
	
	UNION ALL
	
	SELECT
	    google_click_id AS `google click id`,
	    conversion_name AS `conversion name`,
	    concat(cast(from_unixtime(unix_timestamp(conversion_time)+30) as string),'+0000') AS `conversion time`,
	    cast(conversion_value_upload as string) AS `conversion value`,
	    'EUR' AS `conversion currency`
	FROM
	    sem.gclid_conversion_value
	WHERE
	    ymd=${date_to}
	    AND market='AR'
	    AND publisher IN ('dsa_search','google_search')
    ) a
ORDER BY 2 ASC,1,4 DESC,3;

```

3. The table will be stored in the table folder as `000000_0` will be its file name.
4. Next action is to change the file name (optional)
5. Fork the file
6. Specify the sftp server address

```XML
    <action name="Rename">
       <fs>  <!-- renmae the file -->
           <move source='hdfs://nameservice1/user/SEM/export/gclid_artest/000000_0' target='hdfs://nameservice1/user/SEM/export/gclid_artest/gclid_artest.csv'/> 
           <move source='hdfs://nameservice1/user/SEM/export/gclid_intest/000000_0' target='hdfs://nameservice1/user/SEM/export/gclid_intest/gclid_intest.csv'/>
           <move source='hdfs://nameservice1/user/SEM/export/gclid_uktest/000000_0' target='hdfs://nameservice1/user/SEM/export/gclid_uktest/gclid_uktest.csv'/>
       </fs>
       <ok to="Fork_2"/>
       <error to="email_then_kill"/>
   </action>

 	<fork name="Fork_2">
      	<path start="SFTPmove_ar"/>
      	<path start="SFTPmove_in"/>
      	<path start="SFTPmove_uk"/>
  	</fork>

    <action name="SFTPmove_ar" cred="hcat-trivago">
        <java>
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <main-class>com.trivago.bi.java.ftp_sftp.SFTPUpload</main-class>
            <arg>trivago_sftp</arg> <!-- username -->
            <arg>/user/SEM/sem-bidding-wf/co_semDailyProduction/wf_semDailyProduction/wf_biddingFeatures/sftppassword</arg> <!-- password file location -->  
            <arg>18.195.30.145</arg> <!-- ftp host address -->
            <arg>/user/SEM/export/gclid_artest/gclid_artest.csv</arg> <!-- hdfs file path to be moved -->
            <arg>incoming</arg> <!-- the folder in ftp server -->
            <file>/user/BI/lib/java-tools-0.5-all.jar</file>
        </java>      
     	<ok to="join_2"/>
     	<error to="email_then_kill"/>
 	</action>
```

