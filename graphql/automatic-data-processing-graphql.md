# ADP (Automatic Data Processing) GraphQL Schema

## Overview

This conceptual GraphQL schema represents the ADP Human Capital Management (HCM) platform APIs, covering the full spectrum of workforce management capabilities available through the ADP Developer Portal at https://developers.adp.com/. ADP provides REST APIs across payroll, worker demographics, time and labor, benefits, talent management, and recruiting domains. This schema translates those REST resources into a unified GraphQL type system.

## Provider

- **Name**: Automatic Data Processing (ADP)
- **Developer Portal**: https://developers.adp.com/
- **Base URL**: https://api.adp.com
- **Authentication**: OAuth 2.0 with mutual TLS (mTLS) required for production

## Schema Source

This is a conceptual schema derived from the ADP public developer documentation and API reference at:

- https://developers.adp.com/articles/api/payroll
- https://developers.adp.com/articles/api/workers
- https://developers.adp.com/articles/api/time-labor-management
- https://developers.adp.com/articles/api/benefits
- https://developers.adp.com/articles/api/talent-management
- https://developers.adp.com/articles/api/recruiting

## API Domains Covered

### Worker Demographics API
Covers employee personal information, employment records, job assignments, pay rates, and reporting relationships across ADP Workforce Now and ADP Vantage HCM.

### Payroll API
Covers payroll processing including earnings, deductions, pay statements, payroll runs, tax records, and direct deposit configurations.

### Time and Labor API
Covers time entries, schedules, time off requests, PTO accruals, leave management, and labor cost allocation.

### Benefits API
Covers benefits enrollment, plan data, coverage elections, and life event processing.

### Talent Management API
Covers performance reviews, goal management, learning and development, competencies, and succession planning.

### Recruiting API
Covers job postings, candidate management, offer letters, and new hire onboarding workflows.

## Type Summary

| Category | Types |
|---|---|
| Workers | Worker, WorkerDetails, WorkerStatus, Employment, EmploymentDetails, PersonName, Address, Communication |
| Payroll | Payroll, PayrollRunDetails, PayrollResult, Paycheck, PaycheckDetails, EarningRecord, DeductionRecord, TaxRecord, DirectDeposit, AllocationDetails |
| Time and Labor | TimeEntry, Schedule, ScheduleShift, Attendance, AttendanceRecord, PTO, PTOAccrual, Leave, LeaveRequest |
| Benefits | Benefit, BenefitPlan, BenefitElection, Enrollment, EnrollmentDetails, Coverage, Dependent, LifeEvent |
| Organization | Organization, Department, Position, Job, Team, Manager, CostCenter, Location |
| Talent | Goal, GoalProgress, Performance, Review, ReviewCycle, Learning, Course, Competency, CompetencyRating |
| Reporting | Report, ReportDetails, ReportColumn, Analytics, WorkforceMetric |
| Platform | APIKey, Token, Webhook, WebhookEvent, EventSubscription, AuditLog |
| Common | Cost, Money, DateRange, Pagination, Error |

**Total named types: 72**

## Query Capabilities

- Retrieve individual workers and worker lists with filtering by status, department, and location
- Fetch payroll runs, paychecks, and pay statement details
- Query time entries, schedules, and leave balances
- Access benefits enrollment and coverage data
- View performance reviews, goals, and learning records
- Pull organizational structure including departments, positions, and cost centers
- Generate workforce analytics and reports

## Mutation Capabilities

- Create and update worker records and employment details
- Submit payroll inputs and trigger payroll runs
- Log time entries and manage schedules
- Process benefits enrollment and life events
- Create and update goals and performance reviews
- Manage webhook subscriptions and event notifications

## Authentication

ADP APIs use OAuth 2.0 with two flows:

1. **Client Credentials** — for server-to-server integrations accessing non-employee-specific data
2. **Authorization Code** — for integrations requiring employee consent and access to individual records

Production environments require mutual TLS (mTLS) certificate-based connections in addition to OAuth tokens.

## Notes

ADP does not publicly expose a native GraphQL endpoint. This schema is a conceptual representation enabling developers to understand the data model, plan GraphQL gateway implementations (e.g., using Apollo Federation or Hasura), or design wrapper layers over the ADP REST APIs.
