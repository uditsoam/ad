# Active Directory (AD)

## 1. What is Active Directory?

Active Directory Domain Service (AD DS) is a centralized system that stores and manages network objects like users, computers, groups, printers, and shares.

---

## 2. Users

Users are security principals (can authenticate and access resources).

### Types:

* Normal Users (employees)
* Service Accounts (IIS, MSSQL)

---

## 3. Machines

* Every domain-joined computer has an account.
* Example: DC01 → DC01$
* Machine accounts are also security principals.
* Passwords are auto-rotated and strong.

---

## 4. Security Groups

Used to assign permissions.

### Important Groups:

* Domain Admins → Full control
* Server Operators → Manage DC
* Backup Operators → Access all files
* Account Operators → Manage accounts
* Domain Users → All users
* Domain Computers → All machines
* Domain Controllers → DCs

---

## 5. Organizational Units (OUs)

* Containers for users and computers
* Used to apply policies
* A user can belong to only ONE OU

---

## 6. OU vs Groups

| Feature    | OU     | Group       |
| ---------- | ------ | ----------- |
| Purpose    | Policy | Permissions |
| Membership | One    | Multiple    |

---

## 7. AD Users and Computers

GUI tool used to:

* Create users
* Reset passwords
* Manage groups

---

## 8. Default Containers

* Builtin → Default groups
* Computers → New machines
* Users → Default users
* Domain Controllers → DCs

---

## 9. OSCP Perspective

* AD = Central control system
* Groups = Power
* Enumeration is key
* Goal = Domain Admin

---

## 10. Key Takeaway

"AD exploitation is all about understanding relationships between users, groups, and machines."
