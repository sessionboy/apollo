---
title: Validating schema changes
description: How to maintain the schema's contract with CI
---

To recognize the need for schema validation, first understand that a GraphQL schema defines a contract between clients and server that contains the available types and their behavior. When a GraphQL API is deployed, consumers start to request fields and depend on the contract. If the schema is updated, such as adding a field or removing a type, the contract changes. Modifying the schema and contract can have a wide range of impact on clients from positive, more functionality, to adverse, active schema dependencies no longer exist.

The Apollo Platform ensures teams deploy schemas without breaking consumers. To prevent dangerous schema evolution, the `apollo service:check` command compares a proposed schema against the active schema to create a list of changes. This resulting list is matched against the field-level usage data from the previous schema and categorizes the changes by severity. If any change's severity is too high, the team is promptly flagged by the cli or GitHub status with actionable feedback.

<h2 id="cli">Setup `apollo` for schema changes</h2>

To check and validate the difference between the current published schema and a new version, run the `apollo service:check` command during continuous integration.

For basic usage, use the following command, substituting the appropriate GraphQL endpoint URL and an API key obtained from the service _Settings_ menu in the [Apollo UI](https://engine.apollographql.com/):

```bash
npx apollo service:check --key="<API_KEY>" --endpoint="http://localhost:4000/graphql"
```

The command can be placed in any continuous integration pipeline, such as this [example in CircleCI](#check-schema-on-ci). To surface results, `apollo` emits an exit code and can [integrate with GitHub statuses](#github).

> For accuracy, it's best to retrieve the schema from a running GraphQL server (with introspection enabled), though the CLI can also reference a local file. See [config options](../platform/apollo-config.html) for more information.

<h3 id="check-tags">Multiple schemas</h3>

When multiple schemas are [published under separate tags](./schema-registry.html), the `--tag` flag specifies which schema to compare against, such as `prod` or `staging`. Often running checks against different schema tags during continuous integration ensures that all important deployments are accounted for. Checking multiple tags will result in check statuses similar to:

<div style="text-align:center">
![multiple service checks](../img/schema-validation/service-checks.png)
</div>

<h2 id="categorize">Categorizing schema changes</h2>

`apollo` buckets schema changes by impact on consumers. Since consumers choose exactly how to use the GraphQL API, the real effect of changes can be unpredictable when inspecting the change in isolation. To properly categorize these changes, the Apollo Platform matches changes with field usage metrics to determine a change's severity.

<h3 id="severity">Change severity</h3>

The Apollo Platform identifies three change severities and reports them on the command line or within a pull-request status([setup for GitHub](#github)):

1. **Failure**: Either the schema is invalid or the changes _will_ break current clients.
2. **Warning**: There are potential problems that may come from this change, but no clients are immediately impacted.
3. **Notice**: This change is safe and will not break current clients.

The overall status of validation, which modifies the cli's exit code and GitHub status, depends on whether metrics are reported to the service.

<h3 id="change-types">Change types</h3>

The Apollo Platform identifies three types of changes:

- **Addition**: additive change that is transparent to clients, such as field addition
- **Update**: in-place modification of field or type that could change query results, such as kind change or required argument addition
- **Removal**: removal of schema element that consumer use, such as type, field, enum, argument, or other

Each of these overall types contains different changes codes, such as `TYPE_REMOVED`, `ARG_DEFAULT_VALUE_CHANGE`, `ENUM_VALUE_ADDED`, or `FIELD_CHANGED_KIND`, which are displayed in the cli output.

<h3 id="metrics">Assigning severity</h3>

When metrics are sent to a service, the changes are verified in the following manner:

- **Addition**: Automatically marked as a `Notice`
- **Update**: Automatically marked as a `Warning`
- **Removal**: If an operation uses the removed type, field, enum, argument, or other schema element, marked as `Failure`

The overall schema validation check will fail if there are any changes marked as a `Failure`.

> Occasionally a service or schema tag does not contain any metrics. In this case, schema validation does cannot verify changes, so removals are automatically set as `Failure`. Additionally warnings will fail Github status checks.

<h2 id="performing-changes">Strategies for performing schema changes</h2>

Defining strategies to perform these changes with minimal impact on clients enables maintainable evolving a schema in response to rapidly changing product requirements. The insight necessary for these techniques is adding new fields, arguments, queries, or mutations won't introduce any new breaking changes. These changes can be confidently made without consideration about existing clients or field usage metrics, since GraphQL clients receive exactly what they ask for.

_Field rollover_ is a term given to an API change that's an evolution of a field, such as a rename or a change in arguments. Some of these changes can be really small, resulting in many variations and making an API harder to manage.

We'll go over these two kinds of field rollovers separately and show how to make these changes safely.

<h3 id="renaming-or-removing">Renaming or removing a field</h3>

When a field is unused, renaming or removing can be performed immediately without affecting clients. Unfortunately, additional considerations should be made if a client uses the field or a GraphQL deployment doesn't have per-field usage metrics.

For example, with the following `Query` type is in a base schema:

```graphql
type Query {
  user(id: ID!): User
}
```

A possible change is renaming `user` to `getUser` to be more descriptive, like so:

```graphql
type Query {
  getUser(id: ID!): User
}
```

Assuming some clients use `user`, this would be a breaking change, since those clients expecting a `user` query would receive error.

To make this change safely, we can add a new `getUser` field and leave the existing `user` field untouched.

```js
const getUserResolver = (root, args, context) => {
  context.User.getById(args.id);
};

const resolvers = {
  Query: {
    getUser: getUserResolver,
    user: getUserResolver
  }
};
```

> To prevent code duplication, the resolver logic can be shared between the two fields:

<h3 id="deprecating">Deprecating a field</h3>

The previous tactic works well to avoid breaking changes, however consumers don't know to switch to the new field name. To solve this problem and signal the switch, the GraphQL specification provides a built-in `@deprecated` schema directive (sometimes called decorators in other languages):

```
type Query {
  user(id: ID!): User @deprecated(reason: "renamed to 'getUser'")
  getUser(id: ID!): User
}
```

GraphQL-aware client tooling, like [Apollo VScode](./vscode.html), GraphQL Playground, and GraphiQL, use this information to help developers make the right choices. These tools will:

- Provide developers with the helpful deprecation message referring them to the new name.
- Avoid auto-completing the field.

Over time, usage will fall for the deprecated field and grow for the new field.

> the Apollo Platform contains a [trace warehouse](./tracing.html) to enable educated decisions about when to retire a field based on usage data through schema analytics.

<h3 id="versioning">What about versioning?</h3>

Versioning is a technique to prevent necessary changes from becoming breaking changes. Developers who have worked with REST APIs past may have various patterns for versioning the API, commonly by using a different URI (e.g. `/api/v1`, `/api/v2`, etc.) or a query parameter (e.g. `?version=1`). With this technique, an application can easily end up with many different API endpoints over time, and the question of _when_ an API can be deprecated can become problematic. While version a GraphQL API the same way may be tempting, multiple graphql endpoints add exponential complexity to schema development and quickly becomes unmaintainable.

<h3 id="what-about-versioning">How about never make a breaking change?</h3>

To another extreme, teams could choose to avoid making any change that might break an operation, ignoring consumer usage. While a viable strategy for maintaining clients in the short term, this limits the flexibility of the schema. Checking changes against usage enables more improvements to API ergonomics, such as removing fields or default argument updates. Positive API experience leads to better developer experience and more robust client and server interaction.

<h2 id="github">Continuous Integration and GitHub</h2>

Schema validation is best used when integrated in a team's development workflow. To make this easy, Apollo integrates with GitHub to provide status checks on pull requests when schema changes are proposed. To enable schema validation in GitHub, follow these steps:

![GitHub Status View](../img/schema-history/github-check.png)

<h3 id="install-github">Install GitHub application</h3>

Go to [https://github.com/apps/apollo-engine](https://github.com/apps/apollo-engine) and click the `Configure` button to install the Apollo Engine integration on the appropriate GitHub profile or organization.

<h3 id="check-schema-on-ci">Run validation on each commit</h3>

After adding `apollo service:check` in a continuous integration workflow (e.g. CircleCI, etc.), schema validation is performed automatically and potential problems are displayed directly on a pull-request's status checks, providing actionable feedback to developers.

To setup validation, run the `apollo service:check` command targeting a GraphQL server with introspection enabled. An example of is shown below with a CircleCI config:

> Note: with a GitHub status check, ignoring the `apollo`'s error code by appending `|| echo 'validation failed'` allows continuous integration to continue without failing early

```yaml
version: 2

jobs:
  build:
    docker:
      - image: circleci/node:8

    steps:
      - checkout

      - run: npm install

      # Start the GraphQL server.  If a different command is used to
      # start the server, use it in place of `npm start` here.
      - run:
          name: Starting server
          command: npm start
          background: true

      # make sure the server has enough time to start up before running
      # commands against it
      - run: sleep 5

      # This will authenticate using the `ENGINE_API_KEY` environment
      # variable. If the GraphQL server is available elsewhere than
      # http://localhost:4000/graphql, set it with `--endpoint=<URL>`.
      - run: npx apollo service:check
```
