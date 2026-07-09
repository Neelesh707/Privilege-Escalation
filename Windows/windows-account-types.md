# Windows Account Types

Before diving into privilege escalation techniques, it's important to understand the different account types on a Windows system.

## Standard User Groups

Windows systems mainly have two kinds of users, categorized by access level:

| Group | Description |
|---|---|
| **Administrators** | Users with the most privileges. Can change any system configuration parameter and access any file on the system. |
| **Standard Users** | Users who can access the computer but are limited to performing basic tasks. Typically cannot make permanent or essential system changes and are restricted to their own files. |

- Any user with administrative privileges belongs to the **Administrators** group.
- Standard users belong to the **Users** group.

## Built-in Service Accounts

In addition to the groups above, Windows has special built-in accounts used internally by the operating system. These come up often in the context of privilege escalation:

### SYSTEM / LocalSystem
- Used by the OS to perform internal tasks.
- Has **full access** to all files and resources on the host.
- Privileges are **even higher** than those of Administrators.

### Local Service
- Default account used to run Windows services with **minimum privileges**.
- Uses **anonymous connections** over the network.

### Network Service
- Default account used to run Windows services with **minimum privileges**.
- Uses the **computer's credentials** to authenticate over the network.

## Key Notes

- These built-in accounts (`SYSTEM`, `Local Service`, `Network Service`) are created and managed by Windows — they can't be used like regular user accounts.
- However, by exploiting certain services, an attacker may be able to **gain the privileges** of these accounts (most notably `SYSTEM`), which is a common goal in Windows privilege escalation.
