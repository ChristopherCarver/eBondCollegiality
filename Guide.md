# Multi-source eBonding *Draft*
---
[[TOC]]

---
# About

eBonding is a term used to describe the electronic passing of information for business-to-business (B2B) operations. Typically eBonding is used to keep two system of records in sync. A record on one system has the same information for a record on a different system. Each system passes information between each other as information on the record changes; hence both records contain the same information. Businesses setup and configure eBonding as a means to maintain a system of record they control automatically saving time and resources; i.e., billing, audit, automation, etc. etc.. 

Multi-source eBonding is when you have three or more systems coordinating together on the same record from multiple sources. Multi-source eBonding is a mature B2B model for multi-supplier strategies, where one company coordinates with different multiple suppliers that provide various services within an organization. The advantage of a multi-supplier model is the model provides an organization the ability to right-size their enterprise needs across many suppliers that render a variety of services. Versus letting a single supplier render all operational needs within an organization. 

This article is to help ServiceNow admins to setup multi-source eBonding framework for incident tickets. Incidents are normally a prime candidate to be shared across multiple suppliers. When an outage to a service is reported, there could be multiple suppliers working together to resolve the incident. Also if a supplier is utilized as an external call center, then routing incoming incident tickets to the appropriate secondary supplier needs to be performed. Given these two scenarios enterprises typically model, make incidents the first type of ticket to build a multi-source eBond framework. The same framework can be applied to other ticket types in ServiceNow with some minor adjustments.

---
## Progressive thoughts
\
Here are the progressive thoughts as multi-source eBonding is being developed to go back and refer too.
- You enterprise will eBond with suppliers, so therefore is makes sense to create company records in the core_company table for each supplier. 
- Company records need to be associated with the local ServiceNow accounts which are used by the supplier to eBond with ServiceNow. This will help you keep track what ServiceNow account is used by the supplier to make their RESTful API calls.
- eBond enabled companies only need to be associated with accounts that have the role of **eBond Account**. 
- eBonding is triggered based on the **Assignment Group** a record to assigned to. Therefore, group (sys_user_group) records need to associated with the companies that ServiceNow are eBonded with. Then in a business rule, if the assignment group company is eBonded, then continue to eBond operations. 
- Turning off/on eBonding is as simple as toggling the company record in the core_company table. 

---
## Follow along
\
Each section will include an update set for those that wish to install the multi-source eBonding on their instance and the instructions to create the update set. ServiceNow offers many ways to solve a problem or configure an operational model. These instructions are _a_ way and does not represent _the_ way on creating a multi-source eBonding framework. The best way to use these instructions is to read through them and see how the framework can be adopted and modified to fit your organization.  

!!! note Update sets 
    The instructions do not cover how-to use update sets or the nuisances of update sets. You can explore on how to leverage update sets within the online ServiceNow documentation.  

!!! note Terms & Meanings
    - _Supplier:_ The legal entity that ServiceNow connects too. You can substitute supplier for vendor.
    - _Company:_ A record from the core_company table in ServiceNow. 

!!! danger Modifying OOTB tables
    The article focuses on creating a robust framework tied closely to the out of the box (OOTB) tables already existing in ServiceNow. There are plenty of alternative implementation solutions at your disposal to accomplish the same result. Your mileage may vary depending on scope defined by and practices set by your organization.

---
# Foundation setup

The foundation setup is based on the premise that when an incident is created the *Service* and *Service Offering* within an incident are mandatory fields. This is a best practice to follow with a multi-supplier strategy for the enterprise. The reason is this removes the question "who supports this" and focuses on the true issue of "what is broken?" Who solves the incident is not of any concern to most customers and requiring customers or fulfillers to know what assignment group to assign a ticket in large enterprise environments is unreasonable and prone to misrouted tickets. 

**Service Offering** --> **Assignment group** --> **Company** --> **User account**

The *Service Offering* within an incident determines who the ticket is assigned to in the *Assignment group*. The *Assignment group* will be linked to a supplier (company record). This allows enterprises the ability to swap out suppliers across various services within ServiceNow quickly and easily reducing long term run and maintain costs.

!!! note Assignment groups
    Normally an *Assignment group* will contain all the members that support the *Service Offering** within an incident. In the eBond case it is normal if their are no members of an *Assignment group*. It is also normal to have members in an *Assignment group* that is eBonded with a supplier as well; where the enterprise allows the supplier to log into the enterprise ServiceNow to fulfill tickets. Either scenario still works with eBonding.   

The supplier will have a company record that is linked to a user record that represents the eBond account the supplier will use to make inbound RESTful calls. This will become a means to quickly identify what account a company uses for eBonding and creates a "lockout" point for security.

---
## Demo data
\
This how-to provides demo data that you can use to test the multi-source eBonding framework. The scenario is your organization has hired Alpha Co. and Beta Co. to supply IT services across your enterprise. Alpha Co. supplies data center operations and Beta Co. supplies backup and recovery operations for the data center. An incident occurred at the data center where both Alpha Co. and Beta Co. need to work together to resolve the incident. 

---
## Update set
\
*eBond Collegiality v1.0*

---
## Add eBond account role
\
The *eBond Account* role denotes which accounts are used by suppliers to eBond with your instance of ServiceNow. Each supplier should have their own local ServiceNow account that the supplier will use to connect to your instance of ServiceNow passing inbound REST payloads. This role will be used in various ways in the multi-source eBond framework. 

