# Gym Management & Member Engagement Automation

An n8n-based workflow automation system designed to streamline gym member registration, membership record management, welcome communication, and subscription-expiry follow-up.

The workflow connects a registration form with Airtable and Gmail, while also providing automated monitoring and administrator notifications for memberships approaching expiration.

---

## Project Overview

Managing gym memberships manually can involve repetitive activities such as:

- Collecting member information
- Creating membership records
- Calculating subscription expiry dates
- Sending welcome emails
- Checking memberships approaching expiration
- Contacting members for renewal
- Notifying administrators about upcoming expirations

This automation reduces these repetitive activities by connecting the registration process, member database, email communication, and scheduled membership monitoring into automated workflows.

---

## Business Problem

Without automation, gym staff may need to manually:

1. Review new member registrations
2. Enter member information into a database
3. Calculate membership end dates
4. Send welcome emails
5. Regularly check membership records
6. Identify members whose subscriptions are about to expire
7. Contact members about renewal
8. Payment processing
9. Renewal management
10. Notify management about upcoming expirations

These repetitive processes can consume staff time and increase the possibility of missed follow-ups or data-entry errors.

---
# Solution
---
### Workflow 1 — New Member Registration
<details><summary>More</summary>
When a new member submits the registration form:

```text
Registration Form
       ↓
    Webhook
       ↓
JavaScript Data Processing
       ↓
    Airtable
       ↓
      Wait
       ↓
Welcome Email
```

- **Webhook Trigger**

The workflow begins with an n8n Webhook configured to receive a POST request.

The webhook receives registration information submitted from the external registration form.

The incoming data contains information such as:

``` Full Name | Email | Phone Number | Subscription Plan | Profile Picture```

- **JavaScript Data Processing**

The JavaScript Code node extracts the required information from the incoming form submission.

It searches the submitted questions by their names rather than relying only on fixed array positions.

The workflow creates a structured object containing:
```text
{
  "fullName": "...",
  "email": "...",
  "phoneNumber": "...",
  "subscriptionPlan": "...",
  "profilePicture": [...]
}
```
_This makes the incoming form data easier to use in subsequent workflow steps._

- **Airtable Record Creation**

The processed member information is stored in an Airtable database.

The record includes:
```
Full Name | Email | Phone Number | Subscription | Profile Picture | Start Date | End Date
```
_The workflow automatically generates the start date using the current date._

- **Automated Membership End Date**

The workflow determines the membership duration from the selected subscription plan.

The current implementation supports:
```
One-month membership | Two-month membership | Three-month membership
```
The workflow then calculates the membership end date using n8n's date/time functionality.

Example:
```
Start Date
    ↓
Subscription Plan
    ↓
Determine Duration
    ↓
Calculate End Date
```
- **Wait Step**

After creating the Airtable record, the workflow uses an n8n Wait node before sending the welcome email. 
This introduces a controlled delay between record creation and customer communication.

- **Personalized Welcome Email**

The workflow sends a branded HTML welcome email through Gmail.

The email dynamically retrieves information from the member record, including:

``` Member's first name | Start date | Membership type | End date```

The workflow also extracts the member's first name directly within the email expression rather than requiring a separate processing node.
```
$json.fields['Full Name']
  ? $json.fields['Full Name'].split(' ')[0]
  : 'there'
```
_This allows the email to address the member personally._


</details>

---
### Workflow 2 — Membership Expiry Monitoring

<details><summary>More</summary>
  
A scheduled workflow runs every day at 6:00am
```text
Schedule Trigger
       ↓
Search Airtable Records
       ↓
Check Membership End Date
       ↓
Is Expiry Date 3 Days Away?
       ↓
      YES
       ↓
Extract Member Information
       ↓
Send Renewal Email
       ↓
Prepare Admin Notification
       ↓
Send Telegram Notification
```
_This allows the system to proactively identify members approaching subscription expiry._

- **Schedule Trigger**

The second workflow starts with an n8n Schedule Trigger.

The workflow is configured to run daily at: ```6:00 am```

- **Search Airtable**

The workflow retrieves membership records from the Airtable database.

The records contain the information required to determine which memberships are approaching expiration.
- **Conditional Expiry Check**

The workflow uses an IF node to compare each member's membership end date against: ```Current Date + 3 Days```

```
The condition effectively asks:
  Is this member's subscription ending three days from now?
Only records satisfying this condition continue through the renewal reminder branch.
```

- **First Name Extraction**

For members approaching expiration, a Python Code node extracts the member's first name from the full name.

The workflow uses: ```full_name.split()[0]```

The extracted first name is then stored as:
```firstName```

_This value is used to personalize the renewal email._

- **Renewal Reminder Email**
Members whose subscriptions are due to expire in three days receive an automated renewal reminder through Gmail.

The email contains:

```Member's first name | Membership expiry date | Membership type | Start date | Renewal reminder | Personalized motivational messaging```

_The purpose is to encourage members to renew before their membership expires._

- **Administrator Notification**

The workflow also prepares an internal notification containing members whose subscriptions are approaching expiration.

The notification includes:

```Member name | Phone number | Subscription type ```

