# Autoencoders for Anomaly Detection: Using Deep Learning Methods to Monitor the Health of Automated Datasets  

-----  

### What is this project?  
This proposal and proof of concept is a Tyler Tech hackweek project from 2023. The datasets discussed here are data assets that are housed in Tyler customer SaaS platforms as tabular datasets. In the use case described below, datasets are hosted and published in a given customer environment, and have accompanying ETL implemented such that data from their system of record – a separate database hosted elsewhere – is pulled into the Tyler dataset on a given cadence through a scheduled process. The scheduled process is itself another Tyler product, which connects source systems to the Tyler environment, so that data can be visualized and shared.  

The steps outlined in the _Building a Proof of Concept_ section below are laid out as a proposal for a future project to build out this dataset monitoring feature in a test environment. The notebooks contained in thie repo, however, are preliminary research, intended to answer a few foundational questions about whether we can and should pursue development of this feature.  

Ultimately, the performance of this autoencoder was not great – accuracy score peaked at around 45% with a peak recall rate of 85%. I believe that with additional optimization, this could be a viable method of monitoring updates to automated datasets. This was a project that was time-boxed to about a week of development, which included the data collection process, so I had a lot of additional ideas for ways to track and improve model performance that are not implemented here. Those next steps are documented at the end of the `Preprocessing and Modeling` notebook.   

_Some details in this project – like details about the source data – have been omitted for privacy._  

-----  


### Problem

Can we build infrastructure/services that monitor the health of a scheduled dataset?

### Use Case

Our customers often automate datasets that are connected to their own source databases. These datasets are updated automatically, based on some specified cadence. Update failures, when they happen, are typically due to schema mismatches -- when the update data doesn't match the existing dataset in schema -- or connectivity issues to the source system. These errors are caught and made obvious by our established error handling infrastructure.

There are however cases where an update can be faulty in terms of the *content* being published -- fewer rows than expected, erroneous values in a column, etc. -- and there is currently no tracking or error handling available to monitor for these types of issues. In other words, an update can run successfully, and the dataset can publish as expected, but the content of the data is inaccurate. Downstream metrics, vizzes, and reports will be inaccurate as well.

Today, in order to validate the content of dataset updates, customers have to query their datasets themselves, and then proceed to compare the data to a previous update, or to data from the source system (or some other, likely manual, QA process).

This method of issue remediation is costly. Customers must be proactive in monitoring their updates by establishing their own manual, labor-intensive QA process. Internally, Data Integrations spends more time troubleshooting data issues; because these "silent failures" often originate from the customer source system, DI engineers must investigate database architecture remotely, with little to no knowledge of or access to the database itself.

Can we instead provide an opt-in service that automatically checks a dataset each time an update is published in the platform? The service would assess the dataset for completeness, obvious anomalies, or other inconsistencies that we prioritize. If health checks don't pass, the customer can be alerted by the service that their dataset may not have updated properly and take action.  

### Preliminary Research  

Research should be done initially to understand a few foundational questions:  
- Is there enough metadata about dataset updates available through Tyler APIs to track the health of scheduled datasets?  
    - If not, can this metadata be bootstrapped in such a way that we can use it to train a model that will reflect real-world scenarios?
- Can we get to a clear definition of what an anomaly means in this specific context of expected vs unexpected updates?
- Are there models other than autoencoders that are appropriate for this problem?

### Building a Proof of Concept: Tasks for the Build

The testing process and development process for a net-new monitoring service are linked, in that we will need to build an MVP of the service before we can fully understand the requirements to build in production. The approach laid out here is expected to be iterative and change along the way, but at a high level, the testing and development process can be broken out into phases:

1.  **Collect test data**
    1.  In order to understand whether we *can* implement automated dataset monitoring, we need to know which specific events to monitor for, and how to obtain comprehensive data on these events.
        1.  Tasks:
            1.  Identify monitoring criteria.
            2.  Collect a sample of the data. The sample should represent the dataset in its healthy state, as well as in an error state. If possible, the proportion of healthy observations to erroneous ones in the sample should be as realistic as possible. If the anomaly event is rare, there should only be a small number of errors in the sample, and a large number of "valid" records.  
2.  **Create test environment**
    1.  What IT resources are needed for testing? Can this be done with a demo domain and a local dev environment?
        1.  Tasks:
            1.  Identify and set up testing framework
3.  **Create archive dataset** (first time only)
    1.  In order to monitor a dataset, we need to begin capturing metadata on the dataset each time an update happens. Dataset metadata will be captured in an archive dataset which lives on the same domain as the dataset to be monitored.
        1.  Tasks:
            1.  Define the metadata that should be captured on the dataset.
            2.  Create a script that generates an archive dataset for the dataset being monitored, where each column is a specific piece of metadata.
4.  **Capture updates**
    1.  Each time the monitored dataset undergoes an update, the archive repository dataset should capture metadata on the updates.
        1.  Tasks:
            1.  Create a script that runs whenever an update is made to the monitored dataset. The script should capture any of the defined pieces of metadata and append them to the archive repository dataset.
5.  **Analyze updates**
    1.  With each update of the monitored dataset, analyze the archive data and determine the health of the dataset.
        1.  Tasks:
            1.  Set up model competition and analyze performance
                1.  PCA
                2.  Some kind of random forest?
                3.  Some kind of neural network?
                4.  Naive methods?
            2.  Write script to assess the latest dataset update for health, using the best modeling method/s from the model competition. The archive history records are used as training data, and the most recent archive record is the test.
            3.  Write the data from model assessment to a dataset? Maybe back to the archive dataset?
                1.  Could be interesting to create a viz from the results
6.  **If anomaly is detected, notify dataset owners**
    1.  The notification method is dependent on customer needs, and also on the implementation path; do we need input from the customer in order to start this service on their domain, or can we just "turn on" the service?
        1.  It should be said that this is a service meant to empower the dataset owners. The expectation is that customers will use this tool to monitor their own datasets. We internally take on the responsibility of monitoring the monitoring service, but we do not monitor datasets for the customer.
        2.  Considering this, I strongly recommend that we have a requirements-gathering exchange with the customer as a pre-req for turning on the service. This would be a stop-gap to ensure that there is an agreement of customer ownership, giving us the ability to designate a real person who will receive and be custodian of any notifications.
7.  **Operationalize**
    1.  Scripts can run in Airflow, with the DAG cadence set to one of the following:
        1.  A brief time after the dataset is regularly scheduled to update
            1.  pros:
                1.  The job only runs as often as necessary
            2.  cons:
                1.  If a dataset takes too long to finish updating, the DAG could kick off before the update is complete, thereby missing the newest data
                2.  Manual data pulls would not be caught until the next scheduled runtime
        2.  Very frequently
            1.  pros:
                1.  We catch way more of the updates faster
                2.  If any of the scripts takes a long time / heavy resources to execute, the DAG could fail more often