1. Navigate to **User Administration** > **Roles** and click **New**.
1. Under *Role New Record*, fill in the following fields:
    - _Name:_ eBond Account
    - _Description:_ Denotes the sys_user account is to be used for eBonding activities.
1. Click **Submit**, which will create the new role.

!!! note Demo Data - User accounts
    Setting up two accounts that will be used by two suppliers Alpha Co. and Beta Co. to make RESTful calls for eBonding.

    1. Navigate to **User Administration** > **Users**, and click **New**.
    1. Under the **User New Record** section, fill in the following field:
        - _User ID:_ eBondAlpha
        - _First name:_ Alpha Co
        - _Last name:_ eBond Account
    1. In the record header, right-click and select **Save**.
    1. In the **Roles** tab, click "Edit..."
    1. Under the **Collection** list, find the *eBond Account* role and add it **>** to the **Roles List**.
    1. Click **Save**.
    1. Click **New**.
    1. Under the **User New Record** section, fill in the following field:
        - _User ID:_ eBondBeta
        - _First name:_ Beta Co
        - _Last name:_ eBond Account
    1. In the record header, right-click and select **Save**.
    1. In the **Roles** tab, click "Edit..."
    1. Under the **Collection** list, find the *eBond Account* role and add it **>** to the **Roles List**.
    1. Click **Save**.

---
## Changes to core_company table
\
We need two new columns in the core_company table; *eBond account* and *eBonded*. The *eBond account* is a reference to sys_user table that links which account the company uses to eBond with your instance. The *eBonded* is a true or false field that we can use to turn off eBonding at the company level. 

1. Navigate to **System Definition** > **Tables** and open the *core_company* table.
1. Under the **Columns** tab, click **New** to create a new column.
1. Under the **Dictionary Entry New Record** section, fill in the following fields:
    - _Type:_ Reference
    - _Column label:_ eBond account
1. Under the **Reference Specification** tab, fill in the following fields:
    - _Table to reference:_ sys_user
    - _Reference qual condition:_
        - Active is true AND
        - Roles is eBond Account
1. Click **Submit**.
1. Under the **Columns** tab, click **New** to create a new column.
1. Under the **Dictionary Entry New Record** section, fill in the following fields:
    - _Type:_ True/False
    - _Column label:_ eBonded
1. Under the **Default value** tab, fill in the following field:
    - _Default value:_ false
1. Click **Submit**.

**OPTIONAL:** Modify the view on company records to see the *eBond account* and *eBonded* fields.

1. Navigate to **Organization** > **Companies** and click on any of the company records.
1. Navigate to **Additional actions** > **Configure** > **Form Layout**, which will bring up the **Configuring Company form**.
1. Under the **Available** list, find the *eBond account* and *eBonded* fields and add them **>** to the **Selected** list.
1. Move the *eBond account* and *eBonded* fields up the **Selected** list **^** to be below the *Stock price*.
1. Click **Save**.

**OPTIONAL:** Modify the view on company records to see the *eBond account* only if the *eBonded* field is true.
1. Navigate to **System UI** > **UI Polices** and click **New**.
1. Under the **UI Policy New record** section, fill in the following fields:
    - _Table:_ core_company
    - _Short description:_ Show eBond account in company
1. Under the **When to apply** tab, fill in the following fields:
    - _Conditions:_
        - eBonded is true
1. In the record header, right-click and select **Save**.
1. In the **UI Policy Actions**, click **New**.
1. Under the **UI Policy Action New record** section, fill in the following fields:
    - _Field name:_ eBond account
    - _Mandatory:_ True
    - _Visible:_ True
1. Click **Submit**.
1. Click **Update**.

!!! note Demo Data - Company records
    Setting up two company records for the suppliers Alpha Co. and Beta Co..

    1. Navigate to **Organization** > **Companies**, and click **New**.
    1. Under the **Company New Record** section, fill in the following field:
        - _Name:_ Alpha Co.
        - _eBonded:_ True
        - _eBond account:_ Alpha Co eBond Account
    1. Click **Submit**.
    1. Click **New**.
    1. Under the **Company New Record** section, fill in the following field:
        - _Name:_ Beta Co.
        - _eBonded:_ True
        - _eBond account:_ Beta Co eBond Account
    1. Click **Submit**.

---
## Associate groups with companies
\
FILL ME IN
1. Navigate to **System Definition** > **Tables** and open the *sys_usr_group* table.
1. Under the **Columns** tab, click **New** to create a new column.
1. Under the **Dictionary Entry New Record** section, fill in the following fields:
    - _Type:_ Reference
    - _Column label:_ Company
1. Under the **Reference Specification** tab, fill in the following fields:
    - _Table to reference:_ core_company
1. Click **Submit**.

**OPTIONAL:** Modify the view on group records to see the *Company* field.

1. Navigate to **User Administration** > **Groups** and click on any of the group records.
1. Navigate to **Additional actions** > **Configure** > **Form Layout**, which will bring up the **Configuring Group form**.
1. Under the **Available** list, find the *Company* field and add it **>** to the **Selected** list.
1. Move the *Company* field up the **Selected** list **^** to be below the *Name*.
1. Click **Save**.

