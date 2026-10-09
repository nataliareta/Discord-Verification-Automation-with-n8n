# Automated Discord Verification with n8n 🤖

An automated Discord member verification and access-control workflow built using **n8n, Google Forms, Google Sheets, and the Discord API**.

The project was designed to automate participant verification before granting access to a private Discord community. Instead of manually comparing registration data with Discord members and assigning roles one by one, the workflow handles the process automatically — from member matching to role assignment, welcome messaging, retry handling, duplicate protection, and verification logging.

---

## Overview

Participants register through Google Forms and submit several identifiers, including their Discord username.

When they first join the Discord server, they do not immediately receive access to the entire community. New members can only access the onboarding area while the main community channels remain restricted.

The automation periodically reads registration data from Google Sheets and compares the submitted Discord username with members who have joined the Discord server.

If a match is found, the workflow verifies the participant, assigns the appropriate Discord role, unlocks the restricted channels, sends a personalized welcome message, and records the result in a verification log.

If no match is found, the participant remains in the onboarding area and the result is recorded as `NOT_FOUND`.

Users with a `NOT_FOUND` status are automatically retried on the next workflow run. If the participant edits their form response with the correct Discord username, the automation reads the latest value and can verify them without creating a duplicate verification record.

---

## The Problem

For a community with a large number of participants, manually verifying Discord members creates several operational challenges:

- Registration data must be checked one by one.
- Discord roles need to be assigned manually.
- Users can accidentally receive access before verification.
- It becomes difficult to track who has already been verified.
- Incorrect Discord usernames require manual follow-up.
- Repeated verification can result in duplicate welcome messages or unnecessary role assignments.

The goal of this project was to turn that repetitive process into a controlled and monitorable automation.

---

## The Solution

The system uses **Google Sheets as the registration source of truth**, while n8n acts as the orchestration layer between the registration database and Discord.

The automation handles three main outcomes:

- **Matched and not yet verified**  
  Assign the Discord role, send the welcome message, and record the user as `VERIFIED`.

- **Already verified**  
  Stop processing the user to prevent duplicate actions.

- **Discord username not found**  
  Keep the user restricted, record the result as `NOT_FOUND`, and retry the participant on the next workflow execution.

---

## Workflow Overview

The complete verification workflow is orchestrated through n8n.

![Discord Verification Workflow](Workflow%20n8n.jpeg)

At a high level, the process is:

**Google Form → Google Sheets → Filter Unverified Responses → Discord Member Matching → Role Assignment → Welcome Message → Verification Log**

The workflow currently runs on a scheduled basis and includes a date guard so it only processes users during the intended operating period.

---

## Registration Data

Registration data is collected using Google Forms and stored in a worksheet called:

`Form_Responses`

The registration dataset includes:

- Email Address
- Discord Username
- PK
- Full Name

The original form responses remain separate from the operational verification log.

---

## Discord Access Control

Discord access is controlled using roles.

When a new participant joins the server, they initially only have access to the onboarding area:

- `#welcome`
- `#rules-and-guidelines`

The main community category remains private.

Once verification succeeds, the automation assigns the participant the required member role. Discord then automatically unlocks the restricted channels associated with that role.

This keeps access management simple because n8n only needs to manage the user's role instead of modifying individual channel permissions.

---

## Member Verification

n8n retrieves the list of members currently inside the Discord server through the Discord API.

The workflow compares each submitted Discord username against the Discord member list.

When a match is found, the workflow retrieves the user's **Discord User ID** and uses it for role assignment and personalized mentions.

The Discord User ID is retained as a more stable identifier because Discord usernames may change over time while the underlying user ID remains the same.

---

## Retry Handling for Incorrect Usernames

Participants may submit an incorrect Discord username and later edit their Google Form response.

To support this case:

- `VERIFIED` users are skipped.
- `NOT_FOUND` users remain eligible for retry.
- New form responses are processed normally.
- The latest Discord username from `Form_Responses` is always used.
- Existing verification rows are updated using `PK` instead of creating duplicate rows.

This allows a record to transition from:

`NOT_FOUND → VERIFIED`

