# MySQL Data Models & Schema

This document defines the Sequelize models for the Lead Management System.

## Database Configuration

```javascript
// config/database.js
module.exports = {
  development: {
    username: process.env.DB_USER || 'root',
    password: process.env.DB_PASSWORD || 'password',
    database: process.env.DB_NAME || 'travel_leads_dev',
    host: process.env.DB_HOST || 'localhost',
    port: process.env.DB_PORT || 3306,
    dialect: 'mysql',
    logging: console.log,
    pool: {
      max: 10,
      min: 2,
      acquire: 30000,
      idle: 10000
    },
    define: {
      timestamps: true,
      underscored: false,
      freezeTableName: true
    }
  },
  production: {
    username: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    host: process.env.DB_HOST,
    port: process.env.DB_PORT || 3306,
    dialect: 'mysql',
    logging: false,
    pool: {
      max: 20,
      min: 5,
      acquire: 30000,
      idle: 10000
    },
    define: {
      timestamps: true,
      underscored: false,
      freezeTableName: true
    }
  }
};
```

## Sequelize Models

### 1. Organization Model

```javascript
// models/Organization.js
'use strict';
const { DataTypes } = require('sequelize');

module.exports = (sequelize) => {
  const Organization = sequelize.define(
    'Organization',
    {
      org_id: {
        type: DataTypes.UUID,
        defaultValue: DataTypes.UUIDV4,
        primaryKey: true
      },
      org_name: {
        type: DataTypes.STRING(200),
        allowNull: false,
        validate: {
          notEmpty: true,
          len: [3, 200]
        }
      },
      subscription_tier: {
        type: DataTypes.ENUM('FREE', 'STARTER', 'PROFESSIONAL', 'ENTERPRISE'),
        defaultValue: 'STARTER',
        allowNull: false
      },
      createdAt: {
        type: DataTypes.DATE,
        defaultValue: DataTypes.NOW
      },
      updatedAt: {
        type: DataTypes.DATE,
        defaultValue: DataTypes.NOW
      }
    },
    {
      tableName: 'organizations',
      timestamps: true
    }
  );

  Organization.associate = (models) => {
    Organization.hasMany(models.User, {
      foreignKey: 'org_id',
      onDelete: 'CASCADE'
    });
    Organization.hasMany(models.Lead, {
      foreignKey: 'org_id',
      onDelete: 'CASCADE'
    });
    Organization.hasMany(models.Quote, {
      foreignKey: 'org_id',
      onDelete: 'CASCADE'
    });
    Organization.hasMany(models.AuditLog, {
      foreignKey: 'org_id',
      onDelete: 'CASCADE'
    });
  };

  return Organization;
};
```

### 2. User Model

```javascript
// models/User.js
'use strict';
const { DataTypes } = require('sequelize');
const bcrypt = require('bcrypt');

module.exports = (sequelize) => {
  const User = sequelize.define(
    'User',
    {
      user_id: {
        type: DataTypes.UUID,
        defaultValue: DataTypes.UUIDV4,
        primaryKey: true
      },
      org_id: {
        type: DataTypes.UUID,
        allowNull: false,
        references: {
          model: 'organizations',
          key: 'org_id'
        },
        onDelete: 'CASCADE'
      },
      email: {
        type: DataTypes.STRING(254),
        allowNull: false,
        unique: 'unique_email_per_org',
        validate: {
          isEmail: true
        }
      },
      password_hash: {
        type: DataTypes.STRING(255),
        allowNull: false,
        get() {
          return undefined; // Never expose password
        }
      },
      name: {
        type: DataTypes.STRING(100),
        allowNull: false,
        validate: {
          notEmpty: true
        }
      },
      role: {
        type: DataTypes.ENUM('ADMIN', 'MANAGER', 'AGENT', 'VIEWER'),
        defaultValue: 'AGENT',
        allowNull: false
      },
      is_active: {
        type: DataTypes.BOOLEAN,
        defaultValue: true
      },
      last_login_at: {
        type: DataTypes.DATE,
        allowNull: true
      },
      createdAt: {
        type: DataTypes.DATE,
        defaultValue: DataTypes.NOW
      },
      updatedAt: {
        type: DataTypes.DATE,
        defaultValue: DataTypes.NOW
      }
    },
    {
      tableName: 'users',
      timestamps: true,
      hooks: {
        beforeCreate: async (user) => {
          if (user.password_hash) {
            const salt = await bcrypt.genSalt(10);
            user.password_hash = await bcrypt.hash(user.password_hash, salt);
          }
        },
        beforeUpdate: async (user) => {
          if (user.changed('password_hash')) {
            const salt = await bcrypt.genSalt(10);
            user.password_hash = await bcrypt.hash(user.password_hash, salt);
          }
        }
      }
    }
  );

  User.prototype.comparePassword = async function (password) {
    return bcrypt.compare(password, this.password_hash);
  };

  User.associate = (models) => {
    User.belongsTo(models.Organization, {
      foreignKey: 'org_id'
    });
    User.hasMany(models.Lead, {
      as: 'leads_created',
      foreignKey: 'created_by',
      onDelete: 'RESTRICT'
    });
    User.hasMany(models.Lead, {
      as: 'leads_assigned',
      foreignKey: 'assigned_to',
      onDelete: 'SET NULL'
    });
    User.hasMany(models.AuditLog, {
      foreignKey: 'changed_by',
      onDelete: 'RESTRICT'
    });
  };

  return User;
};
```