!!! note Demo Data - Assignment Groups
    Setting up two support groups; *Data Center Operations Support* and *Backup and Recovery Support*. The reason the group names are not associated with the names of the suppliers is a run and maintain operation decision. Most of the time a supplier will support multiple *Service offerings*, when an organization desires to replace a supplier, instead of updating multiple *Service offerings*, only the support group *Company* field need to be updated. This is a minor detail as many organizations choose to name the assignment groups after the supplier and that is perfectly fine.

    1. Navigate to **User Administration** > **Groups**, and click **New**.
    1. Under the **Group New Record** section, fill in the following fields:
        - _Name:_ Data Center Operations Support
        - _Company:_ Alpha Co.
    1. Click **Submit**.
    1. Click **New**.
    1. Under the **Group New Record** section, fill in the following fields:
        - _Name:_ Backup and Recovery Support
        - _Company:_ Beta Co.
    1. Click **Submit**.

---
## Service, service offering, and assignment group alignment in incidents
\
Ticket mis-routing is problem in all services organization and surveys show that customer satisfaction reduces the more a ticket is re-routed. To reduce ticket mis-routes, the service, service offering, and assignment group need to be in alignment. It is unreasonable to expect that fulfillers know which support group supports what services. In large enterprises with hundreds of various services, internal teams, and external suppliers ticket mis-routing is common without alignment between all three fields. 

!!! note Optional setup
    This section is optional, but recommended for any ServiceNow environment; eBonded or not. For organizations that employ a multi-supplier strategy, this section is _highly_ recommended.   

1. Navigate to **System Definition** > **Dictionary**.
1. Under the list view search, fill in the the following search fields and hit enter:
    - _Table:_ task
    - _Column name:_ business_service
1. Open the *task* record.
1. Click on the **Reference Specification** tab, fill in the following field:
    - _Reference qual condition:_
         - Class is not Offering
1. Click **Update**.
1. Navigate to **System Definition** > **Business Rules** and click **New**.
1. Under the **Business Rule New Record** section, fill in the following fields:
    - _Name:_ Incident Auto-assignment Group
    - _Table:_ incident
    - _Advanced:_ True
1. Under the **When to run** tab, fill in the following fields:
    - _Insert:_ True
    - _Update:_ True
    - _Filter Conditions:_
        - Service changes OR
        - Service offering changes
1. Under the **Advanced** tab, fill in the following fields:
    - _Condition:_ gs.isInteractive()
    - _Script:_

        ```` 
        (function executeRule(current, previous /*null when async*/ ) {
        
             if (!gs.nil(current.service_offering.assignment_group))
                 current.assignment_group = current.service_offering.assignment_group;
            else
                 current.assignment_group = current.business_service.assignment_group;
        })(current, previous);
        ```` 
1. Click **Submit**.

!!! note Demo Data - Service and Service Offerings
    Setting up one service and two service offerings for Alpha Co. and Beta Co.
    
    1. Navigate to **Configuration** > **Services**, and click **New**.
    1. Under the **Service New Record** section, fill in the following fields:
        - _Name:_ IT Infrastructure
        - _Service classification:_ Technical Service
    1. In the record header, right-click and select **Save**.
    1. Under the **Offerings** tab, click **New**.
    1. Under the **Offering New Record** section, fill in the following fields:
        - _Name:_ Data Center Operations
        - _Support Group:_ Data Center Operations Support
    1. Click **Submit**.
    1. Under the **Offerings** tab, click **New**.
    1. Under the **Offering New Record** section, fill in the following fields:
        - _Name:_ Backup & Recovery
        - _Support Group:_ Backup and Recovery Support
    1. Click **Submit**.
    1. Click **Update**.

---
## Associate incidents with eBond companies
\
Fulfillers will need the ability to eBond incidents with suppliers. And we have partially started the setup that a trigger point will be the *Assignment group* in an incident. Now we will start to add custom fields to the *incident* table to enhance further functionality that will support and aid in multi-source eBonding.

!!! note Assignment group
    Later on we will configure how changing the *Assignment group* will affect the *eBonded with* field. 

There are two main ways to eBond an incident to a supplier; first via the *Assignment group* and the second via a custom field called *eBonded with*. The *eBonded with* will serve a dual purpose. First, it will represent all the suppliers the ticket is eBonded with in a list. A fulfiller will then have the ability to quickly see what suppliers the ticket is eBonded too. Second, it will toggle eBonding and deBonding operations when suppliers are added or removed from the list. That way a fulfiller can initiate an eBond or deBond with a supplier without having to change the *Assignment group."

!!! note eBonded with
    This field expects the fulfiller to know which suppliers to eBond with and the service offerings the supplier provides. If the fulfiller does not know the proper supplier, then they should use the *Service* and *Service offering* fields to set the *Assignment group* field to the appropriate group.

!!! note deBonding
    deBonding is the opposite of eBonding by breaking the connection between the tickets. After a ticket has been deBonded from a company, the ticket can be re-eBonded to the company by adding the company back to the list. In the event of a re-eBonding, there is no guarantee the company ticket will be the same . 

1. Navigate to **System Definition** > **Tables** and search for the *incident* table under the **Name** field.
1. Open the *Incident* record and click **New** under the **Columns** tab.
1. Under the **Dictionary Entry New Record** section, fill in the following fields:
    - _Type:_ List 
    - _Column label:_ eBonded with
    - _Column name:_ (this should default to u_eBonded_with)
