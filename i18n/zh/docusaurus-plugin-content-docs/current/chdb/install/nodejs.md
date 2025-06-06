---
'title': '为 NodeJS 安装 chDB'
'sidebar_label': 'NodeJS'
'slug': '/chdb/install/nodejs'
'description': '如何为 NodeJS 安装 chDB'
'keywords':
- 'chdb'
- 'embedded'
- 'clickhouse-lite'
- 'NodeJS'
- 'install'
---


# 安装 chDB 用于 NodeJS

## 需求 {#requirements}

安装 [libchdb](https://github.com/chdb-io/chdb):

```bash
curl -sL https://lib.chdb.io | bash
```

## 安装 {#install}

```bash
npm i chdb
```

## GitHub 仓库 {#github-repository}

您可以在 [chdb-io/chdb-node](https://github.com/chdb-io/chdb-node) 找到该项目的 GitHub 仓库。

## 使用 {#usage}

您可以通过导入和使用 chdb-node 模块，在 NodeJS 应用程序中利用 chdb 的强大功能：

```javascript
const { query, Session } = require("chdb");

var ret;

// Test standalone query
ret = query("SELECT version(), 'Hello chDB', chdb()", "CSV");
console.log("Standalone Query Result:", ret);

// Test session query
// Create a new session instance
const session = new Session("./chdb-node-tmp");
ret = session.query("SELECT 123", "CSV")
console.log("Session Query Result:", ret);
ret = session.query("CREATE DATABASE IF NOT EXISTS testdb;" +
    "CREATE TABLE IF NOT EXISTS testdb.testtable (id UInt32) ENGINE = MergeTree() ORDER BY id;");

session.query("USE testdb; INSERT INTO testtable VALUES (1), (2), (3);")

ret = session.query("SELECT * FROM testtable;")
console.log("Session Query Result:", ret);

// Clean up the session
session.cleanup();
```

## 从源代码构建 {#build-from-source}

```bash
npm run libchdb
npm install
npm run test
```
