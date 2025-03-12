# MSLicensesReporter
A repeating report produced for an Azure environment to find current licensing deployment. Includes searches for disabled users with licenses.

## Overview
Function
This logic app is able to send an API request to Microsoft Graph and then retrieve the relevant license
information for each license.

Microsoft Graph displays the licenses using the SKU name, which is slightly different to what we see on Entra
ID. To get the proper names, refer to this #link

The logic app is repeated using a recurrence trigger so you can set the monthly report to weekly or fortnightly etc.

For the permissions, we use a system-assigned identity for the logic app to allow it to view licenses in MS Graph.
