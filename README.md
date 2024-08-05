# arch_pg_render
Render engine extension for bare metal PostgreSQL 15 on Arch Linux.

# 0.) Install YaY
```bash
pacman -Syu git
sudo useradd -m -G wheel user
sudo EDITOR=vim visudo
(Find the and uncomment wheel group without password line 108
passwd user
confirm password
su - user
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

# 1.) Initalize the postgres database
```bash
yay -Syu postgrest-bin postgresql
su postgres
initdb --locale=C.UTF-8 --encoding=UTF8 -D /var/lib/postgres/data --data-checksums
exit
systemctl start postgresql
systemctl enable postgresql
vim /var/lib/postgres/.psql_history
wq
chown postgres /var/lib/postgres/.psql_history
```

# 2.) Copy the MakeFile, arch_pg_render--1.0, arch_pg_render.control
```
yay -Syu git
cd ~/home/user
git clone https://github.com/realderekstevens/arch_pg_render
sudo cp arch_pg_render--1.0.sql /usr/share/postgresql/extension
sudo cp arch_pg_render.control /usr/share/postgresql/extension
```

# 3.) Loadup your postgres
```
su - postgres
psql
```

# Examples
```
-- PostgreSQL extension
create extension arch_pg_render;
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

## Which should give you:
```
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

# Sql.1
'''
-- PostgREST setup
create extension arch_pg_render;
create domain "text/html" as text;
create role anon nologin;
create role writer nologin;

grant usage on schema public to anon;
grant usage on schema public to writer;

create table posts (
    id serial not null primary key,
    title text not null,
    content text not null,
    image_url text,
    likes integer default 0 not null
);

create table templates (
    id text not null primary key,
    template text not null
);

create table analytics (
    post_id integer references posts (id) not null,
    created_at timestamptz not null default now()
);

-- postgrest permissions
grant select on posts to anon;
grant select on templates to anon;
grant select on analytics to anon;
grant insert on analytics to anon;

grant select on posts to writer;
grant update on posts to writer;

--- PAGES ---

-- all posts
create or replace function index(page integer default 1) returns "text/html" as $$
    select render(
        (select template from templates where id = 'layout'),
        (json_build_object(
            'title',   'All Posts',
            'styles',    (select template from templates where id = 'styles.css'),
            'header',    (select template from templates where id = 'header'),
            'footer',    (select template from templates where id = 'footer'),
            'load_more', (select render(
                (select template from templates where id = 'posts/load-more'),
                (json_build_object('next_page', (page + 1), 'children', '<nav></nav>')
            ))),
            'children',  (select render_agg(
                (select template from templates where id = 'posts/list-post'), posts) from
                (select id, title, content, image_url, likes from posts order by id limit 3 offset ($1 - 1) * 3) as posts
            ))
        )
    );
$$ language sql;

-- single post
create or replace function post(id integer)
returns "text/html" as $$
    select render(
        (select template from templates where id = 'layout'),
        (json_build_object(
            'title',     (select title from posts where id = $1),
            'styles',    (select template from templates where id = 'styles.css'),
            'header',    (select template from templates where id = 'header'),
            'footer',    (select template from templates where id = 'footer'),
            -- re-use for analytics
            'load_more', (select render ('<input type="hidden" hx-trigger="load" hx-post="/analytics" name="post_id" value="{{ value }}" hx-headers=''{"Accept": "application/json"}'' />', $1)
            ),
            'children',  (select render(
                (select template from templates where id = 'posts/post'),
                (select to_json(posts) from (
                    select id, title, content, image_url, likes, a.views from posts
                    left join (select post_id, count(*) as views from analytics group by post_id) a on posts.id = a.post_id
                    where id = $1
                ) posts)
            ))
        ))
    );
$$ language sql;

--- RPC ---

create or replace function like(id integer)
returns "text/html" as $$
declare
    updated_likes integer;
begin
    set local role writer;
    update posts set likes = likes + 1 where posts.id = $1 returning likes into updated_likes;
    return updated_likes;
end;
$$ language plpgsql;

create or replace function load_more(page integer default 1)
returns "text/html" as $$
    select render(
        (select template from templates where id = 'posts/load-more'),
        (json_build_object(
            'next_page', (select (page + 1)),
            'children', (select render_agg(
                (select template from templates where id = 'posts/list-post'), posts)
                from (select id, title, content, image_url, likes from posts order by id limit 3 offset ($1 - 1) * 3) as posts
            ))
        )
    );
$$ language sql;
'''

# SQL 2
'''
insert into templates (id, template) values
    ('layout', '
    <!DOCTYPE html>
    <html>
        <head>
            <title>{{ title }}</title>
            <style>{{ styles }}</style>
            <script src="https://unpkg.com/htmx.org@2.0.0-alpha1/dist/htmx.min.js"></script>
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
        </head>
        <body hx-headers=''{"Accept": "text/html"}''>
            {{ header }}
            <main>
                {{ children }}
                {{ load_more }}
            </main>
            {{ footer }}
        </body>
    </html>'
);

insert into templates (id, template) values
    ('header', '
    <header>
        <h1><a href="/">pgrender.org</a></h1>
        <h4>Example site running arch_pg_render, PostgREST and HTMX</h4>
        <code><a href="https://github.com/mkaski/arch_pg_render_example">View source</a></code>
    </header>'
);

insert into templates (id, template) values
    ('footer', '
    <footer>
        <small><a href="https://github.com/realdereksteverns/arch_pg_render">arch_pg_render</a></small>
    </footer>'
);

insert into templates (id, template) values
    ('posts/list-post', '
    <article>
        <form hx-post="/rpc/like" hx-target="find span">
            <button name="id" value="{{ id }}" type="submit">
                Like
            </button>
            <label><span>{{ likes }}</span> likes</label>
        </form>
        <a href="/post/{{ id }}">
            <h2>{{ title }}</h2>
        </a>
        <section>{{ content }}</section>
        <img src="{{ image_url }}" alt="{{ title }}">
    </article>'
);

insert into templates (id, template) values
    ('posts/post', '
    <article>
        <form>
            <span>{{ likes }} likes</span>
            <span>{{ views }} views</span>
        </form>
        <h1>{{ title }}</h1>
        <section>{{ content }}</section>
        <img src="{{ image_url }}" alt="{{ title }}">
    </article>'
);

insert into templates (id, template) values
    ('posts/load-more', '
    {% if children != blank %}
        {{ children }}
        <nav hx-get="/rpc/load_more?page={{ next_page }}" hx-trigger="revealed" hx-swap="afterend" />
    {% endif %}
    <div class="htmx-indicator">Loading...</div>
');

insert into templates (id, template) values
    ('styles.css', '
    body {
        margin: 0;
        padding: 0;
        font-family: system-ui, sans-serif;
    }
    header {
        text-align: center;
    }
    h4 {
        margin-top: -1rem;
        margin-bottom: 2rem;
        font-weight: 300;
    }
    main {
        width: 600px;
        max-width: 100%;
        margin: 1rem auto;
        @media (max-width: 600px) {
            width: 90%;
        }
    }
    article {
        border: 1px solid #888;
        margin-bottom: 1.25rem;
        padding: 1rem;
        opacity: 1;
        transition: opacity 500ms ease;
    }
    article.htmx-added {
        opacity: 0;
    }
    form {
        display: flex;
        align-items: center;
        gap: 1rem;
    }
    button {
        display: flex;
        padding: 0.25rem;
    }
    img {
        width: 100%;
        max-width: 100%;
        max-height: calc((600px * 9) / 16);
        object-fit: cover;
        display: block;
    }
    section {
        margin: 1rem 0;
    }
    footer {
        text-align: center;
        padding: 2rem;
    }
    nav {
        text-align: center;
        position: absolute;
        right: 1rem;
    }
');
'''
