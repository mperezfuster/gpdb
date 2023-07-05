---
title: pgaudit
---

The PostgreSQL Audit Extension, or `pgaudit`, provides detailed session and object audit logging via the standard logging facility provided by PostgreSQL. The goal of PostgreSQL Audit is to provide the tools needed to produce audit logs required to pass certain government, financial, or ISO certification audits.

## <a id="topic_reg"></a>Installing and Registering the Module

The `pgaudit` module is installed when you install Greenplum Database. Before you can use any of the data types defined in the module, you must register the `pgaudit` extension in each database in which you want to use it:

```
CREATE EXTENSION pgaudit;
```

Refer to [Installing Additional Supplied Modules](../../install_guide/install_modules.html) for more information.

## <a id="topic_info"></a>Module Documentation

Refer to the [pgaudit github documentation](https://github.com/pgaudit/pgaudit/blob/master/README.md) for detailed information about using the module.


