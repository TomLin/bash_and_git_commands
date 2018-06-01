
## Reference

[Tianyi's script to export csv and upload to sftp](http://git.trivago.trv/projects/BI/repos/sem-bidding-wf/browse/co_semDailyProduction/wf_semDailyProduction/wf_biddingFeatures/wf_gclid_conversion_value/workflow.xml)

[Mariia's script to export csv and upload to sftp](http://git.trivago.trv/projects/BI/repos/bi-marketing/browse/marketing-wf/co_mdailyBP/wf_mdailyBP/wf_sftp_upload_adphorus_data/workflow.xml)

[Mariia's script to set up a workflow](http://git.trivago.trv/projects/BI/repos/bi-marketing/browse/marketing-wf/co_mdailyBP/wf_mdailyBP/wf_criteo_bp/workflow.xml)


1. When try to create a table on bi-marketing, we need to prepare all the script (table definiton and hive script) to Mariia to merge with the BI-marketing branch.

2. The table location should be `hdfs://nameservice1/user/marketing/files/<personal_defined_table_folder>` Like in this [link](http://git.trivago.trv/projects/BI/repos/bi-marketing/browse/marketing-wf/co_mdailyBP/wf_mdailyBP/wf_gclid_bp/table_gclid_bp_locale)

3. wrtie down the textfile table note for csv output (not finished)

4. software used to look up the files in remote sftp: WinSCP



## Learn how to export table as csv file
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


## Full example 01
```XML
<?xml version="1.0" encoding="UTF-8"?>

<workflow-app name="MI_adphorusPredictions" xmlns="uri:oozie:workflow:0.4">
<credentials>
    <credential name="hcat-trivago" type="hcat-trivago" />
</credentials>
    <start to="hive_adphorusPredictions"/>
	
         <action name="hive_adphorusPredictions" cred="hcat-trivago">
        <hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
              <job-xml>/data/hive/hive-trivago.xml</job-xml>
              <script>hive_adphorus_data.hql</script>
                 <param>crunchDate=${crunchDate}</param>
        </hive>
        <ok to="hive_collectadphorusPredictions"/>
        <error to="email_then_kill"/>
    </action>
 
    <action name="hive_collectadphorusPredictions" cred="hcat-trivago">
      <hive xmlns="uri:oozie:hive-action:0.2">
        <job-tracker>${jobTracker}</job-tracker>
        <name-node>${nameNode}</name-node>
        <prepare>
          <delete path="${wf_folder}/csv_file" />
          <mkdir  path="${wf_folder}/csv_file" />
        </prepare>
        <job-xml>/data/hive/hive-trivago.xml</job-xml>
        <script>hive_collect_adphorus_data.hql</script>
        <param>output_dir=${wf_folder}/csv_file</param>
        <param>HadoopDatabaseToReadFrom=${HadoopDatabaseToReadFrom}</param>
      </hive>
      <ok    to="fs_rename_attachement_to_csv"/>
      <error to="email_then_kill"/>
    </action>

    <action name="fs_rename_attachement_to_csv">
      <fs>
        <move source="${wf_folder}/csv_file/000000_0" target="${wf_folder}/csv_file/adphorus_prediction_data_${crunchDate}.csv" />
      </fs>
      <ok    to="sftp"/>
      <error to="email_then_kill"/>
    </action>

    <action name="sftp">
        <java>
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <main-class>SFTPUpload</main-class>
            <arg>trivago</arg> <!-- username -->
            <arg>/user/BI/lib/adphorus.pass</arg> <!-- hdfs filepath of the password -->
            <arg>adpftp.adphorus.com</arg> <!-- host address  -->
            <arg>${wf_folder}/csv_file/</arg> <!-- table folder in hdfs, you'll need to store it as CSV and force bucketing into 1 file  -->
            <arg>/trivago/</arg> <!-- destination folder  -->
            <file>/user/BI/lib/SFTPUpload.jar</file>
        </java>
    <ok to="end"/>
    <error to="email_then_kill"/>
  </action>

   <action name="email_then_kill">
   	 <email xmlns="uri:oozie:email-action:0.1">
      <to>gabriel.breuer@trivago.com, Mariia.Pankevych@trivago.com</to>
      <subject>Workflow ${wf:name()} failed</subject>
      <body>
        One or more actions from workflow ${wf:name()} failed.
        Please try to restart it once:
        1. Go to http://10.1.17.1:8889/oozie/list_oozie_workflow/${wf:id()}/
        2. Click on "Rerun" button (it's in the left bottom corner)
        3. In the new "window" click on "Submit" button.

        This email is sent each time this workflow fails.
        If it's not the first time seeing this email, please ignore it.

        I may be "just" software but I have feelings too :).
        Cheers, Oozie.
      </body>
   	 </email>
    	<ok to="kill" />
    	<error to="kill" />
  	</action>


    <kill name="kill">
        <message>Action failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>

    <end name="end"/>

</workflow-app>
```

## Full example 02
```XML
<workflow-app name="wf_gclid_conversion_value" xmlns="uri:oozie:workflow:0.5">
	<credentials>
		<credential name="hcat-trivago" type="hcat-trivago" />
	</credentials> 

	<start to="Fork_1"/>

 	<fork name="Fork_1">
    	<path start="stg1"/>
        <path start="stg3"/>
    </fork>

    <action name="stg1" cred="hcat-trivago">
        <hive xmlns="uri:oozie:hive-action:0.2">
	        <job-tracker>${jobTracker}</job-tracker>
	        <name-node>${nameNode}</name-node>
	        <job-xml>/data/hive/hive-trivago.xml</job-xml>
	        <script>hive_stg1</script>
	        <param>date_to=${date_to}</param>
            <param>HadoopSemDatabaseToWriteTo=${HadoopSemDatabaseToWriteTo}</param>
        </hive>        
     	<ok to="stg2"/>
     	<error to="email_then_kill"/>
 	</action>

    <action name="stg2" cred="hcat-trivago">
    	<hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <job-xml>/data/hive/hive-trivago.xml</job-xml>
            <script>hive_stg2</script>
            <param>date_to=${date_to}</param>
            <param>HadoopSemDatabaseToWriteTo=${HadoopSemDatabaseToWriteTo}</param>
        </hive>        
	    <ok to="join_1"/>
	    <error to="email_then_kill"/>
 	</action>

    <action name="stg3" cred="hcat-trivago">
        <hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <job-xml>/data/hive/hive-trivago.xml</job-xml>
            <script>hive_stg3</script>
            <param>date_to=${date_to}</param>
            <param>HadoopSemDatabaseToWriteTo=${HadoopSemDatabaseToWriteTo}</param>
        </hive>        
     	<ok to="join_1"/>
     	<error to="email_then_kill"/>
 	</action>

    <join name="join_1" to="stg4"/>

    <action name="stg4" cred="hcat-trivago">
        <hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <job-xml>/data/hive/hive-trivago.xml</job-xml>
            <script>hive_stg4</script>
            <param>HadoopSemDatabaseToWriteTo=${HadoopSemDatabaseToWriteTo}</param>
        </hive>
        <ok to="gclid_conversion_value"/> 
    	<error to="email_then_kill"/>
 	</action>

    <action name="gclid_conversion_value" cred="hcat-trivago">
        <hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <job-xml>/data/hive/hive-trivago.xml</job-xml>
            <script>hive_gclid_conversion_value</script>
            <param>date_to=${date_to}</param>
            <param>num_partitions=70</param>
            <param>HadoopSemDatabaseToWriteTo=${HadoopSemDatabaseToWriteTo}</param>
        </hive>        
     	<ok to="hive_gclid_test"/>
     	<error to="email_then_kill"/>
 	</action>

    <action name="hive_gclid_test" cred="hcat-trivago">
        <hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <job-xml>/data/hive/hive-trivago.xml</job-xml>
            <script>hive_gclid_test</script>
            <param>date_to=${date_to}</param>
            <param>HadoopSemDatabaseToWriteTo=${HadoopSemDatabaseToWriteTo}</param>
        </hive>        
     	<ok to="Rename"/>
     	<error to="email_then_kill"/>
 	</action>

    <action name="Rename">
       <fs>
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
            <arg>incoming</arg>
            <file>/user/BI/lib/java-tools-0.5-all.jar</file>
        </java>      
     	<ok to="join_2"/>
     	<error to="email_then_kill"/>
 	</action>

    <action name="SFTPmove_in" cred="hcat-trivago">
        <java>
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <main-class>com.trivago.bi.java.ftp_sftp.SFTPUpload</main-class>
            <arg>trivago_sftp</arg> <!-- username -->
            <arg>/user/SEM/sem-bidding-wf/co_semDailyProduction/wf_semDailyProduction/wf_biddingFeatures/sftppassword</arg> <!-- password file location -->  
            <arg>18.195.30.145</arg> <!-- ftp host address -->
            <arg>/user/SEM/export/gclid_intest/gclid_intest.csv</arg> <!-- hdfs file path to be moved -->
            <arg>incoming</arg>
            <file>/user/BI/lib/java-tools-0.5-all.jar</file>
        </java>      
     	<ok to="join_2"/>
     	<error to="email_then_kill"/>
 	</action>

    <action name="SFTPmove_uk" cred="hcat-trivago">
        <java>
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <main-class>com.trivago.bi.java.ftp_sftp.SFTPUpload</main-class>
            <arg>trivago_sftp</arg> <!-- username -->
            <arg>/user/SEM/sem-bidding-wf/co_semDailyProduction/wf_semDailyProduction/wf_biddingFeatures/sftppassword</arg> <!-- password file location -->  
            <arg>18.195.30.145</arg> <!-- ftp host address -->
            <arg>/user/SEM/export/gclid_uktest/gclid_uktest.csv</arg> <!-- hdfs file path to be moved -->
            <arg>incoming</arg>
            <file>/user/BI/lib/java-tools-0.5-all.jar</file>
        </java>      
     	<ok to="join_2"/>
     	<error to="email_then_kill"/>
	</action>

    <join name="join_2" to="end"/>

  	<action name="email_then_kill">
    	<sub-workflow>
      		<app-path>${nameNode}/user/BI/hadoop-wf/co_genericTools/wf_sendNotification
			</app-path>
      		<propagate-configuration/>
      		<configuration>
        		<property>
	          		<name>emailRecipient</name>
          			<value>tianyi.zhou@trivago.com</value>
        		</property>
        		<property>
	          		<name>workflowName</name>
          			<value>${wf:name()}</value>
        		</property>
        		<property>
	          		<name>workflowId</name>
          			<value>${wf:id()}</value>
        		</property>
      		</configuration>
    	</sub-workflow>
    	<ok to="kill" />
    	<error to="kill" />
  	</action>

    <kill name="kill">
        <message>Action failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>
    
    <end name="end"/>

</workflow-app>
```

## Full example 03
```XML
<?xml version="1.0" encoding="UTF-8"?>

<workflow-app name="MI_criteo_bp" xmlns="uri:oozie:workflow:0.4">
<credentials>
    <credential name="hcat-trivago" type="hcat-trivago" />
</credentials>
    <start to="criteo_bp"/>
	
         <action name="criteo_bp" cred="hcat-trivago">
        <hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
              <job-xml>/data/hive/hive-trivago.xml</job-xml>
              <script>hive_criteo_bp.hql</script>
                 <param>crunchDate=${crunchDate}</param>
        </hive>
        <ok to="criteo_bp_format"/>
        <error to="email_then_kill"/>
    </action>
 
       <action name="criteo_bp_format" cred="hcat-trivago">
        <hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
              <job-xml>/data/hive/hive-trivago.xml</job-xml>
              <script>hive_criteo_bp_format.hql</script>
                 <param>crunchDate=${crunchDate}</param>
        </hive>        
     <ok to="Rename"/>
     <error to="email_then_kill"/>
 </action>

    <action name="Rename">
      <fs>
        <move source='hdfs://nameservice1/user/marketing/files/move/wf_criteo_bp/criteo_bp_format/000000_0' 
                     target='hdfs://nameservice1/user/marketing/files/move/wf_criteo_bp/criteo_bp_format/criteo_bp_format.csv'/>
      </fs>
      <ok    to="execute"/>
      <error to="email_then_kill"/>
    </action>

    <action name="execute">
        <shell xmlns="uri:oozie:shell-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>

            <exec>execute.sh</exec>
            <env-var>crunchDate=${crunchDate}</env-var>
	       <file>${wf_folder}/execute.sh</file> 
           <file>${wf_folder}/criteo_api.py</file>	
            <capture-output/>
        </shell>
        <ok to="end"/>
        <error to="email_then_kill"/>
    </action>

   <action name="email_then_kill">
   	 <email xmlns="uri:oozie:email-action:0.1">
      <to>gabriel.breuer@trivago.com, Mariia.Pankevych@trivago.com</to>
      <subject>Workflow ${wf:name()} failed</subject>
      <body>
        One or more actions from workflow ${wf:name()} failed.
        Please try to restart it once:
        1. Go to http://10.1.17.1:8889/oozie/list_oozie_workflow/${wf:id()}/
        2. Click on "Rerun" button (it's in the left bottom corner)
        3. In the new "window" click on "Submit" button.

        This email is sent each time this workflow fails.
        If it's not the first time seeing this email, please ignore it.

        I may be "just" software but I have feelings too :).
        Cheers, Oozie.
      </body>
   	 </email>
    	<ok to="kill" />
    	<error to="kill" />
  	</action>

    <kill name="kill">
        <message>Action failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>

    <end name="end"/>

</workflow-app>
```
