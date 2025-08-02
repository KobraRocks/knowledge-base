# Config Codex Environment with Deno

```bash
# 1. Create install directory
mkdir -p "$HOME/.deno/bin"

# 2. Download the latest Linux x86_64 Deno ZIP from GitHub
curl -fsSL \
  -o deno.zip \
  https://github.com/denoland/deno/releases/latest/download/deno-x86_64-unknown-linux-gnu.zip

# 3. Unzip the binary into ~/.deno/bin and clean up
unzip -q deno.zip -d "$HOME/.deno/bin"
rm deno.zip

# 4. Persist Deno on your PATH (for Bash)
echo 'export PATH="$HOME/.deno/bin:$PATH"' >> "$HOME/.bashrc"
source "$HOME/.bashrc"

# 5. (Optional) Install Bash completions
"$HOME/.deno/bin/deno" completions bash > "$HOME/.deno/deno-completions.sh"
echo 'source "$HOME/.deno/deno-completions.sh"' >> "$HOME/.bashrc"
source "$HOME/.bashrc"
```
Voila!
