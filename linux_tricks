 # To save time

## find and replace


```bash
find . -type f -exec perl -0777 -pi -e 's|<nav>.*?</nav>|<nav>NEW CONTENT</nav>|sg' {} +
```
* -0777 → slurp file whole (multi-line mode).
* s|||sg → substitute across newlines, global.
* .*? → non-greedy match (stops at first </nav>).
