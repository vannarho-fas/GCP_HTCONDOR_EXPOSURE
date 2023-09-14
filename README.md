# Calculating portfolio risk using a montecarlo model run using a HTCondor cluster on Google Cloud Platform. 

This directory contains the source code for calculating portfolio risk using HTCondor and GCP.

## Analyzing portfolio risk using HTCondor and Compute Engine
This tutorial shows you how to use Google Cloud services to run Monte Carlo simulations and analyze the resulting data sets. You use HTCondor to schedule jobs on a cluster of Compute Engine virtual machines that run the simulations. You then use BigQuery to store aggregated simulation results, and Datalab to analyze and visualize those results.

This assumes the following:
* You have some experience with Google Cloud.
* You're familiar with Linux command-line tools.

## Using Monte Carlo simulations in the financial sector
Researchers and data scientists often rely on Monte Carlo simulations to solve physical or mathematical problems that are difficult to solve in other ways. The Monte Carlo method uses repeated random sampling to obtain numerical results. These simulations are typically run as independent calculations, which are distributed across many computers. This type of high-performance computing is often called high throughput as opposed to tightly coupled parallel computation. High-throughput parallel workloads are made up of many smaller calculations that can run independently of each other across many hosts. Individual tasks can be stopped and restarted without affecting other operations in the broader simulation.

The financial services industry relies on Monte Carlo simulations to analyze investment portfolios and to forecast potential losses that might be incurred for risks, such as credit risk, liquidity risk, and market risk. Under the Basel III accords, formal risk modeling is required by all major international banking institutions. With adequate computing infrastructure, the simulations can be automated.

Risk modeling includes a variety of techniques including market risk, value at risk (VaR), historical simulation (HS), or extreme value theory (EVT). This tutorial describes an implementation of a VaR calculation of a hypothetical stock portfolio that is made up of a selection from the Standard & Poor's 500 (S&P 500) stock market index.

There are two major stages in the process:

* Data acquisition and risk scoring
* Data analysis and reporting

In the first stage, you obtain and process the current and historical positions of the equities. Typically, this requires only a few GB of data for the positions, scenarios, and market data.

The second stage in the process simulates the market movement by using a stochastic process, and then analyzes the likely outcomes. You can use a Monte Carlo simulation to look at the potential evolution of asset prices over time, assuming that daily returns follow a normal distribution. This type of price evolution is known as a random walk. The Monte Carlo simulation estimates the expected level of return and volatility of each stock in the portfolio. This data can be estimated from historical prices by using simple methods that assume that past mean return and volatility levels continue in the future. Other more sophisticated methods are typically employed in real-world scenarios, but this tutorial uses the simpler methods for the sake of brevity.

Simulation of the market can be done at a large scale and in parallel with thousands to hundreds of thousands of simulations across thousands of stock entities. The resulting data in aggregate can be easily in the terabytes and can be difficult to manage.

Additional simplifications for this implementation include using end-of-day stock prices over the past 252 days (the number of trading days in a typical year). In a real-life scenario, the number of price points taken into account would be much larger—one per second, or even tick by tick. A tick is a recorded change in the value of a stock. A stock, such as GOOG, ticks about 10 to 20 times a second.

Finally, an integral part of Monte Carlo VaR calculation is simulating the returns of each security in the portfolio and then aggregating returns across the portfolio to construct the return distribution. This tutorial uses a simple summary of positions and doesn't take into account stock-specific risk factors, such as public filings, government policy, and so on, that might have influenced historical returns. This method naively assumes that all of the historical returns are non-factor returns.

These simplifications make the tutorial easier to follow. However, the overall architecture remains the same, and you can modify it by adding data or substituting more sophisticated models and techniques.

## Google Cloud architecture

You obtain the raw market data and portfolio values from either external data sources or from internal financial systems and store them in Cloud Storage.
You use Cloud Deployment Manager to provision a high-performance computing (HPC) cluster and then use the HTCondor job submission commands to submit the Monte Carlo simulation jobs to the cluster of compute nodes. Each job simulates one stock based on its historical movements.
The stock forecasts are automatically uploaded to a Cloud Storage bucket.
When all of the jobs have completed, you ingest the data into BigQuery for analysis.
At this point, you can further analyze or process the aggregated results for applications in machine learning or business intelligence. In this tutorial, you analyze and visualize the data by using Datalab. Other tools you could use include Data Studio, AI Platform, or other data analysis tools.