### 3. Lead Model

```javascript
// models/Lead.js
'use strict';
const { DataTypes } = require('sequelize');

module.exports = (sequelize) => {
  const Lead = sequelize.define(
    'Lead',
    {
      lead_id: {
        type: DataTypes.UUID,
        defaultValue: DataTypes.UUIDV4,
        primaryKey: true
      },
      org_id: {
        type: DataTypes.UUID,
        allowNull: false,
        references: {
          model: 'organizations',
          key: 'org_id'
        },
        onDelete: 'CASCADE'
      },
      lead_name: {
        type: DataTypes.STRING(100),
        allowNull: false,
        validate: {
          notEmpty: true,
          len: [1, 100]
        }
      },
      email: {
        type: DataTypes.STRING(254),
        allowNull: false,
        validate: {
          isEmail: true
        }
      },
      phone: {
        type: DataTypes.STRING(20),
        allowNull: false,
        validate: {
          is: /^\+?[1-9]\d{1,14}$/ // E.164 format
        }
      },
      country_code: {
        type: DataTypes.CHAR(2),
        allowNull: false,
        validate: {
          len: [2, 2]
        }
      },
      travel_date: {
        type: DataTypes.DATE,
        allowNull: false,
        validate: {
          isDate: true,
          isAfterToday(value) {
            if (new Date(value) < new Date()) {
              throw new Error('Travel date must be in the future');
            }
          }
        }
      },
      duration_days: {
        type: DataTypes.INTEGER,
        allowNull: true,
        validate: {
          isInt: true,
          min: 1,
          max: 365
        }
      },
      destination: {
        type: DataTypes.STRING(100),
        allowNull: false,
        validate: {
          notEmpty: true
        }
      },
      travel_type: {
        type: DataTypes.ENUM('DOMESTIC', 'INTERNATIONAL'),
        allowNull: false
      },
      num_travelers: {
        type: DataTypes.INTEGER,
        allowNull: false,
        validate: {
          isInt: true,
          min: 1,
          max: 500
        }
      },
      budget: {
        type: DataTypes.DECIMAL(12, 2),
        allowNull: true,
        validate: {
          isDecimal: true,
          min: 0
        }
      },
      lead_source: {
        type: DataTypes.ENUM('WEBSITE', 'PHONE', 'EMAIL', 'REFERRAL', 'SOCIAL', 'OTHER'),
        allowNull: false
      },
      status: {
        type: DataTypes.ENUM('NEW', 'CONTACTED', 'QUOTED', 'NEGOTIATING', 'BOOKED', 'REJECTED', 'ARCHIVED', 'CLOSED'),
        defaultValue: 'NEW',
        allowNull: false
      },
      notes: {
        type: DataTypes.TEXT,
        allowNull: true
      },
      assigned_to: {
        type: DataTypes.UUID,
        allowNull: true,
        references: {
          model: 'users',
          key: 'user_id'
        },
        onDelete: 'SET NULL'
      },
      created_by: {
        type: DataTypes.UUID,
        allowNull: false,
        references: {
          model: 'users',
          key: 'user_id'
        },
        onDelete: 'RESTRICT'
      },
      last_contacted_at: {
        type: DataTypes.DATE,
        allowNull: true
      },
      createdAt: {
        type: DataTypes.DATE,
        defaultValue: DataTypes.NOW
      },
      updatedAt: {
        type: DataTypes.DATE,
        defaultValue: DataTypes.NOW
      },
      deleted_at: {
        type: DataTypes.DATE,
        allowNull: true
      }
    },
    {
      tableName: 'leads',
      timestamps: true,
      paranoid: false // Manual soft delete handling
    }
  );

  Lead.associate = (models) => {
    Lead.belongsTo(models.Organization, {
      foreignKey: 'org_id'
    });
    Lead.belongsTo(models.User, {
      as: 'creator',
      foreignKey: 'created_by'
    });
    Lead.belongsTo(models.User, {
      as: 'assignee',
      foreignKey: 'assigned_to'
    });
    Lead.hasOne(models.Quote, {
      foreignKey: 'lead_id',
      onDelete: 'RESTRICT'
    });
    Lead.hasMany(models.AuditLog, {
      foreignKey: 'entity_id'
    });
  };

  // Scope for excluding soft-deleted records
  Lead.addScope('active', {
    where: { deleted_at: null }
  });

  return Lead;
};
```

