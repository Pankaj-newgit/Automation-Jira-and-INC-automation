Final solution I recommend

Your actual flow is:

DOP Trigger
    ↓
CI/CD Pipeline
    ↓
Jenkins
    ↓
Code Merge
    ↓
Security
    ↓
Deployment
    ↓
Application / Infrastructure impact
    ↓
Zabbix / Dynatrace alerts
    ↓
ServiceNow INC

During deployment:

Deployment
   │
   ├── CPU High ───────────┐
   ├── Memory High ────────┤
   ├── Disk High ──────────┤
   ├── API 5xx ─────────────┤
   ├── API 4xx ─────────────┤
   └── Microservice issue ──┘
                            ↓
                       ServiceNow
                         INCs

Your automation should know:

DOP12345
Application = Digital_ThanksApp
Environment = PROD
Start = 10:00
End = 10:30
Status = DEPLOYMENT

Then when:

INC20532974
Created = 10:08
Application = Digital_ThanksApp
Source = Zabbix
Alert = CPU High

the automation says:

INC time        → inside deployment window
Application     → matches
Environment     → matches
Activity        → approved deployment

             ↓

DEPLOYMENT RELATED

Then it verifies recovery and resolves the INC with the DOP reference.

PART 1 — Don't start coding yet

This is your Step 1.

First document the existing flow.

You need to collect exactly these things from your organization.

A. ServiceNow

Get:

ServiceNow URL
INC API/table name
INC fields
Authentication method
Required fields for resolving INC
Resolution Code field
Resolution Notes field
DOP/Jira/Vendor reference field

From your screenshot we already know you have:

Resolution Type
Resolution Code
Resolution Notes

and a resolution option:

Alert due to deployment was going on at that time.

So we can reuse those.

PART 2 — Identify the deployment source

This is the most important step.

You told me:

Deployment is triggered through DOP → Jenkins → code merge → security → DOP/deployment.

Therefore, DOP/Jenkins should be the source of deployment activity information.

Do not depend on a person manually entering the DOP number into the INC.

We want:

DOP
 ↓
Deployment Started
 ↓
Automation receives event
 ↓
Create Activity Record

For example:

Activity ID: ACT000123
DOP: DOP12345
Application: Digital_ThanksApp
Environment: PROD
Activity Type: DEPLOYMENT
Start: 10:00
Expected End: 10:30
Status: RUNNING
PART 3 — Create an Activity Registry

This is the heart of the solution.

Initially this can be a small database.

I recommend:

PostgreSQL if your organization allows it.

For a POC, even SQLite is sufficient.

Create an activities table:

activity_id
activity_type
reference_number
application
environment
start_time
expected_end_time
actual_end_time
status
source
created_time

Example:

ACT001
DEPLOYMENT
DOP12345
Digital_ThanksApp
PROD
10:00
10:30
10:27
COMPLETED
DOP

Another:

ACT002
DATABASE
CHG56789
Billing
PROD
11:00
12:00
12:05
COMPLETED
ServiceNow

Another:

ACT003
NETWORK
NET12345
Payment
PROD
14:00
15:00
14:50
COMPLETED
Network
PART 4 — Capture deployment START automatically

This is where your CI/CD comes into the picture.

Your desired flow should become:

DOP
 ↓
Deployment initiated
 ↓
DOP/Jenkins
 ↓
Webhook/API
 ↓
Automation
 ↓
Activity Registry

Automation stores:

DOP12345
Digital_ThanksApp
PROD
10:00
DEPLOYMENT
RUNNING

Then Jenkins continues normally.

The automation does not control or modify the deployment initially.

It only records:

"A deployment is happening."

This is important for safety.

PART 5 — Capture deployment END

When Jenkins/DOP finishes:

Jenkins
 ↓
Deployment completed
 ↓
Webhook/API
 ↓
Automation
 ↓
Activity Registry

Update:

status = COMPLETED
actual_end_time = 10:27

So now:

DOP12345

Start       10:00
End         10:27
Status      COMPLETED
PART 6 — Now monitor ServiceNow INCs

Your automation periodically checks newly generated INCs.

For example:

Every 1 minute
      ↓
ServiceNow API
      ↓
Get new/active monitoring INCs

We don't need to retrieve every historical INC every time.

Initially:

created in last 5 minutes

or use a ServiceNow query/filter based on your actual environment.

PART 7 — Extract information from the INC

