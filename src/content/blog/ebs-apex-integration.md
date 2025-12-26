---
title: "Extending E-Business Suite with APEX"
date: 2024-09-10
author: "JMJ Cloud"
tags: ["EBS", "APEX", "Integration"]
summary: "How to build modern web interfaces on top of your Oracle EBS data using APEX and ORDS, including architecture patterns, SSO integration, and best practices for coexistence."
---

Oracle E-Business Suite remains the backbone of operations for many enterprises. While EBS provides comprehensive functionality, users increasingly expect modern, mobile-friendly interfaces. Oracle APEX offers a path to deliver those experiences without replacing your EBS investment.

## The Architecture Pattern

The recommended approach keeps APEX and EBS as separate but integrated systems:

```
┌─────────────────┐     ┌─────────────────┐
│   APEX Server   │────▶│   ORDS/APEX DB  │
└─────────────────┘     └────────┬────────┘
                                 │
                                 │ DB Links / APIs
                                 ▼
                        ┌─────────────────┐
                        │    EBS Database │
                        └─────────────────┘
```

APEX runs in its own database (or PDB), connecting to EBS data through database links or REST APIs. This separation provides several benefits:

- **Isolation**: APEX upgrades don't affect EBS
- **Performance**: APEX queries don't compete with EBS transactions
- **Security**: Fine-grained control over what EBS data is exposed

## Setting Up the Database Link

Create a database link from your APEX database to EBS:

```sql
CREATE DATABASE LINK ebs_link
CONNECT TO apps_readonly
IDENTIFIED BY "secure_password"
USING 'EBS_PROD';
```

Use a read-only account with access only to the tables and views your APEX application needs.

## Synchronizing Reference Data

For frequently accessed reference data, consider materialized views:

```sql
CREATE MATERIALIZED VIEW mv_ebs_employees
REFRESH COMPLETE ON DEMAND
AS
SELECT employee_id,
       full_name,
       email_address,
       supervisor_id,
       organization_id
FROM hr_employees@ebs_link
WHERE current_employee_flag = 'Y';

-- Refresh on a schedule
BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name        => 'REFRESH_EBS_EMPLOYEES',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'BEGIN DBMS_MVIEW.REFRESH(''MV_EBS_EMPLOYEES''); END;',
    repeat_interval => 'FREQ=HOURLY;INTERVAL=1',
    enabled         => TRUE
  );
END;
/
```

## Single Sign-On Integration

Users shouldn't have to log in twice. If your EBS environment uses Oracle Access Manager or LDAP, configure APEX to use the same identity provider:

```sql
-- Configure APEX authentication scheme
BEGIN
  APEX_AUTHENTICATION.SET_SCHEME(
    p_application_id => 100,
    p_scheme_type    => 'LDAP',
    p_host           => 'ldap.company.com',
    p_port           => 389,
    p_use_ssl        => 'Y',
    p_search_filter  => 'uid=%LDAP_USER%'
  );
END;
/
```

For deeper integration, consider implementing a custom authentication function that validates against EBS's FND_USER table.

## Building Approval Workflows

One of the most common APEX extensions is a mobile-friendly approval interface for EBS workflows. The pattern:

1. Query pending notifications from WF_NOTIFICATIONS via database link
2. Display in a modern APEX interface with swipe actions
3. Call EBS workflow APIs to approve/reject
4. Handle the response and update the UI

```sql
-- Get pending approvals for current user
SELECT notification_id,
       subject,
       begin_date,
       from_user,
       message_type
FROM wf_notifications@ebs_link
WHERE recipient_role = :current_user
  AND status = 'OPEN'
ORDER BY begin_date DESC;
```

## Best Practices for Coexistence

**Don't bypass EBS APIs**: Always use EBS public APIs for write operations. Direct DML against EBS tables can corrupt data and break upgrade paths.

**Respect EBS security**: Honor EBS organization access, set of books assignments, and responsibility-based security in your APEX application.

**Plan for EBS upgrades**: Document all database links, synonyms, and dependencies. Test your APEX application after every EBS patch.

**Monitor performance**: Database link queries add latency. Cache what you can, and use asynchronous patterns for long-running operations.

APEX and EBS together provide the best of both worlds: proven ERP functionality with modern user experiences where it matters most.
