# Associations

This section describes the various association types in sequelize. There are four types of
associations available in Sequelize

1. BelongsTo
2. HasOne
3. HasMany
4. BelongsToMany

## Basic Concepts

### Source & Target

Let's first begin with a basic concept that you will see used in most associations, **source** and **target** model. Suppose you are trying to add an association between two Models. Here we are adding a `hasOne` association between `User` and `Project`.

```js
class User extends Model {}
User.init({
  name: Sequelize.STRING,
  email: Sequelize.STRING
}, {
  sequelize,
  modelName: 'user'
});

class Project extends Model {}
Project.init({
  name: Sequelize.STRING
}, {
  sequelize,
  modelName: 'project'
});

User.hasOne(Project);
```

`User` model (the model that the function is being invoked on) is the __source__. `Project` model (the model being passed as an argument) is the __target__.

### Foreign Keys

When you create associations between your models in sequelize, foreign key references with constraints will automatically be created. The setup below:

```js
class Task extends Model {}
Task.init({ title: Sequelize.STRING }, { sequelize, modelName: 'task' });
class User extends Model {}
User.init({ username: Sequelize.STRING }, { sequelize, modelName: 'user' });

User.hasMany(Task); // Will add userId to Task model
Task.belongsTo(User); // Will also add userId to Task model
```

Will generate the following SQL:

```sql
CREATE TABLE IF NOT EXISTS "users" (
  "id" SERIAL,
  "username" VARCHAR(255),
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  PRIMARY KEY ("id")
);

CREATE TABLE IF NOT EXISTS "tasks" (
  "id" SERIAL,
  "title" VARCHAR(255),
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "userId" INTEGER REFERENCES "users" ("id") ON DELETE
  SET
    NULL ON UPDATE CASCADE,
    PRIMARY KEY ("id")
);
```

The relation between `tasks` and `users` model injects the `userId` foreign key on `tasks` table, and marks it as a reference to the `users` table. By default `userId` will be set to `NULL` if the referenced user is deleted, and updated if the id of the `userId` updated. These options can be overridden by passing `onUpdate` and `onDelete` options to the association calls. The validation options are `RESTRICT, CASCADE, NO ACTION, SET DEFAULT, SET NULL`.

For 1:1 and 1:m associations the default option is `SET NULL` for deletion, and `CASCADE` for updates. For n:m, the default for both is `CASCADE`. This means, that if you delete or update a row from one side of an n:m association, all the rows in the join table referencing that row will also be deleted or updated.

#### underscored option

Sequelize allow setting `underscored` option for Model. When `true` this option will set the
`field` option on all attributes to the underscored version of its name. This also applies to
foreign keys generated by associations.

Let's modify last example to use `underscored` option.

```js
class Task extends Model {}
Task.init({
  title: Sequelize.STRING
}, {
  underscored: true,
  sequelize,
  modelName: 'task'
});

class User extends Model {}
User.init({
  username: Sequelize.STRING
}, {
  underscored: true,
  sequelize,
  modelName: 'user'
});

// Will add userId to Task model, but field will be set to `user_id`
// This means column name will be `user_id`
User.hasMany(Task);

// Will also add userId to Task model, but field will be set to `user_id`
// This means column name will be `user_id`
Task.belongsTo(User);
```

Will generate the following SQL:

```sql
CREATE TABLE IF NOT EXISTS "users" (
  "id" SERIAL,
  "username" VARCHAR(255),
  "created_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  PRIMARY KEY ("id")
);

CREATE TABLE IF NOT EXISTS "tasks" (
  "id" SERIAL,
  "title" VARCHAR(255),
  "created_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  "user_id" INTEGER REFERENCES "users" ("id") ON DELETE
  SET
    NULL ON UPDATE CASCADE,
    PRIMARY KEY ("id")
);
```

With the underscored option attributes injected to model are still camel cased but `field` option is set to their underscored version.

#### Cyclic dependencies & Disabling constraints

Adding constraints between tables means that tables must be created in the database in a certain order, when using `sequelize.sync`. If `Task` has a reference to `User`, the `users` table must be created before the `tasks` table can be created. This can sometimes lead to circular references, where sequelize cannot find an order in which to sync. Imagine a scenario of documents and versions. A document can have multiple versions, and for convenience, a document has a reference to its current version.