From the screenshot, your INC has useful information.

The automation should extract:

INC Number
Created Time
Priority
Impact
Urgency
Application
Business Service
CI
Correlation ID
Short Description
Description
Assignment Group
Alert Source

For example:

INC20532974

Source:
Zabbix

Application:
Digital_ThanksApp

Created:
10:08

Priority:
P2

Short Description:
Airtel-Zabbix_New_Digital_ThanksApp...

Correlation ID:
...
PART 8 — Build the correlation engine

This is the actual intelligence.

Do not use only:

INC created during deployment

Instead use multiple matching conditions.

Rule 1 — Application match
INC Application
       ==
Activity Application

Example:

INC = Digital_ThanksApp
DOP = Digital_ThanksApp

MATCH
Rule 2 — Environment match
INC Environment = PROD
Activity Environment = PROD
Rule 3 — Time match

For example:

Activity Start = 10:00
Activity End   = 10:30

INC created = 10:08

Therefore:

10:00 <= 10:08 <= 10:30

MATCH.

Rule 4 — Alert source

For example:

Zabbix
Dynatrace

Both are monitoring sources.

Rule 5 — Alert type

Deployment-related alerts could include:

CPU High
Memory High
Disk High
Pod Restart
API 5xx
API timeout
Connection timeout
Application unavailable

But don't blindly classify every 5xx as deployment-related.

The activity/application/time correlation must also match.

PART 9 — Use a confidence score

I recommend this because it makes your automation safer.

Example:

Application match        +30
Environment match        +20
Activity time match      +25
DOP/deployment match     +15
Alert source match       +10
                         ----
                          100

Then:

80–100 → HIGH
60–79  → MEDIUM
<60    → LOW

Initially:

HIGH   → candidate for auto-resolution
MEDIUM → NOC review
LOW    → keep open

This is much safer than a simple yes/no rule.

PART 10 — Add recovery verification

This is mandatory.

Suppose:

Deployment
 ↓
API 500
 ↓
INC

We shouldn't immediately resolve it.

Instead:

INC
 ↓
Deployment correlation
 ↓
Wait for deployment completion
 ↓
Check monitoring recovery

For example:

DOP completed at 10:27

Zabbix:
CPU recovered
API 500 recovered
Service healthy

Then:

SAFE TO RESOLVE
PART 11 — Genuine failure protection

Suppose:

Deployment:
10:00–10:30

At 10:15:

API 500

looks deployment-related.

But at 10:20:

Database unavailable

and DB remains down after deployment.

The automation should not close that INC merely because deployment was happening.

Therefore:

Activity match
       +
Recovery confirmed
       +
No continuing failure
       ↓
Auto resolve

If recovery isn't confirmed:

KEEP OPEN
PART 12 — ServiceNow update

When the automation decides that an INC is activity-related, update the existing fields.

For deployment:

Resolution Code

Use your existing value:

Alert due to deployment was going on at that time.
Resolution Notes

Automatically populate something like:

Auto-resolved by NOC automation.

Alert Source: Zabbix
Activity Type: Deployment
DOP Reference: DOP12345
Application: Digital_ThanksApp
Environment: PROD

Alert was generated during the approved deployment activity
window and monitoring recovered after deployment completion.

Activity Start: 10:00 IST
Activity End: 10:27 IST

Correlation Status: Deployment Activity
PART 13 — Add a machine-readable classification

This is something I strongly recommend.

Don't depend only on the human-readable resolution note.

If ServiceNow allows a custom field, create something like:

u_incident_classification

Values:

GENUINE_FAILURE
DEPLOYMENT_ACTIVITY
DATABASE_ACTIVITY
NETWORK_ACTIVITY
VENDOR_ACTIVITY
MAINTENANCE_ACTIVITY
OTHER_ACTIVITY

And:

u_activity_reference

Example:

u_incident_classification = DEPLOYMENT_ACTIVITY

u_activity_reference = DOP12345

Now your Excel reporting becomes extremely easy.

PART 14 — Your final ServiceNow record

For example:

INC20532974

State:
Resolved

Resolution Code:
Alert due to deployment was going on at that time.

Incident Classification:
DEPLOYMENT_ACTIVITY

Activity Reference:
DOP12345

Resolution Notes:
Auto-resolved by NOC automation.

Alert generated during approved deployment activity.