1. Under the **Reference Specification** tab, fill in the following fields:
    - _Reference:_ core_company
    - _Reference qual condition:_
        - eBonded is true AND
        - eBond Account is not empty
1. Click **Submit**.
1. Navigate to **Incident** > **All** and open any incident record.
1. Navigate to **Additional actions** > **Configure** > **Form Layout**, which will bring up the **Configuring Company form**.
1. Under the **Available** list, find the *eBonded with* field and add it **>** to the **Selected** list.
1. Move the *eBonded with* field up the **Selected** list **^** to be below the *Assigned to*.
1. Click **Save**.

---
# Custom tables

eBonding will require the creation of custom tables or the extension of existing tables. ServiceNow sets a limit on the number of custom tables an instance can have before additional costs are accrued/incurred. This how-to will create custom tables as there are no similar tables in ServiceNow that are proper candidates to extend. You however can take any of the base tables in ServiceNow and extend them adding the custom fields and it will work the same.      

---
## eBond relationships
\
The *u_eBond_relationship* table is a multi-relational table that represents a one to many record mapping for an incident ticket to be mapped to multiple supplier tickets. 

The *u_eBond_relationship* table will contain the following custom fields:
|Field|Description|
|-----|-----------|
|Type|The type of eBond that was established with the vendor.|
|Table sys ID|The ServiceNow record sys_id on the instance.|
|Table name|The table the ServiceNow record resides on.|
|Status|The status of the eBond.|
|State|The state of the eBond.|
|Correlation Number|The supplier's ticket number or designation|
|Correlation ID|The supplier's ticket sys id or reference|
|Company|The ServiceNow supplier company record on the instance.|
|URL|Link to the supplier ticket.|
|Reflect|Changes made by the supplier are reflected in the ticket.|

The fields relate the ServiceNow ticket to the various suppliers the ticket is eBonded too.

1. Navigate to **System Definition** > **Tables**.
1. Click **New**.
1. Under the **Table New record** section, fill in the following fields:
    - _Label:_ Relationship
    - _Name:_ u_ebond_relationship
    - _New menu name:_ eBond
1. In the record header, right-click and select **Save**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Choice
    - _Column label:_ Type
    - _Column name:_ (this should default to u_type)
1. In the **Choice List Specification** tab, select *Dropdown with --None--*.
1. In the record header, right-click and select **Save**.
1. In the **Choices** tab, add the following records performing these steps:
    1. Click **New**.
    1. In the **Choice New record** section, fill in the field values listed below.
    1. Click **Submit**.
    |Sequence|Label|Value|
    |--------|-----|-----|
    |100|Mirror|mirror|
    |200|Information|information|
    |300|Transaction|transaction|
1. Click *Update**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Table Name
    - _Column label:_ Source table
    - _Column name:_ (this should default to u_source_table)
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Document ID
    - _Column label:_ Source
    - _Column name:_ (this should default to u_source)
1. Under **Related Links**, click **Advanced view**.
1. Under the **Dependent Field** tab, click **Use dependent field**.
1. In the **Dependent on field**, select **Source table**.
1. Click **Submit**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Choice
    - _Column label:_ Status
    - _Column name:_ (this should default to u_status)
1. In the **Choice List Specification** tab, select *Dropdown with --None--*.
1. In the record header, right-click and select **Save**.
1. In the **Choices** tab, add the following records performing these steps:
    1. Click **New**.
    1. In the **Choice New record** section, fill in the field values listed below.
    1. Click **Submit**.
    |Sequence|Label|Value|
    |--------|-----|-----|
    |100|Queued|queued|
    |200|Success|success|
    |300|Failure|failure|
1. Click *Update**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Choice
    - _Column label:_ State
    - _Column name:_ (this should default to u_state)
1. In the **Choice List Specification** tab, select *Dropdown with --None--*.
1. In the record header, right-click and select **Save**.
1. In the **Choices** tab, add the following records performing these steps:
    1. Click **New**.
    1. In the **Choice New record** section, fill in the field values listed below.
    1. Click **Submit**.
    |Sequence|Label|Value|
    |--------|-----|-----|
    |100|eBond|ebond|
    |200|eBonded|ebonded|
    |300|deBond|debond|
    |300|deBonded|debonded|
1. Click *Update**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Correlation Number
    - _Column name:_ (this should default to u_correlation_number)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Correlation ID
    - _Column name:_ (this should default to u_correlation_id)
    - _Max length:_ 32
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Reference
    - _Column label:_ Company
    - _Column name:_ (this should default to u_company)
1. In the **Reference Specification** tab, fill in the following field:
    - _Reference:_ Company
    - _Reference qual condition:_
        - eBonded is true AND
        - eBond Account is not empty
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ URL
    - _Column label:_ URL
    - _Column name:_ (this should default to u_url)
    - _Max length:_ 4000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ True/False
    - _Column label:_ Reflect
    - _Column name:_ (this should default to u_reflect)
1. Under the **Default Value** tab, , fill in the following field:
    - _Default value:_ false
1. Click **Submit**.
1. Click **Update**.

---
### Relating eBonded tickets
\
Associating the supplier tickets within the incident requires the creation of a relationship between the incident and u_ebond_relationship table.

