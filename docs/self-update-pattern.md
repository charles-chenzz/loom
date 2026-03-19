# Self-Update Pattern Implementation Guide

**Status:** Reference\
**Version:** 1.0\
**Last Updated:** 2025-03-19

---

## 1. Overview

This document extracts the self-update pattern from Loom's distribution system into a reusable blueprint for other applications.

### Key Components

1. **Build Info Embedding** - Embed version/commit info at compile time
2. **Binary Distribution** - Server hosting platform-specific binaries
3. **SHA256 Verification** - Integrity check before applying updates
4. **Atomic Replacement** - Safe binary replacement with backup

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Client (CLI)                                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Get current binary SHA256                                               │
│  2. Fetch remote SHA256 from server                                         │
│  3. Compare: if same, exit (already up to date)                             │
│  4. If different:                                                           │
│     a. Download new binary                                                  │
│     b. Verify SHA256 matches                                                │
│     c. Write to temp file                                                   │
│     d. Set executable permissions (Unix)                                    │
│     e. Atomically replace current binary                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Server                                          │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  GET /bin/{platform}         → Binary file                                  │
│  GET /bin/{platform}.sha256  → SHA256 hash                                  │
│  GET /bin/                   → Directory listing (optional)                  │
│                                                                              │
│  File Layout:                                                                │
│  bin/                                                                        │
│  ├── linux-x86_64                                                            │
│  ├── linux-x86_64.sha256                                                     │
│  ├── macos-aarch64                                                           │
│  ├── macos-aarch64.sha256                                                    │
│  └── ...                                                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Components

### 3.1 Build Info Embedding (Rust)

Use `shadow-rs` crate to embed build information at compile time:

**Cargo.toml:**
```toml
[build-dependencies]
shadow-rs = "0.36"

[dependencies]
shadow-rs = "0.36"
```

**build.rs:**
```rust
fn main() -> shadow_rs::SdResult<()> {
    shadow_rs::new()
        .build_target(env!("LOOM_PLATFORM"))  // Custom platform string
        .build()
}
```

**src/lib.rs:**
```rust
shadow_rs::shadow!(build);

/// Platform string in `{os}-{arch}` format
pub const PLATFORM: &str = env!("LOOM_PLATFORM");

pub struct BuildInfo {
    pub version: &'static str,
    pub git_sha: &'static str,
    pub build_timestamp: &'static str,
    pub platform: &'static str,
}

impl BuildInfo {
    pub const fn current() -> Self {
        Self {
            version: build::PKG_VERSION,
            git_sha: build::SHORT_COMMIT,
            build_timestamp: build::BUILD_TIME,
            platform: PLATFORM,
        }
    }
}
```

**Custom Platform String (optional):**

Add to `.cargo/config.toml`:
```toml
[env]
LOOM_PLATFORM = { value = "linux-x86_64", force = true }
```

Or set via CI/CD:
```bash
export LOOM_PLATFORM="linux-x86_64"
cargo build --release
```

### 3.2 Update Client Implementation