Application: Digital_ThanksApp
Environment: PROD
Alert Source: Zabbix
DOP: DOP12345

Deployment Start: 10:00 IST
Deployment End: 10:27 IST

Monitoring recovered after deployment completion.

Now your NOC doesn't have to manually type everything.

PART 15 — What about Dynatrace?

Exactly the same architecture.

You don't need two completely different automation systems.

             Monitoring
            /           \
       Zabbix          Dynatrace
          \               /
           \             /
            ServiceNow INC
                   ↓
          Correlation Engine
                   ↓
             Activity DB
                   ↓
        ┌──────────┴─────────┐
        ↓                    ↓
     Activity             Genuine
     Related              Failure
        ↓                    ↓
     Resolve              Keep Open

The classifier only needs to understand:

source = Zabbix

or:

source = Dynatrace
PART 16 — Handle multiple INCs from one deployment

This is where your solution will give NOC the biggest benefit.

Suppose:

DOP12345
10:00–10:30
Digital_ThanksApp

creates:

INC001 → CPU High
INC002 → Memory High
INC003 → Disk High
INC004 → API 500
INC005 → API Timeout
INC006 → Pod Restart

Automation identifies:

             DOP12345
                 │
       ┌─────────┼──────────┐
       ↓         ↓          ↓
     INC001    INC002     INC003
       ↓         ↓          ↓
     INC004    INC005     INC006

All are:

DEPLOYMENT_ACTIVITY
DOP12345

Then each INC gets the proper classification/reference.

PART 17 — Your Excel requirement is solved automatically

When NOC exports ServiceNow:

INC	Priority	Application	Source	Classification	Reference	State
INC001	P2	Digital_ThanksApp	Zabbix	DEPLOYMENT_ACTIVITY	DOP12345	Resolved
INC002	P3	Digital_ThanksApp	Dynatrace	DEPLOYMENT_ACTIVITY	DOP12345	Resolved
INC003	P2	Billing	Zabbix	DATABASE_ACTIVITY	CHG12345	Resolved
INC004	P1	Payment	Dynatrace	GENUINE_FAILURE	—	Open

They can simply filter:

Classification = DEPLOYMENT_ACTIVITY

Done.

PART 18 — Exact technology stack

For your first implementation, I would use:

Python
   │
   ├── ServiceNow REST API
   │
   ├── DOP/Jenkins API or Webhook
   │
   ├── Zabbix API
   │
   ├── Dynatrace API
   │
   ├── PostgreSQL
   │
   └── Logging

You don't need AI initially.

You don't need Machine Learning.

You don't need a huge platform.

Start with Python + APIs + rules + database.

PART 19 — Exact Python project structure

I would build the project like this:

incident-automation/
│
├── config/
│   └── config.yaml
│
├── servicenow/
│   ├── client.py
│   ├── get_incidents.py
│   └── update_incident.py
│
├── deployment/
│   ├── dop_client.py
│   ├── jenkins_client.py
│   └── activity_manager.py
│
├── monitoring/
│   ├── zabbix_client.py
│   └── dynatrace_client.py
│
├── correlation/
│   ├── matcher.py
│   ├── rules.py
│   └── classifier.py
│
├── resolution/
│   └── resolver.py
│
├── database/
│   └── db.py
│
├── logs/
│
├── tests/
│
└── main.py
PART 20 — The main program

Conceptually:

START
  ↓
Get active deployment activities
  ↓
Get new ServiceNow INCs
  ↓
For each INC
  ↓
Extract application/environment/time/source
  ↓
Search activity registry
  ↓
Does activity match?
  │
  ├── NO → Genuine candidate → KEEP OPEN
  │
  └── YES
        ↓
    Calculate confidence
        ↓
    HIGH?
        │
        ├── NO → NOC REVIEW
        │
        └── YES
              ↓
        Activity completed?
              │
              ├── NO → WAIT
              │
              └── YES
                    ↓
              Alert recovered?
                    │
                    ├── NO → KEEP OPEN
                    │
                    └── YES
                          ↓
                    Update ServiceNow
                          ↓
                    Resolution Code
                          ↓
                    Resolution Notes
                          ↓
                    Activity Reference
                          ↓
                    Resolve INC
PART 21 — Very important: don't auto-close on Day 1

I would implement this in 4 stages.

Stage 1 — READ ONLY

For 1–2 weeks:

ServiceNow
   ↓
Python
   ↓