The workflow formats these records into a readable message.

Example structure:
```
Hello Admin,

The following members have subscriptions
expiring in the next 3 days:

👤 Member Name | 📞 Phone | 📦 Subscription

Please follow up with them for renewal.
```
_The notification is then sent through Telegram._

</details>

---

### Workflow 3 — Membership Payment Automation

<details><summary>More</summary>
  
## Purpose

The payment workflow automates the process of recording and managing gym membership payments, 
connecting payment information with the member's existing membership record.
```text
Payment Submission
       ↓
Payment Processing
       ↓
Retrieve Member Information
       ↓
Match Membership Record
       ↓
Update Payment Information
       ↓
Payment Confirmation
       ↓
Update Membership Status
```

- **Member Registration Workflow**

The workflow begins when a prospective member submits a Fillout registration form.

The automation extracts:

```
Full Name | Email | Phone Number | Subscription Plan | Profile Picture
```

The data is formatted using an n8n Code node before being processed.

The workflow then searches Airtable using the member's email address to determine whether the person already exists in the database.

**Existing Member**

```
If the email already exists, the workflow sends the member a notification explaining that their registration already exists and provides payment options.
```
**New Member**
```
If no record exists, n8n creates a new Airtable record with:
  Member information
  Subscription selection
  Pending status
  Initial payment tracking fields
  Subscription counters
  Lifetime payment value initialized to zero
```
_This creates the member's record before payment is completed._

- **Payment Workflow**

The payment workflow uses a Stripe webhook to receive payment events.
```
Stripe Payment
      ↓
Webhook
      ↓
Extract Payment Data
      ↓
Identify Subscription Plan
      ↓
Find Member by Email
      ↓
Check Membership Status
      ↓
New User OR Renewal
      ↓
Update Airtable
      ↓
Send Confirmation
      ↓
Notify Administrator
```

**Stripe Payment Processing**

When Stripe sends a payment event, the workflow extracts:
- Customer email
- Payment amount
- Payment status
- Payment date
- Stripe payment link

_A Code node then converts the Stripe timestamp into a readable date format and maps the Stripe payment link to the corresponding membership plan._

**Subscription Mapping**

The workflow supports four subscription levels:

| Plan     | Subscription | Duration |
| -------- | ------------ | -------: |
| Lite     | Twice Weekly |  1 month |
| Basic    | One Month    |  1 month |
| Standard | Two Month    | 2 months |
| Premium  | Quarterly    | 3 months |

_This mapping allows the workflow to determine the correct membership type and duration from the Stripe payment link._

- **New User vs Renewal Routing**

After receiving a payment, the workflow searches Airtable using the customer's email address.

It then checks the member's current status.

```
Payment Received
      ↓
Find Member
      ↓
Check Status
   ┌──┴───────┐
   │          │
Expired     Pending
   │          │
Renewal    New Member
```
- **New Member Payment**

For a new member, the workflow:

```
Determines the selected subscription | Assigns the corresponding plan | Sets the initial subscription cycle | Calculates the membership end date | Sets the member's status to Active | Records the payment value | ends a welcome/activation email | Sends a Telegram notification to the administrator
```
_The workflow therefore turns a Pending registration into an active membership after payment._

- **Renewal Payment Workflow**

For an existing member whose membership has expired, the payment is treated as a renewal.

The automation tracks subscription activity by incrementing the appropriate plan counter:

```Lite | Basic | Standard | Premium```

It also adds the latest payment to the member's cumulative Value field.

The membership record is then updated with:

```New subscription | New start date | New calculated end date | Active status | Updated payment value | Previous subscription information```

_This creates a basic subscription and payment history directly within Airtable._

- **Membership End-Date Calculation**

A particularly useful part of the automation is the automatic calculation of the membership end date.

The workflow takes:

```Payment Date + Subscription Duration = Membership End Date```

For example:
```
Payment Date:       29-Apr-2026
Subscription:       2 Months
                         ↓
Membership End:     29-Jun-2026
```
_The duration is dynamically determined from the subscription plan rather than manually entered._

- **Automated Member Communication**

The system automatically communicates with members at different stages of their membership journey.

**Registration**

Members receive an email containing their available payment options.

**Successful New Membership**

Members receive a welcome email confirming:

**Successful payment**

```Active membership | Subscription plan | Start date | End date | Successful Renewal```

Existing members receive a renewal confirmation containing their updated membership information.

**Membership Expiry**

Members receive an email when their membership expires.

**Upcoming Expiry**

Members receive a reminder three days before their membership expires.

</details>

## Technology Stack
```
Automation: n8n
Database: Airtable
Registration: Fillout
Payment Processing: Stripe
Email: Gmail
Admin Notifications: Telegram
Programming: JavaScript | Python
Data Format: JSON
Integration: Webhooks | REST-style event processing
```

## Key Automation Skills Demonstrated

```
Workflow Automation | n8n | Webhook Integration | Stripe Integration | Airtable Automation |
API/Event Processing | JSON Data Transformation | JavaScript | Python | Conditional Logic |
Data Mapping | Subscription Management | Automated Email | Telegram Notifications | Date Calculation | Business Process Automation
```


