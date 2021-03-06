## Master Page (Reverse, Web, Crypto, 315p)
	tl;dr sqli via domain txt record

In this task, we're provided with a url to a simple login page, after looking around for a bit, we've noticed that `https://masterpage.asis-ctf.ir/.git/` returns error 403, which means there's a repository inside and we can access it

After [dumping](https://github.com/internetwache/GitTools/blob/master/Dumper/gitdumper.sh) the repo, we can easily check that there were 3 files in the repo `config.php`, `index.php` and `query.php`

`index.php` contains the interesting authentication part we're looking for:

``` php
$id = $mysql->filter($_POST['user'], "auth");
$pw = $mysql->filter($_POST['pass'], "pass");
$ip = $mysql->filter($_SERVER['REMOTE_ADDR'], "hash");

/* resolve ip addr. */
$dns = dns_get_record(gethostbyaddr($ip));
$ip = ($dns) ? print_r($dns, true) : ($ip);


/* mysql filtration */
$filter = "_|information|schema|con|\_|ha|b|x|f|@|\"|`|admin|cas|txt|sleep|benchmark|procedure|\^";
foreach($_POST as $_VAR){
    if(preg_match("/{$filter}/i", $_VAR) || preg_match("/{$filter}/i", $ip))
    {
        exit("Blocked!");
    }
}
if(strlen($id.$pw.$ip) >= 1024 || substr_count("(", $id.$pw.$ip) > 2)
{
    exit("Too Long!");
}

/* admin auth */
$query = "SELECT id, pw FROM admin WHERE id='{$id}' AND pw='{$pw}' AND ip='{$ip}';";
$result = $mysql->query($query, 1);

if($result['id'] === "admin" && $result['pw'] === $_POST['pw'])
{
    echo $flag."<HR>";
}
```

It's pretty obvious that the goal is to perform a SQL injection and get the flag, but how do we deal with the filters?

It turns out, that we can easily bypass the `mysql filtration` section by performing a GET request instead of POST, which means, we have to inject the payload into $ip.

We can do that by setting a TXT record of our domain to the needed string

After some debugging, we came up with a working payload: `' union select 'admin',NULL or '1'='`

The query becomes:

`"SELECT id, pw FROM admin WHERE id='NULL' AND pw='{NULL}' AND ip='GARBAGE`   `' union select 'admin',NULL or '1'='`    `GARBAGE';";`

The first part doesn't return any rows, the second select returns one row: `'admin', NULL` which allows it to pass the final check:

`$result['id'] === "admin" && $result['pw'] === $_POST['pw']`

Summing it up, create a TXT record with payload on a controlled domain, make a GET request from said domain and get the flag!