## Objectives
* Run common financial analyses by using HPC on Compute Engine.
* Optimize the HTCondor templates to improve provisioning time.
* Automate data loading into BigQuery and create visualizations by using Datalab or Data Studio.

## Costs
This  uses the following billable components of Google Cloud:

* Compute Engine to run the simulations. For details, read the Compute Engine pricing page.
* BigQuery to store and analyze the aggregate results of the simulations. For details, read the BigQuery pricing page.
* Cloud Storage to store market and portfolio data. For details, read the Cloud Storage pricing page.
* Compute Engine to run Datalab. For details, read the Datalab pricing page.

To generate a cost estimate based on your projected usage, use the pricing calculator.

## Before you begin
In the Cloud Console, on the project selector page, select or create a Cloud project.

Note: If you don't plan to keep the resources that you create in this procedure, create a project instead of selecting an existing project. After you finish these steps, you can delete the project, removing all resources associated with the project.
Go to the project selector page

Make sure that billing is enabled for your Google Cloud project. Learn how to confirm billing is enabled for your project.

If you don't have the Cloud SDK installed, you can use Cloud Shell for this tutorial. Cloud Shell provides a shell environment with the Cloud SDK preinstalled. The rest of this tutorial uses Cloud Shell. If you prefer to use the Cloud SDK, you can run commands in the terminal window of your local computer, unless otherwise noted.
In the Cloud Console, open Cloud Shell.

### OPEN Cloud Shell
When you finish this tutorial, you can avoid continued billing by deleting the resources you created. For more information, see Cleaning up.

### Getting the dataset and code
The sample code for this tutorial is available here. It includes scripts to obtain market data, Deployment Manager templates for creating the HTCondor compute cluster, and a simple Monte Carlo model.

### Get the tools
To run this tutorial, you need tools that are available in a GitHub repository, including a makefile, scripts, and configuration files.

In Cloud Shell, clone the GitHub repository that contains the sample code and change directories to the folder the repository was cloned to:

git clone https://github.com/fordesmith/GCP_HTCONDOR_EXPOSURE.git;

The sample code has a makefile that drives most of the commands you perform. The commands in the makefile are mostly wrappers around gcloud commands. The sample code has directories for the following:

* data/holds the portfolio data including the S&P 500 equities. In this tutorial, you analyze only a small subset, namely the FANG stocks (Facebook, Amazon, Netflix, and Google).
* deploymentmanager/ holds the Deployment Manager templates for creating and launching the HTCondor compute cluster.
* model/ holds the Monte Carlo model that calculates future equity changes based on a random walk.
* startup-scripts/ holds additional scripts for creating the necessary system images for the compute cluster.

### Get historical data
The historical equity closing price data that drives the Monte Carlo simulation comes from Quandl. To get the dataset, do the following:

In a browser, go to the Quandl sign-in page.
Sign up for a personal account. After you register, Quandl gives you a personalized API key to use for accessing public datasets. Keep a copy of the key in a safe place.
In Cloud Shell, download the data. Replace [API_KEY] with the key you downloaded from Quandl.

make download apikey=[API_KEY]

This makefile contains a series of commands to download the data. The dataset is saved to the data/WIKI_PRICES_2018-01-01.csv file.

If you prefer to get the data directly:

Sign up for a Quandl account and get an API key as described in the previous steps.
Go to the following URL, substituting your API key for [API_KEY]:

https://www.quandl.com/api/v3/datatables/WIKI/PRICES?qopts.export=true&api_key=$[API_KEY]&date.gt=2018-01-01

Quandl creates a CSV file that contains a dataset with equity closing prices from January 1, 2018, packages the CSV file in a zip file, and downloads the zip file to your computer.

Unzip the file.

Rename the CSV file to WIKI_PRICES_2018-01-01.csv and put it in the data subdirectory.

Upload historical data and simulation code to Cloud Storage
After you obtain the simulation code and historical data, you make it available in Google Cloud for subsequent processing.

