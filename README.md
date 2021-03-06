# express-es6-rest-starter
Express-es6-rest-starter - is a node.js express app skeleton that allows you to develop faster rest services.
For ORM  it uses [Sequelize](https://github.com/sequelize/sequelize) with a custom query builder that lets you create easily complex queries from your request queries strings.Also uses [node-jwt-simple](https://github.com/hokaccha/node-jwt-simple) for authentication and [node-cache-manager](https://github.com/BryanDonovan/node-cache-manager) for managing cache.

Features
--------

- **ORM** [Sequelize](https://github.com/sequelize/sequelize)
- **Cache** [node-cache-manager](https://github.com/BryanDonovan/node-cache-manager)
- **Jwt Tokens** [node-jwt-simple](https://github.com/hokaccha/node-jwt-simple)
- Complex query builder from request query strings
- Auto basic CRUD controllers and routes
- Easily handle almost everything from your model and develop faster

## Folder tree
```
├── express-es6-rest-starter/
│   ├── src/
|   |     ├── controllers/
|   |     |     ├── authentication.js
|   |     |     ├── generic.js
|   |     |     ├── users.js
|   |     ├── helpers/
|   |     |     ├── baseModel.js
|   |     |     ├── cache.js
|   |     |     ├── error.js
|   |     |     ├── queryBuilder.js
|   |     |     ├── routesConstructor.js
|   |     ├── lib/
|   |     |     ├── index.js
|   |     ├── middlewares/
|   |     |     ├── expired_plan.js
|   |     |     ├── index.js
|   |     |     ├── roles.js
|   |     ├── migrations/
|   |     |     ├── generic.js
|   |     |     ├── index.js
|   |     |     ├── migration.js
|   |     ├── models/
|   |     |     ├── profiles.js
|   |     |     ├── index.js
|   |     |     ├── users.js
|   |     ├── prototypes/
|   |     |     ├── index.js
|   |     ├── services/
|   |     |     ├── passport.js
|   |     ├── config.js
|   |     ├── router.js
├── .gitignore
├── .package.json
├── .README.md
├── .babelrc
├── .env
```
## Get Started

Set up your .env
```
.env
NODE_ENV=development //set enviroment
DB_DIALECT=mysql     //dialect
DB_HOST=localhost    //host
DB_NAME=yourDb       //database name
DB_USER=root         //database user
DB_PASS=password     //database password
POOL_MAX=5           //max pool
POOL_MIN=0           //min pool
POOL_IDLE=10000      //pool idle
CACHE_STORE=memory   //Cache (in the skeleton just 'redis' and 'memory' but if you want you can use all available from node-cache-manager)
```

Set up your config.js
```javascript
src/config.js

export default {
    secret: 'yourSecretKey', //jwt Secret
    expiresIn: 60,           //expiration time in minutes
    ttl: 120                 //refresh time in minutes
}
```

Check out the two models users and profiles which have a relation with each other.You dont need to create a route and a controller the routes are created automatically and for controller they use generic controller which is simple CRUD controler and if you look at override field you can bypass generic controller with a custom function.
```javascript
src/models/users


import Sequelize from "sequelize";
import { baseModel } from '../helpers/baseModel';
import bcrypt from 'bcrypt-nodejs';
import passport from 'passport';
import { update } from '../controllers/users';
import * as middlewares from '../middlewares/index';


const auth = passport.authenticate('jwt', { session: false });


const defaultOptions = { //set up Sequelize options
    freezeTableName: true,
    paranoid: true,
    instanceMethods: {
        toJSON: function() {
            let values = this.get();

            delete values.password;
            return values;
        },
        verifyPassword: function(password, done) {
            return bcrypt.compare(password, this.password, (err, isMatch) => {
                return done(err, isMatch);
            });
        },
    },
    hooks: {
        beforeUpdate: function(user, options, next) {
            bcrypt.genSalt(10, function(err, salt) {
                if (err) {
                    return next(err);
                }

                bcrypt.hash(user.password, salt, null, (err, hash) => {
                    if (err) {
                        return next(err);
                    }

                    user.password = hash;
                    next();
                });
            });
        },
        beforeCreate: function(user, options, next) {
            bcrypt.genSalt(10, function(err, salt) {
                if (err) {
                    return next(err);
                }

                bcrypt.hash(user.password, salt, null, (err, hash) => {
                    if (err) {
                        return next(err);
                    }

                    user.password = hash;
                    next();
                });
            });
        }
    }
};

export const users = { // set up user model
    name: 'users',
    middlewares: { //attach middlewares to autogenerated routes
        general: [auth, middlewares.expired()], //middlewares that applied to all autogenerated user routes
        save:[] //override general middlewares for the save route (possible override routes are list,get,save,update,delete,restore)
    },
    model: Object.assign({}, baseModel, { //create my model fields
        username: { type: Sequelize.STRING, unique: true, allowNull: false },
        password: { type: Sequelize.STRING, allowNull: false },
        role: { type: Sequelize.STRING }
    }),
    options: defaultOptions,
    relations: true, //set relation too true so we can define relations in definedRelations field
    //set up a hasOne relation with profiles model 
    definedRelations: [{ name: 'profiles', type: 'hasOne', foreignKey: 'user_id', as: 'profile' }], 
    controllerGeneric: true, //create autogenerated routes and use generic controller
    filters: [], //custom filters for the specific endpoint
    // override generic controller for a specific route (possible override controllers are list,get,save,update,delete,restore)
    override: { 
        update: update // override update method with a custom method 
    }
}

```

```javascript
src/models/profiles

import Sequelize from "sequelize";
import { baseModel } from '../helpers/baseModel';
import passport from 'passport';
import * as middlewares from '../middlewares/index';


const auth = passport.authenticate('jwt', { session: false });


const defaultOptions = {
    freezeTableName: true,
    paranoid: true,   
};

export const profiles = {
    name: 'profiles',
    middlewares: {
        general: [auth, middlewares.expired()],
    },
    model: Object.assign({}, baseModel, {
        name: { type: Sequelize.STRING, allowNull: false },
        email: {
            type: Sequelize.STRING,
            unique: true,
            allowNull: false,
            validate: { isEmail: { msg: "email is not valid" } },
            set: function(val) {
                this.setDataValue('email', val.toLowerCase());
            }
        }
    }),
    options: defaultOptions,
    relations: true,
    definedRelations: [{ name: 'users', type: 'belongsTo', foreignKey: 'user_id', as: 'user' }],
    controllerGeneric: true,
    filters: [],
    override: {}
}

```

Now you have 12 availables routes

```
GET /api/users
GET /api/users/:id
POST /api/users
PUT /api/users/:id
DELETE /api/users/:id
GET /api/users/restore  ///you can use restore if you use paranoid in your model(soft delete) to restore your deleted data with a query param string eg. /api/users/restore?id=1

GET /api/profiles
GET /api/profiles/:id
POST /api/profiles
PUT /api/profiles/:id
DELETE /api/profiles/:id
GET /api/profiles/restore
```

## API
**Models options**
- **name**: string | table name 
- **middlewares**: array of objects | object with options **general**, **list**,**get**, **save**, **update**, **delete**, **restore**
- **middlewares.general**: array | an array of middlewares to apply to all routes
- **middlewares[avail. options]**: array | override general middleware or apply specific middlewares for this route
- **model**: object | sequelize model
- **options**: object  | sequelize options
- **relations**: boolean 
- **definedRelations**: array of objects | an array with the model relations 
```javascript
definedRelations: [
  { 
    name: //relation name, 
    type: //relation type, available fields sequilize fields, 
    foreignKey: //foreign key
    as: //alias required for the moment
    through: //table for belongsToMany relations 
  }
],
```
- **controllerGeneric**: boolean | autogenerate routes and use generic controller
- **filters**: array of objects | create custom filters for the specific route
example 

```javascript
filters:[
  {
    name:"test",
    fn: () => {return {id:1}}
  }
]

//now call the name as query param ex. /api/users?filter=test or multiple /api/users?filter=test&filter=test2
```
- **override**: object | override generic controller for specific routes available options **general**, **list**,**get**, **save**, **update**, **delete**, **restore**
- **override[options]**: boolean Or function | set it to true if you dont want the route to be created at all or as function to bypass generic controller


**Query builder**

Creates complex queries from request queries string 

  **options**
  - **relation**: the alias of the relation or all for every possible relation
  
  ```bash
  //example
  /api/users?relation=profile  //will return the users with the profile relation
  /api/users?relation=all      //will return the users and every possible relation 
  /api/users?relation=profile&relation=someOtherRelation    //will return the users and the two relation
  /api/users?relation=profile&profile.include=someNestedRelation  //will return the users the profiles and profiles relation
  
  ```
  
  - **search**: search in model fields
  - **model.field**: search for a specif field in the model
  ```bash
  //example
  
  /api/users?username=something 
  /api/users?profile.id=1 // for relation field search
  ```
  - **limit**: limit result 
    ```bash
  //example
  
  /api/users?limit=10
  /api/users?relation=someRelation&someRelation.limit=5  //limit for many to many or many to one relations
  ```
  - **page**: page number started at 1
  - **attributes**: select table fields comma seperated
  ```bash
  //example
  
  /api/users?attributes=username,id  //returns only the username and the id
  /api/users?relation=profile&profile.attributes=email,id //returns users and from profile only the email and the id fields
  ```
  
  - **exclude**: exclude table fields comma seperated
  ```bash
  //example
  
  /api/users?exclude=username,id  //returns excludes the username and the id
  /api/users?relation=profile&profile.exclude=email,id //returns users and from profile excludes the email and the id fields
  ```
  
  - **paranoid**: return soft deleted rows
  - **order**: field to order from
  - **orderType**: ASC or DESC
  - **nested includes**: examples of how to use nested includes
  ```bash
  //example
  
  /api/customers?relation=malfunctions&malfunctions.include=model&malfunctions.model.include=peripherals
  /api/customers?relation=malfunctions&malfunctions.include=model&malfunctions.model.id=1
  /api/customers?relation=malfunctions&malfunctions.include=model&malfunctions.model.attributes=name,serialNumber
  ```
