# Customizing Schema Generation

::: tip INFO
The Marten team's advice is to keep your Postgresql security usage very simple to reduce friction, but if you can't
follow that advice, Marten has <i>some</i> facility to customize the generated DDL for database security rights.
:::

## Table Creation Style

By default, Marten generates the raw schema export from `IDocumentSchema.ToDDL()` by
writing first a `DROP TABLE` statement, then a `CREATE TABLE` statement. If desired, you can direct Marten
to do this generation by doing a single `CREATE TABLE IF NOT EXISTS` statement. That usage is shown below:

<!-- snippet: sample_customizing_table_creation -->
<a id='snippet-sample_customizing_table_creation'></a>
```cs
var store = DocumentStore.For(_ =>
{
    _.Advanced.DdlRules.TableCreation = CreationStyle.CreateIfNotExists;

    // or the default

    _.Advanced.DdlRules.TableCreation = CreationStyle.DropThenCreate;
});
```
<sup><a href='https://github.com/JasperFx/marten/blob/master/src/Marten.Testing/Examples/DDLCustomization.cs#L10-L19' title='Snippet source file'>snippet source</a> | <a href='#snippet-sample_customizing_table_creation' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

## Invoker vs Definer Permissions

Marten does all of its document updating and inserting through the generated "[upsert](https://wiki.postgresql.org/wiki/UPSERT)"
functions. Most of the time you would probably just let these functions execute under the privileges of whatever
schema user is currently running. If necessary, you can opt into Postgresql's ability to say that a function
should run under the privileges of whatever Postgresql user account created the function.

That is shown below:

<!-- snippet: sample_customizing_upsert_rights -->
<a id='snippet-sample_customizing_upsert_rights'></a>
```cs
var store = DocumentStore.For(_ =>
{
    // Opt into SECURITY DEFINER permissions
    _.Advanced.DdlRules.UpsertRights = SecurityRights.Definer;

    // The default SECURITY INVOKER permissions
    _.Advanced.DdlRules.UpsertRights = SecurityRights.Invoker;
});
```
<sup><a href='https://github.com/JasperFx/marten/blob/master/src/Marten.Testing/Examples/DDLCustomization.cs#L24-L33' title='Snippet source file'>snippet source</a> | <a href='#snippet-sample_customizing_upsert_rights' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

See the [Postgresql documentation on creating functions](https://www.postgresql.org/docs/9.5/static/sql-createfunction.html) for more information.

## Database Role

If you want the DDL scripts generated by Marten to run under a specific Postgresql ROLE, you can configure that like so:

<!-- snippet: sample_customizing_role -->
<a id='snippet-sample_customizing_role'></a>
```cs
var store = DocumentStore.For(_ =>
{
    _.Advanced.DdlRules.Role = "ROLE1";
});
```
<sup><a href='https://github.com/JasperFx/marten/blob/master/src/Marten.Testing/Examples/DDLCustomization.cs#L38-L44' title='Snippet source file'>snippet source</a> | <a href='#snippet-sample_customizing_role' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Doing so will wrap any DDL scripts generated or exported by Marten with this pattern:

```sql
SET ROLE [ROLE_NAME];

-- the body of the DDL

RESET ROLE;

```

## Template Files for GRANT's

One of the primary goals of Marten has been to drastically reduce the mechanical work necessary to apply
database schema changes as your system evolved. To that end, we've strived to make the DDL generation
and patch generation be as complete as possible. However, if your database administrators require any kind
of customized security on the document storage tables and upsert functions, you can optionally use the
"template file" feature shown in this section to add GRANT's or other permissions to the database objects
generated by Marten.

To create a DDL template for the document storage tables, name your file with the pattern [template name].table that would look like the following:

```sql
ALTER TABLE %SCHEMA%.%TABLENAME% OWNER TO "SchemaOwners";
GRANT SELECT (%COLUMNS%) ON TABLE %SCHEMA%.%TABLENAME% TO "ServiceAccountRole", "SchemaOwners";
GRANT INSERT (%COLUMNS%) ON TABLE %SCHEMA%.%TABLENAME% TO "SchemaOwners";
GRANT UPDATE (%NON_ID_COLUMNS%) ON TABLE %SCHEMA%.%TABLENAME% TO "SchemaOwners";
GRANT DELETE ON %SCHEMA%.%TABLENAME% TO "ServiceAccountRole";
```

For the Postgresql functions generated by Marten, you would name your file with the pattern [template name].function and it would look something like this:

```sql
ALTER FUNCTION %SCHEMA%.%FUNCTIONNAME%(%SIGNATURE%) OWNER TO "SchemaOwners";
GRANT EXECUTE ON FUNCTION  %SCHEMA%.%FUNCTIONNAME%(%SIGNATURE%) TO "ServiceAccountRole";
```

As you probably surmised, the "%SOMETHING%" strings are the inputs coming from Marten itself (it's just a crude string replacement behind the scenes).
The available substitutions for tables are:

* %SCHEMA% - the database schema holding the table
* %TABLENAME% - the name of the table
* %COLUMNS% - comma delimited list of all the columns in the table
* %NON_ID_COLUMNS% - comma delimited list of all the columns except for "id" in the table
* %METADATA_COLUMNS% - comma delimited list of all the Marten metadata columns

For functions, you have:

* %SCHEMA% - the database schema holding the function
* %FUNCTION% - the name of the function
* %SIGNATURE% - a comma delimited string of all the declared input types for the function. Since Postgresql supports function overloading, you will frequently need to use the function signature to uniquely identify a function.

To apply the templating, first put all your *.table and *.function files in a single directory or the base directory of your application and use code
like that shown below in your `IDocumentStore` initialization:

<!-- snippet: sample_using_ddl_templates -->
<a id='snippet-sample_using_ddl_templates'></a>
```cs
var store = DocumentStore.For(_ =>
{
    // let's say that you have template files in a
    // "templates" directory under the root of your
    // application
    _.Advanced.DdlRules.ReadTemplates("templates");

    // Or just sweep the base directory of your application
    _.Advanced.DdlRules.ReadTemplates();
});
```
<sup><a href='https://github.com/JasperFx/marten/blob/master/src/Marten.Testing/Examples/DDLCustomization.cs#L49-L60' title='Snippet source file'>snippet source</a> | <a href='#snippet-sample_using_ddl_templates' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

**Do note that the `ReadTemplates()` methods only do a shallow search through the given directory and do not consider child directories.

To establish the default table and function DDL template, name your files "default.table" and "default.function". If Marten finds these files,
it will automatically apply that to all document types as the default.

To overwrite the default template on a document by document type basis, you have a couple options. You can opt to use the fluent interface configuration:

<!-- snippet: sample_configure_ddl_template_by_fi -->
<a id='snippet-sample_configure_ddl_template_by_fi'></a>
```cs
var store = DocumentStore.For(_ =>
{
    _.Schema.For<User>().DdlTemplate("readonly");
});
```
<sup><a href='https://github.com/JasperFx/marten/blob/master/src/Marten.Testing/Examples/DDLCustomization.cs#L65-L70' title='Snippet source file'>snippet source</a> | <a href='#snippet-sample_configure_ddl_template_by_fi' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

You can also decorate document types with the `[DdlTemplate("name")]` attribute shown below:

<[sample:configure_template_with_attribute]>

And lastly, you can take advantage of the embedded configuration option to change the Ddl template against the underlying configuration model:

<[sample:configure_template_with_configure_marten]>