1. Navigate to **System Definition** > **Relationships** and click **New**.
1. In the **Relationship New record** section, fill in the following fields:
    - _Name:_ eBond Relationships
    - _Applies to table:_ Global [global]
    - _Queries from table:_ Relationship [u_ebond_relationship]
    - _Query with:_

        ```` 
        current.addQuery('source_table', parent.getTableName());
        current.addQuery('source', parent.sys_id);
        ```` 
1. Click **Submit**.
1. Navigate to **Incident** > **All** and click on any of the incident records.
1. Navigate to **Additional actions** > **Configure** > **Related Lists**, which will bring up the **Configuring related lists on Incident form**.
1. Under the **Available** list, find the *eBond Relationships* relationship and add it **>** to the **Selected** list.
1. Click **Save**.

---
## eBond Properties
\
The *u_ebond_registry* table is similar to that of the *sys_properties* table in that it stores generic properties and values for eBonding. If you choose to use the *sys_properties* table, then make sure to look through the various scripts that reference the *u_ebond_registry* table in this how-to and change as appropriate.

The *u_ebond_registry* table will contain the following custom fields:
|Field|Description|
|-----|-----------|
|Supplier|"All" or the name of the supplier.|
|Key|The key variable name.|
|Value|The property variable value.|

1. Navigate to **System Definition** > **Tables**.
1. Click **New**.
1. Under the **Table New record** section, fill in the following fields:
    - _Label:_ Registry
    - _Name:_ u_ebond_registry
    - _Add module to menu:_ eBond
1. In the record header, right-click and select **Save**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Supplier
    - _Column name:_ (this should default to u_supplier)
    - _Max length:_ 40
1. Under the **Default Value** tab, fill in the following field:
    - _Default value:_ All
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Key
    - _Column name:_ (this should default to u_key)
    - _Max length:_ 4000
1. Click **Submit**.

1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Value
    - _Column name:_ (this should default to u_value)
    - _Max length:_ 4000
1. Click **Submit**.
1. Click **Update**.

---
## eBond Logs
\
The *u_ebond_log* table is similar to that of the *syslog* table in that it stores generic log information. If you choose to use the *syslog* table, then make sure to look through the various scripts that reference the *u_ebond_log* table. The *syslog* table has a granulairty of seconds and ServiceNow performs multiple operations per second. Hence the order of the logs messages become out of order. It is recommended that you use the *u_ebond_log* to help with debugging as the *Index* column is a sequential integer to aid in showing the order of operations.

!!! note Evaluator messages
The *u_ebond_log* table does not capture *Evaluator* messages from faulted JavaScript execution. Use the *syslog* table to check for failed JavaScript executions.

The *u_ebond_log* table will contain the following custom fields:
|Field|Description|
|-----|-----------|
|Index|Auto-incrementing numeric value.|
|Supplier|Name of the supplier.|
|Class|Name of the ServiceNow component, i.e., script include, business rule, workflow, etc. etc..|
|Source|Label of the ServiceNow component.|
|Message|The log message.|
|Level|Choice value of *High*, *Medium*, *Low*.|

1. Navigate to **System Definition** > **Tables**.
1. Click **New**.
1. Under the **Table New record** section, fill in the following fields:
    - _Label:_ Log
    - _Name:_ u_ebond_registry
    - _Add module to menu:_ eBond
1. In the record header, right-click and select **Save**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Integer
    - _Column label:_ Index
    - _Column name:_ (this should default to u_index)
    - _Read only:_ True
1. Under the **Default Value** tab, fill in the following field:
    - _Default value:_ javascript:getNextObjNumberPadded();
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Supplier
    - _Column name:_ (this should default to u_supplier)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Class
    - _Column name:_ (this should default to u_class)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Source
    - _Column name:_ (this should default to u_source)
    - _Max length:_ 80
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Message
    - _Column name:_ (this should default to u_message)
    - _Max length:_ 4000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Choice
    - _Column label:_ Level
    - _Column name:_ (this should default to u_level)
1. In the record header, right-click and select **Save**.
1. Under the **Choices** tab, add the following records performing these steps:
    1. Click **New**.
    1. In the **Choice New record** section, fill in the field values listed below.
    1. Click **Submit**.
    |Sequence|Label|Value|
    |--------|-----|-----|
    |100|High|high|
    |200|Medium|medium|
    |300|Low|low|
1. Click **Update**.
1. Click **Update**.

---
## eBond Data Map
\
The *u_ebond_data_map* table is a translation data table for inbound and outbound communication with suppliers.

The *u_ebond_data_map* table will contain the following custom fields:
|Field|Description|
|-----|-----------|
|Module|*All* or the name of the module.|
|Classification|Name of the classification.|
|Direction|*Inbound*, *Outbound*, or *Duplex*.|
|Supplier|*All* or the name of the supplier.|
|Source Value|The internal value for the instance.|
|Supplier Value|The supplier's value.|
|Note|Freeform to document various notes pertaining to the data map record.|

1. Navigate to **System Definition** > **Tables**.
1. Click **New**.
1. Under the **Table New record** section, fill in the following fields:
    - _Label:_ Data Map
    - _Name:_ u_ebond_data_map
    - _Add module to menu:_ eBond
1. In the record header, right-click and select **Save**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Module
    - _Column name:_ (this should default to u_module)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Classification
    - _Column name:_ (this should default to u_classification)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Choice
    - _Column label:_ Direction
    - _Column name:_ (this should default to u_direction)
    - _Max length:_ 40
