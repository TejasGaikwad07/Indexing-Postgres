# Indexing in Postgres

We’ve created postgres tables many times now. Let’s see how/if indexing helps us speed up queries

- Create a postgres DB locally 

```jsx
docker run  -p 5432:5432 -e POSTGRES_PASSWORD=mysecretpassword -d postgres
```

- Connect to it and create some dummy data in it

```jsx
docker exec -it container_id /bin/bash
psql -U postgres
```

- Create the schema for a simple medium like app

```jsx
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    name VARCHAR(255)
);
CREATE TABLE posts (
    post_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    image VARCHAR(255),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

- Insert some dummy data in

```jsx
DO $$
DECLARE
    returned_user_id INT;
BEGIN
    -- Insert 5 users
    FOR i IN 1..5 LOOP
        INSERT INTO users (email, password, name) VALUES
        ('user'||i||'@example.com', 'pass'||i, 'User '||i)
        RETURNING user_id INTO returned_user_id;

        FOR j IN 1..500000 LOOP
            INSERT INTO posts (user_id, title, description)
            VALUES (returned_user_id, 'Title '||j, 'Description for post '||j);
        END LOOP;
    END LOOP;
END $$;
```

- Try running a query to get all the posts of a user and log the time it took

```jsx
 EXPLAIN ANALYSE SELECT * FROM posts WHERE user_id=1 LIMIT 5;
```

Focus on the `execution time`

- Add an index to user_id

```jsx
CREATE INDEX idx_user_id ON posts (user_id);
```

Notice the `execution time` now. 
What do you think happened that caused the query time to go down by so much?

## How indexing works (briefly)

When you create an index on a field, a new data structure (usually B-tree) is created that stores the mapping from the `index column` to the `location` of the record in the original table. 

Search on the index is usually `log(n)` 

### Without indexes

<img width="1252" alt="without indexs" src="https://github.com/TejasGaikwad07/Indexing-Postgres/assets/70066236/f55f86ea-19f1-40e4-a77b-58b3aa01634d">

### With indexes

<img width="1332" alt="with index" src="https://github.com/TejasGaikwad07/Indexing-Postgres/assets/70066236/c4f358da-4332-4a24-abda-918839a07005">

The data pointer (in case of postgres) is the `page` and `offset` at which this record can be found. 

Think of the index as the `appendix` of a book and the `location` as the `page + offset` of where this data can be found
