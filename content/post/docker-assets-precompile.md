---
draft: true
---

`rake assets:precompile` fail with error

```
rake aborted!
BrowserifyRails::BrowserifyError: Error while running `/app/node_modules/.bin/browserify -t coffeeify --extension=".coffee" --list -o "/app/tmp/cache/browserify-rails/output20160322-6-kr0qpc" -`:
```

install node this way

```
curl -sL https://deb.nodesource.com/setup_5.x | sudo -E bash -
sudo apt-get install --yes nodejs
```

npm is installed along with node, you don;t need to `apt-get` it.  

If you get node: command not found chances are likely you need to link your binary:

`sudo ln -s /usr/bin/nodejs /usr/bin/node`