```js
class Document extends Model {}
Document.init({
  author: Sequelize.STRING
}, { sequelize, modelName: 'document' });
class Version extends Model {}
Version.init({
  timestamp: Sequelize.DATE
}, { sequelize, modelName: 'version' });

Document.hasMany(Version); // This adds documentId attribute to version
Document.belongsTo(Version, {
  as: 'Current',
  foreignKey: 'currentVersionId'
}); // This adds currentVersionId attribute to document
```

However, the code above will result in the following error: `Cyclic dependency found. documents is dependent of itself. Dependency chain: documents -> versions => documents`.

In order to alleviate that, we can pass `constraints: false` to one of the associations:

```js
Document.hasMany(Version);
Document.belongsTo(Version, {
  as: 'Current',
  foreignKey: 'currentVersionId',
  constraints: false
});
```

Which will allow us to sync the tables correctly:

```sql
CREATE TABLE IF NOT EXISTS "documents" (
  "id" SERIAL,
  "author" VARCHAR(255),
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "currentVersionId" INTEGER,
  PRIMARY KEY ("id")
);

CREATE TABLE IF NOT EXISTS "versions" (
  "id" SERIAL,
  "timestamp" TIMESTAMP WITH TIME ZONE,
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "documentId" INTEGER REFERENCES "documents" ("id") ON DELETE
  SET
    NULL ON UPDATE CASCADE,
    PRIMARY KEY ("id")
);
```

#### Enforcing a foreign key reference without constraints

Sometimes you may want to reference another table, without adding any constraints, or associations. In that case you can manually add the reference attributes to your schema definition, and mark the relations between them.

```js
class Trainer extends Model {}
Trainer.init({
  firstName: Sequelize.STRING,
  lastName: Sequelize.STRING
}, { sequelize, modelName: 'trainer' });

// Series will have a trainerId = Trainer.id foreign reference key
// after we call Trainer.hasMany(series)
class Series extends Model {}
Series.init({
  title: Sequelize.STRING,
  subTitle: Sequelize.STRING,
  description: Sequelize.TEXT,
  // Set FK relationship (hasMany) with `Trainer`
  trainerId: {
    type: Sequelize.INTEGER,
    references: {
      model: Trainer,
      key: 'id'
    }
  }
}, { sequelize, modelName: 'series' });

// Video will have seriesId = Series.id foreign reference key
// after we call Series.hasOne(Video)
class Video extends Model {}
Video.init({
  title: Sequelize.STRING,
  sequence: Sequelize.INTEGER,
  description: Sequelize.TEXT,
  // set relationship (hasOne) with `Series`
  seriesId: {
    type: Sequelize.INTEGER,
    references: {
      model: Series, // Can be both a string representing the table name or a Sequelize model
      key: 'id'
    }
  }
}, { sequelize, modelName: 'video' });

Series.hasOne(Video);
Trainer.hasMany(Series);
```

## One-To-One associations

One-To-One associations are associations between exactly two models connected by a single foreign key.

### BelongsTo

BelongsTo associations are associations where the foreign key for the one-to-one relation exists on the **source model**.

A simple example would be a **Player** being part of a **Team** with the foreign key on the player.

```js
class Player extends Model {}
Player.init({/* attributes */}, { sequelize, modelName: 'player' });
class Team extends Model {}
Team.init({/* attributes */}, { sequelize, modelName: 'team' });

Player.belongsTo(Team); // Will add a teamId attribute to Player to hold the primary key value for Team
```

#### Foreign keys

By default the foreign key for a belongsTo relation will be generated from the target model name and the target primary key name.

The default casing is `camelCase`. If the source model is configured with `underscored: true` the foreignKey will be created with field `snake_case`.

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({/* attributes */}, { sequelize, modelName: 'company' });

// will add companyId to user
User.belongsTo(Company);