### 4. Quote Model

```javascript
// models/Quote.js
'use strict';
const { DataTypes } = require('sequelize');

module.exports = (sequelize) => {
  const Quote = sequelize.define(
    'Quote',
    {
      quote_id: {
        type: DataTypes.UUID,
        defaultValue: DataTypes.UUIDV4,
        primaryKey: true
      },
      lead_id: {
        type: DataTypes.UUID,
        allowNull: false,
        unique: true,
        references: {
          model: 'leads',
          key: 'lead_id'
        },
        onDelete: 'RESTRICT'
      },
      org_id: {
        type: DataTypes.UUID,
        allowNull: false,
        references: {
          model: 'organizations',
          key: 'org_id'
        },
        onDelete: 'CASCADE'
      },
      status: {
        type: DataTypes.ENUM('PENDING_APPROVAL', 'SENT', 'ACCEPTED', 'REJECTED', 'EXPIRED'),
        defaultValue: 'PENDING_APPROVAL',
        allowNull: false
      },
      total_amount: {
        type: DataTypes.DECIMAL(12, 2),
        allowNull: true,
        validate: {
          isDecimal: true,
          min: 0
        }
      },
      expires_at: {
        type: DataTypes.DATE,
        defaultValue: () => {
          const date = new Date();
          date.setDate(date.getDate() + 7);
          return date;
        }
      },
      createdAt: {
        type: DataTypes.DATE,
        defaultValue: DataTypes.NOW
      },
      updatedAt: {
        type: DataTypes.DATE,
        defaultValue: DataTypes.NOW
      }
    },
    {
      tableName: 'quotes',
      timestamps: true
    }
  );

  Quote.associate = (models) => {
    Quote.belongsTo(models.Lead, {
      foreignKey: 'lead_id'
    });
    Quote.belongsTo(models.Organization, {
      foreignKey: 'org_id'
    });
  };

  return Quote;
};
```

### 5. AuditLog Model