```rust
use anyhow::{Context, Result};
use sha2::{Digest, Sha256};
use url::Url;

/// Get update base URL from environment
pub fn get_update_base_url() -> Result<Url> {
    // Option 1: Explicit update URL
    if let Ok(raw) = std::env::var("MYAPP_UPDATE_BASE_URL") {
        return Url::parse(&raw).context("invalid update URL");
    }
    
    // Option 2: Derive from API URL
    if let Ok(api_url) = std::env::var("MYAPP_API_URL") {
        let mut base = Url::parse(&api_url).context("invalid API URL")?;
        base.set_path("");
        return Ok(base);
    }
    
    anyhow::bail!("MYAPP_UPDATE_BASE_URL or MYAPP_API_URL must be set")
}

/// Compute SHA256 hash
pub fn compute_sha256(data: &[u8]) -> String {
    hex::encode(Sha256::digest(data))
}

/// Build URL for SHA256 file
pub fn build_sha_url(base_url: &Url, platform: &str) -> Result<Url> {
    base_url
        .join(&format!("bin/{platform}.sha256"))
        .context("failed to construct SHA URL")
}

/// Build URL for binary file
pub fn build_bin_url(base_url: &Url, platform: &str) -> Result<Url> {
    base_url
        .join(&format!("bin/{platform}"))
        .context("failed to construct binary URL")
}

/// Normalize SHA256 string (trim whitespace, lowercase)
pub fn normalize_remote_sha(raw: &str) -> String {
    raw.trim().to_lowercase()
}

/// Check if update is needed
pub fn needs_update(current_sha: &str, remote_sha: &str) -> bool {
    current_sha != remote_sha
}

/// Verify downloaded binary matches expected SHA256
pub fn verify_download(downloaded_bytes: &[u8], expected_sha: &str) -> Result<()> {
    if downloaded_bytes.is_empty() {
        anyhow::bail!("Downloaded binary is empty");
    }

    let downloaded_sha = compute_sha256(downloaded_bytes);
    if downloaded_sha != expected_sha {
        anyhow::bail!(
            "SHA256 mismatch: expected {}..., got {}...",
            &expected_sha[..12],
            &downloaded_sha[..12]
        );
    }

    Ok(())
}

/// Main update flow
pub async fn run_update(http_client: &reqwest::Client, platform: &str) -> Result<()> {
    let base_url = get_update_base_url()?;
    let sha_url = build_sha_url(&base_url, platform)?;
    let bin_url = build_bin_url(&base_url, platform)?;

    // 1. Get current executable SHA256
    let current_exe = std::env::current_exe().context("failed to get current executable")?;
    let current_bytes = std::fs::read(&current_exe).context("failed to read current executable")?;
    let current_sha = compute_sha256(&current_bytes);

    // 2. Fetch remote SHA256
    let sha_response = http_client
        .get(sha_url.clone())
        .send()
        .await
        .context("failed to fetch remote SHA")?;

    if !sha_response.status().is_success() {
        anyhow::bail!("Failed to check for updates: {}", sha_response.status());
    }

    let remote_sha = normalize_remote_sha(&sha_response.text().await?);

    // 3. Check if update needed
    if !needs_update(&current_sha, &remote_sha) {
        println!("Already up to date");
        return Ok(());
    }

    println!("Update available: {}... → {}...", &current_sha[..12], &remote_sha[..12]);

    // 4. Download new binary
    let response = http_client
        .get(bin_url.clone())
        .send()
        .await
        .context("failed to download update")?;

    if !response.status().is_success() {
        anyhow::bail!("Download failed: {}", response.status());
    }

    let bytes = response.bytes().await.context("failed to read update binary")?;

    // 5. Verify SHA256
    verify_download(&bytes, &remote_sha)?;
    println!("Download verified ({} bytes)", bytes.len());

    // 6. Write to temp file
    let tmp_path = current_exe.with_extension("new");
    tokio::fs::write(&tmp_path, &bytes)
        .await
        .context("failed to write temporary binary")?;

    // 7. Set executable permissions (Unix only)
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        let mut perms = std::fs::metadata(&tmp_path)?.permissions();
        perms.set_mode(0o755);
        std::fs::set_permissions(&tmp_path, perms)?;
    }

    // 8. Atomically replace binary
    self_replace::self_replace(&tmp_path).context("failed to replace binary")?;

    // 9. Cleanup
    let _ = std::fs::remove_file(&tmp_path);

    println!("Update complete!");
    Ok(())
}
```

### 3.3 Dependencies

**Cargo.toml:**
```toml
[dependencies]
anyhow = "1"
sha2 = "0.10"
hex = "0.4"
tokio = { version = "1", features = ["fs"] }
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls"] }
url = "2"
self_replace = "1"
tracing = "0.1"
shadow-rs = "0.36"

[build-dependencies]
shadow-rs = "0.36"
```

