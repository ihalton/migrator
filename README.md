# @reactioncommerce/migrator

![npm (scoped)](https://img.shields.io/npm/v/@reactioncommerce/migrator.svg)
 [![CircleCI](https://circleci.com/gh/reactioncommerce/migrator.svg?style=svg)](https://circleci.com/gh/reactioncommerce/migrator)

Command line interface for migrating MongoDB databases.

## Features

Although this CLI tool was created for [Reaction Commerce](https://reactioncommerce.com/), it is a general purpose MongoDB data migration tool that you can use for any project.

- Migrations are scoped to namespaced "tracks", allowing you to track the version of different areas of data within the same database.
- Store all desired versions in a config file, or different config files per environment. Commit these to version control to keep a record of what data versions are in each of your environments.
- Tracks are locked while they are being migrated, preventing two people or two migration runner workers from trying to run the same migration.
- Although each migration step for a single track necessarily runs in series, track migrations happen in parallel using Node worker threads, which means migrating data will take less time.
- Migration progress is reported on screen allowing you to estimate how long migrations will take to finish.
- Migration history is stored and can be viewed or cleared with CLI commands
- One level of version branching is supported. Versions can be either an integer or two integers separated with a dash (e.g., "2-1"). This causes the track to split, so that you can migrate up to "2-1" from "2-0", but if you migrate up to "3-0" from "2-0", the "2-1" migration will not run. This is useful if you need to migrate some data for an older release that you still support without affecting your current release.

### Why we built this

Put simply, we would have loved to use an out-of-the-box solution for MongoDB data versioning and migrations. We looked at and tried a few, including:

https://www.npmjs.com/package/migrate-mongo
https://www.npmjs.com/package/db-migrate
https://www.npmjs.com/package/mongodb-migrations
https://github.com/emmanuelbuah/mgdb-migrator
https://www.npmjs.com/package/mongrator

But none of these had everything we needed. Some were just not a great user experience. Others were not actively maintained.

Some of the specific problems were:

- The need to run `up` or `down` rather than just specifying a desired version
- Insecure configuration
- Difficult to run on remote servers / with GitOps
- No support for multiple versioning tracks
- APIs specific to Mongoose
- No provided version checking function
- No support for migration code living in NPM packages
- Slow
- Doesn't track/display migration history

## Usage

This CLI looks for a config file in the current directory. We recommend that you create a new directory in which this config file and a `package.json` file will live, and commit it to version control.

You can also have different config files per environment, which allows this one "migrations" repo to reflect the current "desired state" of all your data in all your environments.

To create the directory and install this package in it, run the following commands:

```sh
mkdir migrations
cd migrations
echo "12.14.1" > .nvmrc
nvm use
npm init -y
npm i @reactioncommerce/migrator
touch migrator.config.js
```

Then edit `package.json` and set `"type": "module"`.

The main thing in the object exported by the config file is an array of tracks:

```js
// migrator.config.js
export default {
  tracks: [
    // Migrations exported by an NPM package
    {
      namespace: "my-namespace",
      package: "npm-package-name",
      version: 2
    },
    // Ad-hoc migrations located in the current directory
    {
      namespace: "my-namespace",
      path: "./migrations/index.js",
      version: 5
    }
  ]
};
```

## CLI Commands

To see all commands, run any of the following:

```sh
migrator
migrator -h
migrator --help
```

To see additional docs and options for a specific command, run any of the following:

```sh
migrator <command> -h
migrator <command> --help

# Example
migrator migrate --help
```

### migrator report

To view a report of current data versions versus desired data versions and which migrations are needed, edit `migrator.config.js` to set all the versions to your desired versions. Then run:

```sh
MONGO_URL=mongodb://localhost:27017/dbname migrator report
```

*Important:* Set `MONGO_URL` to the MongoDB connection URL with correct database name.

To view the report for a specific environment, edit `migrator.config-<env>.js` and then run:

```sh
MONGO_URL=mongodb://localhost:27017/dbname migrator report <env>
```

### migrator migrate

To view a report of current data versions versus desired data versions and which migrations are needed and then choose whether to run migrations, edit `migrator.config.js` to set all the versions to your desired versions. Then run:

```sh
MONGO_URL=mongodb://localhost:27017/dbname migrator migrate
```

*Important:* Set `MONGO_URL` to the MongoDB connection URL with correct database name.

To view the report for a specific environment, edit `migrator.config-<env>.js` and then run:

```sh
MONGO_URL=mongodb://localhost:27017/dbname migrator migrate <env>
```

If you don't want to be prompted to decide whether to run them (recommended only for CI), add `-y`:

```sh
MONGO_URL=mongodb://localhost:27017/dbname migrator migrate -y
# OR
MONGO_URL=mongodb://localhost:27017/dbname migrator migrate <env> -y
```

### migrator unlock-track

To unlock a track if you get errors about it being locked but you're sure that nothing is running those migrations right now, run:

```sh
MONGO_URL=mongodb://localhost:27017/dbname migrator unlock-track <namespace>
```

### migrator history

To view a list of all previous migration runs for a track, run:

```sh
MONGO_URL=mongodb://localhost:27017/dbname migrator history <namespace>
```

### migrator clear-history

To clear the list of all previous migration runs for a track, run:

```sh
MONGO_URL=mongodb://localhost:27017/dbname migrator clear-history <namespace>
```

## How to Publish a Package with Migrations

To be compatible with this tool, an NPM package with migrations must have an ES module export named "migrations". This must be an object with the following structure:

```js
const migrations = {
  tracks: [
    {
      namespace: "something",
      migrations: {
        2: {
          up(context) {},
          down(context) {}
        }
      }
    }
  ]
}
```

The `namespace` should be something similar to your package name that will not collide with other packages that provide migrations.

The keys of the `migration` object are the database version numbers. These must be a single number (`2`) or two numbers separated by a dash (`2-1`) if you need to branch off your main migration path to support previous major releases. Only one branch level is allowed.

Version `1` is reserved as the assumed version before any migrations run. Versions 0 and below are invalid.

Each migration version must provide an `up` function.

Each migration version must provide one of the following for `down`:

- A `down` function
- `down: "unnecessary"` if a down function isn't needed
- `down: "impossible"` if migrating down isn't possible due to some information having been deleted

Both types of functions receive a migration context, which has a connection to the MongoDB database and a `progress` function for reporting progress.

### How to Migrate Data

The `up` and `down` functions should do whatever they need to do to move data from your N-1 or N+1 schema to your N schema. They must always be written as if there are millions of documents to convert, meaning they should use MongoDB bulk reads and writes and do updates in small batches.

If errors are thrown, they will be caught. In fact, throwing an error is the only way to stop the migration process and mark the migration as failed.

If you return a string from your `up` or `down` function, it will be stored as `result` in the migration history. Do not return anything other than a string, or `undefined`, or `null`. If you throw, the error message will be stored as `result` in the migration history instead.

While running, the migration function can and should report its progress by calling `context.progress(percentDone)`. The migration function must return a Promise and when that promise resolves, the migration is considered done and the version for this namespace in the database is incremented. If the Promise is rejected, the migration is considered failed and the data may be in a partially migrated state.

Additionally, you can and should make use of [MongoDB transactions](https://docs.mongodb.com/manual/core/transactions/) in your function if you are migrating multiple related collections in a way that will cause problems if some updates succeed and others fail.

To avoid issues, we strongly suggest that you write idempotent migration code, that is, code that can be run multiple times and will do nothing, yet succeed, if the data is already migrated.

### How to Check Data Version in App Code

After you've created and exported migrations for you package, the final step is to check the current migration version for each of your namespaces somewhere in your top-level or startup code, after you are connected to MongoDB but before you run any database commands. Do this by depending on the [@reactioncommerce/db-version-check](https://github.com/reactioncommerce/db-version-check) NPM package and calling the function it exports. Refer to the documentation for that package.

## Commit Messages

To ensure that all contributors follow the correct message convention, each time you commit your message will be validated with the [commitlint](https://www.npmjs.com/package/@commitlint/cli) package, enabled by the [husky](https://www.npmjs.com/package/husky) Git hooks manager.

Examples of commit messages: https://github.com/semantic-release/semantic-release

## Publication to NPM

The `@reactioncommerce/migrator` package is automatically published by CI when commits are merged or pushed to the `master` branch. This is done using [semantic-release](https://www.npmjs.com/package/semantic-release), which also determines version bumps based on conventional Git commit messages.

## Developer Certificate of Origin
We use the [Developer Certificate of Origin (DCO)](https://developercertificate.org/) in lieu of a Contributor License Agreement for all contributions to Reaction Commerce open source projects. We request that contributors agree to the terms of the DCO and indicate that agreement by signing-off all commits made to Reaction Commerce projects by adding a line with your name and email address to every Git commit message contributed:
```
Signed-off-by: Jane Doe <jane.doe@example.com>
```

You can sign-off your commit automatically with Git by using `git commit -s` if you have your `user.name` and `user.email` set as part of your Git configuration.

We ask that you use your real full name (please no anonymous contributions or pseudonyms) and a real email address. By signing-off your commit you are certifying that you have the right to submit it under the [Apache 2.0 License](./LICENSE).

We use the [Probot DCO GitHub app](https://github.com/apps/dco) to check for DCO sign-offs of every commit.

If you forget to sign-off your commits, the DCO bot will remind you and give you detailed instructions for how to amend your commits to add a signature.

## License
Copyright 2020 Reaction Commerce

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.

See the License for the specific language governing permissions and
limitations under the License.
