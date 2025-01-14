# sequelize-version-next

Automatically version (audit, log) your sequelize models, tracking all the changes (create, update, delete) by generating a version of the model than can be used for easy
querying, see the changes made, or whatever you want. The version model uses sequelize hooks to persist the data.

##Forked from https://github.com/ivmarcos/sequelize-version

## Installation

```shell
npm i sequelize-version-next --save
```

or

```shell
yarn add sequelize-version-next
```

## Features

- Custom settings (version prefix, suffix, schema and more)
- Supports transaction

## Usage

```js
const Sequelize = require('sequelize');
const Version = require('sequelize-version-next');

const sequelize = new Sequelize(...);

const Person = sequelize.define('Person', ...);

const PersonVersion = new Version(Person);
```

## Options

| Name             | Type            | Default     | Description                                                |
| ---------------- | --------------- | ----------- | ---------------------------------------------------------- |
| prefix           | `string`        | `'version'` | Prefix for table name and version attributes               |
| suffix           | `string`        |             | Table name suffix                                          |
| attributePrefix  | `string`        |             | Overrides prefix for version attribute fields              |
| schema           | `string`        |             | Version model schema, uses origin model schema as default  |
| sequelize        | `sequelize`     |             | Sequelize instance, uses origin model sequelize as default |
| exclude          | `Array<string>` |             | Attributes to ignore                                       |
| tableUnderscored | `boolean`       | `true`      | Use underscore in version table name                       |
| underscored      | `boolean`       | `true`      | Use underscore in version attributes                       |

## Examples

### Checking versions

```js
// let's create a person for test
let person = await Person.build({ name: 'Jack' }).save();

// now we change a name
person.name = 'Jack Johnson';

// update
await person.save();

// and delete
await person.destroy();

// finally, get the versions
const versions = await PersonVersion.findAll({ where: { id: person.id } });

// or, even simpler
const versionsByInstance = await person.getVersions();

// this way too
const versionsByModel = await Person.getVersions({ where: { id: person.id } });

// versions added
console.log(JSON.parse(JSON.stringify(versions)));
/*
[
    {
        version_id: 1,
        version_type: 1,
        version_timestamp: 2017-08-02T16:08:09.956Z,
        id: 1,
        name: 'Jack'
    },
    {
        version_id: 2,
        version_type: 2,
        version_timestamp: 2017-08-02T16:18:09.958Z,
        id: 1,
        name: 'Jack Johnson'
    },
    {
        version_id: 3,
        version_type: 3,
        version_timestamp: 2017-08-02T16:18:09.959Z,
        id: 1,
        name: 'Jack Johnson'
    },
]
*/
```

### Custom options

```js
//customization examples
const customOptions = {

    //table name and version attributes prefix
    prefix: '',

    //table name suffix
    suffix: 'log',

    //attribute prefix (overrides default - prefix)
    attributePrefix: 'revision',

    //version model schema
    schema: 'audit',

    //you can use another sequelize instance (overrides default - sequelize from origin model)
    sequelize: new Sequelize(...),

    //attributes to ignore from origin model
    exclude: ['createdAt', 'updatedAt'],

    //table name with underscore, true as default
    tableUnderscored: true,

    //attributes with underscore, true as default
    underscored: true,

}

// single options
const VersionModel = new Version(Model, customOptions);

// global options
Version.defaults = customOptions;
```

### Transaction

```js
//version uses the origin model transaction
sequelize.transaction(transaction => {
  return Person.build({ name: 'Jack' }).save({ transaction });
});

//or, if you are using cls - http://docs.sequelizejs.com/manual/tutorial/transactions.html#automatically-pass-transactions-to-all-queries
sequelize.transaction(() => {
  return Person.build({ name: 'Jack' }).save();
});
```

### Querying

```js
// default scopes created in version model (created, updated, deleted)
const versionCreated = await VersionModel.scope('created').find({
  where: { id: person.id },
});

const versionUpdates = await VersionModel.scope('updated').findAll();

const versionDeleted = await VersionModel.scope('deleted').find({
  where: { id: person.id },
});

// using origin model
const allVersions = await Person.getVersions({
  where: { name: { like: 'Jack%' } },
});

// using instance from origin model
const person = await Person.findById(1);
const versionsForOnlyThisPerson = await person.getVersions({
  where: { name: { like: '%Johnson' } },
});
```

### Important Notes

This lib uses sequelize hooks to be able to track the changes. When using class methods, it is necessary to use the `individualHooks: true` option to make this possible. In such cases, this can cause a dramatic reduction in performance. See more at: https://sequelize.org/master/manual/hooks.html.

```js
// This will select all records that are about to be deleted and emit `beforeDestroy` and `afterDestroy` on each instance.
await Model.destroy({
  where: { accessLevel: 0 },
  individualHooks: true,
});

// This will select all records that are about to be updated and emit `beforeUpdate` and `afterUpdate` on each instance.
await Model.update(
  { username: 'Jack' },
  {
    where: { accessLevel: 0 },
    individualHooks: true,
  }
);
```

## License

The files included in this repository are licensed under the MIT license.
