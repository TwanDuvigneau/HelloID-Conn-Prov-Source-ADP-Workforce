# HelloID-Conn-Prov-Source-ADP-Workforce

| :information_source: Information |
|:---------------------------|
| This repository contains the connector and configuration code only. The implementer is responsible to acquire the connection details such as username, password, certificate, etc. You might even need to sign a contract or agreement with the supplier before implementing this connector. Please contact the client's application manager to coordinate the connector requirements.       |

<br />

The _'HelloID-Conn-Prov-Source-ADP-Workforce'_ connector needs to be executed 'on-premises'. Make sure you have 'Windows PowerShell 5.1' installed on the server where the 'HelloID agent and provisioning agent' are running, and that the 'Execute on-premises' switch is toggled.

Note that the _'HelloID-Conn-Prov-Source-ADP-Workforce'_ implementation is based on ADP Workforce environments for the Dutch market. If you want to implement the connector for the US market, changes will have to be made within the source code.

### Todo

- [X] Add _departments.ps1_
- [X] Add pagination for the _workerDemographics_ endpoint
- [X] Add logic to obtain an AccessToken inluding a *.pfx certificate

## Table of contents

 - Introduction
 - Prerequisites
 - Getting started
    - Supported PowerShell versions
    - Certificate
    - Paging
    - Custom Fields
    - Caveats
- PowerShell functions
- Setup the PowerShell connector

## Introduction

ADP Workforce is a cloud based HR management platform and provides a set of REST API's that allow you to programmatically interact with it's data. The HelloID source connector uses the API's in the table below.

---

### API's being used by the HelloID connector

| _API_ | _Description_|
| --- | ----------- |
| _WorkerDemographics_ | _Contains the employees personal and contract data_ |
| _OrganizationDepartments_ | _Contains data about the organisation structure_ |

---

## Prerequisites

- Windows PowerShell 5.1 installed on the server where the 'HelloID agent and provisioning agent' are running.

- The public key *.pfx certificate belonging to the X.509 certificate that's used to activate the required API's.

- The password for the public key *.pfx certificate.

- The 'Execute on-premises' switch on the 'System' tab is toggled.

## Getting started

### Supported PowerShell versions

The recommended PowerShell version for the  _'HelloID-Conn-Prov-Source-ADP-Workforce'_ is _Windows PowerShell 5.1_. The connector is not tested on older versions of Windows PowerShell.

_PowerShell 7.0.3 Core_ is not yet supported.

### X.509 certificate / public key

To get access to the ADP Workforce API's, a x.509 certificate is needed. This certificate has to be created by the customer.

The public key belonging to the certificate, must be send ADP. ADP will then generate a ClientID and ClientSecret and will activate the required API's.

There are a few options for creating certificates. One of them being the 'OpenSSL' utility. Available on Linux/Windows. https://www.openssl.org/

APD will register an application that's allowed to access the specified API's. _worker-demographics_ and _organizational_departments_. Other API's within the ADP Workforce environment cannot be accessed.

### X.509 certificate / Private key

The private key (*.pfx) belonging to the X.590 certificate must be used in order obtain an accesstoken.

There are two options available to import the *.pfx:

Option 1 called "Certificatepath" takes the path to the *.pfx on the machine on which the agent is configured.

Option 2 called "Application PFX Certificate" takes a base64 string of the *.pfx file, which powershell converts to a certificate object. This eliminates the need for a local on-premises agent.

Execute the following code to get the base64 of your *.pfx file in your clipboard: [System.Convert]::ToBase64String((get-content "C:\*.pfx" -Encoding Byte)) | Set-Clipboard

To use option 2, leave option 1 empty.

### AccessToken

In order to retrieve data from the ADP Workforce API's, an AccessToken has to be obtained. The AccessToken is used for all consecutive calls to ADP Workforce. To obtain an AccessToken, we will need the ___ClientID___, ___ClientSecret___, ___The path to your pfx certificate___ and the ___password for the pfx certificate___.

Tokens only have access to a certain API scope. Default the scope is set to: 'worker-demographics organization-departments'. Data outside this scope from other API's cannot be retrieved

### Paging

Paging is only supported by ADP for the 'worker-demograpics' endpoint. Paging is not yet implemented in the connector.

### Assignments

If a worker has multiple assignments, each assigment will be imported in HelloID.

### Custom Fields

Both the worker and assigment(s) may contain _custom fields_. Custom fields will be automatically imported in HelloID.