Classification
   ↓
Log only

Example:

INC001 → DEPLOYMENT_ACTIVITY
DOP12345
Confidence = 92%

Action = WOULD_RESOLVE

Nothing is changed in ServiceNow.

Stage 2 — Classification only

Python updates:

Classification
Activity Reference
Work Notes

but doesn't resolve.

NOC validates the result.

Stage 3 — Controlled auto-resolution

Start with:

P3/P4

and only high-confidence cases.

For example:

Confidence >= 85%
+
Deployment completed
+
Monitoring recovered

→ Resolve.

Stage 4 — Expand carefully

After you have sufficient historical validation:

P3/P4 → automated
P2 → controlled automation
P1 → initially manual/NOC approval

This isn't because P1/P2 can never be automated; it's because you need stronger evidence before automatically resolving high-impact incidents.

PART 22 — The most important design decision

Your automation should have two independent data streams.

Stream 1 — Activity
DOP/Jenkins
      ↓
Activity Registry
Stream 2 — Incident
Zabbix/Dynatrace
      ↓
ServiceNow
      ↓
Automation

Then:

Activity Stream
       +
Incident Stream
       ↓
Correlation Engine
       ↓
Classification
       ↓
Recovery Verification
       ↓
ServiceNow Action

This is the architecture I would present to your NOC/DevOps team.

PART 23 — What you should do tomorrow

Don't start by writing 1,000 lines of Python.

Follow these exact steps:

Step 1

Take 5–10 real deployment examples.

For each collect:

DOP number
Application
Start time
End time
Environment
Jenkins job/build
Step 2

For each deployment, collect the INCs generated around that time:

INC number
Created time
Application
Short description
Correlation ID
Source
Priority
Final resolution
Step 3

Take 5 genuine failure examples that happened without deployment.

This is extremely important because these become your negative test cases.

Step 4

Map the fields.

Create a document:

DOP Application
       ↓
ServiceNow Application

DOP Environment
       ↓
INC Environment

DOP Start/End
       ↓
INC Created Time

DOP Number
       ↓
INC Activity Reference
Step 5

Ask the DOP/Jenkins team:

"Can we receive a webhook/API event whenever a deployment starts and completes, including DOP number, application, environment, start time, end time and deployment status?"

This is probably your most important question.

Step 6

Ask the ServiceNow team:

"Can we update the existing Resolution Code and Resolution Notes through ServiceNow REST API, and can we add a machine-readable Incident Classification and Activity Reference field?"

Step 7

Build the Python read-only POC.

Step 8

Test against historical INCs.

Step 9

Run in shadow mode.

Step 10

Only after validation, enable automatic resolution.

Final architecture

This is what I would ultimately aim for:

                         ┌──────────────┐
                         │     DOP      │
                         └──────┬───────┘
                                │
                         Deployment Start
                                │
                                ▼
                         ┌──────────────┐
                         │    Jenkins   │
                         │     CI/CD    │
                         └──────┬───────┘
                                │
                  Code → Security → Deploy
                                │
                                ▼
                       ┌─────────────────┐
                       │ Activity Engine │
                       └────────┬────────┘
                                │
                         Activity Registry
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
              Zabbix                       Dynatrace
                 │                             │
                 └──────────────┬──────────────┘
                                ▼
                       ┌─────────────────┐
                       │   ServiceNow    │
                       │      INC        │
                       └────────┬────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   Correlation   │
                       │     Engine      │
                       └────────┬────────┘
                                │
                ┌───────────────┴────────────────┐
                │                                │
          Activity Match                   No Match
                │                                │
                ▼                                ▼
        Recovery Check                     Genuine Candidate
                │                                │
         ┌──────┴──────┐                         │
         │             │                         ▼
      Recovered    Not Recovered             KEEP OPEN
         │             │
         ▼             ▼
   Auto Resolve     KEEP OPEN
         │
         ▼
 Resolution Code
         +
 Resolution Notes
         +
 DOP/Jira/Vendor Reference
         +
 Classification
The key principle

DOP/Jenkins tells us that an activity is happening.
Zabbix/Dynatrace tells us that something happened.
ServiceNow stores the INC.
Python correlates the two.
Recovery verification decides whether it is safe to resolve.

That is the cleanest way to solve your NOC requirement.

And importantly, your existing ServiceNow resolution options can be reused rather than rebuilding ServiceNow from scratch.