1. In the record header, right-click and select **Save**.
1. Under the **Choices** tab, add the following records performing these steps:
    1. Click **New**.
    1. In the **Choice New record** section, fill in the field values listed below.
    1. Click **Submit**.
    |Sequence|Label|Value|
    |--------|-----|-----|
    |100|Inbound|inbound|
    |200|Outbound|outbound|
    |300|Duplex|duplex|
1. Click **Update**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Supplier
    - _Column name:_ (this should default to u_supplier)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Source value
    - _Column name:_ (this should default to u_source_value)
    - _Max length:_ 4000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Supplier value
    - _Column name:_ (this should default to u_supplier)
    - _Max length:_ 4000
1. Click **Submit**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Note
    - _Column name:_ (this should default to u_note)
    - _Max length:_ 4000
1. Click **Submit**.
1. Click **Update**.

---
## eBond REST Payloads
\
The *u_ebond_rest_payloads* table records all outbound REST API calls and their statuses. This table is also used to retry REST calls in the event of a failure; see **eBond Scheduled Job**. 

The *u_ebond_rest_payloads* table will contain the following custom fields:
|Field|Description|
|-----|-----------|
|Supplier|The name of the supplier.|
|Retry count|The number of retries performed executing the RESTful call to the supplier.|
|Retry|True or False.|
|REST Message|ServiceNow REST message label.|
|Payload|REST data payload for the supplier.|
|Format|The format of the payload; e.g., JSON, XML, SOAP, etc. etc..|
|Number|The source ticket number.|
|HTTP Status|HTTP return code.|
|HTTP Response|Entire HTTP response payload.|
|HTTP Method|ServiceNow REST method label.|
|Endpoint|The URL endpoint for the REST call.|
|Active|True or False|

1. Navigate to **System Definition** > **Tables**.
1. Click **New**.
1. Under the **Table New record** section, fill in the following fields:
    - _Label:_ REST Payload
    - _Name:_ u_ebond_rest_payload
    - _Add module to menu:_ eBond
1. In the record header, right-click and select **Save**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Supplier
    - _Column name:_ (this should default to u_supplier)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Integer
    - _Column label:_ Retry count
    - _Column name:_ (this should default to u_retry_count)
1. Under the **Default Value** tab, , fill in the following field:
    - _Default value:_ 0
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ True/False
    - _Column label:_ Retry
    - _Column name:_ (this should default to u_retry)
1. Under the **Default Value** tab, , fill in the following field:
    - _Default value:_ true
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ REST Message
    - _Column name:_ (this should default to u_rest_message)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Payload
    - _Column name:_ (this should default to u_payload)
    - _Max length:_ 4000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Format
    - _Column name:_ (this should default to u_format)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Number
    - _Column name:_ (this should default to u_number)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Integer
    - _Column label:_ HTTP Status
    - _Column name:_ (this should default to u_http_status)
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ HTTP Response
    - _Column name:_ (this should default to u_http_response)
    - _Max length:_ 4000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ HTTP Method
    - _Column name:_ (this should default to u_http_method)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Endpoint
    - _Column name:_ (this should default to u_endpoint)
    - _Max length:_ 80
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ True/False
    - _Column label:_ Active
    - _Column name:_ (this should default to u_active)
1. Under **Related Links**, click **Advanced view**.
1. Under the **Calculated Value** tab, fill in the following fields:
    - _Calculated:_ true
    - _Calculation:_

        ````
        (function calculatedFieldValue(current) {
            var calc = current.u_retry;

            switch (current.u_http_status.toString()) {
                //1×× Informational
                case '100': // Continue
                case '101': // Switching Protocols
                case '102': // Processing
                case '200': // OK
                case '201': // Created
                case '202': // Accepted
                case '203': // Non-authoritative Information
                case '204': // No Content
                case '205': // Reset Content
                case '206': // Partial Content
                case '207': // Multi-Status
                case '208': // Already Reported
                case '226': // IM Used
                case '300': // Multiple Choices
                case '301': // Moved Permanently
                case '302': // Found
                case '303': // See Other
                case '304': // Not Modified
                case '305': // Use Proxy
                case '307': // Temporary Redirect
                case '308': // Permanent Redirect
                case '400': // Bad Request
                    calc = false;
                    break;
                case '401': // Unauthorized
                case '402': // Payment Required
                case '403': // Forbidden
                case '404': // Not Found
                case '405': // Method Not Allowed
                case '406': // Not Acceptable
                case '407': // Proxy Authentication Required
                case '408': // Request Timeout
                case '409': // Conflict
                case '410': // Gone
                case '411': // Length Required
                case '412': // Precondition Failed
                case '413': // Payload Too Large
                case '414': // Request-URI Too Long
                case '415': // Unsupported Media Type
                case '416': // Requested Range Not Satisfiable
                case '417': // Expectation Failed
                case '418': // I'm a teapot
                case '421': // Misdirected Request
                case '422': // Unprocessable Entity
                case '423': // Locked
                case '424': // Failed Dependency
                case '426': // Upgrade Required
                case '428': // Precondition Required
                case '429': // Too Many Requests
                case '431': // Request Header Fields Too Large
                case '444': // Connection Closed Without Response
                case '451': // Unavailable For Legal Reasons
                case '499': // Client Closed Request
                case '500': // Internal Server Error
                case '501': // Not Implemented
                case '502': // Bad Gateway
                case '503': // Service Unavailable
                case '504': // Gateway Timeout
                    break;
                case '505': // HTTP Version Not Supported
                case '506': // Variant Also Negotiates
                case '507': // Insufficient Storage
                case '508': // Loop Detected
                case '510': // Not Extended
                case '511': // Network Authentication Required
                case '599': // Network Connect Timeout Error
                    calc = false;
                    break;
                default:
                    break;
            }

            // after 48 hours of retrying, turn off active flag
            if (current.u_retry_count >= 48) {
                calc = false;
            }

            return calc; // return the calculated value
        })(current);
        ````
