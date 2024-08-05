# arch_pg_render

Render engine extension for bare metal PostgreSQL.

# Installation
## Arch Linux w/ PostgreSQL 16
```bash

Install YaY
pacman -Syu git
sudo useradd -m -G wheel user
sudo EDITOR=vim visudo
Find the and uncomment wheel group without password line 108
passwd user
confirm password
su - user
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

## Initalize the postgres database
1.) yay -Syu postgrest-bin postgresql
2.) su postgres
3.) initdb --locale=C.UTF-8 --encoding=UTF8 -D /var/lib/postgres/data --data-checksums
4.) exit
5.) systemctl start postgresql
6.) systemctl enable postgresql
7.) vim /var/lib/postgres/.psql_history
8.) :wq
9.) chown postgres /var/lib/postgres/.psql_history





# In PostgreSQL
create extension pg_render;


# Copy the MakeFile

# 

# Examples

```sql
-- PostgreSQL extension
create extension pg_render;
-- Serve /index using postgREST
create function api.index() returns "text/html" as $$
-- Render HTML template with pg_render
select render(
  '<html>
    <head>
      <title>{{ title }}</title>
    </head>
    <body>
      <h1>{{ title }}</h1>
      <p>{{ text }}</p>
      <strong>{{ author }}</strong>
    </body>
  </html>',
  (select to_json(props) from (select title, text, author from posts where id = 1) props)
)
$$;
```

->

```html
# HTTP GET /index
<html>
  <head>
    <title>Example</title>
  </head>
  <body>
    <h1>Example</h1>
    <p>Example text</p>
    <strong>Example author</strong>
  </body>
</html>
```

# Inserts

See more examples in [pg_render_example](https://github.com/mkaski/pg_render_example/blob/master/sql/02_views.sql) project, and how to use pg_render with [PostgREST](https://postgrest.org).

<details>
<summary>Example Data</summary>

```sql
create table posts (id serial primary key, title text not null, text text not null, author text not null);
insert into posts (title, text, author) values
  ('Title 1', 'Example content 1', 'Author 1'),
  ('Title 2', 'Example content 2', 'Author 2'),
  ('Title 3', 'Example content 3', 'Author 3');

create table templates (id text primary key, template text not null);
insert into templates (id, template) values ('example', '<header>{{ title }}</header><article>{{ text }}</article>');
```

</details>

## `render(template text, input json | array | single value)`

Render a template with the query result.

```sql
-- render a single value
select render('Total posts: {{ value }}', (select count(*) from posts));

-- render a single column from a single row
select render('Title: {{ value }}', (select title as value from posts where id = 1));

-- render multiple columns from a single row
select render('Title: {{ title }}, Text: {{ text }}', (select to_json(props) from (select title, text from posts where id = 1) props));

-- render array of values by looping in template
select render('{% for value in values %} {{ value }} {% endfor %}', (select array(select title from posts)));

-- render multiple rows with multiple columns
select
  render(
    '{% for row in rows %} {{ row.title }} - {{ row.text }} - {{ row.author }} {% endfor %}',
    json_agg(to_json(posts.*))
  )
from posts;

-- render from saved template
select render(
  (select template from templates where id = 'example'),
  (select to_json(props) from (
    select title, text 
    from posts 
    where title = 'Title 3') props
  )
);
```

## `render_agg(template text, input record | json | single value)`

Render multiple rows with aggregate render function. Eg. render a template for all posts queried.


```sql
-- render aggregate with single column
select render_agg('{{ value }}', title) from posts where id < 3;

-- render aggregate using derived table
select render_agg('{{ title }} {{ text }}', props) from (select title, text from posts) as props;

-- render aggregate using json_build_object
select render_agg('{{ title }} {{ text }}', json_build_object('title', title, 'text', text)) from posts;
```

```sql
select render_agg('<article><h1>{{ title }}</h1><p>{{ text }}</p></article>', props)
from (select title, text from posts limit 3) as props;
```
->
```html
<article><h1>Title 1</h1><p>Content for Post 1</p></article>
<article><h1>Title 2</h1><p>Content for Post 2</p></article>
<article><h1>Title 2</h1><p>Content for Post 3</p></article>
```

# Development

Made with

- [Liquid](https://shopify.github.io/liquid/) templating language via [liquid-rust](https://github.com/cobalt-org/liquid-rust)
- [pgrx](https://github.com/pgcentralfoundation/pgrx) Rust extension framework for PostgreSQL