### 3.4 Server Implementation (Axum)

```rust
use axum::{
    extract::Request,
    http::StatusCode,
    response::IntoResponse,
};
use tower_http::services::ServeDir;

// Route setup
pub fn bin_routes() -> Router {
    let bin_dir = std::env::var("MYAPP_SERVER_BIN_DIR").unwrap_or_else(|_| "./bin".to_string());
    
    Router::new()
        // Serve static files from bin/
        .nest_service("/bin", ServeDir::new(&bin_dir))
        // Fallback for directory listing
        .fallback(list_bin_directory)
}

/// Handler to list files in the /bin directory
pub async fn list_bin_directory(request: Request) -> impl IntoResponse {
    let request_path = request.uri().path();

    // Only show directory listing for /bin or /bin/
    if request_path != "/" && !request_path.is_empty() {
        return (
            StatusCode::NOT_FOUND,
            "404 Not Found".to_string(),
        );
    }

    let bin_dir = std::env::var("MYAPP_SERVER_BIN_DIR").unwrap_or_else(|_| "./bin".to_string());
    let path = std::path::Path::new(&bin_dir);

    let mut entries = Vec::new();

    if path.exists() && path.is_dir() {
        if let Ok(read_dir) = std::fs::read_dir(path) {
            for entry in read_dir.flatten() {
                let name = entry.file_name().to_string_lossy().to_string();
                let metadata = entry.metadata().ok();
                let is_dir = metadata.as_ref().is_some_and(|m| m.is_dir());
                let size = metadata.as_ref().map(|m| m.len()).unwrap_or(0);
                entries.push((name, is_dir, size));
            }
        }
    }

    entries.sort_by(|a, b| a.0.cmp(&b.0));

    let mut html = String::from(
        r#"<!DOCTYPE html>
<html><head><title>Index of /bin/</title></head>
<body><h1>Index of /bin/</h1><table>
<tr><th>Name</th><th>Size</th></tr>
"#,
    );

    for (name, is_dir, size) in entries {
        let display_name = if is_dir { format!("{name}/") } else { name.clone() };
        let size_str = if is_dir { "-".to_string() } else { format_size(size) };
        html.push_str(&format!(
            r#"<tr><td><a href="/bin/{name}">{display_name}</a></td><td>{size_str}</td></tr>"#
        ));
    }

    html.push_str("</table></body></html>");

    (StatusCode::OK, [("Content-Type", "text/html")], html)
}

fn format_size(size: u64) -> String {
    const KB: u64 = 1024;
    const MB: u64 = KB * 1024;

    if size >= MB {
        format!("{:.1}M", size as f64 / MB as f64)
    } else if size >= KB {
        format!("{:.1}K", size as f64 / KB as f64)
    } else {
        format!("{size}")
    }
}
```

---

## 4. Build & Deployment

### 4.1 Build Script (Local)

```bash
#!/bin/bash
# scripts/build-binaries.sh

set -e

BIN_DIR="${BIN_DIR:-./bin}"
mkdir -p "$BIN_DIR"

build_target() {
    local target=$1
    local platform=$2
    
    echo "Building for $platform ($target)..."
    
    LOOM_PLATFORM="$platform" \
        cargo build --release --target "$target"
    
    cp "target/$target/release/myapp" "$BIN_DIR/$platform"
    
    # Generate SHA256
    sha256sum "$BIN_DIR/$platform" | cut -d' ' -f1 > "$BIN_DIR/$platform.sha256"
}

# Build all platforms (requires cross-compilation toolchains)
build_target "x86_64-unknown-linux-gnu" "linux-x86_64"
build_target "aarch64-unknown-linux-gnu" "linux-aarch64"
build_target "x86_64-apple-darwin" "macos-x86_64"
build_target "aarch64-apple-darwin" "macos-aarch64"
build_target "x86_64-pc-windows-msvc" "windows-x86_64"

echo "Binaries built in $BIN_DIR/"
ls -la "$BIN_DIR"
```

