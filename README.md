# Detailed Authentication Implementation

This article was last updated on 25 Mar 2026

## Contents
- [About](#about)
- [Tables](#tables)
  - [SMILE Table](#smile-table)
  - [Keycloak Table](#keycloak-table)
- [Data Integration](#data-integration)
- [Features Validation Flow](#features-validation-flow)
  - [Login](#login)
  - [Logout](#logout)
  - [Create User](#create-user)
  - [Edit User & Update Status User](#edit-user--update-status-user)
---

## About

This Implementation Guide for Authentication Service provides a detailed overview of how Keycloak is integrated into the SMILE system for authentication and authorisation. It explains the interaction between SMILE’s database and Keycloak, covering data storage, the structure of JWT tokens, the authentication flow, and system integration.

n addition, after a user successfully authenticates and obtains a token from Keycloak via SMILE, the user will access the WMS (Warehouse Management System) application. At this stage, an additional process is performed where the WMS system sets its own authentication context (WMS Auth Token). The token issued by SMILE is used by WMS to retrieve user profile information and other relevant attributes. WMS then enriches this data by adding specific information required for its internal operations, such as roles, permissions, warehouse access, and operational context. This enriched data is then encapsulated into a new WMS-specific token or session, which is used for subsequent interactions within the WMS application.

This document aims to guide developers in implementing Keycloak within SMILE and help users understand the authentication flow, data synchronisation, and overall security mechanisms between the two systems, including the token transformation and enrichment process when transitioning from SMILE authentication to WMS authorization. It is intended for developers, system architects, and administrators responsible for implementing, maintaining, or extending authentication within the SMILE ecosystem.

---

## Tables

The three tables, **SMILE**, **Keycloak**, and **Users WMS**, are utilised for user authentication to validate that the user has the right privilege to access the system.

### SMILE Table

| Variable | Data Type | Constraint | Note |
|--------|-----------|------------|------|
| id | bigint (20) | NOT NULL, AUTO_INCREMENT | |
| username | varchar (255) | | default: NULL |
| email | varchar (255) | | default: NULL |
| firstname | varchar (255) | | default: NULL |
| lastname | varchar (255) | | default: NULL |
| date_of_birth | date | | default: NULL |
| gender | int (11) | | default: NULL |
| mobile_phone | varchar (255) | | default: NULL |
| address | text | | default: NULL |
| created_by | int (11) | | default: NULL |
| updated_by | int (11) | | default: NULL |
| deleted_by | int (11) | | default: NULL |
| created_at | timestamp | NOT NULL | default: current_timestamp() |
| updated_at | timestamp | NOT NULL | default: current_timestamp() on update |
| password | varchar (255) | | default: NULL |
| entity_id | bigint (20) | | default: NULL |
| role | int (11) | | default: NULL |
| village_id | varchar (255) | | default: NULL |
| timezone_id | int (11) | | default: NULL |
| token_login | text | | default: NULL |
| status | tinyint (4) | | default: 1 |
| last_login | datetime | | default: NULL |
| last_device | tinyint (4) | | default: NULL |
| mobile_phone_2 | varchar (255) | | default: NULL |
| mobile_phone_brand | varchar (255) | | default: NULL |
| mobile_phone_model | varchar (255) | | default: NULL |
| imei_number | varchar (255) | | default: NULL |
| sim_provider | varchar (255) | | default: NULL |
| sim_id | varchar (255) | | default: NULL |
| iota_app_gui_theme | varchar (255) | | default: NULL |
| permission | varchar (255) | | default: NULL |
| application_version | varchar (255) | | default: NULL |
| last_mobile_access | datetime | | default: NULL |
| view_only | int (11) | NOT NULL | default: 0 |
| change_password | tinyint (4) | | default: NULL |
| manufacture_id | int (11) | | default: NULL |
| fcm_token | varchar (255) | | default: NULL |
| keycloak_uuid | varchar (100) | | default: NULL |
| user_uuid | varchar (100) | | default: NULL |

---

### Keycloak Table

| Variable | Data Type | Constraint | Note |
|--------|-----------|------------|------|
| id | bigint (20) | NOT NULL, AUTO_INCREMENT | |
| username | varchar (255) | | default: NULL |
| email | varchar (255) | | default: NULL |
| firstname | varchar (255) | | default: NULL |
| lastname | varchar (255) | | default: NULL |
| created_at | timestamp | NOT NULL | default: current_timestamp() |
| password | varchar (255) | | default: NULL |
| role | int (11) | | default: NULL |
| token_login | text | | default: NULL |
| user_uuid | varchar (100) | | default: NULL |
| program_id | int (11) | | default: NULL |

---

### User WMS Table

| Variable | Data Type | Constraint | Note |
|--------|-----------|------------|------|
| id | bigint (20) | NOT NULL, PRIMARY KEY | |
| user_uuid | char(36) | NOT NULL | |
| entity_id | bihint(20) | NOT NULL | |
| username | varchar (255) | | default: NULL |
| email | varchar (255) | | default: NULL |
| firstname | varchar (255) | | default: NULL |
| lastname | varchar (255) | | default: NULL |
| mobile_phone | varchar(20) | | default: NULL |
| gender | int(11) | | default: NULL |
| gender_label | varchar(20) | | default: NULL |
| date_of_birth | date | | default: NULL |
| created_at | timestamp | NOT NULL | default: current_timestamp() |
| password | varchar (255) | | default: NULL |
| role | int (11) | | default: NULL |
| role_id | int (11) | | default: NULL |
| role_label | varchar(50) | | default: NULL |
| view_only | tinyint(1) | | default: NULL |
| status | int(11) | | default: NULL |
| last_device | int(11) | | default: NULL |
| last_login | datetime | | default: NULL |
| integration_client_id | int(11) | | default: NULL |
| keycloak_uuid | char(36) | | default: NULL |
| external_roles | varchar(255) | | default: NULL |
| address | varchar(255) | | default: NULL |
| manufacture_id | int(11) | | default: NULL |
| village_id | varchar(255) | | default: NULL |
| external_properties | longtext | | default: NULL |
| created_at | datetime | NOT NULL | default: current_timestamp() |
| updated_at | datetime | | default: NULL |
| deleted_at | datetime | | default: NULL |
| created_by | bigint(20) | | default: NULL |
| updated_by | bigint(20) | | default: NULL |
| deleted_by | bigint(20) | | default: NULL |
| is_active | tinyint(1) | | default: 1 |

---

## Data Integration

The data integration mechanism is achieved by synchronising the considered variables:

1. `user_uuid` from SMILE User Database ↔ `appUserId` in Keycloak User Database  
2. `keycloak_uuid` from SMILE User Database ↔ `id` from Keycloak User Database  
3. `program_id` from Keycloak User Database ↔ `user_workspaces` table in SMILE Database  
4. `role` from Keycloak User Database ↔ `user_role` table in SMILE Database  

For more information on the `user_workspaces` and `user_role` tables, please refer to the **Database Tables Overview** document.

---

## Features Validation Flow

The Feature Validation Flow explains the process of data validation within the authentication system through a structured validation mechanism, ensuring security, consistency, and compliance with system requirements.

---

### Login

**Features**
- Login | Web
- Login | Mobile

**Validation Flow**
1. User inputs username and password.
2. Keycloak checks if the username exists.
3. Keycloak validates username and password.
4. Upon success, Keycloak issues a JWT token.
5. User is logged in.

---

### Logout

**Features**
- Logout | Web
- Logout | Mobile

**Validation Flow**
1. User clicks Logout.
2. Keycloak validates the authorization token.
3. Keycloak revokes the token.
4. User is logged out.
5. Page redirects to Login.

---

### Create User

**Features**
- Create User | Global Setting

**Validation Flow**
1. User creation in SMILE Database.
2. User creation in Keycloak Database.
3. Data synchronization between both databases.

#### User Creation in SMILE Database
1. Super Admin inputs new user data.
2. System checks username/email uniqueness.
3. System validates input format.
4. User created with `keycloak_uuid = NULL`.

#### User Creation in Keycloak Database
1. Keycloak checks username uniqueness.
2. User data is created.
3. Input format is validated.
4. Role existence is checked.
5. Role is created if not existing.

#### Data Synchronization
1. System matches `user_uuid` with `appUserId`.
2. `keycloak_uuid` is updated.
3. User receives successful creation notification.

---

### Edit User & Update Status User

**Features**
- Edit User | Global Setting
- Update Status User | Global Setting

**Validation Flow**
1. Super Admin inputs updated data.
2. System validates user ID.
3. System validates data format.
4. User data is updated in both databases.

