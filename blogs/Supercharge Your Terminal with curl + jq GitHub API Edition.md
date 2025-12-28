
# Supercharge Your Terminal with curl + jq: GitHub API Edition

Working with APIs can be messy especially when the response comes in **raw JSON**. But with **curl** and **jq**, you can fetch, filter, and transform API data right in your terminal. Today, we'll explore some practical GitHub API examples to see just how powerful this duo can be.


<div align="left">
      <a href="https://www.youtube.com/watch?v=hq_UNyNa0f8">
         <img src="https://img.youtube.com/vi/hq_UNyNa0f8/0.jpg" style="width:100%;">
      </a>
</div>


---

## What Are curl and jq?

- **curl**: A command-line tool to fetch data from URLs (HTTP/HTTPS). Perfect for APIs.
- **jq**: A lightweight JSON processor. It allows you to slice, filter, map, and transform JSON data with ease.

Combined, curl + jq let you work with APIs **without writing a single line of Python or JavaScript**.

---

## 1. Fetching and Pretty-Printing Repository Data

Let’s fetch details of the `stedolan/jq` repository:

```bash
curl -s https://api.github.com/repos/stedolan/jq | jq .
````

**What happens here?**

- `curl -s` fetches the JSON silently (without progress bars).
    
- `jq .` pretty-prints the JSON so it's easy to read.
    

---

## 2. Extract Specific Fields

Suppose you only want the repository **name**, **stars**, and **description**:

```bash
curl -s https://api.github.com/repos/stedolan/jq \
  | jq '{name: .name, stars: .stargazers_count, description: .description}'
```

Output:

```json
{
  "name": "jq",
  "stars": 18800,
  "description": "Command-line JSON processor"
}
```

**Why this is useful:** You avoid sifting through hundreds of fields you don’t need.

---

## 3. List All Repositories of a User

Want to see all repositories for GitHub user `octocat`?

```bash
curl -s https://api.github.com/users/octocat/repos | jq '.[].name'
```

Output (example):

```
Spoon-Knife
Hello-World
octocat.github.io
...
```

**Tip:** `.[].name` iterates through each repository in the JSON array and prints its `name`.

---

## 4. Filter Repositories by Stars

Let’s say you want only popular repos (more than 50 stars):

```bash
curl -s https://api.github.com/users/octocat/repos \
  | jq '.[] | select(.stargazers_count > 50) | {name, stars: .stargazers_count}'
```

Output (example):

```json
{
  "name": "Hello-World",
  "stars": 150
}
```

**Explanation:**

- `select(.stargazers_count > 50)` filters repositories.
    
- The resulting JSON shows only the fields you care about.
    

---

## 5. Export Repository Data as CSV

If you want a CSV of repository **name** and **stars**:

```bash
curl -s https://api.github.com/users/octocat/repos \
  | jq -r '.[] | [.name, .stargazers_count] | @csv'
```

Output (example):

```
"Hello-World",150
"Spoon-Knife",95
"octocat.github.io",60
```

**Why this rocks:** You can pipe it into Excel, Google Sheets, or other scripts instantly.

---

## 6. Count Repositories

Quickly count how many public repositories a user has:

```bash
curl -s https://api.github.com/users/octocat/repos | jq 'length'
```

Output:

```
8
```

---

## Pro Tips

1. **Silent mode**: Always use `-s` with curl for scripting.
    
2. **Raw output**: Use `-r` in jq when you want plain text instead of JSON quotes.
    
3. **Aliases**: Add this to your shell for convenience:
    

```bash
alias cj='curl -s "$@" | jq'
```

Then you can just run:

```bash
cj https://api.github.com/users/octocat/repos
```

---

## Conclusion

With **curl + jq**, the GitHub API (or any JSON API) becomes instantly readable and filterable — all from your terminal. You can:

- Pretty-print JSON
    
- Extract only the fields you need
    
- Filter arrays
    
- Export as CSV
    
- Count or aggregate data
    

No Python, no JavaScript, just **pure terminal magic**.
