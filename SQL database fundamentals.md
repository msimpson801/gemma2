# ⚓ SQL Database Fundamentals — A Pirate's Guide

> _"A ship without a database is just a boat. A pirate without a primary key is just a rascal."_ — Nobody, but they should have.

Welcome aboard, landlubber! You're about to learn the fundamentals of SQL databases — and we're going to do it with pirates. By the time we're done, you'll understand primary keys, foreign keys, relationships, and joins well enough to make even Davy Jones nod approvingly.

Grab your compass. Let's navigate.

---

## 🗺️ What Even Is a Database Table?

Before we hoist the sails, let's get grounded. A **database** is just an organised collection of data. Inside that database, data is stored in **tables** — which look a lot like spreadsheets.

Here's a simple `pirates` table:

|pirate_id|name|rank|
|---|---|---|
|1|Redbeard Rick|Captain|
|2|Salty Sue|Navigator|
|3|One-Eye Pete|Deckhand|

Each **row** is one pirate. Each **column** is a piece of information about them. Simple enough, right?

Now, here's where it gets interesting.

---

## 🔑 Primary Keys — Every Pirate Needs a Unique Identity

Imagine two pirates both named "Black Bob." How do you tell them apart in your database? By name alone — you can't. That's where the **primary key** comes in.

A **primary key** is a column (or group of columns) that **uniquely identifies every row** in a table. No two rows can have the same primary key value. Ever.

```sql
CREATE TABLE pirates (
    pirate_id INT PRIMARY KEY,
    name      VARCHAR(100),
    rank      VARCHAR(50)
);
```

In our `pirates` table, `pirate_id` is the primary key. It doesn't matter if we have a hundred pirates named "Black Bob" — each one gets a unique `pirate_id`, and the database can tell them apart instantly.

**Rules of a primary key:**

- ✅ Must be **unique** — no duplicates
- ✅ Must **never be NULL** — every row needs one
- ✅ Should **never change** — a pirate's ID is for life

Think of it like a pirate's official ship registry number. Their name might change (pirates do love a good alias), but that registry number? That's forever.

---

## 🔗 Foreign Keys — Connecting the Dots Between Tables

Now, a pirate doesn't just exist in isolation. They belong to a ship, they have a parrot, they visit islands. We need a way to **link tables together**.

That's the job of a **foreign key**.

A **foreign key** is a column in one table that **refers to the primary key in another table**. It's the bridge between two tables, and it's how relational databases earn the "relational" part of their name.

Let's add a `ships` table:

|ship_id|ship_name|
|---|---|
|1|The Rusty Anchor|
|2|Queen Anne's Fury|

Now let's update our pirates table to include which ship they sail on:

|pirate_id|name|rank|ship_id|
|---|---|---|---|
|1|Redbeard Rick|Captain|1|
|2|Salty Sue|Navigator|1|
|3|One-Eye Pete|Deckhand|2|

The `ship_id` column in `pirates` is a **foreign key** — it points back to the `ship_id` primary key in `ships`. This is how the database knows that Redbeard Rick and Salty Sue both sail on _The Rusty Anchor_.

```sql
CREATE TABLE pirates (
    pirate_id INT PRIMARY KEY,
    name      VARCHAR(100),
    rank      VARCHAR(50),
    ship_id   INT,
    FOREIGN KEY (ship_id) REFERENCES ships(ship_id)
);
```

The golden rule of foreign keys: **the value must exist in the other table first** (or be NULL). You can't assign a pirate to ship number 99 if ship 99 doesn't exist. The database will throw an error and protect you from bad data. Handy!

---

## 💑 Relationships — How Tables Talk to Each Other

With primary and foreign keys in hand, we can now model **relationships** between things in the real world. There are three main types.

---

### 1️⃣ One-to-One — A Pirate and Their Parrot

A pirate has exactly one parrot. A parrot belongs to exactly one pirate. This is a **one-to-one** relationship.

|parrot_id|name|colour|pirate_id|
|---|---|---|---|
|1|Captain|Red|1|
|2|Crackers|Green|2|
|3|Doubloon|Blue|3|

Each `pirate_id` appears only once in the `parrots` table. Redbeard Rick has Captain the parrot. Salty Sue has Crackers. One-to-one.

**When do you use this?** When two things are so closely tied that they could almost be one table — but you want to keep them separate for clarity or performance.