```javascript
// models/AuditLog.js
'use strict';
const { DataTypes } = require('sequelize');

module.exports = (sequelize) => {
  const AuditLog = sequelize.define(
    'AuditLog',
    {
      audit_id: {
        type: DataTypes.BIGINT,
        primaryKey: true,
        autoIncrement: true
      },
      org_id: {
        type: DataTypes.UUID,
        allowNull: false,
        references: {
          model: 'organizations',
          key: 'org_id'
        },
        onDelete: 'CASCADE'
      },
      entity_type: {
        type: DataTypes.STRING(50),
        allowNull: false, // LEAD, QUOTE, USER
        validate: {
          notEmpty: true
        }
      },
      entity_id: {
        type: DataTypes.STRING(36),
        allowNull: false
      },
      action: {
        type: DataTypes.STRING(50),
        allowNull: false, // CREATE, UPDATE, DELETE, STATUS_CHANGE, CONVERT
        validate: {
          notEmpty: true
        }
      },
      old_value: {
        type: DataTypes.JSON,
        allowNull: true
      },
      new_value: {
        type: DataTypes.JSON,
        allowNull: true
      },
      changed_by: {
        type: DataTypes.UUID,
        allowNull: false,
        references: {
          model: 'users',
          key: 'user_id'
        },
        onDelete: 'RESTRICT'
      },
      ip_address: {
        type: DataTypes.STRING(45),
        allowNull: true
      },
      user_agent: {
        type: DataTypes.STRING(500),
        allowNull: true
      },
      request_id: {
        type: DataTypes.UUID,
        allowNull: true
      },
      changed_at: {
        type: DataTypes.DATE,
        defaultValue: DataTypes.NOW
      },
      retention_until: {
        type: DataTypes.DATE,
        defaultValue: () => {
          const date = new Date();
          date.setFullYear(date.getFullYear() + 5);
          return date;
        }
      }
    },
    {
      tableName: 'audit_logs',
      timestamps: false
    }
  );

  AuditLog.associate = (models) => {
    AuditLog.belongsTo(models.Organization, {
      foreignKey: 'org_id'
    });
    AuditLog.belongsTo(models.User, {
      foreignKey: 'changed_by'
    });
  };

  return AuditLog;
};
```

### 6. Model Index & Associations

```javascript
// models/index.js
'use strict';
const fs = require('fs');
const path = require('path');
const { Sequelize } = require('sequelize');
const config = require('../config/database')[process.env.NODE_ENV || 'development'];

const sequelize = new Sequelize(config);
const db = {};

// Load all models
fs.readdirSync(__dirname)
  .filter((file) => file.endsWith('.js') && file !== 'index.js')
  .forEach((file) => {
    const model = require(path.join(__dirname, file))(sequelize);
    db[model.name] = model;
  });

// Setup associations
Object.keys(db).forEach((modelName) => {
  if (db[modelName].associate) {
    db[modelName].associate(db);
  }
});

db.sequelize = sequelize;
db.Sequelize = Sequelize;

module.exports = db;
```

## Migration Files

### Migration 1: Create Organizations

```javascript
// migrations/20260131100000-create-organizations.js
'use strict';

module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('organizations', {
      org_id: {
        type: Sequelize.UUID,
        defaultValue: Sequelize.UUIDV4,
        primaryKey: true
      },
      org_name: {
        type: Sequelize.STRING(200),
        allowNull: false
      },
      subscription_tier: {
        type: Sequelize.ENUM('FREE', 'STARTER', 'PROFESSIONAL', 'ENTERPRISE'),
        defaultValue: 'STARTER'
      },
      createdAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.NOW
      },
      updatedAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.NOW
      }
    });
  },

  down: async (queryInterface) => {
    await queryInterface.dropTable('organizations');
  }
};
```

### Migration 2: Create Users

```javascript
// migrations/20260131100001-create-users.js
'use strict';

module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('users', {
      user_id: {
        type: Sequelize.UUID,
        defaultValue: Sequelize.UUIDV4,
        primaryKey: true
      },
      org_id: {
        type: Sequelize.UUID,
        allowNull: false,
        references: {
          model: 'organizations',
          key: 'org_id'
        },
        onDelete: 'CASCADE'
      },
      email: {
        type: Sequelize.STRING(254),
        allowNull: false
      },
      password_hash: {
        type: Sequelize.STRING(255),
        allowNull: false
      },
      name: {
        type: Sequelize.STRING(100),
        allowNull: false
      },
      role: {
        type: Sequelize.ENUM('ADMIN', 'MANAGER', 'AGENT', 'VIEWER'),
        defaultValue: 'AGENT'
      },
      is_active: {
        type: Sequelize.BOOLEAN,
        defaultValue: true
      },
      last_login_at: {
        type: Sequelize.DATE
      },
      createdAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.NOW
      },
      updatedAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.NOW
      }
    });

    await queryInterface.addConstraint('users', {
      fields: ['org_id', 'email'],
      type: 'unique',
      name: 'unique_email_per_org'
    });

    await queryInterface.addIndex('users', ['org_id', 'role']);
  },

  down: async (queryInterface) => {
    await queryInterface.dropTable('users');
  }
};
```

