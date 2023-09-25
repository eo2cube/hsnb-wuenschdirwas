# HSNB Wünsch dir was

To put everything on the server:

```
npm run build -- --base=/hsnb/
scp -r -i /path/to/cube dist/* eocube@phenocube.org:/var/www/hsnb
```

That's it!
