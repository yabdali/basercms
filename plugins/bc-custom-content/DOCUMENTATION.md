# bc-custom-content Plugin: Technical Documentation

This document provides a detailed technical overview of the `bc-custom-content` plugin for baserCMS. It covers the plugin's core concepts, architecture, data model, and the system for creating custom field types.

---

## 1. Core Concepts

The `bc-custom-content` plugin is built around a set of core concepts that work together to provide a flexible and extensible custom content system.

*   **Custom Content (`CustomContent`):** Represents the content item itself, linked to the core `Content` entity of baserCMS. It holds configuration like the template to use, list count, etc. This is the bridge between a custom table definition and a specific page on the website.

*   **Custom Table (`CustomTable`):** Represents a custom content type. It defines the name, title, and other settings for the content type. Each `CustomTable` corresponds to a new database table that stores the entries for that content type.

*   **Custom Field (`CustomField`):** Represents a reusable field that can be added to a `CustomTable`. It defines the field's name, type (e.g., text, textarea, file), validation rules, and other metadata.

*   **Custom Link (`CustomLink`):** This is the pivot entity that links a `CustomField` to a `CustomTable`. It allows a field to be used in multiple tables with different settings (e.g., different title, validation rules). It also defines the layout and display options for the field within the context of a specific table.

*   **Custom Entry (`CustomEntry`):** This entity represents a single entry/record of a `CustomTable`. The actual data for the custom fields is stored in a dynamically created table.

---

## 2. High-Level Architectural Overview

The plugin integrates tightly with the baserCMS core, extending its generic content management capabilities to provide a powerful custom content system.

#### Architectural Diagram

```
+-----------------+      +--------------------+      +-------------------+      +-----------------+
|   Controllers   |----->|      Services      |----->|       Tables      |----->|    Database     |
|  (Admin & API)  |      | (Business Logic)   |      | (ORM / Data Access)|      |(Physical Metadata|
+-----------------+      +--------------------+      +-------------------+      |     Tables)     |
        |                        ^       ^                       |
        |                        |       |                       v
        v                        |       |      +--------------------------------+
+-----------------+              |       |      | Dynamically Created Entry Tables |
|      Views      |              |       |      | (e.g., custom_entries_1_blogs) |
| (Admin Templates|              |       |      +--------------------------------+
| in bc-admin-third)|            |       |
+-----------------+              |       |
        ^                        |       +------------------------------------+
        |                        |                                            |
        |  Renders UI based on   |                                            |
        |  config & data         |            +-----------------------------+ |
        |                        |            | baser-core `Contents` Table | |
        +------------------------+            +-----------------------------+ |
                                                        ^                       |
                                                        | Polymorphic Relation  |
                                                        | (plugin, type,        |
                                                        |  entity_id)           |
                                                        v                       |
                                              +---------------------+           |
                                              | `custom_contents`   |           |
                                              +---------------------+           |
                                                                                |
                       Loads Config from <--------------------------------------+
                                         |
                +--------------------------+
                | Field Type Sub-Plugins   |
                | (e.g., BcCcText)         |
                +--------------------------+

```

#### How It All Works Together: A Request Flow Example

1.  A user creates a new "Custom Content" page in the admin panel.
2.  A new record is created in the core `contents` table with `plugin` = `'BcCustomContent'` and `type` = `'CustomContent'`.
3.  A corresponding record is created in the `custom_contents` table, which stores the settings for this page (e.g., which `CustomTable` to use). The `entity_id` of the `contents` record points to the ID of this `custom_contents` record.
4.  When a request is made to the page's URL, baserCMS routes it to the `bc-custom-content` plugin.
5.  The plugin reads the `custom_contents` record to get the settings.
6.  It then fetches the structure from the `custom_tables` and `custom_links` tables.
7.  Finally, it queries the dynamic `custom_entries_*` table for the data and renders the page using the specified template.

---

## 3. Database Schema

