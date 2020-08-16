### Speaking with a PHP-FPM socket via command line

- https://www.dynatrace.com/news/blog/7-minute-workout-does-your-apache-web-server-need-love/
- https://serverfault.com/questions/516373/what-is-the-meaning-of-ah00485-scoreboard-is-full-not-at-maxrequestworkers
- https://easyengine.io/tutorials/php/fpm-status-page/

```bash
#SCRIPT_NAME=/status SCRIPT_FILENAME=/status REQUEST_METHOD=GET cgi-fcgi -bind -connect /var/run/php/php7.2-fpm.sock
export SCRIPT_NAME=/status 
export SCRIPT_FILENAME=/status 
export REQUEST_METHOD=GET 
cgi-fcgi -bind -connect /var/run/php/php7.2-fpm.sock
```