---

### 2️⃣ One-to-Many — Many Pirates to One Ship

A ship can have _many_ pirates. But each pirate sails on only _one_ ship at a time. This is a **one-to-many** relationship — and it's the most common type you'll encounter.

We've already modelled this! The `ship_id` foreign key on the `pirates` table does exactly this. One ship, many pirates pointing at it.

```
ships            pirates
---------        ---------------
ship_id: 1  <──  ship_id: 1  (Redbeard Rick)
                 ship_id: 1  (Salty Sue)

ship_id: 2  <──  ship_id: 2  (One-Eye Pete)
```

The "many" side always holds the foreign key. That's the rule.

---

### 🔀 Joins — Bringing the Tables Together

Now that our data is spread across multiple tables, we need a way to **query them together**. That's what a **JOIN** does — it combines rows from two (or more) tables based on a related column.

Let's say we want to see each pirate alongside the name of their ship. We'd write:

```sql
SELECT pirates.name, ships.ship_name
FROM pirates
JOIN ships ON pirates.ship_id = ships.ship_id;
```

This is called an **INNER JOIN** (or just `JOIN`). It returns only rows where there's a match in **both** tables.

**Result:**

|name|ship_name|
|---|---|
|Redbeard Rick|The Rusty Anchor|
|Salty Sue|The Rusty Anchor|
|One-Eye Pete|Queen Anne's Fury|

Great! But wait... what about edge cases? What if a pirate has no ship? What if a ship has no pirates?

---

### 🏝️ The Stranded Pirate — LEFT JOIN

Meet **Marooned Maria**. She's a pirate, but she got stranded on a desert island. She has no ship. Her `ship_id` in the database is `NULL`.

|pirate_id|name|ship_id|
|---|---|---|
|4|Marooned Maria|NULL|

If we run our plain `INNER JOIN` from before, **Marooned Maria won't appear at all**. The INNER JOIN only returns rows where a match exists in both tables. No ship? No row.

But what if we _want_ to see all pirates, even the ones without a ship? That's a **LEFT JOIN**:

```sql
SELECT pirates.name, ships.ship_name
FROM pirates
LEFT JOIN ships ON pirates.ship_id = ships.ship_id;
```

**Result:**

|name|ship_name|
|---|---|
|Redbeard Rick|The Rusty Anchor|
|Salty Sue|The Rusty Anchor|
|One-Eye Pete|Queen Anne's Fury|
|Marooned Maria|NULL|

A **LEFT JOIN** returns **all rows from the left table** (pirates), and fills in `NULL` for anything it can't find on the right side (ships). Marooned Maria is included — she just has no ship name, because she has no ship.

> 💡 **Left vs Right**: The "left" and "right" just refer to the order tables appear in your query. `FROM pirates LEFT JOIN ships` — pirates is the left table, ships is the right.

---

### 👻 The Ghost Ship — RIGHT JOIN (or a Flipped LEFT JOIN)

Now imagine **The Flying Dutchman** — a legendary ghost ship with no living pirates aboard. It exists in the `ships` table, but no pirate has `ship_id` pointing to it.

|ship_id|ship_name|
|---|---|
|3|The Flying Dutchman|

A regular `LEFT JOIN` (pirates on the left) would _not_ include this ship, because there are no pirates linking to it from the left table.

To get all ships — even empty ones — we can either:

**Option A: RIGHT JOIN** (get all from the right table)

```sql
SELECT pirates.name, ships.ship_name
FROM pirates
RIGHT JOIN ships ON pirates.ship_id = ships.ship_id;
```

**Option B: Flip it! LEFT JOIN with ships first**

```sql
SELECT pirates.name, ships.ship_name
FROM ships
LEFT JOIN pirates ON pirates.ship_id = ships.ship_id;
```

Both give the same result:

|name|ship_name|
|---|---|
|Redbeard Rick|The Rusty Anchor|
|Salty Sue|The Rusty Anchor|
|One-Eye Pete|Queen Anne's Fury|
|NULL|The Flying Dutchman|

The ghost ship appears! Its pirate name is `NULL` because no pirates are aboard. Spooky — but useful.

---

### 🌐 FULL JOIN — Show Everything, Leave No Pirate (or Ship) Behind

What if you want _everything_? All pirates, all ships — even the stranded ones and the ghost ships?