The plugin's functionality is built on a set of metadata tables that define the structure and a set of dynamic tables that store the data.

#### ER Diagram

```
+-----------------+      +-------------------+
|    contents     |      | custom_contents   |
| (baser-core)    |----->| (PK: id)          |
+-----------------+      | custom_table_id(FK)|
| id (PK)         |      | ...               |
| plugin          |      +-------------------+
| type            |               |
| entity_id (FK)  |               |
| ...             |               v
+-----------------+      +-------------------+      +-------------------+      +--------------------+
                       |   custom_tables   |      |   custom_links    |      |    custom_fields   |
                       +-------------------+      +-------------------+      +--------------------+
                       | id (PK)           |      | id (PK)           |      | id (PK)            |
                       | name              |---<  | custom_table_id(FK)|      | name               |
                       | title             |      | custom_field_id(FK)|----->| title              |
                       | ...               |      | ...               |      | type               |
                       +-------------------+      +-------------------+      | ...                |
                                                                             +--------------------+
```

### Core Integration Table

*   **`contents`**: This is a core table in baserCMS. `bc-custom-content` creates records here to register its pages with the core system. The `entity_id` column links to the `custom_contents` table.

### Plugin Metadata Tables

*   **`custom_contents`**: Stores the configuration for each custom content page, such as the template to use, list count, and a foreign key to the `custom_tables` table.
*   **`custom_tables`**: Defines the custom content types, including their name and title.
*   **`custom_fields`**: Stores the definitions for the reusable custom fields.
*   **`custom_links`**: Acts as a pivot table, linking `custom_fields` to `custom_tables`.

### Dynamic Data Tables

*   **`custom_entries_{id}_{name}`**: For each record in `custom_tables`, a new database table is created with this naming convention. These tables have columns that correspond to the `CustomLink`s defined for that `CustomTable`, and they store the actual content entries.

---

## 4. Core Logic and Service Layer

The plugin's logic is cleanly organized into a service layer.

*   **`CustomTablesService`**: Manages the metadata of the `custom_tables`. It orchestrates the creation and deletion of the physical entry tables by calling the `CustomEntriesService`.
*   **`CustomLinksService`**: Manages the fields within a custom table. When a field is created or deleted, it calls `CustomEntriesService` to add or remove the corresponding column from the physical database table.
*   **`CustomEntriesService`**: The workhorse of the plugin. It directly manages the `custom_entries_*` tables, using the `BcDatabaseService` for low-level schema manipulations and handling all CRUD operations for the entries.

---

## 5. The Custom Field System

The plugin is highly extensible, allowing developers to add new field types by creating simple sub-plugins.

### 5.1. Structure of a Custom Field Plugin

A custom field plugin is a standard CakePHP plugin with a specific structure:

*   **`config.php`**: Contains basic metadata. The `type` must be `['BcCustomContentPlugin']`.
*   **`config/setting.php`**: The core of the sub-plugin, defining the field's properties.
*   **`src/Plugin.php`**: The main plugin class, often minimal.
*   **`templates/` (Optional)**: For providing custom UI for the field's settings or form control.

### 5.2. The `setting.php` File

This file defines how the field type behaves:

```php
'BcCustomContent' => [
    'fieldTypes' => [
        'BcCcText' => [ // Key must match the plugin name
            'label' => 'テキスト',
            'columnType' => 'string', // DB column type
            'controlType' => 'text',  // FormHelper control type
            'useSize' => true,        // Enables the 'size' setting in the admin
        ]
    ]
]
```

### 5.3. How to Create a New Custom Field Plugin

1.  **Create the Plugin Directory:** In `plugins/bc-custom-content/plugins/`, create a new directory for your plugin.
2.  **Add `config.php` and `src/Plugin.php`:** Create the basic plugin files.
3.  **Create `config/setting.php`:** Define the `label`, `columnType`, `controlType`, and any `use...` flags for your new field.
4.  **Activate the Plugin:** Your new field type will now be available in baserCMS.