1. Click **Submit**.
1. Click **Update**.

---
# Multi-source Inbound Handling

Inbound incident handling could be setup in an agnostic fashion based on a mutual push-push model setup between the enterprise and suppliers. The mutual push-push model is where the each party sets how they shall receive messages and no party performs a pull/get operation. There are major advantages for this model over one party dictating both how it will send and receive messages. 

!!! note OEM & OSP
    Original equipment manufacturers (OEM) and original service providers (OSP) rarely fit into the mutual push-push model as their clientele is vast and the reliance of the enterprise on the OEMs and OSPs do not warrant a collaborative effort between the enterprise ServiceNow developers and the OEM/OSP. In the case of true partnership between enterprise and suppliers should allow for a mutual push-push model.    

The biggest advantage to all parties is long term run and maintain of eBonding operations. If one party changes their fields or data, the other party is not forced into changing their eBonding framework to accommodate; a decision from one party should not impact or cause work for the other. Another advantage is resource utilization. Pull requests are expensive and if an enterprise is being pulled by dozens of suppliers, this places an undue burden on the enterprise. It is far more agreeable that if a change is to occur in one party's record, then that party will pass (push) the delta change to the agreed parties.

---
## Quick Rundown
\
Suppliers will write to a staging table called *[u_ebond_incident_staging]*. The table will be an extension of the *[sys_import_set_row]* and will have the ability to take advantage of the OOTB ServiceNow transform functionality. The *transform maps* will validate the incoming data and then create or update the appropriate records within the system.

---
## Incident staging table
\
Normally the REST inbound message handling aspect for eBonding is developed using *Scripted REST API*. For multi-source eBonding a different model will be developed using a staging table. Suppliers will directly write to a staging table instead of a Scripted REST API.

ServiceNow *Transform Maps* provide an elegant and efficient means to do much of what a *Scripted REST API* can do with the use of *Transform Scripts*. The **onStart** is used for preparation of global variables, *onBefore* is used for data validation, and *onAfter* for data operations. All of this efficiently designed by Servicenow in the import set framework to handle inbound REST messages.

!!! note Staging other tables
    For each type of destination ServiceNow table for multi-source eBonding will require a unique staging table. For example, if the requirement is to have a multi-source eBond with change requests, a similar staging table that that below will need to be created.

The *u_ebond_incident_staging* table holds the field values passed by the supplier for incidents.

The *u_ebond_incident_staging* table will contain the following custom fields:
|Field|Description|
|-----|-----------|
|Work Note|Support work note, customer not visible.|
|Subcategory|Subcategory for the incident.|
|State|The state of the incident.|
|Short Description|The title (or short description) of the incident.|
|Service Offering|The service offering the incident is impacting.|
|Service|The service the incident is impacting.|
|Priority|The priority level of the incident.|
|Sys ID|The sys_id for the local incident ticket.|
|Number|The number for the local incident ticket.|
|Hold Reason|The reason the incident ticket was placed on hold.|
|Description|The description of the incident.|
|Contact Type|How the incident was reported.|
|Comment|Customer visible comment.|
|External Reference|The supplier's incident ticket unique identifier.|
|External Number|The supplier's incident ticket number.|
|CMDB CI|The configuration item the incident is reported against.|
|Close Notes|Incident closure notes.|
|Close Code|Incident closure code.|
|Category|Category for the incident|
|Caller|The caller who reported the incident.|
|Assignment Group|The assignment group working the incident.|

1. Navigate to **System Definition** > **Tables**.
1. Click **New**.
1. Under the **Table New record** section, fill in the following fields:
    - _Label:_ Incident Staging
    - _Name:_ u_ebond_incident_staging
    - _Extends table:_ Import Set Row [sys_import_set_row]
    - _Add module to menu:_ eBond
1. In the record header, right-click and select **Save**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Work Note
    - _Column name:_ (this should default to u_work_note)
    - _Max length:_ 4000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String    
    - _Column label:_ Subcategory
    - _Column name:_ (this should default to u_subcategory)
    - _Max length:_ 80
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String   
    - _Column label:_ State
    - _Column name:_ (this should default to u_state)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String    
    - _Column label:_ Short Description
    - _Column name:_ (this should default to u_short_description)
    - _Max length:_ 1000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Service Offering
    - _Column name:_ (this should default to u_service_offering)
    - _Max length:_ 255
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Service
    - _Column name:_ (this should default to u_service)
    - _Max length:_ 255
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Priority
    - _Column name:_ (this should default to u_priority)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Sys ID
    - _Column name:_ (this should default to u_sys_id)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Number
    - _Column name:_ (this should default to u_number)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Hold Reason
    - _Column name:_ (this should default to u_hold_reason)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Description
    - _Column name:_ (this should default to u_description)
    - _Max length:_ 4000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Contact Type
    - _Column name:_ (this should default to u_contact_type)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Comment
    - _Column name:_ (this should default to u_comment)
    - _Max length:_ 4000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ External Reference
    - _Column name:_ (this should default to u_external_reference)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ External Number
    - _Column name:_ (this should default to u_external_number)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ CMDB CI
    - _Column name:_ (this should default to u_cmdb_ci)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Close Notes
    - _Column name:_ (this should default to u_close_notes)
    - _Max length:_ 1000
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Close Code
    - _Column name:_ (this should default to u_close_code)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Category
    - _Column name:_ (this should default to u_category)
    - _Max length:_ 80
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Caller
    - _Column name:_ (this should default to u_caller)
    - _Max length:_ 40
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Assignment Group
    - _Column name:_ (this should default to u_assignment_group)
    - _Max length:_ 40
