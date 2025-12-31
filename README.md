```bash
git clone https://github.com/Mahin556/voting-app.git

cd voting-app

openssl rand -base64 756 > mongo-keyfile
chmod 400 mongo-keyfile
chown 999:999 mongo-keyfile

docker compose up -d mongo-0 mongo-1 mongo-2

docker exec -it root-mongo-0-1 mongosh \
  -u root \
  -p example \
  --authenticationDatabase admin

use admin

rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-0:27017" },
    { _id: 1, host: "mongo-1:27017" },
    { _id: 2, host: "mongo-2:27017" }
  ]
})

use langdb;

db.languages.insert({"name" : "csharp", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 5, "compiled" : false, "homepage" : "https://dotnet.microsoft.com/learn/csharp", "download" : "https://dotnet.microsoft.com/download/", "votes" : 0}});
db.languages.insert({"name" : "python", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 3, "script" : false, "homepage" : "https://www.python.org/", "download" : "https://www.python.org/downloads/", "votes" : 0}});
db.languages.insert({"name" : "javascript", "codedetail" : { "usecase" : "web, client-side", "rank" : 7, "script" : false, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "download" : "n/a", "votes" : 0}});
db.languages.insert({"name" : "go", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 12, "compiled" : true, "homepage" : "https://golang.org", "download" : "https://golang.org/dl/", "votes" : 0}});
db.languages.insert({"name" : "java", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 1, "compiled" : true, "homepage" : "https://www.java.com/en/", "download" : "https://www.java.com/en/download/", "votes" : 0}});
db.languages.insert({"name" : "nodejs", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 20, "script" : false, "homepage" : "https://nodejs.org/en/", "download" : "https://nodejs.org/en/download/", "votes" : 0}});

db.languages.find().pretty();

exit

docker compose up -d backend

docker compose up -d frontend
```

---

