---
title: advanced_password_check 
---

The `advanced_password_check` module allows you to strenghten password policies for Greenplum Database.

The Greenplum Database `advanced_password_check` module is based on the `passwordcheck_extra` module, which enhances the PostgreSQL `passwordcheck` module to support user-defined policies to strengthen `passwordcheck`'s minimum password requirements.

## <a id="topic_reg"></a>Loading the Module 

The `advanced_password_check` module provides no SQL-accessible functions. To use it, enable the extension as a preloaded library and restart Greenplum Database server:

```
gpconfig -c shared_preload_libraries -v advanced_password_check 
gpstop -ar 
```

Then install the extension in your database:

```
CREATE EXTENSION advanced_password_check;
```

## <a id="topic_using"></a>Using the advanced\_password\_check Module 

`advanced_password_check` is a Greenplum Database module that you can enable and configure to check password strings against one or more user-defined policies. You can configure policies that:

-   Set a minimum password string length.
-   Set a maximum password string length.
-   Set a maximum password age before it requires renewing.
-   Set rules for reusing an old password: number of days or number of previous passwords before a user can reuse a password.
-   Set maximum number of failed log in attempts before a user is locked, and number of minutes the user remains locked.
-   Define a custom list of special characters.
-   Define rules for special character, upper/lower case character, and number inclusion in the password string.
-   Use UDFs to manage an exception list for each feature so certain users can bypass the current restrictions.

The `advanced_password_check` module defines server configuration parameters that you set to configure password setting policies. These parameters include:

|Parameter Name|Type|Default Value|Description|
|--------------|----|-------------|-----------|
|lockout_duration|int|0|Number of minutes a user is locked after failing to log in the number of times set by password_login_attempts. If set to 0, user is locked indefinitely.|
|minimum\_length|int|8|The minimum allowable length of a Greenplum Database password.|
|maximum\_length|int|15|The maximum allowable length of Greenplum Database password.|
|password_login_attempts|int|0|Number of failed log in attempts before user is locked. If set to 0, this feature is disable.|
|password_max_age|int|0|The maximum amount of days before the password expires. If set to 0, password does not expire.|
|password_reuse_days|int|0|Number of days before a user can reuse a password. If set to 0, user can reuse any password.|
|password_reuse_history|int|0|Number of previous passwords user must not reuse. If set to 0, user can reuse any password.|
|special\_chars|string|!@\#$%^&\*\(\)\_+\{\}\|<\>?=|The set of characters that Greenplum Database considers to be special characters in a password.|
|restrict\_upper|bool|true|Specifies whether or not the password string must contain at least one upper case character.|
|restrict\_lower|bool|true|Specifies whether or not the password string must contain at least one lower case character.|
|restrict\_numbers|bool|true|Specifies whether or not the password string must contain at least one number.|
|restrict\_special|bool|true|Specifies whether or not the password string must contain at least one special character.|

After you define your password policies, you run the `gpconfig` command for each configuration parameter that you must set. When you run the command, you must qualify the parameter with the module name. For example, to configure Greenplum Database to remove any requirements for a lower case letter in the password string, you run the following command:

```
gpconfig -c advanced_password_check.restrict_lower -v false
```

After you set or change module configuration in this manner, you must reload the Greenplum Database configuration:

```
gpstop -u
```

The `advanced_password_check` module provides the following user-defined functions (UDF):

Use the `manage_exception_list()` user-defined function to ------. The function takes three arguments: the action to take, the role name and the name of the exception. The value of `action` can be add, remove, or show. The value of `exception_type` can be password_max_age, password_reuse_days, password_reuse_history, password_login_attempts, or an empty string to represent all of the available exception types. When the action is show, you may specify `role_name` as an empty string to include all roles.

The following example 

```
SELECT manage_exception_list('add', 'test_user', 'password_max_age');
```
 

|Function Signature|Arguments|Description|
|--------|----------|-----------|
|`manage_exception_list(action, role_name, exception_type)`|`action`= `add`/`remove`/`show` <br> `role_name` <br>`exception_type`=`password_max_age`/`password_reuse_days`/`password_reuse_history`/`password_login_attempts` |Adds, removes, and shows roles in the exception list for the different password features.|
|unblock_account(role_name)|`role_name`|Unblocks a user.|
|status()||Lists names and values of the password policies in place.|




## <a id="topic_example"></a>Example 

Suppose that you have defined the following password policies:

-   The password must contain a minimum of 10 characters and a maximum of 18.
-   The password must contain a mixture of upper case and lower case characters.
-   The password must contain at least one of the default special characters.
-   The are no requirements that the password contain a number.

You would run the following commands to configure Greenplum Database to enforce these policies:

```
gpadmin@gpmaster$ gpconfig -c advanced_password_check.minimum_length -v 10
gpadmin@gpmaster$ gpconfig -c advanced_password_check.maximum_length -v 18
gpadmin@gpmaster$ gpconfig -c advanced_password_check.restrict_numbers -v false
gpadmin@gpmaster$ gpstop -u
```

After loading the new configuration, passwords that the Greenplum superuser sets must now follow the policies, and Greenplum returns an error for every policy that is not met. Note that Greenplum checks the password string against all of the policies, and concatenates together the messages for any errors that it encounters. For example \(line breaks added for better viewability\):

```
# testdb=# CREATE role r1 PASSWORD '12345678901112';
ERROR:  Incorrect password format: lower-case character missing, upper-case character
missing, special character missing (needs to be one listed in "<list-of-special-chars>")
```

## <a id="topic_info"></a>Additional Module Documentation 

Refer to the [passwordcheck](https://www.postgresql.org/docs/9.4/passwordcheck.html) PostgreSQL documentation for more information about this module.