### Migration 3: Create Leads

```javascript
// migrations/20260131100002-create-leads.js
'use strict';

module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('leads', {
      lead_id: {
        type: Sequelize.UUID,
        defaultValue: Sequelize.UUIDV4,
        primaryKey: true
      },
      org_id: {
        type: Sequelize.UUID,
        allowNull: false,
        references: {
          model: 'organizations',
          key: 'org_id'
        },
        onDelete: 'CASCADE'
      },
      lead_name: {
        type: Sequelize.STRING(100),
        allowNull: false
      },
      email: {
        type: Sequelize.STRING(254),
        allowNull: false
      },
      phone: {
        type: Sequelize.STRING(20),
        allowNull: false
      },
      country_code: {
        type: Sequelize.CHAR(2),
        allowNull: false
      },
      travel_date: {
        type: Sequelize.DATE,
        allowNull: false
      },
      duration_days: {
        type: Sequelize.INTEGER
      },
      destination: {
        type: Sequelize.STRING(100),
        allowNull: false
      },
      travel_type: {
        type: Sequelize.ENUM('DOMESTIC', 'INTERNATIONAL'),
        allowNull: false
      },
      num_travelers: {
        type: Sequelize.INTEGER,
        allowNull: false
      },
      budget: {
        type: Sequelize.DECIMAL(12, 2)
      },
      lead_source: {
        type: Sequelize.ENUM('WEBSITE', 'PHONE', 'EMAIL', 'REFERRAL', 'SOCIAL', 'OTHER'),
        allowNull: false
      },
      status: {
        type: Sequelize.ENUM('NEW', 'CONTACTED', 'QUOTED', 'NEGOTIATING', 'BOOKED', 'REJECTED', 'ARCHIVED', 'CLOSED'),
        defaultValue: 'NEW'
      },
      notes: {
        type: Sequelize.TEXT
      },
      assigned_to: {
        type: Sequelize.UUID,
        references: {
          model: 'users',
          key: 'user_id'
        },
        onDelete: 'SET NULL'
      },
      created_by: {
        type: Sequelize.UUID,
        allowNull: false,
        references: {
          model: 'users',
          key: 'user_id'
        },
        onDelete: 'RESTRICT'
      },
      last_contacted_at: {
        type: Sequelize.DATE
      },
      createdAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.NOW
      },
      updatedAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.NOW
      },
      deleted_at: {
        type: Sequelize.DATE
      }
    });

    // Create indexes
    await queryInterface.addIndex('leads', ['org_id', 'status'], {
      where: { deleted_at: null }
    });
    await queryInterface.addIndex('leads', ['org_id', { name: 'createdAt', order: 'DESC' }], {
      where: { deleted_at: null }
    });
    await queryInterface.addIndex('leads', ['assigned_to', 'status'], {
      where: { deleted_at: null }
    });
    await queryInterface.addIndex('leads', ['travel_date']);
    await queryInterface.addIndex('leads', ['destination']);
    await queryInterface.addIndex('leads', ['email', 'phone']);
  },

  down: async (queryInterface) => {
    await queryInterface.dropTable('leads');
  }
};
```

### Migration 4: Create Quotes

