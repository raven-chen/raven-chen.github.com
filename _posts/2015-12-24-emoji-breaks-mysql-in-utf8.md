# Mysql2::Error: Incorrect string value

Recently, Got some errors about "Incorrect string value" in my site, Since the site is mostly used by Chinese, I thought it caused by incorrect database encoding. After check the database, All tables are set in UTF-8. Finally I found there're some emoji in the data that user posted. This needs `utf8mb4` encoding. So use

```sh
  ALTER TABLE table_name CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```

resolved it.

The problem might be complicate in some case, [here](http://blog.arkency.com/2015/05/how-to-store-emoji-in-a-rails-app-with-a-mysql-database/) is a nice article that introduced how to support emoji in database.
