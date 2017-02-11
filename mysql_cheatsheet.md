Create database
	create database wordpress;

Create user
	create user wordpress@localhost identified by '<pass>';

Grant privileges. As root user execute:
	grant all privileges on *.* to 'root'@'%' identified by '<pass>';
	
Grant privileges.
	grant all privileges on <database>.* to <user>@<host> identified by '<pass>';

Example
	grant all privileges on wordpress.* to wordpress@localhost identified by '<pass>';

List users
	select User,Password,Host from user;

flush privileges;
use <database>
show tables;
show databases;

show processlist;
show full processlist;
show create table <table>;
explain <table>;

