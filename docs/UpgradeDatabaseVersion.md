# Upgrade Database Version Documentation

### `!!! Warning only for Electron platform, iOS & Android will come later`

An API method has been added `addUpgradeStatement` to define the new structure of the database.

It has to be called prior to open the database with the new version number

ie: assuming `this._SQLiteService` is referring to the sqlite plugin and the current database version is 1

```js
...

let result:any = await this._SQLiteService.addUpgradeStatement("test-sqlite",
{fromVersion: 1, toVersion: 2, statement: schemaStmt, set: setArray});
result = await this._SQLiteService.openDB("test-sqlite",false,"no-encryption",2);
console.log('openDB result.result ' + result.result);
if(result.result) {
    ...
}
```

## Usage

As input parameters for the `addUpgradeStatement`, one must defined: - fromVersion : current database version - toVersion: new database version to be created - statement: a number of statements describing the new structure of database version (including table's definition even for those which doesn't change). - set: set of statements (`INSERT` and `UPDATE`) to upload only new data in the database

### Example

Assuming the Version 1 of the database was created as follows:

```js
let result: any = await this._SQLiteService.openDB('test-sqlite');
if (result.result) {
  // create tables
  const sqlcmd: string = `
    BEGIN TRANSACTION;
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY NOT NULL,
        email TEXT UNIQUE NOT NULL,
        name TEXT,
        company TEXT,
        size FLOAT,
        age INTEGER
    );
    CREATE INDEX IF NOT EXISTS users_index_name ON users (name);
    COMMIT TRANSACTION;
    `;
  result = await this._SQLiteService.execute(sqlcmd);
  if (result.changes.changes === 0 || result.changes.changes === 1) {
    // Insert some Users
    const row: Array<Array<any>> = [
      ['Whiteley', 'Whiteley.com', 30],
      ['Jones', 'Jones.com', 44],
    ];
    let sqlcmd: string = `
        BEGIN TRANSACTION;
        DELETE FROM users;
        INSERT INTO users (name,email,age) VALUES ("${row[0][0]}","${row[0][1]}",${row[0][2]});
        INSERT INTO users (name,email,age) VALUES ("${row[1][0]}","${row[1][1]}",${row[1][2]});
        COMMIT TRANSACTION;
        `;
    await this._SQLiteService.execute(sqlcmd);
    // Close the Database
    await this._SQLiteService.close('test-sqlite');
  }
} else {
  console.log('*** Error: Database test-sqlite not opened');
}
```

Upgrading to Version 2 will be done as below

        WARNING do not need to include these SQL commands

```js
        PRAGMA foreign_keys = OFF;
        BEGIN TRANSACTION;
        COMMIT TRANSACTION;
        PRAGMA foreign_keys = ON;
```

        they are managed by the plugin

```js
const schemaStmt: string = `
    ALTER TABLE users RENAME TO _temp_users;
    CREATE TABLE users (
    id INTEGER PRIMARY KEY NOT NULL,
    email TEXT UNIQUE NOT NULL,
    name TEXT,
    company TEXT,
    country TEXT,
    size FLOAT,
    age INTEGER
    );
    INSERT INTO users (id, email, name, company, size, age)
    SELECT id, email, name, company, size, age
    FROM _temp_users;
    DROP TABLE _temp_users;
    CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY NOT NULL,
    userid INTEGER,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    FOREIGN KEY (userid) REFERENCES users(id) ON DELETE SET DEFAULT
    );
    CREATE TABLE IF NOT EXISTS images (
    id INTEGER PRIMARY KEY NOT NULL,
    name TEXT UNIQUE NOT NULL,
    type TEXT NOT NULL,
    size INTEGER,
    img BLOB
    );
    CREATE INDEX IF NOT EXISTS users_index_name ON users (name);
    CREATE INDEX IF NOT EXISTS messages_index_name ON messages (userid);
    `
const setArray: Array<any> = [
    { statement:"INSERT INTO messages (userid,title,body) VALUES (?,?,?);",
        values:[1,"test message 1","content test message 1"]
    },
    { statement:"INSERT INTO messages (userid,title,body) VALUES (?,?,?);",
        values:[2,"test message 2","content test message 2"]
    },
    { statement:"INSERT INTO messages (userid,title,body) VALUES (?,?,?);",
        values:[1,"test message 3","content test message 3"]
    },
    { statement: "INSERT INTO images (name,type,img) VALUES (?,?,?)",
        values:["meowth","png",Images[0]]
    },
    { statement: "INSERT INTO images (name,type,img) VALUES (?,?,?)",
        values:["feather","png",Images[1]]
    },
    { statement:"UPDATE users SET country = ?  WHERE id = ?;",
        values:["United Kingdom",1]
    },
    { statement:"UPDATE users SET country = ?  WHERE id = ?;",
        values:["Australia",2]
    },

]
// call addUpgradeStatement
let result:any = await this._SQLiteService.addUpgradeStatement(
    "test-sqlite",
    {
        fromVersion: 1,
        toVersion: 2,
        statement: schemaStmt,
        set: setArray
    });
// open the database
result = await this._SQLiteService.openDB("test-sqlite",false,"no-encryption",2);
if(result.result) {
console.log("*** Database test-sqlite Opened");
    ...
}
```

The images used in `setArray`constant are for example:

