# Summary
This is the default template for a webhook that registers an email recipient to an event with one click. A unique URL is sent to each email recipient with a `Yes` and a `No` button, and their unique `Contact_GUID` and the `Event_ID` are included in the link. The webhook takes the parameters from the URL and sends them to Power Automate (or any webhook destination) on page load. Power Automate then creates the event participant record.

The purpose of this webhook is to create a simple, one-click event registration. This can be useful for events where you simply need an RSVP/headcount so you can order snacks or food.

# Prerequisites
- A basic understanding of HTML and JavaScript is necessary. You shouldn't put any code on your website that you don't understand.
- You need something to receive the webhook. I use Power Automate to update my MinistryPlatform SQL database, and I'll briefly explain the Power Automate flow, but this codespace focuses on the webhook.

# Detailed Steps
There are three parts to this process: 
1. An email gets sent to a contact that includes special URL parameters.
2. A webhook on a landing page reads those parameters and sends them to Power Automate.
3. Power Automate looks up the appropriate info in the database and creates a new event participant record.

## Step 1. Send the User an Email
In addition to the body of the email and details/invitation for the event, your email should include two links—one if the user is RSVPing yes and one if the user is declining the invitation (why is it important to decline the invitation? So you can see who has declined, just in case you want to send a follow-up invitation to everyone who hasn't responded). I like to have two different landing pages to make it easier to put a message like "Thank you for your RSVP" or "Sorry you can't make it!" on the landing page. So, for example, your pages to RSVP with the webhook might be `https://example.com/rsvp-yes` to register 'YES' and `https://example.com/rsvp-no` to register 'NO'.

The links are where the magic happens. Since `Contact_GUID` is a really easy variable to put into an MP email, I prefer to use that to identify the person. Now there's not a great way (at least that I've found) to add the `Event_ID` as a variable to the email, so you'll need to hardcode it in the email. Let's imagine the event we want to use the registration for is `Event_ID 1234`. So in our MP email template, we want to use this as the link to pass the `Contact_GUID` and `Event_ID` to the register yes webhook: `https://example.com/rsvp-yes/?event_id=1234&contact_guid=[Contact_GUID]`. And as you probably guessed, we'll use this for no: `https://example.com/rsvp-no/?event_id=1234&contact_guid=[Contact_GUID]`.

> **Please Note:** We're not using `Contact_ID`! We're using ***GU**ID*. Personally, my preference is to always use GUID for Contacts, but it's more of a preference than having a good reason.

> **Also Note:** The links in the email will only and always work for the Contact they were sent to, so be aware of that if people try to forward the invitation.

## Step 2. Configure the Landing Page
When someone clicks the link to the `/rsvp-yes` page, the link they click will include the `Event_ID` and the `Contact_GUID` as URL parameters. Copy the code from the Basic Webhook file. You'll add this to your webpage as either raw HTML or JavaScript. This JavaScript reads the URL parameters from the link in the email and includes them in a webhook.

This code includes an `<h2>` block that is updated by the JavaScript. It gives a wait message until the webhook completes or throws an error. The content of the confirmation and error messages can be edited in the variables. You can use your own element with the `id="user-message"` or this can be removed completely. 

The webhook code has plenty of comments in it to explain what is going on, but the following is a *brief* overview.

### H2 User Message
This H2 is all that the user will see. The JavaScript replaces the content of the H2 with the variables at the beginning of the script. This entire beginning section can be removed if you don't want to use the user facing message. Alternatively, you can use your own HTML element as long as it has the `id="user-message"` property. 

> **Note:** If you use your own HTML element, add it here or at least remove the one from this code. If you don't, the script will just find the first match with the ID and modify that one.

### Variables
The first few sections of the script set variables. The first three are designed to be modified by YOU.
#### Webhook URL
This code begins with setting the webhook destination. You ***MUST*** replace the placeholder URL in the supplied code with your webhook URL. 
#### Message Variables
The `user_confirmation` variable is the message that will replace the content of the above `<h2>` upon successful completion of the webhook. The `user_error` message displays if there is an error (or in the console if you opt not to have an element with the `id="user-message"`).
#### URL Params
The next variables are the URL parameters, in this case `event_id` and `contact_guid`. Note that the `event_id` is converted to an integer because that is the format that MP expects to receive it.