### 4.2 CI/CD (GitHub Actions)

```yaml
# .github/workflows/build-release.yml

name: Build Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    strategy:
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            platform: linux-x86_64
            os: ubuntu-latest
          - target: aarch64-unknown-linux-gnu
            platform: linux-aarch64
            os: ubuntu-24.04-arm
          - target: x86_64-apple-darwin
            platform: macos-x86_64
            os: macos-13
          - target: aarch64-apple-darwin
            platform: macos-aarch64
            os: macos-14
          - target: x86_64-pc-windows-msvc
            platform: windows-x86_64
            os: windows-latest

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Build
        env:
          LOOM_PLATFORM: ${{ matrix.platform }}
        run: cargo build --release --target ${{ matrix.target }}

      - name: Prepare artifact
        shell: bash
        run: |
          mkdir -p bin
          if [ "${{ matrix.os }}" == "windows-latest" ]; then
            cp target/${{ matrix.target }}/release/myapp.exe bin/${{ matrix.platform }}.exe
            certutil -hashfile bin/${{ matrix.platform }}.exe SHA256 | head -2 | tail -1 > bin/${{ matrix.platform }}.sha256
          else
            cp target/${{ matrix.target }}/release/myapp bin/${{ matrix.platform }}
            sha256sum bin/${{ matrix.platform }} | cut -d' ' -f1 > bin/${{ matrix.platform }}.sha256
          fi

      - uses: actions/upload-artifact@v4
        with:
          name: binary-${{ matrix.platform }}
          path: bin/

  release:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - uses: actions/download-artifact@v4
        with:
          path: bin/
          pattern: binary-*
          merge-multiple: true

      - name: Create bundle
        run: |
          mkdir -p dist
          mv bin/* dist/

      - uses: softprops/action-gh-release@v2
        with:
          files: dist/*
```

### 4.3 Server Deployment

```bash
# On server: /var/lib/myapp/
myapp-server
bin/
├── linux-x86_64
├── linux-x86_64.sha256
├── linux-aarch64
├── linux-aarch64.sha256
├── macos-x86_64
├── macos-x86_64.sha256
├── macos-aarch64
├── macos-aarch64.sha256
├── windows-x86_64.exe
└── windows-x86_64.sha256
```

Environment variable:
```bash
export MYAPP_SERVER_BIN_DIR=/var/lib/myapp/bin
```

---

## 5. Security Considerations

### 5.1 Current Implementation

- **SHA256 Verification**: Verifies downloaded binary integrity
- **Atomic Replacement**: Uses `self_replace` crate for safe updates
- **Backup**: Old binary backed up as `.old` automatically

### 5.2 Recommended Enhancements

| Feature | Description | Priority |
|---------|-------------|----------|
| **Signed Binaries** | Sign with private key, verify signature before update | High |
| **HTTPS Only** | Enforce TLS for all downloads | High |
| **Version Manifest** | `GET /bin/manifest.json` with version info | Medium |
| **Update Channels** | `stable`, `beta`, `nightly` channels | Medium |
| **Delta Updates** | Download only changed bytes | Low |

### 5.3 Signing Implementation (Future)

```rust
// Generate signing key
// openssl genpkey -algorithm Ed25519 -out private.pem
// openssl pkey -in private.pem -pubout -out public.pem

// Sign binary
// openssl pkeyutl -sign -inkey private.pem -out binary.sig -rawin -in binary

// Verify signature in client
use ed25519_dalek::{Signature, VerifyingKey, Verifier};

pub fn verify_signature(binary: &[u8], signature: &[u8], public_key: &VerifyingKey) -> Result<()> {
    let sig = Signature::from_slice(signature)?;
    public_key.verify(binary, &sig)?;
    Ok(())
}
```

---

