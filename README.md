# MSLicensesReporter
A repeating report produced for an Azure environment to find current licensing deployment. Includes searches for disabled users with licenses. The report can be sent to an email address.

## Overview
![img](https://github.com/kgao826/MSLicensesReporter/blob/main/images/MS%20Licenses%20Flow%20Diagram.png)

Function
This logic app is able to send an API request to Microsoft Graph and then retrieve the relevant license information for each license.

Microsoft Graph displays the licenses using the SKU name, which is slightly different from what we see on Entra ID. To get the proper SKU names refer to this [link](https://learn.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-service-plan-reference)

The logic app is repeated using a recurrence trigger so you can set the monthly report to weekly or fortnightly, etc.

![img](https://github.com/kgao826/MSLicensesReporter/blob/main/images/L1.png)

I have also added a time converter so it runs local time (using convert time zone) at midnight. You can also set the time to run at when the office opens e.g. at 9am so it is most up to date. Remember by default is UTC timezone.

![img](https://github.com/kgao826/MSLicensesReporter/blob/main/images/L2.png) ![img](https://github.com/kgao826/MSLicensesReporter/blob/main/images/L3.png)

For the permissions, we use a system-assigned identity for the logic app to allow it to view licenses in MS Graph.

## Creating the Logic App
First, we deploy an Azure Logic app to a chosen resource group. On the left-hand side pane, make sure to go to **Identity** and turn on System Managed Identity. We will need to assign permissions to this logic app so that it can read user licensing information.
You will need to assign the permission of Microsoft Graph **Directory.Read.All**. This will allow the app to view everything associated with MSGraph. If you do not need to get the total users in the organisation, then you can search for a more granular permission in the Graph API.

Create a group and start with the HTTP request to get MS Licenses.

![img](https://github.com/kgao826/MSLicensesReporter/blob/main/images/L4.png)

|HTTP| Value |
|--|--|
| Method | GET |
| URI | https://graph.microsoft.com/v1.0/subscribedSkus |
| Headers | Content-type: application/json |
| Authorisation | Managed Identity |

![img](https://github.com/kgao826/MSLicensesReporter/blob/main/images/L5.png)

Paste the above URI into the [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) and have a look yourself at what the output is.

From that output, you can create a Parse JSON function next and parse the JSON output, and you will notice in the JSON API response that the **consumedUnits** is the amount of licenses used and that in **prepaidUnits**, the **enabled** value is the total amount of licenses.

![img](https://github.com/kgao826/MSLicensesReporter/blob/main/images/L6.png)

To get the scheme you must feed in sample data (e.g. from Graph Explorer) of the returned JSON so it can automtically generate a schema for you. **Select Use sample payload to generate schema**.

Then, we put the output into an HTML table so that it is formatted properly (see below). The license is the **skuPartNumber**, the amount used is the **consumedUnits** and the total is **enabled**.

![img](https://github.com/kgao826/MSLicensesReporter/blob/main/images/L7.png)

## Total Members in a group
Depending on your organisation, you may only want to run licenses on a specific group. In this case, we need to create another scope (group) of functions in the logic app to get the total amount of users in a group. Our group is a dynamic group, so during the time of this logic app's creation, there was no easy method to get total users in a group without looping through the entire group. Hopefully, there are easier solutions in the future.
We can use the Entra ID connector to get all the users in that dynamic group. The scope is called Total Employees:

We use the Top parameter to ensure all the users are retrieved, otherwise it only retireves 100 by default.

We then just do a simple loop of all the users in that group and increment the variable Total Members. This will return the total members in that group. Hopefully, there will be something more simple in the future like get group count.

##Disabled Accounts with Licenses
This one is a bit more complicated as we directly put a filter in the URI of the HTTP request.

![image.png](/.attachments/image-3ed259d8-7277-406a-9ef5-cf56d97ac811.png)


|HTTP| Value |
|--|--|
| Method | GET |
| URI | https://graph.microsoft.com/v1.0/users?$select=displayName,userPrincipalName,accountEnabled,assignedLicenses&$filter=accountEnabled%20eq%20false%20and%20assignedLicenses/$count%20ne%200&$count=true |
| Headers | ConsistencyLevel: eventual |
| Authorisation | Manage Identity |

You will first notice that the URI has a lot of %. This is because the logic app HTTP cannot parse some characters like spaces and brackets unlike the [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer). To filter disabled accounts with licenses we use two functions: $Select and $Filter

The raw URI is as follows:
```
https://graph.microsoft.com/v1.0/users?$select=displayName,userPrincipalName,accountEnabled,assignedLicenses&$filter=accountEnabled eq false and assignedLicenses/$count ne 0&$count=true
```

Convert to HTTP format is (you can have a look at HTTP encoding [here](https://www.w3schools.com/tags/ref_urlencode.ASP)):
```
https://graph.microsoft.com/v1.0/users?$select=displayName,userPrincipalName,accountEnabled,assignedLicenses&$filter=accountEnabled%20eq%20false%20and%20assignedLicenses/$count%20ne%200&$count=true
```

**IMPORTANT!**
You will notice in the Headers that the **ConsistencyLevel** is set to **eventual**. This allows MS Graph API to parse $Select and $Filter functions. Otherwise, an error will be returned.

##Licenses
#315956 
To get the product name, ID number and string ID. Refer to this [link](https://learn.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-service-plan-reference).

The following licenses are included in the logic app, if you want to add any more simply add to this list.

| **String ID** | **Product Name** | **ID Number** |
|--|--|--|
| VISIOCLIENT | Visio Online Plan 2 | 71a9d935-2760-4973-b514-a5c33566cc4b_c5928f49-12ba-48f7-ada3-0d743a3601d5 |
| INTUNE_A_D | Microsoft Intune Plan 1| 71a9d935-2760-4973-b514-a5c33566cc4b_2b317a4a-77a6-4188-9437-b68a77b4e2c6 |
| POWER_BI_PRO | Power BI Pro | 71a9d935-2760-4973-b514-a5c33566cc4b_f8a1db68-be16-40ed-86d5-cb42ce701560 |
| IDENTITY_THREAT_PROTECTION | Microsoft 365 E5 Security | 71a9d935-2760-4973-b514-a5c33566cc4b_26124093-3d78-432b-b5dc-48bf992543d5 |
| PROJECTPREMIUM | Project Online Premium | 71a9d935-2760-4973-b514-a5c33566cc4b_09015f9f-377f-4538-bbb5-f75ceb09358a |
| EXCHANGESTANDARD | Exchange Online (Plan 1) | 71a9d935-2760-4973-b514-a5c33566cc4b_4b9405b0-7788-4568-add1-99614e613b69 |
| Microsoft_Teams_Exploratory_Dept | Microsoft Teams Exploratory | 71a9d935-2760-4973-b514-a5c33566cc4b_e0dfc8b9-9531-4ec8-94b4-9fec23b05fc8 |
| O365_BUSINESS_PREMIUM | Microsoft 365 Business Standard | 71a9d935-2760-4973-b514-a5c33566cc4b_f245ecc8-75af-4f8e-b61f-27d8114de5f3 |
| PBI_PREMIUM_PER_USER | Power BI Premium Per User | 71a9d935-2760-4973-b514-a5c33566cc4b_c1d032e0-5619-4761-9b5c-75b6831e1711 |
| MCOMEETADV | Microsoft 365 Audio Conferencing | 71a9d935-2760-4973-b514-a5c33566cc4b_0c266dff-15dd-4b49-8397-2bb16070ed52 |
| SPE_E3 | Microsoft 365 E3 | 71a9d935-2760-4973-b514-a5c33566cc4b_05e9a617-0261-4cee-bb44-138d3ef5d965 |
| O365_BUSINESS_ESSENTIALS | Microsoft 365 Business Basic | 71a9d935-2760-4973-b514-a5c33566cc4b_3b555118-da6a-4418-894f-7df1e2096870 |
| EXCHANGEENTERPRISE | Exchange Online (Plan 2) | 71a9d935-2760-4973-b514-a5c33566cc4b_19ec0d23-8335-4cbd-94ac-6050e30712fa |
| Microsoft_Teams_Rooms_Pro | Microsoft Teams Rooms Pro | 71a9d935-2760-4973-b514-a5c33566cc4b_4cde982a-ede4-4409-9ae6-b003453c8ea6 |

We then put the HTTP request in this format to get the number of licenses and assigned licenses:
|HTTP| Value |
|--|--|
| Method | GET |
| URI | https://graph.microsoft.com/v1.0/subscribedSkus/<ID Number> |
| Headers | Content-type: application/json |
| Authorisation | Manage Identity |

All the JSON response will be in the same format as shown below:
```
{
    "properties": {
        "@@microsoft.graph.tips": {
            "type": "string"
        },
        "@@odata.context": {
            "type": "string"
        },
        "accountId": {
            "type": "string"
        },
        "accountName": {
            "type": "string"
        },
        "appliesTo": {
            "type": "string"
        },
        "capabilityStatus": {
            "type": "string"
        },
        "consumedUnits": {
            "type": "integer"
        },
        "id": {
            "type": "string"
        },
        "prepaidUnits": {
            "properties": {
                "enabled": {
                    "type": "integer"
                },
                "lockedOut": {
                    "type": "integer"
                },
                "suspended": {
                    "type": "integer"
                },
                "warning": {
                    "type": "integer"
                }
            },
            "type": "object"
        },
        "servicePlans": {
            "items": {
                "properties": {
                    "appliesTo": {
                        "type": "string"
                    },
                    "provisioningStatus": {
                        "type": "string"
                    },
                    "servicePlanId": {
                        "type": "string"
                    },
                    "servicePlanName": {
                        "type": "string"
                    }
                },
                "required": [
                    "servicePlanId",
                    "servicePlanName",
                    "provisioningStatus",
                    "appliesTo"
                ],
                "type": "object"
            },
            "type": "array"
        },
        "skuId": {
            "type": "string"
        },
        "skuPartNumber": {
            "type": "string"
        },
        "subscriptionIds": {
            "items": {
                "type": "string"
            },
            "type": "array"
        }
    },
    "type": "object"
}
```

Just copy and paste that JSON format into the Parse JSON and it will get all the right attributes for you. Remember to name your Actions with good names otherwise good luck trying to select the right parameters.

Now to get the users assigned to the license we can use the following format:
Important! We use the license half ID so for example, Exchange Online (Plan 2):
```
71a9d935-2760-4973-b514-a5c33566cc4b_19ec0d23-8335-4cbd-94ac-6050e30712fa
```

We only use the ID after the underscore:
```
19ec0d23-8335-4cbd-94ac-6050e30712fa
```

Raw HTTP:
```
https://graph.microsoft.com/v1.0/users?$select=displayName,userPrincipalName,accountEnabled&$filter=assignedLicenses/any(u:u/skuId eq <License Half ID>)
```

Encoded format:
```
https://graph.microsoft.com/v1.0/users?%24select=displayName,userPrincipalName&%24filter=assignedLicenses/any%28u%3Au/skuId%20eq%20<License Half ID>%29
```

Example for Exchange Online (Plan 2):
```
https://graph.microsoft.com/v1.0/users?%24select=displayName,userPrincipalName&%24filter=assignedLicenses/any%28u%3Au/skuId%20eq%2019ec0d23-8335-4cbd-94ac-6050e30712fa%29
```

Try the above in [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) using **GET**

The parse JSON format will be the same:
```
{
    "properties": {
        "@@odata.context": {
            "type": "string"
        },
        "value": {
            "items": {
                "properties": {
                    "displayName": {
                        "type": "string"
                    },
                    "userPrincipalName": {
                        "type": "string"
                    }
                },
                "required": [
                    "displayName",
                    "userPrincipalName"
                ],
                "type": "object"
            },
            "type": "array"
        }
    },
    "type": "object"
}
```

This allows us to create a nice table for the output of the licence that into the HTML table.

##Email
Finally, the email format is very important, make sure we select the correct parameter as they are all called the same. It can be a bit confusing with everything the same but what you can do is search for the parameter e.g. "consumedUnits" then just make sure you click on the right header. For example:

![image.png](/.attachments/image-9b10843d-50ad-4c6e-a50c-69a127e203be.png)

**Email Format:**

|**Row**|
|--|
| License Name |
| Used: _consumedUnits_|
| Total: _enabled_|
| Assigned Users |
| HTML List of users |

##Adding another license
To add another license GET request simply follow the same format for each scope. You need to do the same two steps:
- Get license amount by parsing skuID
- Get assigned licenses users list

Parse the outputs and then create an HTML table for the users list. This will create the format as in the email.
