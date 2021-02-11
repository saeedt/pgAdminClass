# pgAdminClass
I use [PostgreSQL] (https://www.postgresql.org/about/) for research and teaching because it is the best DBMS ever made! I have set it up on a virtual machine running CentOS Linux and my students connect through [pgAdmin](https://www.pgadmin.org/).

## pgAdmin Issues
Most of the times, I set up a personal database for each student and grant them all privileges on their personal database. pgAdmin still shows all databases on the server altough they cannot connect to them. Additionally, 'Login/Group Roles' and 'Tablespaces' are visible to everyone.
In some cases, I set up a database for the entire class to work with. Students are given 'CONNECT' and 'SELECT' privileges on the shared databse to work with the database. In this case, I set up a shared account on pgAdmin for everyone to connect and I would not like students to be able to change the password or be able to remove the DBMS server or create new connections. 
I understand that pgAdmin is not made for such environments and my requirements are quite specific, so I tried to address the issues using some 'JavaScript' code.

## Setting up a shared Database
To set up a shared database 'sharedDB' and a shared account 'sharedUser' I do the following. 
First create the databse.
'''
CREATE DATABASE sharedDB;
'''
Then swith to sharedDB revoke all privileges on it from public.
'''
\c sharedDB
REVOKE ALL PRIVILEGES ON SCHEMA public FROM public;
REVOKE ALL PRIVILEGES ON DATABASE sharedDB FROM public;
...
Then create the sharedUser and allow it to use the sharedDB and public schema.
'''
CREATE USER sharedUser WITH PASSWORD '*****'
GRANT CONNECT ON DATABASE sharedDB TO sharedUser;
GRANT USAGE ON SCHEMA public TO sharedUser;
'''
Finally, revoke all privileges from public and sharedUser and just allow them to select. I also grant all privileges to 'myself' in case I would like to add or modify tables or data.
'''
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM public, sharedUser;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO sharedUser;
GRANT ALL PRIVILEGES ON DATABASE sharedDB TO myself;
'''
Every time I add new tables, I run the following to ensure sharedUser can use it.
'''
GRANT SELECT ON ALL TABLES IN SCHEMA public TO sharedUser; 
'''

## pgAdmin Workarounds
I would not like the modifications I make here to affect all users including myself. Therefore, I add a '.' to usernames who should not be affected by any of this, and therefore users like 'sharedUser' without a dot in their username will be affected by these modifications.
### Disabling the main menu   
To prevent 'sharedUser' from changing the password, or using any of the tools, I disabled all items on the [navigation bar menu] (https://www.pgadmin.org/styleguide/themes/menus/) for the sharedUSer. 
'''
$(document).ready(function(){
	const user = $("#navbar-user").text().split('@')[0].trim();	
	if (!(user.includes('.'))){
		console.log(user);
		$( ".dropdown-toggle" ).addClass("disabled");
	}
});
'''
After making sure this works in the console, I ended up injecting it into the browser template in the 'window.onload' function.
'''
/usr/lib/python2.7/site-packages/pgadmin4-web/pgadmin/browser/templates/browser/index.html
'''
To see the changes, the web server needs to be restarted using 'systemctl restart httpd' on my CentOS server.
### Disabling the context menu and hiding redundant features
I needed to disable the context menu (right click) and hide redundant databases (other than 'sharedDB') as well as 'Login/Group Roles' and 'Tablespaces' I added the following to 'acitree' event handler function. 
'''
if (n == 'loaded'){
const user = $("#navbar-user").text().split('@')[0].trim();
	if (!user.includes('.')){
		$( ".aciTreeLine" ).on('contextmenu', function(e) {return false;});
		$( ".aciTreeLine" ).each(function(e){
			let ctr = 1
			let mLevel = $(this).attr('aria-level')
			if (mLevel==3){ 
				if (ctr ==1)
					$(this).hide();
				ctr++
				} else if (mLevel==4){ 
					if(!$(this).find('.aciTreeText').text().includes(user)){
						$(this).hide();
					}		
				}
			});
		}
}	
'''
I added the code in 'pgadmin_commons.js' script right after '("#tree").on("acitree",function(e,t,i,n,s)'. 
'''
/usr/lib/python2.7/site-packages/pgadmin4-web/pgadmin/static/js/generated/pgadmin_commons.js
'''