class User extends Model {}
User.init({/* attributes */}, { underscored: true, sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({
  uuid: {
    type: Sequelize.UUID,
    primaryKey: true
  }
}, { sequelize, modelName: 'company' });

// will add companyUuid to user with field company_uuid
User.belongsTo(Company);
```

In cases where `as` has been defined it will be used in place of the target model name.

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class UserRole extends Model {}
UserRole.init({/* attributes */}, { sequelize, modelName: 'userRole' });

User.belongsTo(UserRole, {as: 'role'}); // Adds roleId to user rather than userRoleId
```

In all cases the default foreign key can be overwritten with the `foreignKey` option.
When the foreign key option is used, Sequelize will use it as-is:

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({/* attributes */}, { sequelize, modelName: 'company' });

User.belongsTo(Company, {foreignKey: 'fk_company'}); // Adds fk_company to User
```

#### Target keys

The target key is the column on the target model that the foreign key column on the source model points to. By default the target key for a belongsTo relation will be the target model's primary key. To define a custom column, use the `targetKey` option.

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({/* attributes */}, { sequelize, modelName: 'company' });

User.belongsTo(Company, {foreignKey: 'fk_companyname', targetKey: 'name'}); // Adds fk_companyname to User
```

### HasOne

HasOne associations are associations where the foreign key for the one-to-one relation exists on the **target model**.

```js
class User extends Model {}
User.init({/* ... */}, { sequelize, modelName: 'user' })
class Project extends Model {}
Project.init({/* ... */}, { sequelize, modelName: 'project' })

// One-way associations
Project.hasOne(User)

/*
  In this example hasOne will add an attribute projectId to the User model!
  Furthermore, Project.prototype will gain the methods getUser and setUser according
  to the first parameter passed to define. If you have underscore style
  enabled, the added attribute will be project_id instead of projectId.

  The foreign key will be placed on the users table.

  You can also define the foreign key, e.g. if you already have an existing
  database and want to work on it:
*/

Project.hasOne(User, { foreignKey: 'initiator_id' })

/*
  Because Sequelize will use the model's name (first parameter of define) for
  the accessor methods, it is also possible to pass a special option to hasOne:
*/

Project.hasOne(User, { as: 'Initiator' })
// Now you will get Project.getInitiator and Project.setInitiator

// Or let's define some self references
class Person extends Model {}
Person.init({ /* ... */}, { sequelize, modelName: 'person' })

Person.hasOne(Person, {as: 'Father'})
// this will add the attribute FatherId to Person

// also possible:
Person.hasOne(Person, {as: 'Father', foreignKey: 'DadId'})
// this will add the attribute DadId to Person

// In both cases you will be able to do:
Person.setFather
Person.getFather

// If you need to join a table twice you can double join the same table
Team.hasOne(Game, {as: 'HomeTeam', foreignKey : 'homeTeamId'});
Team.hasOne(Game, {as: 'AwayTeam', foreignKey : 'awayTeamId'});

Game.belongsTo(Team);
```

Even though it is called a HasOne association, for most 1:1 relations you usually want the BelongsTo association since BelongsTo will add the foreignKey on the source where hasOne will add on the target.

#### Source keys

The source key is the attribute on the source model that the foreign key attribute on the target model points to. By default the source key for a `hasOne` relation will be the source model's primary attribute. To use a custom attribute, use the `sourceKey` option.

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({/* attributes */}, { sequelize, modelName: 'company' });

// Adds companyName attribute to User
// Use name attribute from Company as source attribute
Company.hasOne(User, {foreignKey: 'companyName', sourceKey: 'name'});
```

### Difference between HasOne and BelongsTo

In Sequelize 1:1 relationship can be set using HasOne and BelongsTo. They are suitable for different scenarios. Lets study this difference using an example.

Suppose we have two tables to link **Player** and **Team**. Lets define their models.

```js
class Player extends Model {}
Player.init({/* attributes */}, { sequelize, modelName: 'player' })
class Team extends Model {}
Team.init({/* attributes */}, { sequelize, modelName: 'team' });
```

When we link two models in Sequelize we can refer them as pairs of **source** and **target** models. Like this

Having **Player** as the **source** and **Team** as the **target**

```js
Player.belongsTo(Team);
//Or
Player.hasOne(Team);
```

Having **Team** as the **source** and **Player** as the **target**

```js
Team.belongsTo(Player);
//Or
Team.hasOne(Player);
```

HasOne and BelongsTo insert the association key in different models from each other. HasOne inserts the association key in **target** model whereas BelongsTo inserts the association key in the **source** model.

Here is an example demonstrating use cases of BelongsTo and HasOne.

```js
class Player extends Model {}
Player.init({/* attributes */}, { sequelize, modelName: 'player' })
class Coach extends Model {}
Coach.init({/* attributes */}, { sequelize, modelName: 'coach' })
class Team extends Model {}
Team.init({/* attributes */}, { sequelize, modelName: 'team' });
```

Suppose our `Player` model has information about its team as `teamId` column. Information about each Team's `Coach` is stored in the `Team` model as `coachId` column. These both scenarios requires different kind of 1:1 relation because foreign key relation is present on different models each time.

When information about association is present in **source** model we can use `belongsTo`. In this case `Player` is suitable for `belongsTo` because it has `teamId` column.

```js
Player.belongsTo(Team)  // `teamId` will be added on Player / Source model
```

When information about association is present in **target** model we can use `hasOne`. In this case `Coach` is suitable for `hasOne` because `Team` model store information about its `Coach` as `coachId` field.

```js
Coach.hasOne(Team)  // `coachId` will be added on Team / Target model
```

## One-To-Many associations (hasMany)

One-To-Many associations are connecting one source with multiple targets. The targets however are again connected to exactly one specific source.

```js
class User extends Model {}
User.init({/* ... */}, { sequelize, modelName: 'user' })
class Project extends Model {}
Project.init({/* ... */}, { sequelize, modelName: 'project' })

// OK. Now things get more complicated (not really visible to the user :)).
// First let's define a hasMany association
Project.hasMany(User, {as: 'Workers'})
```

This will add the attribute `projectId` to User. Depending on your setting for underscored the column in the table will either be called `projectId` or `project_id`. Instances of Project will get the accessors `getWorkers` and `setWorkers`.

Sometimes you may need to associate records on different columns, you may use `sourceKey` option:

```js
class City extends Model {}
City.init({ countryCode: Sequelize.STRING }, { sequelize, modelName: 'city' });
class Country extends Model {}
Country.init({ isoCode: Sequelize.STRING }, { sequelize, modelName: 'country' });

// Here we can connect countries and cities base on country code
Country.hasMany(City, {foreignKey: 'countryCode', sourceKey: 'isoCode'});
City.belongsTo(Country, {foreignKey: 'countryCode', targetKey: 'isoCode'});
```

So far we dealt with a one-way association. But we want more! Let's define it the other way around by creating a many to many association in the next section.

## Belongs-To-Many associations

Belongs-To-Many associations are used to connect sources with multiple targets. Furthermore the targets can also have connections to multiple sources.

```js
Project.belongsToMany(User, {through: 'UserProject'});
User.belongsToMany(Project, {through: 'UserProject'});
```

This will create a new model called UserProject with the equivalent foreign keys `projectId` and `userId`. Whether the attributes are camelcase or not depends on the two models joined by the table (in this case User and Project).

Defining `through` is **required**. Sequelize would previously attempt to autogenerate names but that would not always lead to the most logical setups.

This will add methods `getUsers`, `setUsers`, `addUser`,`addUsers` to `Project`, and `getProjects`, `setProjects`, `addProject`, and `addProjects` to `User`.

Sometimes you may want to rename your models when using them in associations. Let's define users as workers and projects as tasks by using the alias (`as`) option. We will also manually define the foreign keys to use:

```js
User.belongsToMany(Project, { as: 'Tasks', through: 'worker_tasks', foreignKey: 'userId' })
Project.belongsToMany(User, { as: 'Workers', through: 'worker_tasks', foreignKey: 'projectId' })
```

`foreignKey` will allow you to set **source model** key in the **through** relation.
`otherKey` will allow you to set **target model** key in the **through** relation.

```js
User.belongsToMany(Project, { as: 'Tasks', through: 'worker_tasks', foreignKey: 'userId', otherKey: 'projectId'})
```

Of course you can also define self references with belongsToMany:

```js
Person.belongsToMany(Person, { as: 'Children', through: 'PersonChildren' })
// This will create the table PersonChildren which stores the ids of the objects.

```

#### Source and target keys

If you want to create a belongs to many relationship that does not use the default primary key some setup work is required.
You must set the `sourceKey` (optionally `targetKey`) appropriately for the two ends of the belongs to many. Further you must also ensure you have appropriate indexes created on your relationships. For example:

```js
const User = this.sequelize.define('User', {
  id: {
    type: DataTypes.UUID,
    allowNull: false,
    primaryKey: true,
    defaultValue: DataTypes.UUIDV4,
    field: 'user_id'
  },
  userSecondId: {
    type: DataTypes.UUID,
    allowNull: false,
    defaultValue: DataTypes.UUIDV4,
    field: 'user_second_id'
  }
}, {
  tableName: 'tbl_user',
  indexes: [
    {
      unique: true,
      fields: ['user_second_id']
    }
  ]
});

const Group = this.sequelize.define('Group', {
  id: {
    type: DataTypes.UUID,
    allowNull: false,
    primaryKey: true,
    defaultValue: DataTypes.UUIDV4,
    field: 'group_id'
  },
  groupSecondId: {
    type: DataTypes.UUID,
    allowNull: false,
    defaultValue: DataTypes.UUIDV4,
    field: 'group_second_id'
  }
}, {
  tableName: 'tbl_group',
  indexes: [
    {
      unique: true,
      fields: ['group_second_id']
    }
  ]
});

User.belongsToMany(Group, {
  through: 'usergroups',
  sourceKey: 'userSecondId'
});
Group.belongsToMany(User, {
  through: 'usergroups',
  sourceKey: 'groupSecondId'
});
```

If you want additional attributes in your join table, you can define a model for the join table in sequelize, before you define the association, and then tell sequelize that it should use that model for joining, instead of creating a new one:

```js
class User extends Model {}
User.init({}, { sequelize, modelName: 'user' })
class Project extends Model {}
Project.init({}, { sequelize, modelName: 'project' })
class UserProjects extends Model {}
UserProjects.init({
  status: DataTypes.STRING
}, { sequelize, modelName: 'userProjects' })

User.belongsToMany(Project, { through: UserProjects })
Project.belongsToMany(User, { through: UserProjects })
```

To add a new project to a user and set its status, you pass extra `options.through` to the setter, which contains the attributes for the join table

```js
user.addProject(project, { through: { status: 'started' }})
```

By default the code above will add projectId and userId to the UserProjects table, and _remove any previously defined primary key attribute_ - the table will be uniquely identified by the combination of the keys of the two tables, and there is no reason to have other PK columns. To enforce a primary key on the `UserProjects` model you can add it manually.

```js
class UserProjects extends Model {}
UserProjects.init({
  id: {
    type: Sequelize.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  status: DataTypes.STRING
}, { sequelize, modelName: 'userProjects' })
```

With Belongs-To-Many you can query based on **through** relation and select specific attributes. For example using `findAll` with **through**

```js
User.findAll({
  include: [{
    model: Project,
    through: {
      attributes: ['createdAt', 'startedAt', 'finishedAt'],
      where: {completed: true}
    }
  }]
});
```

Belongs-To-Many creates a unique key when primary key is not present on through model. This unique key name can be overridden using **uniqueKey** option.

```js
Project.belongsToMany(User, { through: UserProjects, uniqueKey: 'my_custom_unique' })
```

## Naming strategy

By default sequelize will use the model name (the name passed to `sequelize.define`) to figure out the name of the model when used in associations. For example, a model named `user` will add the functions `get/set/add User` to instances of the associated model, and a property named `.user` in eager loading, while a model named `User` will add the same functions, but a property named `.User` (notice the upper case U) in eager loading.

As we've already seen, you can alias models in associations using `as`. In single associations (has one and belongs to), the alias should be singular, while for many associations (has many) it should be plural. Sequelize then uses the [inflection][0] library to convert the alias to its singular form. However, this might not always work for irregular or non-english words. In this case, you can provide both the plural and the singular form of the alias:

```js
User.belongsToMany(Project, { as: { singular: 'task', plural: 'tasks' }})
// Notice that inflection has no problem singularizing tasks, this is just for illustrative purposes.
```

If you know that a model will always use the same alias in associations, you can provide it when creating the model

```js
class Project extends Model {}
Project.init(attributes, {
  name: {
    singular: 'task',
    plural: 'tasks',
  },
  sequelize,
  modelName: 'project'
})

User.belongsToMany(Project);
```

This will add the functions `add/set/get Tasks` to user instances.

Remember, that using `as` to change the name of the association will also change the name of the foreign key. When using `as`, it is safest to also specify the foreign key.

```js
Invoice.belongsTo(Subscription)
Subscription.hasMany(Invoice)
```

Without `as`, this adds `subscriptionId` as expected. However, if you were to say `Invoice.belongsTo(Subscription, { as: 'TheSubscription' })`, you will have both `subscriptionId` and `theSubscriptionId`, because sequelize is not smart enough to figure that the calls are two sides of the same relation. 'foreignKey' fixes this problem;

```js
Invoice.belongsTo(Subscription, { as: 'TheSubscription', foreignKey: 'subscription_id' })
Subscription.hasMany(Invoice, { foreignKey: 'subscription_id' })
```

## Associating objects

Because Sequelize is doing a lot of magic, you have to call `Sequelize.sync` after setting the associations! Doing so will allow you the following:

```js
Project.hasMany(Task)
Task.belongsTo(Project)

Project.create()...
Task.create()...
Task.create()...

// save them... and then:
project.setTasks([task1, task2]).then(() => {
  // saved!
})

// ok, now they are saved... how do I get them later on?
project.getTasks().then(associatedTasks => {
  // associatedTasks is an array of tasks
})

// You can also pass filters to the getter method.
// They are equal to the options you can pass to a usual finder method.
project.getTasks({ where: 'id > 10' }).then(tasks => {
  // tasks with an id greater than 10 :)
})

// You can also only retrieve certain fields of a associated object.
project.getTasks({attributes: ['title']}).then(tasks => {
  // retrieve tasks with the attributes "title" and "id"
})
```

To remove created associations you can just call the set method without a specific id:

```js
// remove the association with task1
project.setTasks([task2]).then(associatedTasks => {
  // you will get task2 only
})

// remove 'em all
project.setTasks([]).then(associatedTasks => {
  // you will get an empty array
})

// or remove 'em more directly
project.removeTask(task1).then(() => {
  // it's gone
})

// and add 'em again
project.addTask(task1).then(() => {
  // it's back again
})
```

You can of course also do it vice versa:

```js
// project is associated with task1 and task2
task2.setProject(null).then(() => {
  // and it's gone
})
```

For hasOne/belongsTo it's basically the same:

```js
Task.hasOne(User, {as: "Author"})
Task.setAuthor(anAuthor)
```

Adding associations to a relation with a custom join table can be done in two ways (continuing with the associations defined in the previous chapter):

```js
// Either by adding a property with the name of the join table model to the object, before creating the association
project.UserProjects = {
  status: 'active'
}
u.addProject(project)

// Or by providing a second options.through argument when adding the association, containing the data that should go in the join table
u.addProject(project, { through: { status: 'active' }})


// When associating multiple objects, you can combine the two options above. In this case the second argument
// will be treated as a defaults object, that will be used if no data is provided
project1.UserProjects = {
    status: 'inactive'
}

u.setProjects([project1, project2], { through: { status: 'active' }})
// The code above will record inactive for project one, and active for project two in the join table
```

When getting data on an association that has a custom join table, the data from the join table will be returned as a DAO instance:

```js
u.getProjects().then(projects => {
  const project = projects[0]

  if (project.UserProjects.status === 'active') {
    // .. do magic

    // since this is a real DAO instance, you can save it directly after you are done doing magic
    return project.UserProjects.save()
  }
})
```

If you only need some of the attributes from the join table, you can provide an array with the attributes you want:

```js
// This will select only name from the Projects table, and only status from the UserProjects table
user.getProjects({ attributes: ['name'], joinTableAttributes: ['status']})
```

## Check associations

You can also check if an object is already associated with another one (N:M only). Here is how you'd do it:

```js
// check if an object is one of associated ones:
Project.create({ /* */ }).then(project => {
  return User.create({ /* */ }).then(user => {
    return project.hasUser(user).then(result => {
      // result would be false
      return project.addUser(user).then(() => {
        return project.hasUser(user).then(result => {
          // result would be true
        })
      })
    })
  })
})

// check if all associated objects are as expected:
// let's assume we have already a project and two users
project.setUsers([user1, user2]).then(() => {
  return project.hasUsers([user1]);
}).then(result => {
  // result would be true
  return project.hasUsers([user1, user2]);
}).then(result => {
  // result would be true
})
```

## Advanced Concepts

### Scopes

This section concerns association scopes. For a definition of association scopes vs. scopes on associated models, see [Scopes](scopes.html).

Association scopes allow you to place a scope (a set of default attributes for `get` and `create`) on the association. Scopes can be placed both on the associated model (the target of the association), and on the through table for n:m relations.

#### 1:n

Assume we have models Comment, Post, and Image. A comment can be associated to either an image or a post via `commentableId` and `commentable` - we say that Post and Image are `Commentable`

```js
class Post extends Model {}
Post.init({
  title: Sequelize.STRING,
  text: Sequelize.STRING
}, { sequelize, modelName: 'post' });

class Image extends Model {}
Image.init({
  title: Sequelize.STRING,
  link: Sequelize.STRING
}, { sequelize, modelName: 'image' });

class Comment extends Model {
  getItem(options) {
    return this[
      'get' +
        this.get('commentable')
          [0]
          .toUpperCase() +
        this.get('commentable').substr(1)
    ](options);
  }
}

Comment.init({
  title: Sequelize.STRING,
  commentable: Sequelize.STRING,
  commentableId: Sequelize.INTEGER
}, { sequelize, modelName: 'comment' });

Post.hasMany(Comment, {
  foreignKey: 'commentableId',
  constraints: false,
  scope: {
    commentable: 'post'
  }
});

Comment.belongsTo(Post, {
  foreignKey: 'commentableId',
  constraints: false,
  as: 'post'
});

Image.hasMany(Comment, {
  foreignKey: 'commentableId',
  constraints: false,
  scope: {
    commentable: 'image'
  }
});

Comment.belongsTo(Image, {
  foreignKey: 'commentableId',
  constraints: false,
  as: 'image'
});
```

`constraints: false` disables references constraints, as `commentableId` column references several tables, we cannot add a `REFERENCES` constraint to it.

Note that the Image -> Comment and Post -> Comment relations define a scope, `commentable: 'image'` and `commentable: 'post'` respectively. This scope is automatically applied when using the association functions:

```js
image.getComments()
// SELECT "id", "title", "commentable", "commentableId", "createdAt", "updatedAt" FROM "comments" AS
// "comment" WHERE "comment"."commentable" = 'image' AND "comment"."commentableId" = 1;

image.createComment({
  title: 'Awesome!'
})
// INSERT INTO "comments" ("id","title","commentable","commentableId","createdAt","updatedAt") VALUES
// (DEFAULT,'Awesome!','image',1,'2018-04-17 05:36:40.454 +00:00','2018-04-17 05:36:40.454 +00:00')
// RETURNING *;

image.addComment(comment);
// UPDATE "comments" SET "commentableId"=1,"commentable"='image',"updatedAt"='2018-04-17 05:38:43.948
// +00:00' WHERE "id" IN (1)
```

The `getItem` utility function on `Comment` completes the picture - it simply converts the `commentable` string into a call to either `getImage` or `getPost`, providing an abstraction over whether a comment belongs to a post or an image. You can pass a normal options object as a parameter to `getItem(options)` to specify any where conditions or includes.

#### n:m

Continuing with the idea of a polymorphic model, consider a tag table - an item can have multiple tags, and a tag can be related to several items.

For brevity, the example only shows a Post model, but in reality Tag would be related to several other models.

```js
class ItemTag extends Model {}
ItemTag.init({
  id: {
    type: Sequelize.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  tagId: {
    type: Sequelize.INTEGER,
    unique: 'item_tag_taggable'
  },
  taggable: {
    type: Sequelize.STRING,
    unique: 'item_tag_taggable'
  },
  taggableId: {
    type: Sequelize.INTEGER,
    unique: 'item_tag_taggable',
    references: null
  }
}, { sequelize, modelName: 'item_tag' });

class Tag extends Model {}
Tag.init({
  name: Sequelize.STRING,
  status: Sequelize.STRING
}, { sequelize, modelName: 'tag' });

Post.belongsToMany(Tag, {
  through: {
    model: ItemTag,
    unique: false,
    scope: {
      taggable: 'post'
    }
  },
  foreignKey: 'taggableId',
  constraints: false
});

Tag.belongsToMany(Post, {
  through: {
    model: ItemTag,
    unique: false
  },
  foreignKey: 'tagId',
  constraints: false
});
```

Notice that the scoped column (`taggable`) is now on the through model (`ItemTag`).

We could also define a more restrictive association, for example, to get all pending tags for a post by applying a scope of both the through model (`ItemTag`) and the target model (`Tag`):

```js
Post.belongsToMany(Tag, {
  through: {
    model: ItemTag,
    unique: false,
    scope: {
      taggable: 'post'
    }
  },
  scope: {
    status: 'pending'
  },
  as: 'pendingTags',
  foreignKey: 'taggableId',
  constraints: false
});

post.getPendingTags();
```

```sql
SELECT
  "tag"."id",
  "tag"."name",
  "tag"."status",
  "tag"."createdAt",
  "tag"."updatedAt",
  "item_tag"."id" AS "item_tag.id",
  "item_tag"."tagId" AS "item_tag.tagId",
  "item_tag"."taggable" AS "item_tag.taggable",
  "item_tag"."taggableId" AS "item_tag.taggableId",
  "item_tag"."createdAt" AS "item_tag.createdAt",
  "item_tag"."updatedAt" AS "item_tag.updatedAt"
FROM
  "tags" AS "tag"
  INNER JOIN "item_tags" AS "item_tag" ON "tag"."id" = "item_tag"."tagId"
  AND "item_tag"."taggableId" = 1
  AND "item_tag"."taggable" = 'post'
WHERE
  ("tag"."status" = 'pending');
```

`constraints: false` disables references constraints on the `taggableId` column. Because the column is polymorphic, we cannot say that it `REFERENCES` a specific table.

### Creating with associations

An instance can be created with nested association in one step, provided all elements are new.

#### BelongsTo / HasMany / HasOne association

Consider the following models:

```js
class Product extends Model {}
Product.init({
  title: Sequelize.STRING
}, { sequelize, modelName: 'product' });
class User extends Model {}
User.init({
  firstName: Sequelize.STRING,
  lastName: Sequelize.STRING
}, { sequelize, modelName: 'user' });
class Address extends Model {}
Address.init({
  type: Sequelize.STRING,
  line1: Sequelize.STRING,
  line2: Sequelize.STRING,
  city: Sequelize.STRING,
  state: Sequelize.STRING,
  zip: Sequelize.STRING,
}, { sequelize, modelName: 'address' });

Product.User = Product.belongsTo(User);
User.Addresses = User.hasMany(Address);
// Also works for `hasOne`
```

A new `Product`, `User`, and one or more `Address` can be created in one step in the following way:

```js
return Product.create({
  title: 'Chair',
  user: {
    firstName: 'Mick',
    lastName: 'Broadstone',
    addresses: [{
      type: 'home',
      line1: '100 Main St.',
      city: 'Austin',
      state: 'TX',
      zip: '78704'
    }]
  }
}, {
  include: [{
    association: Product.User,
    include: [ User.Addresses ]
  }]
});
```

Here, our user model is called `user`, with a lowercase u - This means that the property in the object should also be `user`. If the name given to `sequelize.define` was `User`, the key in the object should also be `User`. Likewise for `addresses`, except it's pluralized being a `hasMany` association.

#### BelongsTo association with an alias

The previous example can be extended to support an association alias.

```js
const Creator = Product.belongsTo(User, { as: 'creator' });

return Product.create({
  title: 'Chair',
  creator: {
    firstName: 'Matt',
    lastName: 'Hansen'
  }
}, {
  include: [ Creator ]
});
```

#### HasMany / BelongsToMany association

Let's introduce the ability to associate a product with many tags. Setting up the models could look like:

```js
class Tag extends Model {}
Tag.init({
  name: Sequelize.STRING
}, { sequelize, modelName: 'tag' });

Product.hasMany(Tag);
// Also works for `belongsToMany`.
```

Now we can create a product with multiple tags in the following way:

```js
Product.create({
  id: 1,
  title: 'Chair',
  tags: [
    { name: 'Alpha'},
    { name: 'Beta'}
  ]
}, {
  include: [ Tag ]
})
```

And, we can modify this example to support an alias as well:

```js
const Categories = Product.hasMany(Tag, { as: 'categories' });

Product.create({
  id: 1,
  title: 'Chair',
  categories: [
    { id: 1, name: 'Alpha' },
    { id: 2, name: 'Beta' }
  ]
}, {
  include: [{
    association: Categories,
    as: 'categories'
  }]
})
```

***

[0]: https://www.npmjs.org/package/inflection
