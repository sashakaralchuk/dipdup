# Hasura integration

DipDup uses this optional section to configure the Hasura engine to track your tables automatically.

```yaml
hasura:
  url: http://hasura:8080
  admin_secret: ${HASURA_ADMIN_SECRET:-changeme}
```

If you have enabled this integration, DipDup will generate Hasura metadata based on your DB schema and apply it using [Metadata API](https://hasura.io/docs/latest/graphql/core/api-reference/metadata-api/index.html).

Hasura metadata is all about data representation in GraphQL API. The structure of the database itself is managed solely by Tortoise ORM.

Metadata configuration is idempotent: each time you call `run` or `hasura configure` command, DipDup queries the existing schema and updates metadata if required. DipDup configures Hasura after reindexing, saves the hash of resulting metadata in the `dipdup_schema` table, and doesn't touch Hasura until needed.

<!-- Configuration is performed in the following order:

1. db models -> graphql files -> hasura json -> camelcase -> store hash -->

## Database limitations

The current version of Hasura GraphQL Engine treats `public` and other schemas differently. Table `schema.customer` becomes `schema_customer` root field (or `schemaCustomer` if `camel_case` option is enabled in DipDup config). Table `public.customer` becomes `customer` field, without schema prefix. There's no way to remove this prefix for now. You can track related [issue](https://github.com/hasura/graphql-engine/issues/3606) on Hasura's GitHub to know when the situation will change. Starting with 3.0.0-rc1, DipDup enforces `public` schema name to avoid ambiguity and issues with the GenQL library. You can still use any schema name if Hasura integration is not enabled.

## Unauthorized access

DipDup creates `user` role that allows querying `/graphql` endpoint without authorization. All tables are set to read-only for this role.

You can limit the maximum number of rows such queries return and also disable aggregation queries automatically generated by Hasura:

```yaml
hasura:
  select_limit: 100
  allow_aggregations: False
```

Note that with limits enabled, you have to use either offset or cursor-based pagination on the client-side.

## Convert field names to camel case

For those of you from the JavaScript world, it may be more familiar to use _camelCase_ for variable names instead of _snake\_case_ Hasura uses by default. DipDup now allows to convert all fields in metadata to this casing:

```yaml
hasura:
  camel_case: true
```

Now this example query to hic et nunc demo indexer...

```graphql
query MyQuery {
  hic_et_nunc_token(limit: 1) {
    id
    creator_id
  }
}
```

...will become this one:

```graphql
query MyQuery {
  hicEtNuncToken(limit: 1) {
    id
    creatorId
  }
}
```

All fields auto-generated by Hasura will be renamed accordingly: `hic_et_nunc_token_by_pk` to `hicEtNuncTokenByPk`, `delete_hic_et_nunc_token` to `deleteHicEtNuncToken` and so on. To return to defaults, set `camel_case` to False and run `hasura configure --force`.

Remember that "camelcasing" is a separate stage performed after all tables are registered. So during configuration, you can observe fields in `snake_case` for several seconds even if conversion to camel case is enabled.

```admonish info title="See Also"
* {{ #summary config/hasura.md }}
* {{ #summary cli-reference.md#hasura-configure }}
```

## Custom Hasura Metadata

There are some cases where you want to apply custom modifications to the Hasura metadata. For example, assume that your database schema has a view that contains data from the main table, in which case you cannot set a foreign key between them. Then you can place files with a `.json` extension in the `hasura` directory of your project with the content in Hasura query format, and DipDup will execute them in alphabetical order of file names when the indexing is complete.

The format of the queries can be found in the [Metadata API](https://hasura.io/docs/latest/api-reference/metadata-api/index/) documentation.

Feature flag `allow_inconsistent_metadata` set in `hasura` configuration section allows users to modify the behavior of the requests error handling. By default, this value is False.
