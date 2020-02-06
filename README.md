# sqlite-to-postgresql
migrate sqlite to postgresql
h1. Redmine sqlite to postgresql

## Install necessary packages 

<pre><code class="text">
apt-get install sqlite3 ruby ruby-dev make libsqlite3-dev libpq-dev postgresql pgloader  build-essential     
</code></pre>

Go to the folder where your SQLite database resides and make a copy of it. If the Redmine server’s still running.

<pre><code class="text">
sqlite3 redmine.sqlite3
</code></pre>

<pre><code class="text">
.backup /tmp/redmine_backup.sqlite3
.exit
</code></pre>


Which will make sure there’s no corruption (i.e. no writes running in the background while you’re making the copy). If the server’s not running you don’t have to deal with this and just copy the file manually:

<pre><code class="text">
cp redmine.sqlite3 /tmp/redmine_backup.sqlite3
cp /var/lib/dbconfig-common/sqlite3/redmine/instances/default/redmine_default /tmp/redmine_backupdb.sqlite3

</code></pre>


Now install various gems. The 1.5.2 version of the rack gem is breaking database conversion so stick with 1.4

<pre><code class="text">
gem install rack -v 1.4.5
gem install sqlite3 pg taps
</code></pre>

## Edit Gemfile

setup env variables http,https,ftp
go to App file where Gemfile locate 

<pre><code class="text">
cd /usr/share/redmine/
</code></pre>

<pre><code class="text">
gem update
</code></pre>
 
<pre><code class="text">
#gem for migration database
group :production do
 gem "pg", "~> 0.19"
 gem  "rake"
gem "taps"
end
</code></pre>

## Update Gemfile

change network setting to NAT

<pre><code class="text">
cd /usr/share/redmine 
bundle install

</code></pre>
check all gem installed fine


!bundle%20ins.PNG!

## Create a role and database for migrate

Now switch to the postgres user and enter the PostgreSQL command prompt:

<pre><code class="text">
cd 
su postgres
psql -h localhost
</code></pre>



<pre><code class="sql">
CREATE ROLE redmine LOGIN ENCRYPTED PASSWORD 'Qwe12345!' NOINHERIT VALID UNTIL 'infinity';
CREATE DATABASE redmine WITH ENCODING='UTF8' OWNER=redmine;
\q
exit
</code></pre>

## Authentication 

<pre><code class="text">
nano /etc/postgresql/10/main/pg_hba.conf
</code></pre>

<pre><code class="text">
# "local" is for Unix domain socket connections only
local   all             all                                     trust
local   all             all                                     peer
</code></pre>

## Edit database.yml

<pre><code class="text">
nano /etc/redmine/default/database.yml
</code></pre>


<pre><code class="yaml">
development:
  adapter: postgresql
  encoding: unicode
  database: redmine
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: redmine
  password: Qwe12345!

test:
  adapter: postgresql
  database: redmine_test
  username: redmine
  password: Qwe12345!
  host: localhost

production:
  adapter: postgresql
  encoding: unicode
  database: redmine
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: redmine
  password: Qwe12345!
  host: localhost
</code></pre>

## Migrate 

*db:migrate* runs (single) migrations that have not run yet.
*db:setup* does db:create, db:schema:load, db:seed

<pre><code class="text">
cd /usr/share/redmine
rake db:migrate
rake db:setup
</code></pre>

###  Or using sequel

Before using sequel please drop psql database

<pre><code class="sql">
SELECT pg_terminate_backend (pg_stat_activity.pid) FROM pg_stat_activity WHERE pg_stat_activity.datname = 'projectportal';
drop database projectportal;
</code></pre>


<pre><code class="text">
gem install sequel
rake db:create
</code></pre>


<pre><code class="text">
sequel -C sqlite:/tmp/redmine_backupdb.sqlite3 postgres://postgres:"12345678"@localhost/projectportal

</code></pre>

<pre><code class="text">
pgloader --with "data only"  /tmp/redmine_backupdb.sqlite3  postgresql://postgres:"12345678"@localhost/projectportal
</code></pre>

## Alter table tokens 

for easy step  go to phppgadmin
token -> alter column *created_on* and *updated_on* -> *timestamp without time zone
*

## Load postgres db with data from sqlite  

pgLoader is a program that can load data into a PostgreSQL database from a variety of different sources
<pre><code class="text">
cd 
 pgloader --with "data only"  /tmp/redmine_backupdb.sqlite3  postgresql://redmine:"Qwe12345!"@localhost/redmine

</code></pre>


!pgloadout.PNG!


please check  Reset Sequences  

<pre><code class="text">
service apache2 restart
</code></pre>

### For confirmation 

<pre><code class="text">
su postgres
psql
could not change directory to "/root": Permission denied
psql (10.10 (Ubuntu 10.10-0ubuntu0.18.04.1))
Type "help" for help.

</code></pre>

<pre><code class="sql">
postgres=# \c redmine
You are now connected to database "redmine" as user "postgres".
redmine=# select * from tokens
redmine-# ;
 id | user_id | action  |                  value                   |         created_on         |         updated_on
----+---------+---------+------------------------------------------+----------------------------+----------------------------
  3 |       1 | session | b73ac30b89ac9a895345b562592f71230dec33f0 | 2019-10-22 12:52:51.581302 | 2019-10-22 12:54:31.181062
  4 |       1 | feeds   | db3ae976e2567b590ba8e880d474a12bce8f9da1 | 2019-10-22 12:52:51.63742  | 2019-10-22 12:52:51.63742
(2 rows)

redmine=# \q
</code></pre>
if you see any 00 end of the column created_on and updated_on +that means not alter+  
<pre><code class="text">
exit
</code></pre>

### Re- login

go to redmine information page you can find  
<pre><code class="xml">
Database adapter               PostgreSQL
</code></pre>

[[http://192.168.56.124/redmine/admin/info]]


!browser.PNG!




# ---------------------------The End------------------------------------------

## Sqlite queries 

<pre><code class="text">
sqlite3 redmine.sqlite3
</code></pre>

<pre><code class="sql">
.tables # view tables 
select * from table_name; # view table data
.exit

</code></pre>

## Psql queries

<pre><code class="text">
su postgres
psql
</code></pre>

<pre><code class="sql">
\c database_name 
\dt  # view tables 
select * from table-name # view table data;
\l #list all database
ALTER TABLE table_name ADD column_name datatype;

</code></pre>
exit
