+++
title = "Nagios plugin to parse JSON from an HTTP response"
date = 2011-03-04
+++

***Update 2015-10-07*:** This plugin has evolved - please check the latest [README](https://github.com/phrawzty/check_http_json/blob/master/README.md) for up to date details.
 
Hello all !  I wrote a plugin for Nagios that will parse JSON from an HTTP response.  If that sounds interesting to you, feel free to check out my [check\_http\_json repo](https://github.com/phrawzty/check_http_json) on Github.  The plugin has been tested with Ruby 1.8.7 and 1.9.3.  Pull requests welcome !

```bash
Usage: ./check_http_json.rb -u  -e  -w  -c 
-h, --help                       Help info.
-v, --verbose                    Additional human output.
-u, --uri URI                    Target URI. Incompatible with -f.
    --user USERNAME              HTTP basic authentication username.
    --pass PASSWORD              HTTP basic authentication password.
-f, --file PATH                  Target file. Incompatible with -u.
-e, --element ELEMENT            Desired element (ex. foo=>bar=>ish is foo.bar.ish).
-E, --element_regex REGEX        Desired element expressed as regular expression.
-d, --delimiter CHARACTER        Element delimiter (default is period).
-w, --warn VALUE                 Warning threshold (integer).
-c, --crit VALUE                 Critical threshold (integer).
-r, --result STRING              Expected string result. No need for -w or -c.
-R, --result_regex REGEX         Expected string result expressed as regular expression. No need for -w or -c.
-W, --result_warn STRING         Warning if element is [string]. -C is required.
-C, --result_crit STRING         Critical if element is [string]. -W is required.
-t, --timeout SECONDS            Wait before HTTP timeout.
```

The `--warn` and `--crit` arguments conform to the Nagios [threshold format guidelines](http://nagiosplug.sourceforge.net/developer-guidelines.html).
If a simple result of either string or regular expression (`-r` or `-R`) is specified :

* A match is OK and anything else is CRIT.
* The warn / crit thresholds will be ignored.

If the warn and crit results (`-W` and `-C`) are specified :

* A match is WARN or CRIT and anything else is OK.
* The warn / crit thresholds will be ignored.

Note that (`-r` or `-R`) and (`-W` and `-C`) are mutually exclusive.
Note also that the response must be pure JSON. Bad things happen if this isn't the case.
How you choose to implement the plugin is, of course, up to you.  Here's one suggestion:

```bash
# check json from http
define command{
 command_name    check_http_json-string
 command_line    /etc/nagios3/plugins/check_http_json.rb -u 'http://$HOSTNAME$:$ARG1$/$ARG2$' -e '$ARG3$' -r '$ARG4$'
}
define command{
 command_name    check_http_json-int
 command_line    /etc/nagios3/plugins/check_http_json.rb -u 'http://$HOSTNAME$:$ARG1$/$ARG2$' -e '$ARG3$' -w '$ARG4$' -c '$ARG5$'
}

# make use of http json check
define service{
 service_description     elasticsearch-cluster-status
 check_command           check_http_json-string!9200!_cluster/health!status!green
}
define service{
 service_description     elasticsearch-cluster-nodes
 check_command           check_http_json-int!9200!_cluster/health!number_of_nodes!4:!3:
}
```