That's a **FULL OUTER JOIN**:

```sql
SELECT pirates.name, ships.ship_name
FROM pirates
FULL OUTER JOIN ships ON pirates.ship_id = ships.ship_id;
```

|name|ship_name|
|---|---|
|Redbeard Rick|The Rusty Anchor|
|Salty Sue|The Rusty Anchor|
|One-Eye Pete|Queen Anne's Fury|
|Marooned Maria|NULL|
|NULL|The Flying Dutchman|

Nobody gets left out. Every pirate, every ship — matched where possible, NULL where not.

---

### Quick JOIN Cheat Sheet

|JOIN Type|What it returns|
|---|---|
|`INNER JOIN`|Only rows with a match in **both** tables|
|`LEFT JOIN`|All rows from the **left** table + matched rows from right|
|`RIGHT JOIN`|All rows from the **right** table + matched rows from left|
|`FULL JOIN`|All rows from **both** tables, matched where possible|

---

## 🗺️ Many-to-Many — Pirates, Islands, and the Junction Table

Right, here's where things get a little more interesting. Arrr, but don't panic!

A pirate can **visit many islands**. An island can be **visited by many pirates**. This is a **many-to-many** relationship — and it breaks the simple foreign key model we've been using.

Why? Because if we put `island_id` on the `pirates` table, a pirate could only link to _one_ island. And if we put `pirate_id` on the `islands` table, an island could only link to _one_ pirate. Neither works.

**The solution: a Junction Table** (also called a _bridge table_ or _join table_).

We create a third table whose entire purpose is to record _each individual visit_:

```sql
CREATE TABLE pirate_island_visits (
    pirate_id INT,
    island_id INT,
    PRIMARY KEY (pirate_id, island_id),
    FOREIGN KEY (pirate_id) REFERENCES pirates(pirate_id),
    FOREIGN KEY (island_id) REFERENCES islands(island_id)
);
```

Now let's say we have these islands:

|island_id|island_name|
|---|---|
|1|Tortuga|
|2|Skull Island|
|3|Dead Man's Cove|

And these visits:

|pirate_id|island_id|
|---|---|
|1|1|
|1|2|
|2|1|
|3|3|
|3|1|

Reading this: Redbeard Rick (pirate 1) visited Tortuga and Skull Island. Salty Sue visited Tortuga. One-Eye Pete visited Dead Man's Cove and Tortuga.

Now we can query it all together:

```sql
SELECT pirates.name, islands.island_name
FROM pirates
JOIN pirate_island_visits ON pirates.pirate_id = pirate_island_visits.pirate_id
JOIN islands ON pirate_island_visits.island_id = islands.island_id;
```

**Result:**

|name|island_name|
|---|---|
|Redbeard Rick|Tortuga|
|Redbeard Rick|Skull Island|
|Salty Sue|Tortuga|
|One-Eye Pete|Dead Man's Cove|
|One-Eye Pete|Tortuga|

The junction table acts as the **glue** between the two tables. It turns an impossible many-to-many into two perfectly normal one-to-many relationships.

> 💡 **Bonus tip**: Junction tables can hold _extra data_ about the relationship! You could add a `visit_date` column to `pirate_island_visits` to record _when_ each pirate visited each island. The relationship itself has data — and now you have somewhere to put it.

---

## 🧭 Summary — The Treasure Map of What You've Learned

Let's recap the whole voyage:

|Concept|What it does|
|---|---|
|**Primary Key**|Uniquely identifies every row in a table|
|**Foreign Key**|Links a row in one table to a row in another|
|**One-to-One**|One pirate, one parrot — modelled with a FK on one side|
|**One-to-Many**|One ship, many pirates — FK lives on the "many" side|
|**Many-to-Many**|Pirates and islands — solved with a junction table|
|**INNER JOIN**|Only matched rows from both tables|
|**LEFT JOIN**|All rows from left table, NULLs where no match on right|
|**RIGHT JOIN**|All rows from right table, NULLs where no match on left|
|**FULL JOIN**|Everything from both tables|

---

## 🏴‍☠️ Final Words from the Captain

Databases aren't scary — they're just organised ways to store and connect information. The pirate might change ships, acquire a new parrot, or discover a new island. With primary keys, foreign keys, and the right joins, your database can model all of it without breaking a sweat.

Now get out there and write some SQL. The seven seas of data await. ⚓