# Summary
This guide provides details for Power Automate. It assumes you already have the Ministry Platform Power Automate Connector set up. For more information about setting up the connector for the first time, see [MP Help Center](https://help.acst.com/en/ministryplatform/help-topics/platform/customizations/extending-the-platform/power-automate).

This webhook starts with `Contact_GUID` and `Event_ID`. The goal of the Power Automate Flow is to create an Event Participant with a registered status (which means we'll need to find the `Participant_ID`).

To get started, create a Power Automate Cloud Flow and use the trigger "When an HTTP request is received" in the Request category. Usually, there is a page to select the trigger and name the flow when you click "Create new flow." If you can't find the HTTP request trigger, click "Skip" and add it later in the flow.

See below for details on how each step is configured in the flow:

# Power Automate Configuration
This flow has several main sections, each with outlined steps:
1. [Trigger](#trigger)
2. [Set Variables](#set-variables)
3. [Query MP for Participant_ID](#query-mp-for-participant_id)
4. [Query MP for Existing Event Registrations](#query-mp-for-existing-event-registrations)
5. [Update or Create Event Participant Records](#update-or-create-event-participant-records)

## Trigger
### 1. When an HTTP Request Is Received
There are three important fields in the trigger: 
1. Set "*Who can trigger the flow?*" to `Anyone`. Without this, the flow won't accept a webhook from your website.
2. The "*HTTP URL*" will populate after you save the flow for the first time. Copy this URL into your webhook variables.
3. "*Request Body JSON Schema*" should not be left blank. Adding the schema allows you to reference individual fields in the flow. Otherwise, you'll need an extra step to parse the JSON. Use the following schema:

```json
{
    "type": "object",
    "properties": {
        "contact_guid": {
            "type": "string"
        },
        "Event_ID": {
            "type": "integer"
        }
    }
}
```

> **NOTE:** This trigger may rename itself to *manual*. This seems to happen with the new web editor. Let it stay named *manual* and proceed.

### 2. Print the Webhook Body
This step is optional but helpful for debugging. Add a "*Compose*" step and name it "Print Webhook Body." Use descriptive titles for your steps, as this is the only place to add comments. Set the input to `Body` from the previous step.

## Set Variables
Initialize all variables at the beginning of the flow for better readability and management. Using variables instead of directly referencing the webhook step makes them easier to find later in the flow.

### Initialize and Set Contact_GUID
Add the "*Initialize variable*" step to the flow. Name it "Initialize and Set Contact_GUID."
- **Name:** `Contact_GUID`
- **Type:** `String` (the webhook sends a string, and the MP field is also a string)
- **Value:** `contact_guid` (click the lightning bolt icon select this from the first trigger step, don't type it manually)

### Initialize and Set Event_ID
This step is similar to the previous one.

Add the "*Initialize variable*" step to the flow. Name it "Initialize and Set Event_ID."
- **Name:** `Event_ID`
- **Type:** `Integer` (the webhook sends an integer, and the MP field is also an integer)
- **Value:** `event_id` (select this from the first trigger step, don't type it manually)

### Initialize Participant_ID
This step is slightly different, as the value is left blank.

Add the "*Initialize variable*" step to the flow. Name it "Initialize Participant_ID."
- **Name:** `Participant_ID`
- **Type:** `Integer`
- **Value:** Leave this blank, as we don't have a `Participant_ID` yet.

## Query MP for Participant_ID
These next steps lookup the contact record in MP to find and set the Particiapnt_ID variable in our flow.
### Get Contact Record
Finally we get to add a step from our MP Power Automate connector! Add the "TABLE Get Collection" step from the MinistryPlatform connector in the custom category. Name the step "Get_Contact_Record".  Here are the details:
- **Table:** `Contacts`
- **$filter:** `[Contact_GUID] = '@{variables('Contact_GUID')}'`
    1. **Column Name:** Use the column name wrapped in brackets `[ ]`.
    2. **Operator:** Use the equals operator `=` in this case, but other SQL comparisons like `>=` or `BETWEEN` can also be used.
    3. **Value:** The value being searched. Since this is a string, enclose it in single quotes `' '`. If it were an integer, single quotes would not be necessary.
    - Don't worry - you don't need to write out all of `@{variables('Contact_GUID')}` part. Simply type in `[Contact_GUID] = ''`, put your cursor between the single quotes, and use the menu to put in the Contact_GUID variables.

> #### [Swagger UI](#swagger-ui)
> A great way to test your filter content without running the flow a thousand times over and over is to use the Swagger UI provided by MP. To test this step, go to the Tables section of the Swagger page and open `Get /tables/{table}`. Fill out the `table` and `$filter` fields with the content that you expect to be used in this step. For example, you'd want to look up your `Contact_GUID` in MP and confirm you're seeing your contact record in the result. For more info, see the MP [Swagger UI](https://help.acst.com/en/ministryplatform/developer-resources/developer-resources/swagger-ui) help page.
Add the "Parse JSON" step to your flow. We'll use this step to take the JSON output from the previous step to turn the results into use data pieces. Name this step "Parse Contact_GUID Record". 
- **Content:** `@{first(body('Get_Contact_Record'))}` - *I know that looks like a lot, so let's break it down.*
    - In the **Content** field, click the fx icon instead of the lightning bolt. This allows you to insert a function and manipulate the output of a previous step.
    - The previous step returns an array, but since `Contact_GUID` values are unique, there will only be one result.
    - To extract this single record, use the `first()` function. After clicking the fx icon, you can access a menu of functions and data from previous steps. Start typing `first` and select it from the menu.
    - Inside the parentheses of `first()`, click the Dynamic Content tab and insert the body output of the previous step (`Get_Contact_Record`). This ensures that only the first record from the array is used.
        > Without `first()`, the flow would automatically create a "For Each" loop, as arrays are treated as collections of objects.
- **Schema:** I have supplied the schema below. We really only care about the Participant_Record field, so I've removed all others. Note also that there are no `[ ]` around this code to indicate that we're looking at a single object instead of an array. This is because we've selected the first item in the array in the content field above.
```json
{
    "type": "object",
    "properties": {
        "Participant_Record": {
            "type": "integer"
        }
    }
}
```
> #### **Schema notes:** 
>    - Use the "Use sample payload..." link below the schema section to generate the schema. You can obtain the JSON response using [Swagger UI](#swagger-ui) and paste it into the sample area to create the schema.
>    - Alternatively, you can save and test the flow before adding the parse step, then use the output from the test as the sample. This method may take more time.
>    - Regardless of how you generate the sample, ensure you remove the `[ ]` from the outside of the sample if you are using the `first()` function (as in this case). Also, delete any unnecessary fields, keeping only what is necessary for the flow (the `Participant_ID`) in this case. Extra fields can complicate the output and increase the chances of flow failure if the data is missing or not in the expected format.

### Set Participant_ID Variable
The previous parse step makes the `Participant_Record` field on the contact record available for use in the flow. Now we will set the `Participant_ID` variable to the value of the `Particiapnt_Record` from the contact.

> **Note:** For some reason, `Participant_ID` is named `Participant_Record` on the contact record. Whenever you're accessing the contact record, use `Participant_Record`. Otherwise, use `Participant_ID`.

Add the step "Update Variable" to your flow. 
- **Name:** Select `Participant_ID`
- **Value:** Select `Body Participant_Record` from the previous step

## Query MP for Existing Event Registrations
With the Event_ID and Participant_ID variables set, we can now check for existing event registrations. While it is possible to register a person for the event each time they go through this flow, doing so could result in duplicate or conflicting records.

So before we move to create a registration, we're going to check to see if this Participant is already associated with this event.

### Get Existing Event Registrations
Add the "TABLE Get Collection" step from the MinistryPlatform connector in the custom category. Name the step "Get_Existing_Event_Registrations." Here are the details:
- **Table:** `Event_Participants`
- **$filter:** `[Event_ID] = @{variables('Event_ID')} AND [Participant_ID] = @{variables('Participant_ID')}`
    - **Column Names:** Use the column names wrapped in brackets `[ ]`.
    - **Operator:** Use the equals operator `=` for both conditions because we're looking for exact matches.
    - **Logical Operator:** Use `AND` to combine the conditions because we're looking for records where both of these are true.
    - **Values:** Use the `Event_ID` and `Participant_ID` variables. These can be selected from the dynamic content menu (*i.e.* click the lightning bolt).

## Update or Create Event Participant Records
This final section of the flow checks whether an existing Event Participant exists. If it does, the flow updates the existing record(s); otherwise, it creates a new record.

### Condition: Check for Existing Event Registrations
Add a "Condition" step to your flow to evaluate whether any records were returned from the previous step. Name this step "Does an event participant already exist?".
> **Note:** When you create a condition, always use a question-like name. That way, when you read the condition name, it is clear what True or False mean!

Use the following configuration for the condition:
- **Condition:** 
    - The condition consists of three parts: (1) a value to evaluate, (2) a comparison type, and (3) another value.
    - Use the `length()` function to count the number of items in the array returned by the body of the parse step. To do this, add the `length()` function, put your cursor in the parenthesis and add the Dynmic Content of the body of the `Get Existing Event Registrations` step. So altogether, the function will read: `length(body('Get_Existing_Event_Registrations'))`.
    - Set the comparison type to "greater than."
    - Set the comparison value to `0`.
    - This configuration checks if the array contains more than 0 results, indicating that an event participant already exists.
    - **True Path:** Proceed to update the existing event participants.
    - **False Path:** Create a new event Participant record

| | *True Path Steps*  | False Path Step |
|-|---------|-------|
| Step 1: | *Parse All Event Records* | Create Event Participant Registration|
| Step 2: | *Update Event Registration(s) to Registered* ||

### True Path Step 1: Parse Each Existing Event Registration Record
Add the "Parse JSON" step to your flow. We'll use this step to process the JSON output from the previous step. Name this step "Parse Existing Event Registration Records."
- **Content:** `@{body('Get_Existing_Event_Registrations')}`
    - Select `Body` from the Get Existing Event Registrations step
    - We don't need to use `first()` this time since we're going to updated every existing record.
- **Schema:** Included below, but remember [Schema Notes](#schema-notes) above.
    - Note this time we're including the entire array since we wand to update *every* existing record.
    - We're including these 4 fields because we know they will always exist. 
        - `Event_Participant_ID` is the Key/ID record for this table. So this will be used in the next step to update the record.
        - `Participation_Status_ID` is included because we're going to set it to `2` (registered). If we include it here, we can more easily see what it was set to before we updated it (although you could obviously see this in the MP audit log, so maybe this is superfluous). 

```json
{
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "Event_Participant_ID": {
                "type": "integer"
            },
            "Event_ID": {
                "type": "integer"
            },
            "Participant_ID": {
                "type": "integer"
            },
            "Participation_Status_ID": {
                "type": "integer"
            }
        }
    }
}
```

### True Path Step 2: Update Existing Event Participant Records
Add the `TABLE Update Record` step from the MP custom connector to your flow. Name this step "Register Each Event Participant Record". 
- **Table:** `Event_Participants`
- **Content-Type:** `application/json`
- **Accept:** `application/json`
- **Body:** 
```json
[
  {
    "Event_Participant_ID": [Insert Body Event_Participant_ID from the previous step],
    "Participation_Status_ID": 2
  }
]
```
![Schema Screenshot](<Screenshots/Update EP Body Screenshot.png>)

When you select the Event_Participant_ID from the previous parse step, Power Automate will automatically create a "For Each" loop. This happens because the output being used from parse step is from an array of records, so the loop runs *for each* object in the array.

The "For Each" loop will iterate through each record and update its status to "Registered" (Participation_Status_ID = 2).

### False Path Step: Create Event Participant
When the answer is false to "*Are there exising registrations for this participant?*", we'll need to create a new one.

Add the `TABLE Create Record` step from the MP custom connector to your flow. Name this step "Create Event Participant Registration".
- **Table:** `Event_Participants`
- **Content-Type:** `application/json`
- **Accept:** `application/json`
- **Body:** 
```json
[
  {
    "Event_ID": @{variables('Event_ID')},
    "participant_id": @{variables('Participant_ID')},
    "Participation_Status_ID": 2
  }
]
```
![Create EP Screenshot](<Screenshots/Create EP Body Screenshot.png>)

This step creates a new Event Participant record with the following details:
- `Event_ID`: The event for which the participant is being registered.
- `Participant_ID`: The unique identifier of the participant.
- `Participation_Status_ID`: Set to `2`, which corresponds to the "Registered" status.

Once this step is executed, the participant will be successfully registered for the event.
