# Week 2 - Linux Administration Fundamentals
## - Took a long break because of work however I am now back into it and will try to stay consistent.

## Objective

Develop an administrator's understanding of how Linux manages users, permissions, ownership, and software while learning to troubleshoot using a structured approach.

## Topics Covered

### Linux Filesystem

* Continued navigating the Linux filesystem using absolute and relative paths.
* Improved confidence working within the directory structure.
* Reinforced the importance of inspecting the current location before making changes.

### File Management

* Created, copied, moved, renamed and removed files and directories.
* Developed an understanding of source and destination when performing file operations.
* Followed a consistent workflow of inspecting, modifying and verifying changes.

### Linux Permissions

Rather than memorising permission values, I focused on understanding why Linux permissions exist.

Key concepts learned:

* Linux is a multi-user operating system.
* Every access request is based on user identity.
* Linux evaluates permissions in the following order:

  1. Owner
  2. Group
  3. Others

I also learned the difference between execute permissions on files and directories, and why they behave differently.

### Ownership

Explored how Linux separates file ownership from group ownership.

Example:

```text
-rw-rw---- 1 alice developers app.py
```

From this I learned:

* The **owner** represents the individual responsible for the file.
* The **group** represents the team responsible for collaborating on the file.
* Ownership and permissions are separate concepts that work together to control access.

### Administrative Privileges

Developed an understanding of why Linux administrators normally work as a standard user instead of logging in as `root`.

Topics explored:

* Principle of Least Privilege
* Accountability
* Why the `root` account exists
* Why changing file ownership is considered an administrative task
* Why temporary privilege elevation is more secure than working as `root` continuously

### Package Management

Before learning package management commands, I explored the problems they are designed to solve.

I learned why Linux distributions use package repositories to provide:

* Trusted software sources
* Dependency management
* Software version tracking
* Security updates
* Consistent software deployment across multiple systems

## Linux Skills Acquired

* Interpreted Linux permission strings.
* Converted symbolic and numeric permissions.
* Applied file permissions using symbolic and numeric notation.
* Distinguished between file ownership and group ownership.
* Understood the purpose of administrative privileges.
* Developed a conceptual understanding of Linux package management.

## Troubleshooting Mindset

Throughout this week I focused on following a consistent administrative workflow before making any changes.

**Inspect → Change → Verify**

Rather than immediately executing commands, I learned to first determine:

* Who owns the file?
* Which group owns the file?
* What permissions are currently assigned?
* Which identity Linux is evaluating when deciding whether access should be granted?

This approach has helped me become more methodical when troubleshooting Linux systems.

## Reflection

This week shifted my focus away from memorising Linux commands and towards understanding how Linux is designed.

The most valuable lesson was recognising that Linux commands are tools used to solve administrative problems rather than isolated commands to memorise.

I also began thinking more like a systems administrator by considering security, accountability, least privilege, troubleshooting, and how Linux scales in enterprise environments.
