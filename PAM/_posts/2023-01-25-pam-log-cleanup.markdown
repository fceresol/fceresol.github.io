---
layout: post_with_sidebar
title: How to setup LogCleanupCommand job using commandline in PAM
date: 2023-01-25 14:48
category: PAM
subcategory: General Tips

tags: [PAM, HowTo ]

---

In the following post I'll explain how to schedule a LogCleanup job using commandline tools like cURL in order to prevent excessive database growth.

In order to cleanup the historical data from PAM Database, a job could be scheduled on kie-server:

create a file named jobRequest.xml with the following content:

~~~ xml

<?xml version="1.0" encoding="UTF-8" standalone="yes"?\>
<job-request-instance>
    <job-command>org.jbpm.executor.commands.LogCleanupCommand</job-command>
    <scheduled-date>2016-02-11T00:00:00-02:00</scheduled-date>
    <data>
        <entry>
            <key>OlderThanPeriod</key>
            <value xsi:type="xs:string" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">60d</value>
        </entry>
        <entry>
            <key>SingleRun</key>
            <value xsi:type="xs:string" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">true</value>
        </entry>

        <!-- optional, if specified only this process will be cleaned -->

        <!-- <entry>
            <key>ForProcess</key>
            <value xsi:type="xs:string" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">xxxx.yyyyy</value>
          </entry>-->
    </data>
</job-request-instance>

~~~

the parameters used in this sample are:

  - Job Parameters:
    
      - **job-command**: must be `org.jbpm.executor.commands.LogCleanupCommand`
      - **scheduled-date**: the timestamp for the scheduling

  - Job Configuration
    
      - **SingleRun**: if is true the job will be executed only once, if
        it’s false or not specified the job will be rescheduled every
        24h (the delay between two jobs could be configured using the
        NextRun parameter
      - **OlderThanPeriod**: this parameter is used to specify the
        retention, if 60d is specified the logs older than 60 days will
        be removed
      - **ForProcess**: this parameter is used to specify a single
        Process ID to be cleaned, if not specified, the logs for every
        process are removed.
      - **Further parameters** are available on the [official documentation](https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.8/html/designing_business_processes_in_business_central/manage-log-file-proc)

in order to schedule on a kie-server the job, after creating the xml above, the following command could be issued:

~~~ bash

$ curl -X POST --data @<XML_FILE> -u '<KIE_SERVER_USER>:<KIE_SERVER_PASSWORD>' -H 'content-type:application/xml' 'http://<KIE_SERVER_ADDRESS>/services/rest/server/jobs/'

~~~

using the following parameters:

  - **XML_FILE**: the XML created
  - **KIE_SERVER_USER**: a user having kie-server role
  - **KIE_SERVER_PASSWORD**: the password for the user
  - **KIE_SERVER_ADDRESS**: the listen address for the kie-server 

{%- capture note -%}   
the kie-server url specified works on OpenShift, on a on-premise setup the context-root may be different (the default is http://\<KIE\_SERVER\_ADDRESS\>/kie-server/services/rest/server/jobs/)   
   
if the server address uses SSL, remember to change schema from <b>http</b> to <b>https</b> and add <b>-k</b> to the curl command parameters   
{%- endcapture -%}

{%- include note.html content=note -%}

which will output with the created Job ID:

~~~ xml

<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<long-type>
    <value>4</value>
</long-type>

~~~
In order to verify the jobs and their status the following command could
be issued:

~~~ bash
$ curl -u '<KIE_SERVER_USER>:<KIE_SERVER_PASSWORD>*' -H 'Accept:application/json' 'http://<KIE_SERVER_ADDRESS>/services/rest/server/jobs?status=QUEUED\&status=DONE\&status=CANCELLED\&status=ERROR\&status=RETRYING\&status=RUNNING'
~~~

using the following parameters:

  - **KIE_SERVER_USER**: a user having kie-server role
  - **KIE_SERVER_PASSWORD**: the password for the user
  - **KIE_SERVER_ADDRESS**: the listen address the kie-server
  - status=......: filter by status, the example filter shows every
    status.

{%- capture note -%}   
the kie-server url specified works on OpenShift, on a on-premise setup the context-root may be different (the default is http://\<KIE\_SERVER\_ADDRESS\>/kie-server/services/rest/server/jobs/)   
   
if the server address uses SSL, remember to change schema from <b>http</b> to <b>https</b> and add <b>-k</b> to the curl command parameters   
{%- endcapture -%}

{%- include note.html content=note -%}