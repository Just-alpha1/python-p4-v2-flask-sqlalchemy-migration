# Flask-Migrate Tutorial: Complete Schema Migration Guide

This project demonstrates how to use Flask-Migrate (which uses Alembic behind the scenes) to manage database schema changes in a Flask application.

## Learning Goals

- Use Flask-Migrate to manage changes to a database schema
- Create migrations for different types of schema modifications
- Upgrade a schema from an old version to a newer version
- Roll back, or downgrade a schema from a newer version to an older one

## Key Concepts

### Schema
The blueprint of a database. Describes how data relates to other data in tables, columns, and relationships between them.

### Schema Migration
The process of moving a schema from one version to another.

## Project Structure

```
server/
├── app.py                    # Flask application
├── models.py                 # Database models
├── instance/
│   └── app.db               # SQLite database file
├── migrations/              # Flask-Migrate directory
│   ├── README              # Migration documentation
│   ├── alembic.ini         # Alembic configuration
│   ├── env.py              # Migration environment setup
│   ├── script.py.mako      # Template for new migration files
│   └── versions/           # Individual migration scripts
│       ├── 15537423c56d_initial_migration.py
│       ├── 51f20aa4768b_add_department.py
│       ├── 1694ecedb24d_rename_department_wrong.py
│       ├── 1694ecedb24d_rename_department_corrected.py
│       ├── 76f31678b786_rename_address_wrong.py
│       └── 76f31678b786_rename_address_corrected.py
└── testing/
    └── codegrade_test.py   # Test files
```

## Setup Instructions

1. **Install Dependencies**
   ```bash
   pipenv install
   pipenv shell
   ```

2. **Configure Environment**
   ```bash
   cd server
   export FLASK_APP=app.py
   export FLASK_RUN_PORT=5555
   ```

3. **Initialize Migration Repository**
   ```bash
   flask db init
   ```

## Tutorial Progression

### 1. Initial Migration - Employee Model

**Migration File:** `15537423c56d_initial_migration.py`

Creates the initial `employees` table with:
- `id` (Integer, Primary Key)
- `name` (String, Not Null)
- `salary` (Integer, Nullable)

**Commands:**
```bash
flask db migrate -m "Initial migration."
flask db upgrade head
```

**Add Sample Data:**
```bash
flask shell
>>> from models import db, Employee
>>> db.session.add(Employee(name="Kai Uri", salary=89000))
>>> db.session.add(Employee(name="Alena Lee", salary=125000))
>>> db.session.commit()
>>> Employee.query.all()
```

### 2. Second Migration - Department Model

**Migration File:** `51f20aa4768b_add_department.py`

Adds a new `department` table (intentionally using singular name to demonstrate fixing naming errors).

**Commands:**
```bash
flask db migrate -m "add Department"
flask db upgrade head
```

### 3. Third Migration - Table Rename

**Problem:** Flask-Migrate can't auto-generate table renames correctly.

**Wrong Migration:** `1694ecedb24d_rename_department_wrong.py`
- Tries to DROP the table and CREATE a new one
- **This would lose all data!**

**Corrected Migration:** `1694ecedb24d_rename_department_corrected.py`
- Uses `op.rename_table('department', 'departments')`
- Preserves all data

**Edit the migration file to use the corrected version:**
```python
def upgrade():
    op.rename_table('department', 'departments')

def downgrade():
    op.rename_table('departments', 'department')
```

### 4. Fourth Migration - Column Rename

**Problem:** Flask-Migrate can't auto-generate column renames correctly.

**Wrong Migration:** `76f31678b786_rename_address_wrong.py`
- Uses `batch_alter_table` to ADD and DROP columns
- **This would lose all data in the column!**

**Corrected Migration:** `76f31678b786_rename_address_corrected.py`
- Uses `op.alter_column()` with `new_column_name` parameter
- Preserves all data

**Edit the migration file to use the corrected version:**
```python
def upgrade():
    op.alter_column('departments', 'address', new_column_name='location')

def downgrade():
    op.alter_column('departments', 'location', new_column_name='address')
```

## Flask-Migrate Capabilities

### What Flask-Migrate CAN do automatically:
- ✅ Creating and dropping tables
- ✅ Creating and dropping columns
- ✅ Most indexing tasks
- ✅ Adding/removing constraints (named)

### What Flask-Migrate CANNOT do automatically:
- ❌ Table name changes (use `op.rename_table()`)
- ❌ Column name changes (use `op.alter_column()` with `new_column_name`)
- ❌ Adding/removing unnamed constraints
- ❌ Converting unsupported Python data types

## Migration Commands Reference

```bash
# Initialize migration repository
flask db init

# Create new migration
flask db migrate -m "descriptive message"

# Apply migrations (upgrade to latest)
flask db upgrade

# Apply migrations to specific version
flask db upgrade <revision_id>

# Apply migrations to head (latest version)
flask db upgrade head

# Revert migrations (downgrade)
flask db downgrade

# Revert migrations to specific version
flask db downgrade <revision_id>

# Show current version
flask db current

# Show migration history
flask db history

# Show help
flask db --help
```

## Best Practices

1. **Always review migration scripts** before applying them
2. **Never apply migrations directly to production** without testing
3. **Backup your database** before running migrations in development
4. **Use descriptive migration messages**
5. **Test migrations in development first**
6. **Be aware of data loss risks** with auto-generated migrations

## Common Migration Patterns

### Adding a new table:
```python
def upgrade():
    op.create_table('new_table',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('name', sa.String(), nullable=False),
        sa.PrimaryKeyConstraint('id')
    )
```

### Adding a new column:
```python
def upgrade():
    with op.batch_alter_table('existing_table') as batch_op:
        batch_op.add_column(sa.Column('new_column', sa.String()))
```

### Renaming a table:
```python
def upgrade():
    op.rename_table('old_table_name', 'new_table_name')
```

### Renaming a column:
```python
def upgrade():
    op.alter_column('table_name', 'old_column_name', 
                   new_column_name='new_column_name')
```

### Adding a foreign key:
```python
def upgrade():
    op.create_foreign_key('fk_name', 'child_table', 'parent_table',
                         ['parent_id'], ['id'])
```

## Understanding Migration Files

Each migration file contains:
- **Revision ID**: Unique identifier for this migration
- **Revises**: The ID of the previous migration this builds upon
- **upgrade()**: Function to apply the migration
- **downgrade()**: Function to revert the migration

The `down_revision` chain ensures migrations are applied in the correct order.

## Data Preservation

**ALWAYS** prefer data-preserving operations:
- Use `op.rename_table()` instead of `DROP TABLE + CREATE TABLE`
- Use `op.alter_column()` with `new_column_name` instead of `DROP COLUMN + ADD COLUMN`
- Test migrations with sample data to ensure data integrity

## Conclusion

Flask-Migrate is a powerful tool for managing database schema changes in Flask applications. While it can automatically handle many common operations, understanding when and how to manually edit migration scripts is crucial for maintaining data integrity and implementing advanced schema changes.

The examples in this project demonstrate both correct and incorrect approaches to various migration scenarios, providing a comprehensive learning resource for database schema management in Flask applications.