```javascript
// migrations/20260131100003-create-quotes.js
'use strict';

module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('quotes', {
      quote_id: {
        type: Sequelize.UUID,
        defaultValue: Sequelize.UUIDV4,
        primaryKey: true
      },
      lead_id: {
        type: Sequelize.UUID,
        allowNull: false,
        unique: true,
        references: {
          model: 'leads',
          key: 'lead_id'
        },
        onDelete: 'RESTRICT'
      },
      org_id: {
        type: Sequelize.UUID,
        allowNull: false,
        references: {
          model: 'organizations',
          key: 'org_id'
        },
        onDelete: 'CASCADE'
      },
      status: {
        type: Sequelize.ENUM('PENDING_APPROVAL', 'SENT', 'ACCEPTED', 'REJECTED', 'EXPIRED'),
        defaultValue: 'PENDING_APPROVAL'
      },
      total_amount: {
        type: Sequelize.DECIMAL(12, 2)
      },
      expires_at: {
        type: Sequelize.DATE,
        defaultValue: () => {
          const date = new Date();
          date.setDate(date.getDate() + 7);
          return date;
        }
      },
      createdAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.NOW
      },
      updatedAt: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.NOW
      }
    });

    await queryInterface.addIndex('quotes', ['org_id', 'status']);
  },

  down: async (queryInterface) => {
    await queryInterface.dropTable('quotes');
  }
};
```

### Migration 5: Create Audit Logs

```javascript
// migrations/20260131100004-create-audit-logs.js
'use strict';

module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('audit_logs', {
      audit_id: {
        type: Sequelize.BIGINT,
        primaryKey: true,
        autoIncrement: true
      },
      org_id: {
        type: Sequelize.UUID,
        allowNull: false,
        references: {
          model: 'organizations',
          key: 'org_id'
        },
        onDelete: 'CASCADE'
      },
      entity_type: {
        type: Sequelize.STRING(50),
        allowNull: false
      },
      entity_id: {
        type: Sequelize.STRING(36),
        allowNull: false
      },
      action: {
        type: Sequelize.STRING(50),
        allowNull: false
      },
      old_value: {
        type: Sequelize.JSON
      },
      new_value: {
        type: Sequelize.JSON
      },
      changed_by: {
        type: Sequelize.UUID,
        allowNull: false,
        references: {
          model: 'users',
          key: 'user_id'
        },
        onDelete: 'RESTRICT'
      },
      ip_address: {
        type: Sequelize.STRING(45)
      },
      user_agent: {
        type: Sequelize.STRING(500)
      },
      request_id: {
        type: Sequelize.UUID
      },
      changed_at: {
        type: Sequelize.DATE,
        defaultValue: Sequelize.NOW
      },
      retention_until: {
        type: Sequelize.DATE,
        defaultValue: () => {
          const date = new Date();
          date.setFullYear(date.getFullYear() + 5);
          return date;
        }
      }
    });

    await queryInterface.addIndex('audit_logs', ['org_id', 'entity_type', 'entity_id']);
    await queryInterface.addIndex('audit_logs', [{ name: 'changed_at', order: 'DESC' }]);
    await queryInterface.addIndex('audit_logs', ['changed_by']);
  },

  down: async (queryInterface) => {
    await queryInterface.dropTable('audit_logs');
  }
};
```

## Usage in Application

```javascript
// app.js - After database initialization
const db = require('./models');

// Sync models (development only)
if (process.env.NODE_ENV === 'development') {
  db.sequelize.sync({ alter: true });
}

// Query examples
// Find a lead with associated user
const lead = await db.Lead.findByPk(leadId, {
  include: [
    { association: 'creator', attributes: ['user_id', 'name', 'email'] },
    { association: 'assignee', attributes: ['user_id', 'name', 'email'] }
  ]
});

// List leads with pagination
const { count, rows } = await db.Lead.findAndCountAll({
  where: { org_id: orgId, deleted_at: null },
  include: [{ association: 'creator', attributes: ['name'] }],
  limit: 25,
  offset: 0,
  order: [['createdAt', 'DESC']]
});

// Create audit log
await db.AuditLog.create({
  org_id: orgId,
  entity_type: 'LEAD',
  entity_id: leadId,
  action: 'UPDATE',
  old_value: { status: 'NEW' },
  new_value: { status: 'CONTACTED' },
  changed_by: userId,
  ip_address: req.ip,
  user_agent: req.get('user-agent'),
  request_id: req.id
});
```

---

**Version:** 1.0.0  
**Last Updated:** 2026-01-31
