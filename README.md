
### Step 1: Install Pixie CLI

Download and install the official Pixie CLI binary and ensure it is available in your `$PATH`:

---

> **Note:** The installer places the `px` binary in `~/.pixie/bin` and updates your shell profile automatically. If `px version` returns the client version, you are ready for the next step.

```sh {"terminalRows":"19"}
echo "=================================================="
echo "🚀 Installing Pixie CLI with sudo..."
echo "=================================================="

printf "y\n/usr/local/bin\n" | sudo bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"


echo ""
echo "=================================================="
echo "✅ Verifying Installation..."
echo "=================================================="
px version
```

....