# Postgraphile jwt RLS

This is a simple [expressjs](https://expressjs.com/) demo project created based on [Row Level Security in Postgraphile](https://www.graphile.org/postgraphile/postgresql-schema-design/#row-level-security)
Mixing with [RLS](https://www.postgresql.org/docs/9.6/user-manag.html) and [RBAC](https://www.postgresql.org/docs/9.6/ddl-rowsecurity.html) from [PostgreSQL](https://www.postgresql.org/).

## Requirements

- Node 10.15+
- PostgreSQL 9.6+

## Installation

```
npm install
npm run db
```

## Running

```
npm start
```

## Structure

You can check `db.sql` file to know the sql structure.

## Testing authentication

First visit `http://localhost:3131/login` and you should get something like this back:

```
{
"1": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoidXNlcl9sb2dpbiIsInVzZXJfaWQiOjEsImlhdCI6MTU1MDUwMDgwMH0.wqSPESwLzs671yVKyBD0WK_Ppm8oXJOi06UeA7sn7Oc",
"2": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoidXNlcl9sb2dpbiIsInVzZXJfaWQiOjIsImlhdCI6MTU1MDUwMDgwMH0.gGP7YH84vdsLYiwiF7QK3FV63-qs0A63VvPgEfwoDjo",
"admin": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoidXNlcl9hZG1pbiIsInVzZXJfaWQiOjAsImlhdCI6MTU1MDUwMDgwMH0.0G1aHgGJcTwCoWCDHBY6pFhZUlb_ML-1t11DZqNTuCM"
}
```

Here we have for each user id a jwt token created with their roles inside. Also have one for `admin` in order to see all available data.

Now tht we have all data we need, we can test if user with id `1` can see all data it suppose to:

```
curl --request POST \
  --url http://localhost:3131/graphql \
  --header 'authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoidXNlcl9sb2dpbiIsInVzZXJfaWQiOjEsImlhdCI6MTU1MDUwMDgwMH0.wqSPESwLzs671yVKyBD0WK_Ppm8oXJOi06UeA7sn7Oc' \
  --header 'content-type: application/json' \
  --data '{"query":"{\n  allUsers {\n    nodes {\n      id\n      name\n      familyName\n    }\n  }\n}"}'
```

Here we are requesting all users but we would get only user id `1` as results:

```
 {
  "data": {
    "allUsers": {
      "nodes": [
        {
          "id": 1,
          "name": "Majid",
          "familyName": "Garmaroudi"
        }
      ]
    }
  }
}
```

If you repeat the same, you will get only user id `2`.

Now it is time to see what `Admin` sees. Just replace the JWT token in the last call with Admin one. You should see as results:

```
 {
  "data": {
    "allUsers": {
      "nodes": [
        {
          "id": 1,
          "name": "Majid",
          "familyName": "Garmaroudi"
        },
        {
          "id": 2,
          "name": "Paul",
          "familyName": "Rolland"
        }
      ]
    }
  }
}
```

So far we tested 2 different roles for a same query with 3 different results.
You can register a new user without have any access which means, you do not need any jwt token when you call graphql:

```
curl --request POST \
  --url http://localhost:3131/graphql \
  --header 'content-type: application/json' \
  --data '{"query":"mutation {\n  register(input: {\n    firstName: \"johan\"\n    lastName: \"zoidberg\"\n    email: \"johan@zoidberg.com\"\n    password: \"futurama\"\n  }) {\n    user {\n      id\n      name\n    }\n  }\n}"}'
```

More complex queries can be done in `meme` table where we want users have access to everyones meme but can only update their own.

```
curl --request POST \
  --url http://localhost:3131/graphql \
  --header 'authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoidXNlcl9sb2dpbiIsInVzZXJfaWQiOjEsImlhdCI6MTU1MDUwNDA3NH0._aM0Z_9F0LXG10yHLThsKtMD0QRPD_VOOH2bbkJep3g' \
  --header 'content-type: application/json' \
  --data '{"query":"{\n  allUserMemes {\n    nodes {\n      id\n      userId\n      memeUrl\n    }\n  }\n}"}'
```

This will return all memes no matter what jwt token we use.

```
{
  "data": {
    "allUserMemes": {
      "nodes": [
        {
          "id": 1,
          "userId": 1,
          "memeUrl": "http://meme1"
        },
        {
          "id": 2,
          "userId": 1,
          "memeUrl": "http://meme2"
        },
        {
          "id": 3,
          "userId": 1,
          "memeUrl": "http://meme4"
        },
        {
          "id": 4,
          "userId": 2,
          "memeUrl": "http://meme5"
        },
        {
          "id": 5,
          "userId": 2,
          "memeUrl": "http://meme6"
        }
      ]
    }
  }
}
```

But if we try to update meme number `1` with user id `2` jwt token, it would fail:

```
curl --request POST \
  --url http://localhost:3131/graphql \
  --header 'authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoidXNlcl9sb2dpbiIsInVzZXJfaWQiOjEsImlhdCI6MTU1MDUwNDA3NH0._aM0Z_9F0LXG10yHLThsKtMD0QRPD_VOOH2bbkJep3g' \
  --header 'content-type: application/json' \
  --data '{"query":"mutation {\n  updateUserMemeById(input:{id: 2,userMemePatch: {memeUrl:\"http://newmeme1\"}}) {\n    userMeme {\n      id\n      userId\n      memeUrl\n    }\n  }\n}"}'
```

You will get an error that there is nothing to update:

```
{
  "errors": [
    {
      "extensions": {
        "exception": {}
      },
      "message": "No values were updated in collection 'user_memes' because no values were found.",
      "locations": [
        {
          "line": 2,
          "column": 3
        }
      ],
      "path": [
        "updateUserMemeById"
      ],
      "stack": "Error: No values were updated in collection 'user_memes' because no values were found.\n    at commonCodeRenameMe (/private/var/www/vntrs/graphileauth/node_modules/graphile-build-pg/node8plus/plugins/PgMutationUpdateDeletePlugin.js:122:17)\n    at process._tickCallback (internal/process/next_tick.js:68:7)"
    }
  ],
  "data": {
    "updateUserMemeById": null
  }
}
```

## Contributors

- [Paul Rolland](https://github.com/PaulRolland68)

## Note

Thanks [Benjie Gillam](https://github.com/benjie) for the guides, clarifications and patiently answering all our questions.
