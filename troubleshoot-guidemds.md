# Manufacturing Data Solutions Troubleshooting Guide

This page contains details on how to troubleshoot different areas of MDS.

## Table of Contents

- [Manufacturing Data Solutions Troubleshooting Guide](#manufacturing-data-solutions-troubleshooting-guide)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Deployment](#deployment)
    - [Initial checks](#initial-checks)
    - [Debugging failed deployment](#debugging-failed-deployment)
      - [Commonly encountered issues](#commonly-encountered-issues)
      - [In case of any other intermittent issues](#in-case-of-any-other-intermittent-issues)
  - [Troubleshoot custom entity CRUD](#troubleshoot-custom-entity-crud)
    - [Initial checks](#initial-checks-1)
    - [Debugging failed custom entity CRUD](#debugging-failed-custom-entity-crud)
    - [Troubleshoot via logs](#troubleshoot-via-logs)
      - [Commonly encountered errors](#commonly-encountered-errors)
    - [Troubleshooting MDS with logs analytics packs](#troubleshooting-mds-with-logs-analytics-packs)
  - [Troubleshoot Data Mapping](#troubleshoot-data-mapping)
    - [Initial checks](#initial-checks-2)
  - [Troubleshoot Data Ingestion](#troubleshoot-data-ingestion)
    - [Initial checks](#initial-checks-3)
    - [Debugging ingestion validation errors](#debugging-ingestion-validation-errors)
      - [Batch Ingestion](#batch-ingestion)
      - [Sample CSV File](#sample-csv-file)
      - [Stream Ingestion](#stream-ingestion)
      - [Sample File](#sample-file)
      - [OPCUA Stream Ingestion](#opcua-stream-ingestion)
    - [Debugging other ingestion errors](#debugging-other-ingestion-errors)
    - [Troubleshoot via logs](#troubleshoot-via-logs-1)
      - [For Twins](#for-twins)
      - [For Relationships](#for-relationships)
      - [Commonly encountered errors](#commonly-encountered-errors-1)
      - [Common Errors Related to OPCUA](#common-errors-related-to-opcua)
    - [Debugging response of Health API](#debugging-response-of-health-api)
      - [Unable to access health API](#unable-to-access-health-api)
      - [Failed status in health API response](#failed-status-in-health-api-response)
  - [Troubleshoot Control plane issues](#troubleshoot-control-plane-issues)
    - [Initial checks](#initial-checks-4)
      - [Check metrics emitted by Azure resources using Azure Monitor](#check-metrics-emitted-by-azure-resources-using-azure-monitor)
      - [How to view metrics in Azure Monitor](#how-to-view-metrics-in-azure-monitor)
    - [Debugging Control Plane issues](#debugging-control-plane-issues)
      - [How to change network settings](#how-to-change-network-settings)
      - [How to grant Database access](#how-to-grant-database-access)
  - [Troubleshoot Copilot](#troubleshoot-copilot)
    - [Initial checks](#initial-checks-5)
    - [Debugging Copilot issues](#debugging-copilot-issues)
      - [Troubleshoot via logs](#troubleshoot-via-logs-2)
      - [Commonly encountered issue](#commonly-encountered-issue)

## Introduction

This troubleshooting guide is designed to help you identify and resolve common issues that may arise during and after the deployment of Manufacturing Data Solutions service.

## Deployment

### Initial checks

- You can check if your deployment succeeded or not by looking at status of the resource.

### Debugging failed deployment

If status of your MDS resource is `Failed`, go to Managed Resource group for that resource named something like `{your-MDS-instance-name}-HostedResources-{unique-id}`.
You can also refer to Resource JSON details to fetch the details of Managed Resource group.

> [!NOTE]
> Minimum access needed - **Reader** role is required on above Managed Resource group.

![A screenshot showing Deployment Resource Json](../media/troubleshoot-deployment-resource-json.png)

Go to above resource group and then to **Deployments** tab on left-hand menu and check the deployment errors that might have occurred.

![A screenshot showing post deployment errors](../media/troubleshoot-deployment-errors.png)

Deployment can fail due to various reasons:

If you get `BadRequest` as error code, you have sent deployment values that don't match what is expected by Resource Manager.
Please check inner message for more details.

#### Commonly encountered issues

| Error Code                                                                                           | Meaning                                                | Mitigation                                                                                                                                                                                                                                                           |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Failed to assign role to UMI : `<your-UMI>`. Please check if UMI has required roles to assign roles. | Missing permissions for the User Managed Identity used | If the User Assigned Managed Identity doesn't have the Owner role assigned at the subscription scope, the role assignments won't work and causes the deployment to fail. Also, User managed identity’s service principal need to have Owner role assigned to itself. |
| DeploymentQuotaExceeded                                                                              | Insufficient quotas and limits for Azure services      | Azure resources have certain quotas and limits for various azure services like Azure Data Explorer, Open AI etc. Make sure you have enough quota for all the resources before starting the MDS deployment.                                                           |
| NoRegisteredProviderFound/ SubscriptionNotRegistered/ MissingSubscriptionRegistration                | Required Resource Providers aren't registered          | One of the prerequisites for the deployment is that you must register the Resource Providers listed in the deployment guide. Ensure that all the resource providers are properly registered before starting the deployment.                                          |
| Only one User Assigned Managed Identity is supported                                                 | You can’t add multiple User Managed Identities in MDS  |                                                                                                                                                                                                                                                                      |

Documentation of [common ARM deployment errors can be found on MS Learn](https://learn.microsoft.com/en-us/azure/azure-resource-manager/troubleshooting/common-deployment-errors)

#### In case of any other intermittent issues

Please try to redeploy the solution once. If problem persists you need to get in touch with Microsoft Support Team

## Troubleshoot custom entity CRUD

### Initial checks

Check status of create Entity API by following below steps:

1. Call the POST API. The response will include an `Operation-Location` header containing a URL.
   For example, it is trying to register device entity

    ```bash
    curl -X POST https://{{baseurl}}/dmm/entities \
    -H "Authorization: {BEARER_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
        "name": "Device",
        "columns": [
            {
                "name": "id",
                "description": "A unique id",
                "type": "String",
                "mandatory": "true",
                "semanticRelevantFlag": "true",
                "groupBy": "true",
                "primaryKey": "true"
            },
            {
                "name": "description",
                "description": "Additional information about the equipment",
                "type": "String",
                "mandatory": "false",
                "semanticRelevantFlag": "true",
                "groupBy": "false",
                "primaryKey": "false"
            },
            {
                "name": "hierarchyScope",
                "description": "Identifies where the exchanged information",
                "type": "String",
                "mandatory": "false",
                "semanticRelevantFlag": "true",
                "groupBy": "false",
                "primaryKey": "false"
            },
            {
                "name": "equipmentLevel",
                "description": "An identification of the level in the role-based equipment hierarchy.",
                "type": "Enum",
                "mandatory": "false",
                "semanticRelevantFlag": "true",
                "groupBy": "false",
                "primaryKey": "false",
                "enumValues": [
                    "enterprise",
                    "site",
                    "area",
                    "workCenter",
                    "workUnit",
                    "processCell",
                    "unit",
                    "productionLine",
                    "productionUnit",
                    "workCell",
                    "storageZone",
                    "Storage Unit"
                ]
            },
            {
                "name": "operationalLocation",
                "description": "Identifies the operational location of the equipment.",
                "type": "String",
                "mandatory": "false",
                "semanticRelevantFlag": "true",
                "groupBy": "false",
                "primaryKey": "false"
            },
            {
                "name": "operationalLocationType",
                "description": "Indicates whether the operational",
                "type": "Enum",
                "mandatory": "false",
                "semanticRelevantFlag": "true",
                "groupBy": "false",
                "primaryKey": "false",
                "enumValues": [
                    "Description",
                    "Operational Location"
                ]
            },
            {
                "name": "assetsystemrefid",
                "description": "Asset/ERP System of Record Identifier for Equipment (AssetID)",
                "type": "Alphanumeric",
                "mandatory": "false",
                "semanticRelevantFlag": "false",
                "groupBy": "false",
                "primaryKey": "false"
            },
            {
                "name": "messystemrefid",
                "description": "Manufacturing system Ref ID",
                "type": "Alphanumeric",
                "mandatory": "false",
                "semanticRelevantFlag": "false",
                "groupBy": "false",
                "primaryKey": "false"
            },
            {
                "name": "temperature",
                "description": "temperature of device",
                "type": "Double",
                "mandatory": "false",
                "semanticRelevantFlag": "true",
                "groupBy": "false",
                "primaryKey": "false"
            },
            {
                "name": "pressure",
                "description": "pressure of device",
                "type": "Double",
                "mandatory": "false",
                "semanticRelevantFlag": "true",
                "groupBy": "false",
                "primaryKey": "false"
            }
        ],
        "tags": {
            "ingestionFormat": "Batch",
            "ingestionRate": "Hourly",
            "storage": "Hot"
        },
        "dtdlSchemaUrl": "https%3A%2F%2Fl2storage.blob.core.windows.net%2Fcustomentity%2FDevice.json",
        "semanticRelevantFlag": true
    }'
    ```

    It will populate header's `Operation-Location` with the url to check status of API.

2. Decode the URL provided in the `Operation-Location` header in previous call to check the status via GET request.

    ```bash
    curl -X GET "https://{hostname}/dmm/entities/status/RegisterEntity/efd66e795001492fb67def8a94336384" \
    -H "Authorization: Bearer {BEARER_TOKEN}"
    ```

    | Check        | Action                                                                                                                                                                                                                                                                                                                                                                                              |                                         |
    | :----------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------- |
    | API Response | Ensure you receive a 202 response from the API and check the status using the `Operation-Location` header. Perform a GET request on the URL provided in the header and verify that the status in the response payload indicates success. <br> `"createdDateTime": "2024-05-27T03:22:50.1367676Z",`<br>`"resourceLocationUrl": "https://{hostname}/dmm/entities/Device"`,<br>`"status": "Succeeded"` | Path: `https://{hostname}/dmm/entities` |

### Debugging failed custom entity CRUD

1. Please check the response generated by the API to debug the error.
    For more instructions follow [Entity regstration and Update document](../Use/Entity-Registration-and-Updation.md)

### Troubleshoot via logs

<!-- markdownlint-disable MD028 -->
<!-- These are two different quote blocks-->
> [!NOTE]
> The instructions are written for a technical/IT professional with familiarity with Azure services.
> If you are not familiar with these technologies, you may need to work with your IT professional to complete the steps.

> [!IMPORTANT]
> Make sure that you at a minimum have [Log Analytics Reader role to the deployed Log Analytics Workspace instance](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-access?tabs=portal#log-analytics-reader)
<!-- markdownlint-enable MD028 -->

1. Go to **Application Insights** Resource in the Managed Resource Group

    ![The Azure Portal showing Application Insights](../media/troubleshoot-application-insights.png)

1. Select **Logs** under Monitoring navigation section on left hand side

    ![The Azure Portal showing Logs section in Application Insights](../media/troubleshoot-application-insights-logs.png)

1. Run the KQL in Application Insights provided in Table:

    For cases where CRUD API returned failed status

    | Check                      | Action                                                                                                                                                                                                                                                                                                                        |                                                                                                      |
    | -------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------- |
    | API Response               | Ensure you receive a 202 response from the API and check the status using the `Operation-Location` header. Perform a GET request on the URL provided in the header and if the status is `Failed`                                                                                                                              | Path: `{hostname}/dmm/entities`                                                                      |
    | Application Insights Check | Use the following query to check whether it is Redis issue, run below query in Application Insights: <br> `traces \| sort by timestamp \| where message contains "Error while creating entry in Redis"`.<br>If it is a Redis issue check the actual exception and check Redis resource is up and running.                     | Expected Result - Should return a single entry if an entity was registered within the specified time |
    | Application Insights Check | Use the following query to check whether it is Cosmos DB issue, run below query in Application Insights: <br> `traces \| sort by timestamp \| where message contains "Error occured while updating the status in Cosmos".<br>If it is a Cosmos DB issue check the inner exception and verify the Cosmos DB is up and running. | Expected Result - Should return a single entry if an entity was registered within the specified time |

    For cases where CRUD API returned faulted status -

    | Check                      | Action                                                                                                                                                                                            |                                 |
    | :------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------ |
    | API  Response              | Ensure you receive a 202 response from the API and check the status using the "Operation-Location" header. Perform a GET request on the URL provided in the header and if the status is `Faulted` | Path: `{hostname}/dmm/entities` |
    | Application Insights Check | This occurs when Another job is already processing this entity<br><code>traces \| sort by timestamp \| where message contains " Another job is already processing this entity"                    | </code>                         |

#### Commonly encountered errors

| Scenario                                  | Condition                                                                                                                      | Response Code | Response Body Detail                                                                                                                                         |
| :---------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------- | :------------ | :----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Entity Already Exists                     | Attempting to register an entity that already exists                                                                           | 400           | `{"Detail": "An entity already exists with the name Device. Please retry with a different name"}`                                                            |
| Invalid DTDL Schema Location              | Providing a location where the DTDL schema is not present                                                                      | 400           | `{"Detail": "Failed to get the DTDL schema from URL. Error: Response status code does not indicate success: 404 (The specified blob does not exist.)."}`     |
| Invalid DTDL Schema URL                   | Providing an invalid dtdlschemaUrl                                                                                             | 400           | `{"Detail": "Failed to get the DTDL schema from URL. Error: Response status code does not indicate success: 404 (The specified resource does not exist.)."}` |
| Response of GET job status API is Faulted | This occurs when Another job is already processing this entity                                                                 | 202           | `{"createdDateTime": "2024-05-27T03:22:50.1367676Z",`<br>`"resourceLocationUrl": "https://{hostname}/dmm/entities/Device",`<br>`"status": "Faulted"}`        |
| Response of GET job status API is Failed  | **Case 1:** Redis error:Error while creating entry in Redis.<br>**Case 2:** Cosmos error: Error occurred while updating cosmos | 202           | `{"createdDateTime":"2024-05-27T03:22:50.1367676Z",`<br>`"resourceLocationUrl": "https://{hostname}/dmm/entities/Device",`<br>`"status": "Failed"}` 

##### Troubleshooting MDS with logs analytics packs

#### Troubleshoot logs via Log analytics Pack

**Note:** The instructions are written for a technical/IT professional
with familiarity with Azure services. If you are not familiar with these
technologies, you may need to work with your IT professional to complete
the steps.

Make sure that you at a minimum have [Log Analytics Reader role to the
deployed Log Analytics Workspace
instance](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-access?tabs=portal#log-analytics-reader)

Below query is available in Log Analytics Workspace. To run and navigate
to the query pack please follow the below steps.

```kql
let startTime = datetime(\'2024-07-04T00:00:00\');

let endTime = datetime(\'2024-07-05T00:00:00\');

let reqs = AppRequests

\| where TimeGenerated between (startTime .. endTime)

\| where AppRoleInstance startswith \"aks-dmmcopilot-\"

\| where Url has \'copilot/v3/query\'

\| project OperationId, TimeGenerated, ResultCode, DurationMs;

let exc = AppExceptions

\| where TimeGenerated between (startTime .. endTime)

\| where AppRoleInstance startswith \"aks-dmmcopilot-\"

\| distinct OperationId, OuterMessage

\| summarize exception_list = make_list(OuterMessage, 10) by
OperationId;

let copilot_traces = AppTraces

\| where TimeGenerated between (startTime .. endTime)

\| where AppRoleInstance startswith \"aks-dmmcopilot-\";

let instructions = copilot_traces

\| where Message has \'The filtered instructions after removing\'

\| extend instructions =
parse_json(Properties)\[\'filteredInstructionIdsCSV\'\]

\| project OperationId, instructions;

let alias = copilot_traces

\| where Message has \'AliasDictionary -\'

\| extend alias = tostring(parse_json(Properties)\[\'customContext\'\])

\| where isnotempty(alias)

\| project OperationId, customAlias = alias;

let retries = copilot_traces

\| where Message has \'Total Number of Retry Attempts\'

\| summarize retries = make_list(Message) by OperationId;

let query = copilot_traces

\| where Message startswith \'User input -\'

\| extend query = tostring(parse_json(Properties)\[\'input\'\])

\| extend intent = iff(Message contains \'invalid\', \'invalid\',
\'valid\')

\| distinct OperationId, query, intent;

let tokens = copilot_traces

\| where Message has \'Prompt tokens:\' and Message has \'Total
tokens:\'

\| extend prompt_tokens =
toint(parse_json(Properties)\[\'PromptTokens\'\])

\| extend completion_tokens =
toint(parse_json(Properties)\[\'CompletionTokens\'\])

\| summarize make_list(prompt_tokens), make_list(completion_tokens) by
OperationId;

let total_tokens = copilot_traces

\| where Message has \'Prompt tokens:\' and Message has \'Total
tokens:\'

\| extend total_tokens =
toint(parse_json(Properties)\[\'TotalTokens\'\])

\| summarize total_token = sum(total_tokens) by OperationId;

let kql = copilot_traces

\| where Message has \'Generated Graph KQL Query -\'

\| extend KQLquery = parse_json(Properties)\[\'kqlQuery\'\]

\| summarize kqls = make_list(KQLquery) by OperationId

\| project OperationId, kqls;

let kql_sanitized = copilot_traces

\| where Message has \'Sanitized Graph KQL Query -\'

\| extend KQLquery = parse_json(Properties)\[\'kqlQuery\'\]

\| summarize s_kqls = make_list(KQLquery) by OperationId

\| project OperationId, s_kqls;

reqs

\| extend operation_id_req = OperationId

\| join kind=leftouter exc on \$left.operation_id_req ==
\$right.OperationId

\| join kind=leftouter query on \$left.operation_id_req ==
\$right.OperationId

\| join kind=leftouter instructions on \$left.operation_id_req ==
\$right.OperationId

\| join kind=leftouter alias on \$left.operation_id_req ==
\$right.OperationId

\| join kind=leftouter tokens on \$left.operation_id_req ==
\$right.OperationId

\| join kind=leftouter total_tokens on \$left.operation_id_req ==
\$right.OperationId

\| join kind=leftouter kql on \$left.operation_id_req ==
\$right.OperationId

\| join kind=leftouter kql_sanitized on \$left.operation_id_req ==
\$right.OperationId

\| join kind=leftouter retries on \$left.operation_id_req ==
\$right.OperationId

\| project

TimeGenerated,

OperationId,

ResultCode,

query,

DurationMs,

intent,

exception_list,

instructions,

customAlias,

list_prompt_tokens,

list_completion_tokens,

total_token,

kqls,

s_kqls,

retries

\| order by TimeGenerated desc
```

1\. Go to Log Analytics workspace Resource in the Managed Resource Group

![A screenshot of a computer Description automatically
generated](./media/image1.png)

2\. Select \"Logs\" under Monitoring navigation section on left hand
side 

![A screenshot of a computer Description automatically
generated](./media/image2.png)

3\. Click on query hub and select the Query Pack.

![A screenshot of a computer Description automatically
generated](./media/image3.png)

4\. To access saved queries, go to Query Hub and select label under
dropdown. In this case, we have saved query inside a label named
\"MDSTroubleshootingQueries\".

![A screenshot of a computer Description automatically
generated](./media/image4.png)

The above-mentioned query is available under this name: MDS Get All
Details

![A screenshot of a computer Description automatically
generated](./media/image5.png)

5\. Click on Run. To run the query.

![A screenshot of a computer Description automatically
generated](./media/image6.png)
         |

## Troubleshoot Data Mapping

### Initial checks

1. Please get in touch with Microsoft Support Team to help you in verifying that the mapping is done correctly.
   Make sure you understand the mapping via examples [mentioned in the mapping page](../Configure/Map-your-Data.md).

## Troubleshoot Data Ingestion

### Initial checks

1. Match the actual Twin and relationship count to expected one to use below API:

    ```bash
    curl -X GET "https://{hostname}/dmm/query/ingestionStatus?operator=BETWEEN&endDate=2024-05-20&startDate=2024-05-01" \
        -H "Authorization: Bearer {BEARER_TOKEN}"
    ```

    Check below link for more detailed information:

    [Ingestion Status API](../Use/Consume-via-Structured-Queries.md#ingestion-status-api)

2. Check that you get a healthy response from MDS MDS Health check API

    ```bash
    curl -H "Authorization: Bearer {BEARER_TOKEN}" https://{{hostname}}/dmm/health
    ```

    Output:

    ```json
    {
        "message": "MDS is ready to serve the requests.",
        "setupStatus": "Succeeded",
        "setupInfo": {
            "registerEntityStatus": "Succeeded"
        },
        "errorMessage": [],
        "id": "f26dcdeb-6453-4521-b28e-610171db3997",
        "version": 1
    }
    ```

### Debugging ingestion validation errors

> [!IMPORTANT]
> Minimum Access need to access validation errors in Fabric is **Workspace member**

#### Batch Ingestion

When files are uploaded to the Lakehouse, they undergo validation checks.

For batch ingestion, we write the first 200 validation failures per entity.
These errors are stored in a subfolder named **ValidationErrors** within the specified lake path provided by the customer.
A CSV file is then created in this folder, where the errors are recorded following the format provided below in the sample CSV file.
During re-ingestion, a new file is created with a new timestamp, and any validation errors encountered are recorded in this new file.

We are not doing cleanup of files in ValidationErrors. On reingestion, new files will get created with latest timestamp. It is user's responsibility to cleanup older files if not needed.

#### Sample CSV File

```csv
"File Name","Entity Name","File Type" , "Error Message", "Record"
"Operations Request_20240501134049.csv", "Operations Request", "CSV","DataTypeMisMatch: Cell Value(s) do(es) not adhere to the agreed upon data contract",""ulidoperationsrequest3762901per","YrbidoperationsRequest3762901", "descriptionoperationsRequest732483", "Production", "hierarchyscopeoperationsRequest577992", "2024-02-13T06:00:04.12", "2019-06-13T00:00:00","priorityoperationsRequest646820", "aborted""
```

#### Stream Ingestion

When stream of data is sent to MDS, in validation layer we capture the errors in single file per day.

On performing multiple reingestion, errors will be appended in the same file.

We have put the limit on size of file to 10MB, if file reaches to this maximum size, we no more write errors to the file per day.

#### Sample File

```csv
"TimeStamp", "Entity Name", "DataFormat" , "ValidationErrorDataCategory" ,"Error Message", "Properties","Relationships"
"20240501130605", "Operations Request", "JSON", "Property", "Invalid property value as there is mismatch with its type ", "{"id":"0128341001001","description":"EXTRUDE","hierarchyScope":"BLOWNLINE06","startTime":"2023-09-24T02:43","endTime":"2023-09-24T19:23:48","requestState":"completed"}", "[]"
"20240501130623", "Equipment Property", "JSON", "Property", "Mandatory Property Cannot be null or empty", "{"id":"","description":"Address of the hierarchy scope object","value":"Contoso Furniture  HQ  Hyderabad  India","valueUnitOfMeasure":"NA","externalreferenceid":"externalreferenceid7ET2VF39EH","externalreferencedictionary":"externalreferencedictionaryVfMZahPNcL","externalreferenceuri":"https://www.bing.com/"}", "[]"
"20240501130624", "Operations Request", "JSON", "Property", "Invalid property value as there is mismatch with its type ", "{"id":"0128341001001","description":"EXTRUDE","hierarchyScope":"BLOWNLINE06","startTime":"2023-09-24T02:43","endTime":"2023-09-24T19:23:48","requestState":"completed"}", "[]"
"20240501130625", "", "JSON", "Relationship", "Provide a valid entity name in the relationship", "{}", "[{"targetEntityName":"","keys":{"id":"FHnAddress1","value":"NA","valueUnitOfMeasure":"NA"},"relationshipType":null}]"
"20240501130627", "", "JSON", "Relationship", "Provide a valid entity name in the relationship", "{}", "[{"targetEntityName":"","keys":{"id":"FHnAddress1","value":"NA","valueUnitOfMeasure":"NA"},"relationshipType":null}]"
"20240501130628", "Equipment Property", "JSON", "Property", "Mandatory Property Cannot be null or empty", "{"id":"","description":"Ownership Type of the estabslishment","value":"Private Limited Company","valueUnitOfMeasure":"NA","externalreferenceid":"externalreferenceid6U5CFU5OVT","externalreferencedictionary":"externalreferencedictionarylxzpGxVYEN","externalreferenceuri":"https://www.bing.com/"}", "[]"
"20240501130630", "", "JSON", "Relationship", "Provide a valid entity name in the relationship", "{}", "[{"targetEntityName":"","keys":{"id":"EZWProducts Handled1","value":"NA","valueUnitOfMeasure":"NA"},"relationshipType":null}]"
"20240501134049", "Operations Request", "JSON", "Property", "Invalid property value as there is mismatch with its type ", "{"id":"0128341001001","description":"EXTRUDE","hierarchyScope":"BLOWNLINE06","startTime":"2023-09-24T02:43","endTime":"2023-09-24T19:23:48","requestState":"completed"}", "[]"
```

#### OPCUA Stream Ingestion

When stream of opcua data or metadata is sent to MDS, in validation layer we capture the errors in single file per day.

On performing multiple reingestion, errors will be appended in the same file.
The *ValidationErrorDataCategory* field indicates whether the issue is with metadata or data.

We have put the limit on size of file to 10MB, if file reaches to this maximum size, we no more write errors to the file per day.

```csv
"TimeStamp", "DataFormat" , "ValidationErrorDataCategory" ,"Error Message", "DataMessage"
"20240513112048", "OPCUA", "OpcuaData", "[MDS_Ingestion_Response_Code : MDS_IG_OPCUA_MappingDataFieldsNotMatching] : Ingestion failed as Field key not matching with field in metadata with namespaceUri", "{"DataSetWriterId":200,"Payload":{"new key":{"Value":"Contoso Furniture  HQ  Hyderabad  Indianew"}},"Timestamp":"2023-10-10T09:27:39.2743175Z"}"
"20240508115927", "OPCUA", "OpcuaMetadata", "[MDS_Ingestion_Response_Code : MDS_IG_OPCUAMetadataPublisherIdMissing]: Validation failed as publisherId is missing", "{"MessageType":"ua-metadata","PublisherId":"","MessageId":"600","DataSetWriterId":200,"MetaData":{"Name":"urn:assembly.seattle;nsu=http://opcfoundation.org/UA/Station/;i=403","Fields":[{"Name":"Address Hyderabad","FieldFlags":0,"BuiltInType":6,"DataType":null,"ValueRank":-1,"MaxStringLength":0,"DataSetFieldId":"b6738567-05eb-4b48-8cd4-f38c9b120129"}],"ConfigurationVersion":null}}"
"20240508120004", "OPCUA", "OpcuaMetadata", "[MDS_Ingestion_Response_Code : MDS_IG_OPCUAMetadataPublisherIdMissing]: Validation failed as publisherId is missing", "{"MessageType":"ua-metadata","PublisherId":"","MessageId":"600","DataSetWriterId":200,"MetaData":{"Name":"urn:assembly.seattle;nsu=http://opcfoundation.org/UA/Station/;i=403","Fields":[{"Name":"Address Hyderabad","FieldFlags":0,"BuiltInType":6,"DataType":null,"ValueRank":-1,"MaxStringLength":0,"DataSetFieldId":"b6738567-05eb-4b48-8cd4-f38c9b120129"}],"ConfigurationVersion":null}}"
"20240508120329", "OPCUA", "OpcuaMetadata", "[MDS_Ingestion_Response_Code : MDS_IG_OPCUAMetadataPayloadDeserializationFailed] : OPCUAMetadataIngestionFailed as JsonDeserialization to MetadataMessagePayload payload failed", "{"MessageType":"ua-metadata","PublisherId":"publisher.seattle","MessageId":"602","MetaData":{"Name":"urn:assembly.seattle;nsu=http://opcfoundation.org/UA/Station/;i=405","Fields":[{"Name":"Address Peeniya","FieldFlags":0,"BuiltInType":6,"DataType":null,"ValueRank":-1,"MaxStringLength":0,"DataSetFieldId":"b6738567-05eb-4b48-8cd4-f38c9b120129"}],"ConfigurationVersion":null}}"
"20240508121318", "OPCUA", "OpcuaMetadata", "[MDS_Ingestion_Response_Code : MDS_IG_OPCUAMetadataPayloadDeserializationFailed] : OPCUAMetadataIngestionFailed as JsonDeserialization to MetadataMessagePayload payload failed", "{"MessageType":"ua-metadata","PublisherId":"publisher.seattle","MessageId":"602","MetaData":{"Name":"urn:assembly.seattle;nsu=http://opcfoundation.org/UA/Station/;i=405","Fields":[{"Name":"Address Peeniya","FieldFlags":0,"BuiltInType":6,"DataType":null,"ValueRank":-1,"MaxStringLength":0,"DataSetFieldId":"b6738567-05eb-4b48-8cd4-f38c9b120129"}],"ConfigurationVersion":null}}"
```

### Debugging other ingestion errors

### Troubleshoot via logs

> [!Note]
> The instructions are written for a technical/IT professional with familiarity with Azure services.
> If you are not familiar with these technologies, you may need to work with your IT professional to complete the steps.

Make sure that you at a minimum have [Log Analytics Reader role to the deployed Log Analytics Workspace instance](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-access?tabs=portal#log-analytics-reader).

Go to **Application Insights** Resource in the Managed Resource Group

![A screenshot with Azure Portal showing Application Insights](../media/troubleshoot-application-insights.png)

Select "**Logs**" under "**Monitoring**" section on left hand side

![A screenshot with Azure Portal showing Logs section in Application Insights](../media/troubleshoot-application-insights-logs.png)

Run the following KQL in Application Insights to get more information

```kql
traces
| where message contains "ingestion failed"
```

Example of Batch ingestion csv row failure in Application Insights

#### For Twins

> [05:46:50 ERR] Ingestion failed for row:
> [rbsawmill1,KupSawMill1,Production Line for wood sawing operations,Large Wood Saw Medium Wood Saw,workUnit,ProductionLine,Description,assetsystemrefidYKUQX8MDDD,messystemrefidBA3XM6UOAL,dkjsfhdks] at 04/29/2024 05:46:50 for entity Equipment with filename : sampleLakehouse.Lakehouse/Files/dpfolder/Equipment_202404291105102034.csv due to There is a mismatch in total number of columns defined.

#### For Relationships

> [12:09:41 ERR] Ingestion failed for row:
> [Person,dhyardhandler11Test,,bbhandlinglocationyard3511,hasValuesOf] at 04/30/2024 12:09:41 with filename : sampleLakehouse.Lakehouse/Files/dpfolder/Person_Mapping_202403190947077573.csv due to Source Type and Target Type cannot be empty for the record.

#### Commonly encountered errors

| Error Code                                                              | Meaning                                                                            | Mitigation                                                                                            |
| :---------------------------------------------------------------------- | :--------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------- |
| DMM_IG_FileOnlyHasHeaderInformation                                     | The file only has header                                                           | Ensure that the file contains data in addition to the header.                                         |
| DMM_IG_InvalidHeader                                                    | Column Names in the header do not match with registered columns                    | Check and correct the column names in the file header.                                                |
| DMM_IG_NotAllMandatoryColumnsArePresent                                 | Header Validation Error: Mandatory Column is Missing                               | Include all mandatory columns defined during entity registration in the file.                         |
| DMM_IG_RowIdColumnNotPresent                                            | Header Validation Error: ROW ID is missing                                         | Ensure that the ROW ID, which is a mandatory column, is present in the entity file.                   |
| DMM_IG_ColumnCountRowCountMismatch                                      | There is a mismatch in total number of columns defined                             | Ensure that the number of columns in the file matches the expected count.                             |
| DMM_IG_MandatoryColumnNotNullNotEmpty                                   | Cell Value(s) at Mandatory Column(s) are either null or empty                      | Provide non-null and non-empty values for mandatory columns.                                          |
| DMM_IG_DataTypeMisMatch                                                 | DataTypeMisMatch: Cell Value(s) do(es) not adhere to the agreed upon data contract | Ensure that cell values conform to the specified data types.                                          |
| DMM_IG_RowIdValueNotPresent (487)                                       | ROW ID value is missing                                                            | Ensure that the ROW ID value, which is mandatory, is present and not null or empty.                   |
| DMM_IG_MandatoryPropertiesOfEntityAreMissingInRelationships (489)       | Mandatory Properties of entity are missing in relationships                        | Include all mandatory properties in the entity relationships.                                         |
| DMM_IG_InvalidPropertyValue (490)                                       | Invalid property value as there is mismatch with its type                          | Ensure that property values match their specified types.                                              |
| DMM_IG_ProvideAValidTargetEntityNameInTheRelationship (493)             | Provide a valid entity name in the relationship                                    | Specify a valid target entity name in the relationship.                                               |
| DMM_IG_TargetEntityNameIsNotRegisteredInDMM (494)                       | Target entity name is not registered in MDS                                        | Register the target entity in MDS before defining relationships.                                      |
| DMM_IG_PrimaryKeyPropertiesOfTargetEntityAreMissingInRelationship (495) | Properties marked as primary in target entity are missing in relationship          | Include all primary key properties in the relationship.                                               |
| DMM_IG_MandatoryPropertyCannotBeNullOrEmpty (496)                       | Mandatory Property Cannot be null or empty                                         | Provide non-null and non-empty values for mandatory properties.                                       |
| DMM_IG_CSVMappingFileSourceAndTargetShouldNotBeEmpty (499)              | Source Type and Target Type cannot be empty for the record                         | Provide valid source and target types in the mapping file.                                            |
| DMM_IG_CSVMappingNumberOfCellsDoNotMatchNumberOfColumns (501)           | Row data count is not matching with the header count of the mapping file           | Ensure that the number of cells in each row matches the number of columns in the mapping file header. |
| DMM_IG_EntityNameIsNotRegisteredInDMM (502)                             | Entity is not registered in MDS                                                    | Register the entity in MDS before ingestion.                                                          |

#### Common Errors Related to OPCUA

| Error Code                                      | Meaning                                                                                                                                                                                        | Mitigation                                                                                         |
| :---------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------- |
| DMM_IG_OPCUAInvalidDataNoRelatedTargetTwinExist | [DMM_Ingestion_Response_Code :DMM_IG_OPCUAInvalidDataNoRelatedTargetTwinExist] Unable to update OPCUA telemetry data as no related twin or target twin exist                                   | Ensure that a related twin or target twin exists before attempting to update OPCUA telemetry data. |
| DMM_IG_OPCUAMetadataPublisherIdMissing          | [DMM_Ingestion_Response_Code : DMM_IG_OPCUAMetadataPublisherIdMissing]: Validation failed as publisherId is missing                                                                            | Include a valid publisherId in the metadata before reingesting the data.                           |
| DMM_IG_OPCUAMetadataDataSetWriterIdIdMissing    | [DMM_Ingestion_Response_Code : DMM_IG_OPCUAMetadataDataSetWriterIdIdMissing]: Validation failed as DatasetWriterId is missing                                                                  | Provide a valid DatasetWriterId in the metadata before reingesting the data                        |
| DMM_IG_OPCUAMetadataUnsupportedMessageType      | [DMM_Ingestion_Response_Code : DMM_IG_OPCUAMetadataUnsupportedMessageType]: Validation failed as provided MessageType is not supported. The supported MessageTypes are ua-data or ua-metadata. | Ensure the MessageType is either ua-data or ua-metadata before reingesting the data.               |
| DMM_IG_OPCUAMetadataNameMissing                 | [DMM_Ingestion_Response_Code : DMM_IG_OPCUAMetadataNameMissing]: Validation failed as Metadata Name is null or empty                                                                           | Provide a non-null and non-empty Metadata Name before reingesting the data.                        |
| DMM_IG_OPCUAMetadataFieldsMissing               | [DMM_Ingestion_Response_Code : DMM_IG_OPCUAMetadataFieldsMissing]: Validation failed as Metadata fields cannot be empty                                                                        | Ensure that Metadata fields are populated with valid entries before reingesting the data.          |
| DMM_IG_OPCUA_MappingMissing                     | [DMM_Ingestion_Response_Code : DMM_IG_OPCUA_MappingMissing]: Ingestion failed due to missing mapping                                                                                           | Ensure that the mapping is uploaded prior sending telemetry events                                 |

### Debugging response of Health API

Various Issues encountered related to health API

#### Unable to access health API

After the deployment is completed, you can call the health API to check if the system is ready state.
There can be Authentication or Authorization issue while trying to invoke the health API.
Ensure that you have created the App registration as per the [documentation provided](../Deploy/deploy-mds-resource.md/#create-a-new-app-registration) and has necessary roles assigned.
You need to fetch the access token against the App ID that you mentioned as part of the deployment and use it as the Bearer token while invoking any of the MDS APIs.

#### Failed status in health API response

Sometimes the MDS health API can return the response with status as `Failed`.
The response also contains the actual error message.
Depending on the error message, you need to get in touch with the Microsoft support team.

## Troubleshoot Control plane issues

### Initial checks

#### Check metrics emitted by Azure resources using Azure Monitor

Diagnostic settings are enabled by default to capture additional metrics for our Infrastructure resources.
These metrics are enabled for following resources and send to Log Analytics workspace which gets procured as part of deployment.
These settings are enabled for all MDS SKUs.

1. Azure Data explorer
2. Event-hub
3. Function App
4. Azure Kubernetes Service
5. Cosmos DB
6. Azure Open AI
7. App Service Plan
8. Azure Redis Cache

> [!NOTE]
> The retention period for these Metrics in Log Analytics workspace is 30 days.

#### How to view metrics in Azure Monitor

[Follow the guidance on Azure Monitor in MS Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/monitor-azure-resource).

### Debugging Control Plane issues

Common encountered scenarios

#### How to change network settings

- Ensure you have at least **Contributor** role on the Resource.
  Go to Networking tab for the required resource, add your IP address.

#### How to grant Database access

Pre-requisite: Your IP is added in networking settings

1. Ensure you have corresponding role required to access the Resource.
    For example, if you were to access storage account, ensure you have **Storage Blob Data Reader** or **Storage Blob Data Contributor** on the Storage account.
    Similarly, for Cosmos Database.

2. For Azure Data Explorer, you would need **Database Reader** permissions to query the data.
   [Follow the documentation on MS Learn to achieve this](https://learn.microsoft.com/en-us/azure/data-explorer/manage-cluster-permissions)

## Troubleshoot Copilot

### Initial checks

1. Try to fetch information from a table by running Copilot Query API

```txt
POST https://{hostname}/copilot/v3/query?api-version=2024-06-30-preview
```

```jsonc
{
    "ask": "string", // natural language question
}
```

### Debugging Copilot issues

#### Troubleshoot via logs

> [!NOTE]
> The instructions are written for a technical/IT professional with familiarity with Azure services.
> If you are not familiar with these technologies, you may need to work with your IT professional to complete the steps.

Make sure that you at a minimum have [Log Analytics Reader role to the
deployed Log Analytics Workspace
instance](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-access?tabs=portal#log-analytics-reader)

1. Run the following KQL in Application Insights with updated start and end time to get more information

    ```kql
    let startTime = datetime('2024-07-04T00:00:00');
    let endTime = datetime('2024-07-05T00:00:00');
    let reqs = requests
        | where timestamp between (startTime .. endTime)
        | where cloud_RoleInstance startswith "aks-dmmcopilot-"
        | where url has 'copilot/v3/query'
        | project operation_Id, timestamp, resultCode, duration;
    let exc = exceptions
        | where timestamp between (startTime .. endTime)
        | where cloud_RoleInstance startswith "aks-dmmcopilot-"
        | distinct operation_Id, outerMessage
        | summarize exception_list = make_list(outerMessage, 10) by operation_Id;
    let copilot_traces = traces
        | where timestamp between (startTime .. endTime)
        | where cloud_RoleInstance startswith "aks-dmmcopilot-";
    let instructions = copilot_traces
        | where message has 'The filtered instructions after removing'
        | extend instructions = parse_json(customDimensions)['filteredInstructionIdsCSV']
        | project operation_Id, instructions;
    let alias = copilot_traces
        | where message has 'AliasDictionary -'
        | extend alias = tostring(parse_json(customDimensions)['customContext'])
        | where isnotempty(alias)
        | project operation_Id, customAlias = alias;
    let retries = copilot_traces
        | where message has 'Total Number of Retry Attempts'
        | summarize retries = make_list(message) by operation_Id; 
    let query = copilot_traces
        | where message startswith 'User input -'
        | extend query = tostring(parse_json(customDimensions)['input'])
        | extend intent = iff(message contains 'invalid', 'invalid', 'valid')
        | distinct operation_Id, query, intent; 
    let tokens = copilot_traces
        | where message has 'Prompt tokens:' and message has 'Total tokens:'
        | extend prompt_tokens = toint(parse_json(customDimensions)['PromptTokens'])
        | extend completion_tokens = toint(parse_json(customDimensions)['CompletionTokens'])
        | summarize make_list(prompt_tokens), make_list(completion_tokens) by operation_Id;
    let total_tokens = copilot_traces
        | where message has 'Prompt tokens:' and message has 'Total tokens:'
        | extend total_tokens = toint(parse_json(customDimensions)['TotalTokens'])
        | summarize total_token = sum(total_tokens) by operation_Id;
    let kql = copilot_traces
        | where message has 'Generated Graph KQL Query -'
        | extend KQLquery = parse_json(customDimensions)['kqlQuery'] 
        | summarize kqls = make_list(KQLquery) by operation_Id
        | project operation_Id, kqls;
    let kql_sanitized = copilot_traces
        | where message has 'Sanitized Graph KQL Query -'
        | extend KQLquery = parse_json(customDimensions)['kqlQuery'] 
        | summarize s_kqls = make_list(KQLquery) by operation_Id
        | project operation_Id, s_kqls;
    reqs
    | extend operation_id_req = operation_Id
    | join kind=leftouter exc on $left.operation_id_req == $right.operation_Id
    | join kind=leftouter query on $left.operation_id_req == $right.operation_Id
    | join kind=leftouter instructions on $left.operation_id_req == $right.operation_Id
    | join kind=leftouter alias on $left.operation_id_req == $right.operation_Id
    | join kind=leftouter tokens on $left.operation_id_req == $right.operation_Id
    | join kind=leftouter total_tokens on $left.operation_id_req == $right.operation_Id
    | join kind=leftouter kql on $left.operation_id_req == $right.operation_Id
    | join kind=leftouter kql_sanitized on $left.operation_id_req == $right.operation_Id
    | join kind=leftouter retries on $left.operation_id_req == $right.operation_Id
    | project
        timestamp,
        operation_Id,
        resultCode,
        query,
        duration,
        intent,
        exception_list,
        instructions,
        customAlias,
        list_prompt_tokens,
        list_completion_tokens,
        total_token,
        kqls,
        s_kqls,
        retries
    | order by timestamp desc
    ```

2. Go to **Application Insights** Resource in the Managed Resource Group

    ![A screenshot with Azure Portal showing Application Insights](../media/troubleshoot-application-insights.png)

3. Select "**Logs**" under Monitoring navigation section on left hand side  
   ![A screenshot with Azure Portal showing Logs section in Application Insights](../media/troubleshoot-application-insights-logs.png)

#### Commonly encountered issue

|                                          Exception Message                                          | Meaning                                                                                                                                                 |                               Path                                |
| :-------------------------------------------------------------------------------------------------: | :------------------------------------------------------------------------------------------------------------------------------------------------------ | :---------------------------------------------------------------: |
|                        No OperationDoc found for operationId: {operationId}                         | No API Response with provided operationId could be found                                                                                                |                   GET /operations/{operationId}                   |
|                                Unable to perform operation {Reason}                                 | API cannot be executed at the moment                                                                                                                    |                            POST /query                            |
|                               User ask is a invalid intent to process                               | User Query is not related to Manufacturing Operations Management Domain                                                                                 |                            POST /query                            |
|                        Unable to generate valid KQL query for the user input                        | Copilot couldnot generate a valid KQL Query. Please run the provided Application Insights query for more information                                    |                            POST /query                            |
|                           Error occurred while executing query {kqlQuery}                           | Encountered some error while executing the KQL Query. Please run the provided Application Insights query for more information                           |                            POST /query                            |
|                            Summary generation failed for the input query                            | Copilot couldnot generate a valid KQL Query using the provided session history. Please run the provided Application Insights query for more information |                            POST /query                            |
|                  Failed to Generate a Context Aware Query for the user input query                  | Copilot couldnot generate summary for the user query. Please run the provided Application Insights query for more information                           |                            POST /query                            |
|                                Unable to perform operation {Reason}                                 | API cannot be executed at the moment                                                                                                                    |                         GET /instructions                         |
|                                Unable to perform operation {Reason}                                 | API cannot be executed at the moment                                                                                                                    |                 GET /instructions/{instructionId}                 |
|       No Instruction with InstructionId : {instructionId} and Version : {version} is present        | No Instruction exists with the given InstructionId                                                                                                      |                 GET /instructions/{instructionId}                 |
|                                Unable to perform operation {Reason}                                 | API cannot be executed at the moment                                                                                                                    |                         PUT /instructions                         |
|                    Validation failed for Instruction request. \nError: {Message}                    | Validation Error in payload. Please refer to Error: {exception Message} for more details                                                                |                         PUT /instructions                         |
|                                Unable to perform operation {Reason}                                 | API cannot be executed at the moment                                                                                                                    | PATCH /instructions/{instructionId}/versions/{instructionVersion} |
|                 Validation failed for Instruction Patch request. \nError: {Message}                 | Validation Error in payload. Please refer to Error: {exception Message} for more details                                                                | PATCH /instructions/{instructionId}/versions/{instructionVersion} |
|    No Instruction found with InstructionId : {InstructionId} and Version : {InstructionVersion}     | No Instruction exists with the given InstructionId and InstructionVersion                                                                               | PATCH /instructions/{instructionId}/versions/{instructionVersion} |
|            Can not find the following linked instruction id registered: {InstructionId}             | No Linked Instruction exists with the given InstructionId                                                                                               |                         GET /exampleQuery                         |
|                       Example query doc with exampleId {exampleId} not found                        | No Example exists with the given exampleId                                                                                                              |                   GET /exampleQuery/{exampleId}                   |
|                          Invalid file type. Only json files are supported.                          | Wrong API payload format                                                                                                                                |                         PUT /exampleQuery                         |
|                       Can not cast the given file to example query {Message}                        | Validation Error in payload. Please refer to Error: {exception Message} for more details                                                                |                         PUT /exampleQuery                         |
|                 The given query has admin command {Admin Command} at index {Index}                  | Uploaded Example contains unsupported admin commands. Please check again                                                                                |                         PUT /exampleQuery                         |
|            Can not find the following linked instruction id registered: {InstructionId}             | No Linked Instruction exists with the given InstructionId                                                                                               |                         PUT /exampleQuery                         |
|           The exampleID {ExampleId} is part of internal examples. Please use any other id           | Exception raised on use of restricted keword "Internal" for exampleId                                                                                   |                         PUT /exampleQuery                         |
| An example with the same UserQuestion, SampleQuery, LinkedInstructions and exampleId already exists | Found an existing example with same UserQuestion, SampleQuery, LinkedInstructions and exampleId                                                         |                         PUT /exampleQuery                         |
|                       Example query doc with exampleId {exampleId} not found                        | No Example exists with the given exampleId                                                                                                              |                  PATCH /exampleQuery/{exampleId}                  |
|                   An example with id {exampleId} has the same linked instructions                   | Found an existing example with same LinkedInstructions                                                                                                  |                  PATCH /exampleQuery/{exampleId}                  |
|           Can not find the following linked instruction id registered: {Instruction Ids}            | No Linked Instruction exists with the given InstructionId                                                                                               |                  PATCH /exampleQuery/{exampleId}                  |
|                       Example query doc with exampleId {exampleId} not found                        | No Example exists with the given exampleId                                                                                                              |                 DELETE /exampleQuery/{exampleId}                  |
|                            Alias info not found related to the id: {id}                             | No Alias information could be found for the given Id                                                                                                    |                         GET /aliases/{id}                         |
|                            Error saving custom alias with the key {key}                             | Alias information could not be saved at the moment                                                                                                      |                           PUT /aliases                            |
|                            Alias info not found related to the id: {id}                             | No Alias information could be found for the given Id                                                                                                    |                       DELETE /aliases/{id}                        |
|            Custom Alias Dictionary with the {id} is Internal. Hence, cannot be deleted.             | Unable to delete the Alias Dictionary provided as it is Internal                                                                                        |                       DELETE /aliases/{id}                        |

---