without creating a second verification record.

---

## Duplicate Protection

Before processing a participant, the workflow compares form responses with the `Discord_Verification` log.

Participants already marked as:

`Verification Status = VERIFIED`

are filtered out before Discord processing begins.

The participant's `PK` is used as the unique matching key.

This prevents:

- repeated role assignments,
- duplicate welcome messages,
- and duplicate verification records.

---

## Automated Welcome Message

After successful verification, the bot sends a personalized confirmation message in the Discord welcome channel and mentions the newly verified member.

Example:

> 🎉 You’re verified, @username!  
> Welcome! Your server access is now active — you can start exploring the channels. See you around! 🚀

---

## Verification Logging

Automation results are stored in:

`Discord_Verification`

| Field | Description |
| --- | --- |
| Timestamp | Original registration timestamp |
| Email Address | Registered participant email |
| Username Discord | Latest Discord username submitted |
| PK | Unique participant identifier used as the matching key |
| Full Name | Participant name |
| Discord User ID | Unique Discord account identifier |
| Verification Status | `VERIFIED` or `NOT_FOUND` |
| Role Status | Role assignment result |
| Welcome Status | Welcome message delivery result |
| Last Checked | Latest verification timestamp |
| Error Message | Error information when applicable |

Successful verification:

`Verification Status = VERIFIED`  
`Role Status = ASSIGNED`  
`Welcome Status = SENT`  
`Error Message = empty`

Failed match:

`Verification Status = NOT_FOUND`  
`Role Status = NOT_ASSIGNED`  
`Welcome Status = NOT_SENT`

The workflow uses **Append or Update Row** with `PK` as the matching key, so each participant keeps a single verification row.

---

## Automation Schedule

The workflow is currently configured to run automatically every day at:

**11:00 WIB (UTC+7)**

The schedule can be adjusted through the n8n Schedule Trigger.

---

## Tech Stack

- **n8n** — workflow orchestration
- **Google Forms** — participant registration
- **Google Sheets** — registration database and verification logs
- **Google Sheets API** — data integration
- **Google Drive API** — Google file access
- **Google Cloud OAuth 2.0** — authentication for Google services
- **Discord API** — member retrieval, role assignment, and messaging
- **Discord Bot** — automated Discord actions

---

## How to Use

This project requires an existing Discord server, a Discord bot, a Google Form connected to Google Sheets, and an n8n instance.

### 1. Prepare the Registration Form

Create a Google Form containing:

- Email Address
- Discord Username
- PK
- Full Name

Connect the form to Google Sheets.

### 2. Prepare the Discord Server

Create a dedicated role for verified members.

Configure the server so new members can only access onboarding channels, while the main channels require the verified-member role.

The Discord bot needs permission to view members, manage roles, and send messages.

Make sure the bot role is positioned above the role that will be assigned automatically.

### 3. Connect Google Sheets to n8n

Connect the registration spreadsheet to n8n using Google OAuth 2.0.

Create a separate `Discord_Verification` worksheet for automation logging.

### 4. Configure the Discord Integration

Configure the required Discord identifiers in n8n:

```env
DISCORD_SERVER_ID=
DISCORD_ROLE_ID=
WELCOME_CHANNEL_ID=
```

Store the Discord Bot Token securely using n8n Credentials or environment variables.

---

## Security

Production credentials and participant information are not included in this repository.

Never commit:

- Discord bot tokens
- Google OAuth Client Secrets
- OAuth access or refresh tokens
- real participant emails
- real PK values
- production registration data

Any exported n8n workflow should be reviewed and sanitized before being committed to a public repository.

---

## Current Features

- Scheduled Discord verification
- Google Forms and Google Sheets integration
- Discord member matching
- Discord User ID retrieval
- Role-based server access
- Automatic role assignment
- Personalized welcome messages
- Verification logging
- `NOT_FOUND` retry handling
- Editable-form retry support
- Duplicate verification protection
- Single-row verification history using `PK`
- Configurable execution period

---

Thanks for checking out this project! Feel free to reach out if you need the full setup tutorial. 🚀