Custom fields can be selected in both the _person_ and _contract_ mapping.

### Mappings

A basic person and contract mapping is provided. Make sure to further customize these accordingly.

### Caveats

__[worker.businessCommunication]__

The _[worker.businessCommunication]_ array contains information about the:

- Fixed Phone Number
- Mobile Phone Number
- Email Address

All three are array's. Implying that they may contain multiple items.

Since the demo data doesn't have array's with multiple items and since there's no way to determine which item is 'primary'. At this point it's hardcoded to always pick the first __[0]__ based item in the array.

```powershell
if ($null -ne $worker.businessCommunication.landLines){
    $PhoneNumberFixed = $worker.businessCommunication.landLines[0].formattedNumber
}
```

__[worker.assignment.reportsTo]__

The _[worker.assignment.reportsTo]_ array for an assignment contains the information about the manager(s) a worker reports to.

The array may contain multiple items (managers) for an assignment. There's no way to determine which manager is the 'primary' manager for a particular contract/assignment. At this point it's hardcoded to always pick the first __[0]__ based item in the array.

```powershell
if ($null -ne $assignment.reportsTo){
    for ($i = 0; $i -lt $assignment.reportsTo.Length; $i++) {
        $manager = @{
            FormattedName = $assignment.reportsTo[0].reportsToWorkerName.formattedName
            WorkerID = $assignment.reportsTo[0].workerID.idValue
            AssociateOID = $assignment.reportsTo[0].associateOID
            RelationShipCode = $assignment.reportsTo[0].reportsToRelationshipCode.longName
        }
    }
}
```

## PowerShell functions

All PowerShell functions have comment based help. Both in the sourcecode and within the Github repository. <https://github.com/Tools4everBV/HelloID-Conn-Prov-Source-ADP-Workforce/tree/main/docs/en-US>

### Sample data

If you want to customize the connector according to your own needs, you can use the demo data from ADP.

Workers: <https://github.com/marketplace-esi/postman-samples/blob/master/workforce/hr/workers-v2-demographics/success/workers-v2-demographics-al-workers-http-200-response.json>

Department: <https://github.com/marketplace-esi/postman-samples/blob/master/workforce/core/success/core-organization-departments-http-200-response.json>

The connector configuration supports an import from a JSON file for both persons and departments.

### Usage in VSCode

If you need to test your code in VSCode, make sure to fill the _'$splatGetADPWorkers'_ object containing the following fields:

```powershell
$splatGetADPWorkers = @{
    BaseUrl = ''
    ClientID = ''
    ClientSecret = ''
    CertificatePath = ''
    CertificatePassword = ''
    ProxyServer = ''
    WorkerJson = ''
    ImportFile = ''
}
```
Comment out the _'$connectionSettings = ConvertFrom-Json $configuration'_ section.

To execute to code, use the following command:

```powershell
Get-ADPWorkers @splatGetADPWorkers
```

## Setup the PowerShell connector

1. Make sure you can access the ADP Workforce API's.

Obtain the accesstoken:

```powershell
$authorization = "$($CientID):$($clientSecretString)"
$base64String = [System.Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($authorization))

$headers = New-Object "System.Collections.Generic.Dictionary[[String],[String]]"
$headers.Add("grant_type", "client_credentials&scope=worker-demographics organization-departments")
$headers.Add("Authorization", "Basic $base64String)

$response = Invoke-RestMethod 'https://accounts.dex.adp.com/auth/oauth/v2/token' -Method 'POST' -Headers $headers
$response | ConvertTo-Json
$response.access_token
```

Test access to the ADP Workforce API's, replace the value from _$headers.Add("Authorization", "Bearer _your_access_token_")_ with the value from _$response.access_token_.

```powershell
$headers = New-Object "System.Collections.Generic.Dictionary[[String],[String]]"
$headers.Add("Authorization", "Bearer _your_access_token_")

$response = Invoke-RestMethod 'https://api.dex.adp.com/test ' -Method 'GET' -Headers $headers
$response | ConvertTo-Json
```

2. Add a new 'Source System' to HelloID and make sure to import all the necessary files.

    - [ ] configuration.json
    - [ ] personMapping.json
    - [ ] contractMapping.json
    - [ ] persons.ps1
    - [ ] departments.ps1

3. Fill in the required fields on the 'Configuration' tab.

---

# HelloID Docs
The official HelloID documentation can be found at: https://docs.helloid.com/
