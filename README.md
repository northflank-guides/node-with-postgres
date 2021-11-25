# Connecting to a PostgreSQL database using Node.js

Persisting data is essential for many Node applications. Postgres is a popular object-relational datastore which allows to store and query data in a structured manner. This guide introduces you to how to connect your Node application to a Postgres database and shows how to read and write data.

## Goals

At the end of this guide you will be able to create a Node project, setup a connection to a Postgres database and read and write some data. You will also learn how to expose this functionality in a webserver.

## Prerequisites

- Your local machine with Node & npm installed https://nodejs.org/
- Create a new directory and initialize an empty node project with `npm init`
- A running instance of Postgres with a database and user

> Need to spin up a Postgres instance? [Learn how to set up a free PostgreSQL database in minutes with Northflank](#use-northflank)

## Project structure

```
node-with-postgres/
├─ connect.js          <-- sets up postgres connection
├─ get-client.js       <-- reuse client connections
├─ setup-table.js      <-- example of creating a table in your DB
├─ add-data.js         <-- example of writing to your tables
├─ read-data.js        <-- example of reading from your tables
├─ package.json        <-- created by `npm init`, set dependency versions
├─ index.js            <-- http API server
├─ .env                <-- optional - sets up your local environment variables
└─ .gitignore          <-- optional - avoid pushing node_modules and secrets to git
```

The full source code used in this guide can be found in [this git repository](https://github.com/northflank-examples/node-with-postgres).

## Node.js + PostgreSQL

### <a name="connecting"></a>Connecting to the database

We will be using the `pg` package from NPM to open a connection. Install with `npm install pg`.

To handle sensitive connection data appropriately, we also add the the `dotenv` package: `npm install dotenv`. This allows us to locally load the sensitive data into environment variables. Create a file for the environment variables `.env` in your project directory. Here we add the connection details for the Postgres database:

```
PG_HOST=<postgres hostname>
PG_PORT=<postgres port>
PG_USER=<postgres database user>
PG_PASSWORD=<postgres database password>
PG_DATABASE=<postgres database name>
```

Set the variables to the values for your database.

Create a file `connect.js` in your project directory. Add the following code to this file to connect to Postgres and print basic information about this Postgres instance:

```javascript
const { Client } = require('pg');
require('dotenv').config();

(async () => {
  const client = new Client({
    host: process.env.PG_HOST,
    port: process.env.PG_PORT,
    user: process.env.PG_USER,
    password: process.env.PG_PASSWORD,
    database: process.env.PG_DATABASE,
    ssl: true,
  });
  await client.connect();
  const res = await client.query('SELECT $1::text as connected', ['Connection to postgres successful!']);
  console.log(res.rows[0].connected);
  await client.end();
})();
```

This script creates a Postgres client which allows you to run queries, adding and reading data to and from your database. In this example, a simple query which returns the input is run and printed to the console. In the end the `client.end()` method is called to terminate the database connection properly.

> SSL/TLS: if your database is not running with SSL/TLS, set `ssl: false` in the above example.

Now you're ready to run the script with `node ./connect.js`. This should print a message to show that the connection to Postgres was successful. 🚀

Having setup the connection to Postgres, we can now go on to add actual data to the database.

But first, to simplify things, we'll add a new file `get-client.js` which will provide a function `getClient()` and will allow to easily get the Postgres client in other files.

```javascript
const { Client } = require('pg');
require('dotenv').config();

module.exports.getClient = async () => {
  const client = new Client({
    host: process.env.PG_HOST,
    port: process.env.PG_PORT,
    user: process.env.PG_USER,
    password: process.env.PG_PASSWORD,
    database: process.env.PG_DATABASE,
    ssl: true,
  });
  await client.connect();
  return client;
};
```

### Creating a table

Now we can create a new file which makes use of this function and sets up a Postgres table: `setup-table.js`

```javascript
const { getClient } = require('./get-client');

(async () => {
  const client = await getClient();
  let createTableQuery = `
    CREATE TABLE IF NOT EXISTS my_table(
      id BIGSERIAL PRIMARY KEY NOT NULL ,
      name varchar,
      date TIMESTAMP NOT NULL DEFAULT current_timestamp
    );
  `;
  const res = await client.query(createTableQuery);
  console.log(`Created table.`);
  await client.end();
})();
```

Run it with `node ./setup-table.js`.

### Adding data

Now that we have created a table, we can go ahead and add some data. Create a new file `add-data.js`:

```javascript
const { getClient } = require('./get-client');

(async () => {
  const client = await getClient();
  const name = process.argv[2] ?? 'john';
  let insertRow = await client.query('INSERT INTO my_table(name) VALUES($1);', [`${name}`]);
  console.log(`Inserted ${insertRow.rowCount} row`);
  await client.end();
})();
```

This script will insert a row into our newly created table. Run with: `node ./add-data.js`.

You can also run this script with an extra argument which allows you to specify a custom name: `node ./add-data.js <name>` e.g. `node ./add-data.js bob`

### Reading data

To make use of the inserted data, we need to read it. Create a new script which does that `read-data.js`:

```javascript
const { getClient } = require('./get-client');

(async () => {
  const client = await getClient();

  const name = process.argv[2] ?? 'john';
  const entries = await client.query('SELECT * FROM my_table WHERE name = $1;', [name]);
  console.log(`Database entries for ${name}: ${entries.rowCount} row(s)`);
  console.log(Object.keys(entries.rows?.[0]).join('\t'));
  console.log(`${entries.rows.map((r) => Object.values(r).join('\t')).join('\n')}`);
  await client.end();
})();
```

This script will read all entries in the database with a specific name which you can specify on the command line. Test it with: `node ./read-data.js <name>` e.g. `node ./read-data.js bob`

### Putting it together with a webserver (optional)

As Node.js a app usually provides a HTTP interface of some sort, in this section we will explain how to wrap our previous examples into a web server, exposing an API with different endpoints to manipulate and read data.
The web server will contain three endpoints, one for adding a row `/add?name=<your-name>` (corresponding to `add-data.js`), one for reading a row `/read?name=<your-name>` (similar to `read-data.js`) and one for reading all data.

It makes use of the Node.js `http` package which is installed by default. As this example is slightly more involved, there will be detailed comments directly in the code: 

```javascript
const http = require('http');
const url = require('url');
const { getClient } = require('./get-client');

// This is our main function which handles serving HTTP requests
(async () => {
  const client = await getClient(); // Similar to our previous examples, we need a database client.
  await setupTable(client); // Ensure that our database table is setup correctly.

  const server = http.createServer(); // Initializing the HTTP server
  const address = { port: 8080, host: '0.0.0.0' };

  // request handler function: each request to the HTTP server will be handled here
  let requestHandler = async function (req, res) {
    // Utility function helping to return a json response and doing request logging, we can also specify the HTTP status
    // code here which allows to signal if a request was successful or give a hint why it failed.
    const jsonResponse = (responseObject, responseCode = 200) => {
      res.writeHead(responseCode, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(responseObject));

      console.log(new Date(), '-- Handled request:', req.method, req.url, responseCode);
    };

    const requestUrl = new URL(req.url, 'http://localhost:8080'); // Parse the URL from which the request came from.
    const name = requestUrl.searchParams.get('name') ?? 'john'; // We mainly need the URL to check if a specific name was passed as a query string

    // Here we setup the simple request routing, depending on which URL path was specified, we handle the request differently.
    try {
      if (requestUrl.pathname === '' || requestUrl.pathname === '/') {
        // Default URL path - we query all entries in our table without filtering
        const entries = await client.query('SELECT * FROM my_table;');
        jsonResponse(entries.rows);
      } else if (requestUrl.pathname === '/read') {
        // Read URL path - we filter by the name which was specified as query string
        const entries = await client.query('SELECT * FROM my_table WHERE name = $1;', [name]);
        jsonResponse(entries.rows);
      } else if (requestUrl.pathname === '/add') {
        // Add URL path - a new entry is inserted into the database with the specified name.
        let insertRow = await client.query('INSERT INTO my_table(name) VALUES($1);', [`${name}`]);
        jsonResponse({ success: true, message: `Inserted ${insertRow.rowCount} row with name '${name}'` });
      } else {
        // If no of our know paths is returned, we will respond with the standard "Not Found" response with HTTP response code 404
        jsonResponse("The requested route doesn't exist :(", 404);
      }
    } catch (e) {
      // If anything fails during the request handling, we handle the error gracefully by responding with the standard
      // HTTP response code 500 which stands for "Internal Server Error"
      jsonResponse(`Some error happened :(( -- (error: ${e.message})`, 500);
    }
  };
  server.on('request', requestHandler); // Here we assign our previously defined request handler to our server instance
  server.listen(address.port, address.host); // Final step: starting the server and waiting for requests.
  console.log(`Listening on: http://${address.host}:${address.port}`); // Enter the URL in your browser and check if request are handled.
})();

async function setupTable(client) { // Function with same functionality as in the 'setup-table.js' file
  let createTableQuery = `
    CREATE TABLE IF NOT EXISTS my_table(
      id BIGSERIAL PRIMARY KEY NOT NULL ,
      name varchar,
      date TIMESTAMP NOT NULL DEFAULT current_timestamp
    );
  `;
  return await client.query(createTableQuery);
}

```

The server can be started with `node index.js`.

Feel free to explore the different endpoints. Try adding new rows with different names and see how the response changes. You can also experiment with the code - for example, you can try to add an endpoint which deletes all rows corresponding to a specific name. The Postgres [SQL query](https://www.postgresql.org/docs/10/sql-delete.html) for that would be: `DELETE FROM my_table WHERE name='<your-name>';`.

> The `http` package is the most basic package for creating HTTP servers. For more complex applications with many endpoints,
it is recommended to use a more powerful web framework such as [Express](https://expressjs.com) or [Koa](https://koajs.com/).
Those frameworks allow to neatly organize request routing and to use middlewares to handle commonly needed functionality such as authentication
and many other features.

### Using Git and .gitignore (optional)

You do not want to upload your secrets and environment variables to source control. Create a `.gitignore` file with the following contents:

```
**/*.env*
**/node_modules
```

## Summary

In this how-to guide, we have shown how to use Node.js to connect to a PostgreSQL instance and how to manipulate and read data. 

In the first step, a client is instantiated. This client is then used to create a database table and rows are inserted into the table. Then we showed how to read and filter the inserted data.
In the final step, the database calls are wrapped in a web API to make it possible to access the functionality using HTTP requests. 

### <a name="use-northflank"></a>_Using Northflank to connect Node.js to PostgreSQL for free_

Northflank allows you to spin up a PostgreSQL database and a Node.js service within minutes. Sign up for a Northflank account and create a free project to get started.

> [Get started here!](https://app.northflank.com/signup)

1. Make sure your project is pushed to your preferred git provider, e.g. GitHub
2. [Sign up for a free Northflank account](https://app.northflank.com/signup)
3. Create a free Northflank project
4. Within the same project, create a PostgreSQL addon
5. Create a secret group to pass your postgres secrets to your service:
   1. Select your addon in the 'Linked addons' step
   2. Select the `host`, `port`, `database`, `username` and `password` variables and add aliases for each according to the names [here](#connecting)
6. Within your project, create a combined service:
    1. Select your git repository and branch
    2. Select Buildpack as build type
    3. Add port 8080 as a public port