#### User-Message Function
This function finds the element with `id="user-message"` and saves that in a variable. It then checks to see if the element exists and if not, it logs the message error messages to the console instead of modifying the HTML element.

#### Check for required URL Parameters
This section checks to see that the required parameters are supplied in the URL. This keeps you from a bunch of flows running until they time out in Power Automate.

> **REMINDER:** You need to replace the placeholder URL variable with your actual URL for your webhook.

### Send the Webhook
The next bit of the script sends the webhook. First the content of the webhook is logged to the console. This is just to help you see what is being sent. Next, the webhook is sent using `fetch`. The destination URL is pulled in from the variable you set! Don't forget to do that! Then the body of the webhook is constructed from the URL param variables created above. This is the payload that gets sent to Power Automate.

### Error Handling
This is simple, rudimentary error handling. First, we simply log the response from Power Automate. In general, Power Automate will return a 202 response if the data was accepted. If it returns an error (e.g. 4xx or 5xx errors), the error message is displayed in the `<h2>` and more details logged to the console.

### Success
If an error isn't thrown, then the script replaces the user message with the confirmation variable and a success message is logged to the console.

### Catch
The catch returns the error message if there are any other errors.

## Step 3. Configure Power Automate
In MP, to "register" for an event, we need an Event Participant (EP) record, a registration status (02 for registered), and an Event ID. This webhook sends the `Event_ID` and the `Contact_GUID`. Since we're starting with 2 of the three pieces of information we need, we need to leverage Power Automate (PA) to find the `Participant_ID` and create the new record.

In Power Automate, the trigger will be when an HTTPS request is received. I like to set the fields we'll be referencing a lot as variables, so I'm going to initialize `Contact_GUID`, `Participant_ID`, and `Event_ID` all as variables and set the `Event_ID` and `Contact_GUID` with the values received in the webhook.

Next, use the `Contact_GUID` to pull up the Contact Record and then parse the `Participant_Record` (yes, on the Contacts table, `Participant_Record` is the same as `Participant_ID` used everywhere else). Use the `Participant_Record` to set the `Participant_ID` variable.

Finally, create an Event Participant Record with the Event ID. Since I often forget how to do this, here's the code to create the new Event Participant in Power Automate. This would be the Create Record with the MP Connector:
```
[
  {
    "event_id": @{variables('event_id')},
    "participant_id": @{variables('Participant_ID')},
    "Participation_Status_ID": 2
  }
]
```

### Power Automate Overview
Receive Webhook --> Initialize Variables --> Lookup Contact (to find Participant_ID) --> Create Event Participant Record

### Additional Power Automate Considerations
- Another step that can be helpful is to look and see if a registration already exists. That way you don't end up with a bunch of duplicate registrations.
- You could also send an email to a staff member every time someone registers. While I personally don't think this is scalable or preferable, some staff people prefer it.

## Additional Notes/Other Examples
- You could in theory have a single landing page and use a URL parameter for `yes` or `no`. In that case, you'd use a condition to send the webhook to either the yes or the no PA flow ***OR*** you could use the URL parameter in PA to specify if you're going to create an `02 Registered` or an `05 Canceled` Event Participant.
- I have done RSVPs in the past that include all heads in the household. This is useful in the event that spouses are invited and you're trying to get a headcount. In this case, I present the recipient with three options, `Yes (Just Me)`, `Yes (and my spouse)`, or `No`. If they pick household, that goes to a special page that triggers a PA flow that looks up each head in the household of the `Contact_GUID` and creates an Event Participant for each of them.
- Add some kind of alert if the webhook isn't successful.
- Other Examples:
  - Updating a record (using the table ID number)
  - Joining a group (using Group_ID and Participant_ID)