1. Click **Submit**.
1. Click **Update**.

---
## Inbound Transform
\
The inbound *transform map* uses *transform scripts* to perform various operations on the data directly passed by the supplier. The *transform scripts* can be set to *when* they are executed against the data in the transform. 

- **onStart:** Sets up the global variables used by the other transform scripts. Global variables will be capitalized for ease of development. 
- **onBefore:** Validates security and the data passed by the supplier. The order of scripts are important as pre-checks must complete before continuing the transform.  
- **on:After** Creates or updates the needed fields within the records. 
- **on:Complete** Verifies all operations and updates return information to the supplier. 

1. Navigate to **System Import Sets** > **Transform Maps**.
1. Click **New**.
1. In the **Table Transform Map New record** section, fill in the following fields:
    - _Name:_ eBond Incident Transform
    - _Source table:_ Incident Staging [u_ebond_incident_staging]
    - _Target table:_ Relationship [u_ebond_relationship]
1. In the record header, right-click and select **Save**.
1. In the **Field Maps** tab, click **New**.
1. In the **Field Map New record** section, fill in the following fields:
    - _Source field:_ External Number [u_external_number]
    - _Target field:_ Correlation Number [u_correlation_number]
    - _Coalesce:_ True
1. Click **Submit**.

    !!! note Coalesce field not indexed
        If a pop-up box appears titled *Coalesce field not indexed*, then click **OK**, fill in the followup contact information and click **OK**.

1. In the **Field Maps** tab, click **New**.
1. In the **Field Map New record** section, fill in the following fields:
    - _Source field:_ External reference [u_external_reference]
    - _Target field:_ Correlation ID [u_correlation_id]
    - _Coalesce:_ false
1. Click **Submit**.
1. In the **Field Maps** tab, click **New**.
1. In the **Field Map New record** section, click **Use source script** and fill in the following fields:
    - _Choice action:_ ignore
    - _Target field:_ company [core_Company]
    - _Coalesce field:_ true
    - _Script:_ 

        ````
        answer = (function transformEntry(source) {

            // coalesce script is called at various times in the transform
            // ignore the time(s) when the row is not even being evaluated
            if (source.sys_created_by == "") {
                return;
            }

    	    // find the company the account is associated with
            var company = new GlideRecord('core_company');
            company.addQuery('u_ebond_account', source.sys_created_by);
            company.query();
            if (company.next()) {
    		    // make sure the supplier is still eBond approved
                if (company.u_ebonded) {
                    return company;
                }

                // supplier is not eBond enabled
                var log = new GlideRecord('u_ebond_log');
                log.u_class = 'eBond Incident Transform map';
                log.u_source = '[Field Map] Company';
                log.u_supplier = company.name;
                log.u_message = 'Security Violation: The supplier ' + company.name + ' is not eBond enabled. Reference: ' + source.sys_id;
                log.u_level = 'Low';
                log.insert();

                error = true;
                error_message = "Security violation.";
                return "";
            }

            // creator is not associated with a supplier
            var log = new GlideRecord('u_ebond_log');
            log.u_class = 'eBond Incident Transform map';
            log.u_source = '[Field Map] Company';
            log.u_supplier = 'Unknown';
            log.u_message = 'Security Violation: An eBond account (' + source.sys_created_by + ') has written to the u_ebond_incident_staging table with no reference to a record entry in the core_company table. Reference: ' + source.sys_id;
            log.u_level = 'High';
            log.insert();

            error = true;
            error_message = "Security violation.";

            return ""; // return the value to be put into the target field

        })(source);
        ````
1. Click **Submit**.

    !!! note Coalesce fields
        Only the *target field* fields *u_correlation_number* and *u_company* fields are coalesce fields. The *u_company* is derived internally from the *sys_created_by* field that holds the account that wrote the record. This removes the means to supplant company information. The *u_correlation_number* by itself is not sufficient to make it unique enough with multiple eBonded suppliers. The combination of the *u_correlation_number* and *u_company* is required; as the assumption is the supplier will not duplicate their ticket numbers. However, two or more suppliers could have the same ticket numbers unrelated to each other.

1. In the **Transform Scripts** tab, click **New**.
1. In the **Transform Script New record** section, fill in the following fields:
    - _When:_ onStart
    - _Script:_

        ````
        // Initialize global variables
        //                       
        var LOGING = false;
        var COMPONENT = "Transform Map";
        var SOURCE = "eBond Incident Transform";
        ````
1. Click **Submit**.

---
# Multi-source Outbound Handling