In Cloud Shell, create a Cloud Storage bucket and copy the data and code to the bucket. Replace [YOUR_BUCKET_NAME] with a unique name for your Cloud Storage bucket.

make upload bucketname=[YOUR_BUCKET_NAME]

This command uploads the data and the HTCondor job scripts to the Cloud Storage bucket that you specified.

Alternatively, you can manually create the bucket and copy the data to it by using gcloud commands:

gsutil mb gs://[YOUR_BUCKET_NAME]
gsutil cp data/WIKI_PRICES_2018-01-01.csv gs://[YOUR_BUCKET_NAME]/data/
gsutil cp data/companies.csv gs://[YOUR_BUCKET_NAME]/data/
gsutil cp model/* gs://[YOUR_BUCKET_NAME]/model/
gsutil cp htcondor/* gs://[YOUR_BUCKET_NAME]/htcondor/

### Creating an HTCondor cluster by using Deployment Manager templates
The Deployment Manager examples in the GitHub repository provides a template to use for HTCondor clusters. For this tutorial, you use a modified Deployment Manager template that is in the deploymentmanager/ subdirectory.

The modified template contains the following changes:

The template adds support for using preemptible virtual machines (VMs) in addition to regular VMs. Preemptible VMs reduce the cost of Compute Engine instances by up to 80%, but might be reclaimed without notice. This is acceptable for this use case. If a preemptible VM is reclaimed, the HTCondor scheduler treats the reclaimed nodes as failed nodes and reschedules jobs on other nodes.

To keep jobs moving, the template creates two sets of instance groups: one for preemptible VMs and one for regular VMs. For different scenarios, you can vary the ratio of node types as needed.

The cluster nodes are configured without public internet access. In financial computing, you need to reduce the risk that any node can be compromised. The original HTCondor templates use a generic base operating system image and download the required software the first time the node is booted. This configuration requires each node in the cluster to have an internet connection in order to contact the public repository for the HTCondor software. The modified templates contain a set of custom images that have the required software preinstalled for the compute cluster. The nodes therefore don't need internet connectivity.

The nodes need access to Cloud Storage, so the network is configured to give you private access to Google APIs.

The cluster configuration is set by using the deploymentmanager/condor-cluster.yaml file in the cloned repository. The default values in this configuration file are sufficient for this tutorial. However, you can modify the cluster configuration as needed. The underlying code is available, and you can adapt it to support other configurations, such as additional GPUs or additional software packages.

For example, you might change the following values:

Minimum core size, which is the number of cores that the cluster provisions.
Instance type, which is the node instance type to provision.
Before you can create the cluster, you need to create custom images with all of the required software preinstalled.

In Cloud Shell, create the custom images:

make createimages

This command creates three custom images in your project: condor-master, condor-submit, and condor-compute.

Alternatively, you can use the following commands to create each image:

gcloud compute instances create condor-master-template \
    --zone=us-east1-b \
    --machine-type=n1-standard-1 \
    --image=debian-9-stretch-v20181210 \
    --image-project=debian-cloud \
    --boot-disk-size=10GB \
    --metadata-from-file \
         startup-script=startup-scripts/condor-master.sh
sleep 300
gcloud compute instances stop \
    --zone=us-east1-b condor-master-template
gcloud compute images create condor-master  \
    --source-disk condor-master-template   \
    --source-disk-zone us-east1-b   \
    --family htcondor-debian
gcloud compute instances delete \
    --quiet \
    --zone=us-east1-b condor-master-template

These commands do the following tasks:

Create a template VM for condor-master that uses a startup script that downloads and configures HTCondor.
Stop the VM and create a snapshot for the image.
Delete the template VM.
This can take upwards of 30 minutes to complete.

If you run individual gcloud commands, run the set of commands three times, once each for the condor-master, condor-compute, and condor-submit hosts. Change the name in the commands to match the host that you're configuring, for example, condor-compute-template, condor-submit-template.

Edit the parameters for the HTCondor cluster in the deploymentmanager/condor-cluster.yaml file. In particular, set the cluster size. For small-scale testing, ensure that the count and pvmcount values are both set to 2. Those settings provision a four-node cluster that's sufficient for this tutorial.

### Deploy an HTCondor cluster into your project:

make createcluster

Alternatively, run the following command:

gcloud deployment-manager deployments create condor-cluster \
    --config deploymentmanager/condor-cluster.yaml

It might take a few minutes to create the deployment and provision the nodes.

When the process is complete, use ssh to log into the condor-submit host:

gcloud compute ssh condor-submit

At the command line of the host, determine how many cores are available in the cluster:

condor_status

The output is similar to the following:

     Total Owner Claimed Unclaimed Matched Preempting Backfill Drain
X86_64  16     0       0        16       0          0        0     0
Total   16     0       0        16       0          0        0     0

The output shows that a total of 16 cores are available because you created 4 nodes that each have 4 cores. Applications can request single cores or multiple cores for jobs as they are submitted to the scheduler.

Running the Monte Carlo simulations
Now that the hosts are deployed, you can use the HTCondor submission command to submit jobs to the scheduler.

But first, you must copy the application files from the Cloud Shell working directory into the submit host.

Log off from the condor-submit host.
In Cloud Shell, copy the application data and code to the htcondor-submit host:

make ssh bucketname=[YOUR_BUCKET_NAME]

Or:

gcloud compute scp htcondor/* data/* model/* condor-submit:

Use ssh to connect to the condor-submit host again to submit the Monte Carlo simulations:

gcloud compute ssh condor-submit

The HTCondor job specification is in the montecarlo-submit-job file:

# htcondor submit script for runrandom
executable              = run_montecarlo.sh
arguments               = $(Process)
transfer_input_files    = randomwalk.py,companies.csv,WIKI_PRICES_2018-01-01.csv
should_transfer_files   = IF_NEEDED
Transfer_Output_Files   = ""
when_to_transfer_output = ON_EXIT
log                     = run.$(Process).log
Error                   = err.$(Process)
Output                  = out.$(Process)
queue 4

This file defines the job for the scheduler to schedule on the compute nodes.

The executable is the run_montecarlo.sh file, which is a bash script that runs a Monte Carlo simulation.
The transfer_input_files specifies the set of data files that the scheduler copies to the compute nodes when the job is scheduled. You transfer the Monte Carlo model, the portfolio, and the historical pricing.
The transfer_output_files is null because the bash script copies the output to the Cloud Storage bucket.
The Error and Output specify the naming convention for the stderr and stdout files. You add the variable$(Process) in the names to avoid name collision.
The last line submits 4 jobs that correspond to the first 4 stocks in the stock portfolio.
Each Monte Carlo simulation is started by using the run_montecarlo.sh shell script file:

'#! /bin/bash
'# script that initiates the random walk.
'# takes one argument which is the index into the sp500.csv to
'# identify which stock to simulate.
'#
index=$(($1 + 2))
stockfile=companies.csv
stock=$(awk "NR == ${index} {print; exit}" ${stockfile} | cut -d, -f1)
export HOME=`pwd`

./randomwalk.py -c ${stock} --from-csv WIKI_PRICES_2018-01-01.csv > ${stock}.csv
CLOUDSDK_PYTHON=/usr/bin/python gsutil -m cp ${stock}.csv gs://[YOUR_BUCKET_NAME]/output/${stock}.csv

This process identifies a particular stock symbol from the companies.csv file and then runs the Monte Carlo simulation by using the historical pricing data in WIKI_PRICES_2018-01-01.csv.

After the process completes, the script uploads the output data to the Cloud Storage bucket that you created.

Edit the run_montecarlo.sh shell script, and in the line that contains gs://[YOUR_BUCKET_NAME]/output/${stock}.csv, replace [YOUR_BUCKET_NAME] with the name of the Cloud Storage bucket that you created earlier.

Submit the job to the Condor scheduler:

condor_submit montecarlo-submit-job

The command returns a job ID.

To see the progress of the jobs, run the following Condor commands:

Use the condor_q command to show jobs that are queued, but not yet executed.
Use the condor_history command to show already completed jobs. Replace [JOB_ID] with the job ID retrieved in the preceding step.

condor_history [JOB_ID]

When the simulations are complete, you can deprovision the cluster.

Disconnect from the condor-submit host.

In Cloud Shell, deprovision the cluster:

make destroycluster

Or:

gcloud deployment-manager deployments delete condor-cluster

Upload outputs to BigQuery
The next step is to load the data from Cloud Storage into BigQuery.

In Cloud Shell, load the data from Cloud Storage into a new table in BigQuery. Replace [YOUR_BUCKET_NAME] with the name of the Cloud Storage bucket you created earlier.

If you are using the makefile:

make bq bucketname=[YOUR_BUCKET_NAME]

Or use the Cloud SDK commands:

bq mk montecarlo_outputs
bq load --autodetect --source_format=CSV montecarlo_outputs.vartable
gs://[YOUR_BUCKET_NAME]/output/*.csv
cat bq-aggregate.sql | bq query --destination_table montecarlo_outputs.portfolio

### Analyze the simulation data by using Datalab

To analyze the data that's now stored in BigQuery, you can use BigQuery directly and create SQL statements to generate the data you're interested in. You can also use Data Studio to build a dashboard that is updated daily.

gcloud components install datalab

In this tutorial, you use Datalab to analyze the data by using Python and popular data science libraries, such as Pandas. Datalab gives you a managed Jupyter notebook that can do both the aggregations and the visualizations.

The notebook is available in the GitHub repository as a reference, but we recommend going through the details yourself.

To analyze the data in BigQuery:

In Cloud Shell, create a Datalab instance named montecarlo:

datalab create montecarlo

Configure Cloud Shell to access the Datalab notebooks.

In Datalab, click + Notebook to create a new notebook, and then add the following code to load the BigQuery Python libraries into the notebook. For more details, read using Datalab with BigQuery.

%load_ext google.cloud.bigquery

Connect to the Monte Carlo output table and load the data into a Pandas dataframe:

%%bigquery df
SELECT*
FROM `montecarlo_outputs.portfolio`

Visualize the growth in your portfolio:

matplotlib inline
df.T.plot(legend=False)

The results are similar to the following graph:

Graph that visualizes growth in portfolio

Each line in the graph represents the value of the portfolio—in this case, the sum of the shares of the FANG stocks—over 252 trading days based on the historical values for each stock. The graph visually shows the tremendous spread in possible values.

Instead of plotting each of the 1000 simulations, you might want to look at the mean of the portfolio values with corresponding confidence intervals. For example:

import matplotlib.pyplot as plt

mean = df.mean().reset_index().iloc[:,-1]
std  = df.std().reset_index().iloc[:,-1]
plt.errorbar(mean.index, mean, xerr=0.5, yerr=2*std, linestyle='-')
plt.show()

Graph that visualizes the mean of the portfolio values

To get better insight, you can create a histogram of the values on a particular date:

df.iloc[:,-1].hist(bins=100)

The resulting graph shows a histogram of values at day 252.

Data as a histograph

Get descriptive statistics for that day:

df.T[252].describe()

count     1000.000000
mean     27958.274051
std       9295.381959
min       9446.052887
25%      21106.335794
50%      26438.180608
75%      33112.044098
max      74756.152636
Name: f251_, dtype: float64

The result shows the descriptive statistics for that day, with a mean of about 30k and a standard deviation of 9k.

### Cleaning up
To avoid incurring charges to your Google Cloud Platform account for the resources used in this tutorial:

### Delete the project
Caution: Deleting a project has the following effects:
Everything in the project is deleted. If you used an existing project for this tutorial, when you delete it, you also delete any other work you've done in the project.
Custom project IDs are lost. When you created this project, you might have created a custom project ID that you want to use in the future. To preserve the URLs that use the project ID, such as an appspot.com URL, delete selected resources inside the project instead of deleting the whole project.
In the Cloud Console, go to the Manage resources page.
Go to the Manage resources page

In the project list, select the project that you want to delete and then click Delete delete.
In the dialog, type the project ID and then click Shut down to delete the project.
Delete the individual resources
Alternatively, if you can't delete the project, you can delete the various resources manually:

Delete the HTCondor cluster.
Delete instances templates and the images.
Delete the Cloud Storage bucket.
Delete the Datalab notebook.
Delete the BigQuery tables.