```js
const Images: Array<string> = [
  'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACgAAAAkCAYAAAD7PHgWAAAEcElEQVRYR8WYP2hTQRzHfx10aQchi0JcLGpBSBcrlTrpIjoFiy6FDipOHVz8Q0HrUGxdg1N1KBRBackiVoQ6FMVIuzQgpEpdjOiSLUXQIfK9976X37t3l6RNxVuS3Hvv7nPf3+/3vcvraTQaDdlFK4z3yMT8rh7d0Ww97QAzfX12wFq9br4buOk7UpicaQm5F4toCajh9LKnLm23Bex0Ee3k7ArwS/mVvH5elqEzzWmGr0dhDwGGFs3ouMAdA7491y+Dhw5KZuG9UEEA1r6XZfhUPOxgQ0pzPQJIDTi11NtOKOkKkHCcpfDrjQlxaXnGdFE1fAcg2to7sWmgAfVYWCzbPwO06imNHt0Tyd/IyfDlrYRy7kI3fvyUsyvRPbsCxIPIGQ6MAdFWD5RbKnjxZhTSWn0+AqyuS2agEPWNjZhPjrUngBgQkABDQ3hNOJdnmvkXa5UZ6W2CxXBaRoBiLLR2cLgnUSRIbOSLlptVx8LQk7k5iHutah44Pks12+VfApBVh04YsAbV1yR7sslYXU+oSPUK46NWZWPmseJdATLfTJ5UJsxYBNXqoc+EeX7RgpbmRmX1pcjsSq95VkP5AM1czMl63ViS27iNen2QYSUoH+bWVq1WpTh5OAFp1ekbtz7JRVJBPH/+Sk6O5i4YQCxc57Sbq0i1loA2R6hKfDho7rFLqZWzYvXiqCKgSi/6LSC+o7l2ZCIWz5UChHqfH2alvPVVRp/sT4Q7P/1NstmssZ6okNKAyD803+5BICjohjm90qgnAajhcNEHiP7BgQHZqFQkK49FF40uDtyHrZAKEQ6/NWDIoAkcBAQcmpuHoZWG+l1IwlHBjgGp3rP1zchi4kpG3vi+7wQUkMgz5p8tKIwdnzHbhtiatALTRcLvtBnmmc/ANQCuo3JxLGMF6+tmHFUULqgJsUl6Bwy/jXr1elQUWlGnj37JyfQksBhWL/tpM/itK9kHanOQ3rd47bcZxxSIkl97ow67u2Lfouh/+l6EnIvXuU5/TNkMAAjnA7RhUf9RQkWkTRhh9TUCuuO6kUooCMBc/xHzzLG71ZYJjAUhPD6TDUERxoXTC7CRiqOXAIRBZ/J5e3/oXxvhdE6FqpA2g+sslFaA3iLRMmvfYz6l8ixWD/3adF0bwXUNiN87gcP9qfOg72jkepVWkIC6ELQZu5BdAWIwbSl6F9AWQEAXRB8GtOpaxa4BCan3Tp3cemJ3G9R+R/g9DbGenDtLCJQVHIL0AeqKb7fFkaWjdzMIrz4+afdvpWKoslks+Lx9YltufQy/hPICUj1OQAOHR9KGeABwAfk6xOeFOmdrxaI5c6Ktffgjs5/4VzV6QRVUkKcafRMHQh8hQ9udPrm4ChJQw7n3EJYp4D0PPl3YlKtjx+0K3UEAiZ3G9T3fATWRd5UJ8cEBCm3o9D47Fc8CKUCEEw/om/kUD7H4zY2e+Vh8UJb8/fTrDt+BA8/rfZ/j63m9gLSYUHL7Ks99ndZpdYZew3Fub4hbVd3/uvYXfqiMwjPten8AAAAASUVORK5CYII=',
  'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAMAAABEpIrGAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAU1QTFRFNjtAQEVK////bG9zSk9T/v7+/f39/f3+9vf3O0BETlJWNzxB/Pz8d3t+TFFVzM3O1NXX7u/vUldbRElNs7W3v8HCmZyeRkpPW19j8vLy7u7vvsDC9PT1cHR3Oj9Eo6WnxsjJR0tQOD1Bj5KVgYSHTVFWtri50dLUtLa4YmZqOT5D8vPzRUpOkZOWc3Z64uPjr7Gzuru95+jpX2NnaGxwPkNHp6mrioyPlZeadXh8Q0hNPEBFyszNh4qNc3d6eHx/OD1Cw8XGXGBkfoGEra+xxcbIgoaJu72/m52ggoWIZ2tu8/P0wcLE+vr7kZSXgIOGP0NIvr/BvL6/QUZKP0RJkpWYpKaoqKqtVVldmJqdl5qcZWhstbe5bHB0bnJ1UVVZwsTF5ubnT1RYcHN3oaSm3N3e3NzdQkdLnJ+h9fX1TlNX+Pj47/DwwsPFVFhcEpC44wAAAShJREFUeNq8k0VvxDAQhZOXDS52mRnKzLRlZmZm+v/HxmnUOlFaSz3su4xm/BkGzLn4P+XimOJZyw0FKufelfbfAe89dMmBBdUZ8G1eCJMba69Al+AABOOm/7j0DDGXtQP9bXjYN2tWGQfyA1Yg1kSu95x9GKHiIOBXLcAwUD1JJSBVfUbwGGi2AIvoneK4bCblSS8b0RwwRAPbCHx52kH60K1b9zQUjQKiULbMDbulEjGha/RQQFDE0/ezW8kR3C3kOJXmFcSyrcQR7FDAi55nuGABZkT5hqpk3xughDN7FOHHHd0LLU9qtV7r7uhsuRwt6pEJJFVLN4V5CT+SErpXt81DbHautkpBeHeaqNDRqUA0Uo5GkgXGyI3xDZ/q/wJMsb7/pwADAGqZHDyWkHd1AAAAAElFTkSuQmCC',
];
```
