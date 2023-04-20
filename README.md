# Azure-Databricks
## Getting Started with Databricks on Azure
1. Signup for the Azure Account
2. Login and Increase Quotas for regional vCPUs in Azure
3. Create and Launch Azure Databricks Workspace
4. Create Azure Databricks Single Node Cluster
5. Upload Data using Azure Databricks UI
6. Create Notebook and Validating Files using %sh to make sure our file location
7. Develop Spark Application using Azure Databricks Notebook. Create ETL pipeline for read and do some transformation of the table using pyspark dataframe API and spark sql to get specific information we need. Then wirte back to the dbfs filestore using a new name.
8. Export Azure Databricks Notebooks in the dbc format
## Mount ADLS on to Azure Databricks to access files from Azure Blob Storage
1. Set up and configure Databricks CLI for Azure Darabricks by generate new token \
Host:[**https://adb-3658115624069835.15.azuredatabricks.net/**](https://adb-3658115624069835.15.azuredatabricks.net/) \
Token:dapi5e58fdf8e4b02a4e6780c7bb8855e1e4-3 (user-setting: new token) \
Using Databricks CLI for configure \
enter Host and Token to make sure it works
```
databricks configure --token  --profile itvadlsdb
```
 2. Register an Azure Active Directory Application \
 Go to Azure Active Directory and create a new Active Directory named "ADLS-Databricks-Mount-Demo" generate a new client secret for   further use. Make sure to copy the Application (client) ID and Object (tenant) ID for further use. 
 3. Create Databricks Secret for AD Application Client Secret \
 Using Databricks CLI to create a new secret scope named "itvadlsdbdemoscop" and put the client secret we just generated into this scope
 ```
 databricks secrets create-scope \
--scope itvadlsdbdemoscop \
--profile itvadlsdb \
--initial-manage-principal "users"
```
```
databricks secrets put \
--scope itvadlsdbdemoscop \ 
--key itvadlsdbdemokey \
--profile itvadlsdb
```
4. Create ADLS Storage Account \
First use terminal to log into the Azure account and write Databricks CLI for create ADLS Storage Account.
```
#see the resource group
az group list --query '[*].name'
#see the storage account
az storage account list --query '[*].name'
#create a storage account for specific resource group
az storage account create \
>  -n itvadlsdb \
>  -g itvadlsdbrgdemo \
>  -l eastus \
>  --sku Standard_LRS
```
5. Assign IAM Role on Storage Account to Azure AD \
Go to the Storage Account I just created and then click on Access Control(IAM), add role assignment which the role name is Storage Blob Data Contributor. Then select members "ADLS-Databricks-Mount-Demo" which we just create. Finally click on Review+assign.
6. Setup Retail DB Dataset \
Clone Dataset to our local environment from GitHub using git clone. Make sure the retail_db dataset looks good. Also Delete .git folder from the repository.
7. Create ADLS Container or File System and Upload Data \
Use Azure CLI to create container by name **data** in the storage account **itvadlsdbd**.
```
az storage fs create -n data --account-name itvadlsdbdemo
az storage fs list --account-name itvadlsdbd

az storage fs directory upload \
  -f data \
  --account-name itvadlsdbd \
  -s "/Users/yefan_nn/retail_db" \
  --recursive

az storage fs directory list \
  -f data \
  --account-name itvadlsdbd
```
8. Mount ADLS Storage Account on to Azure Databricks
```
configs = {"fs.azure.account.auth.type": "OAuth",
          "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
          "fs.azure.account.oauth2.client.id": "aa997fe9-d4c6-4407-b0ce-bfeb7df54422",
          "fs.azure.account.oauth2.client.secret": dbutils.secrets.get(scope="itvadlsdbdemoscop",key="itvadlsdbdemokey"),
          "fs.azure.account.oauth2.client.endpoint": "https://login.microsoftonline.com/44467e6f-462c-4ea2-823f-7800de5434e3/oauth2/token"}

dbutils.fs.mount(
  source = "abfss://data@itvadlsdb.dfs.core.windows.net/",
  mount_point = "/mnt/data",
  extra_configs = configs)
```
9. Validate ADLS Mount point 
```
%fs ls /mnt/data/retail_db

df = spark.read.csv(
  '/mnt/data/retail_db/orders',
  schema='order_id INT, order_date STRING, order_customer_id INT, order_status STRING')
display(df)

df.write.json('/mnt/data/retail_db_json/orders')

%fs ls /mnt/data/retail_db_json/orders
```