## 6. Usage Examples

### 6.1 CLI Usage

```bash
# Set update URL
export MYAPP_UPDATE_BASE_URL=https://myapp.example.com

# Check and apply update
myapp update

# Or derive from API URL
export MYAPP_API_URL=https://api.myapp.example.com
myapp update
```

### 6.2 Programmatic Usage

```rust
#[tokio::main]
async fn main() {
    let args: Vec<String> = std::env::args().collect();
    
    if args.get(1).map(|s| s.as_str()) == Some("update") {
        let http_client = reqwest::Client::new();
        let platform = BuildInfo::current().platform;
        
        match run_update(&http_client, platform).await {
            Ok(()) => println!("Update successful!"),
            Err(e) => eprintln!("Update failed: {e}"),
        }
    }
}
```

---

## 7. Testing

### 7.1 Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn prop_sha256_deterministic(data: Vec<u8>) {
            let sha1 = compute_sha256(&data);
            let sha2 = compute_sha256(&data);
            prop_assert_eq!(sha1, sha2);
        }

        #[test]
        fn prop_sha256_length(data: Vec<u8>) {
            let sha = compute_sha256(&data);
            prop_assert_eq!(sha.len(), 64);
        }

        #[test]
        fn prop_normalize_sha_idempotent(s in "[a-fA-F0-9 \n\t]{0,100}") {
            let normalized = normalize_remote_sha(&s);
            let twice = normalize_remote_sha(&normalized);
            prop_assert_eq!(normalized, twice);
        }
    }

    #[test]
    fn test_needs_update_same() {
        assert!(!needs_update("abc123", "abc123"));
    }

    #[test]
    fn test_needs_update_different() {
        assert!(needs_update("abc123", "def456"));
    }

    #[test]
    fn test_verify_download_success() {
        let data = b"test binary content";
        let sha = compute_sha256(data);
        assert!(verify_download(data, &sha).is_ok());
    }

    #[test]
    fn test_verify_download_mismatch() {
        let data = b"test binary content";
        let result = verify_download(data, "0000000000000000000000000000000000000000000000000000000000000000");
        assert!(result.is_err());
    }
}
```

### 7.2 Integration Test

```rust
#[tokio::test]
async fn test_update_flow() {
    use wiremock::{MockServer, Mock, ResponseTemplate};
    use wiremock::matchers::{method, path};

    let mock_server = MockServer::start().await;

    // Mock SHA256 response
    let binary = b"fake binary content";
    let sha = compute_sha256(binary);

    Mock::given(method("GET"))
        .and(path("/bin/linux-x86_64.sha256"))
        .respond_with(ResponseTemplate::new(200).set_body_string(sha.clone()))
        .mount(&mock_server)
        .await;

    Mock::given(method("GET"))
        .and(path("/bin/linux-x86_64"))
        .respond_with(ResponseTemplate::new(200).set_body_bytes(binary))
        .mount(&mock_server)
        .await;

    // Test verification
    verify_download(binary, &sha).expect("verification should pass");
}
```

---

## 8. Checklist for New Applications

- [ ] Add `shadow-rs` dependency for build info embedding
- [ ] Implement `BuildInfo` struct with version, git SHA, platform
- [ ] Create update client with SHA256 verification
- [ ] Add `self_replace` dependency for atomic binary replacement
- [ ] Set up server static file serving for `/bin/` directory
- [ ] Create build script for multi-platform compilation
- [ ] Set up CI/CD pipeline for release builds
- [ ] Generate SHA256 files during build process
- [ ] Test update flow on each platform
- [ ] Document environment variables (`MYAPP_UPDATE_BASE_URL`)

---

## 9. References

- **shadow-rs**: https://docs.rs/shadow-rs/
- **self_replace**: https://docs.rs/self_replace/
- **Loom Implementation**: `crates/loom-cli/src/update.rs`
- **Distribution Spec**: `specs/distribution.md`
