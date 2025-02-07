# Tauri Plugin printer

Repositories which I forked from:

[Alfian Lensun's Original Plugin Repository](https://github.com/alfianlensundev/tauri-plugin-printer)

[adao99's Fork for Tauri V2 Beta](https://github.com/adao99/tauri-plugin-printer-v2)

[catfo435's Fork for Tauri V2 More-Stable](https://github.com/catfo435/tauri-plugin-printer)

Tauri V2 stable requires a different cofiguration with newly added features like permissions, capabilities which makes the code in the previous repos backwards-incompatible.

Development Worflow:

### 1. Created Plugin Scaffold 
```bash
npx @tauri-apps/cli plugin new printer
```

### 2. Copied the code from the old repo to this project

### 3. Setup permissions
Add allow clauses for **ALL** commands in ```permissions/default.toml```

```toml
[default]
description = "Default permissions for the plugin"
permissions = ["allow-create-temp-file", "allow-remove-temp-file", "allow-get-printers", "allow-get-printers-by-name", "allow-print-pdf", "allow-get-jobs", "allow-get-jobs-by-id", "allow-resume-job", "allow-restart-job", "allow-pause-job", "allow-remove-job"]
```
⚠️ There might be a better way to do this (I tried using `build.rs` it did not set default permissions, it set for each command separately)


### 4. Add plugin to your project
- If you are using the plugin locally, you have to add `printer:default` to your `capabilities/default.json` under permissions

    ``` json
    {
        "permissions": [
            "printer:default"
        ]
    }
  ```
- If you use `npx tauri add`, you dont need to. (I think so, didn't check.)

### 5. How Felipe use it

I am not a Rust or even frontend developer (basically I work with RoR more focused on backend), so I am not sure if this is the best way to use it. But it works for me.

As I was using other rust plugin to print for UNIX and get printers, I didn't want to change the way that I do (all related to printing to be selected on Tauri/Rust).

- To get printers, and print on UNIX or other systems, I use https://github.com/talesluna/rust-printers

- Import using `use tauri_plugin_printer;` on rust file that you want to use it.

- I am also using this to create temp files on Tauri/Rust `tempfile` and `base64` to encode the file to base64.

- My capabilities
```json
    {
        "permissions": [
            "printer:default",
            "printer:allow-print-pdf",
            "printer:allow-custom-get-printers-by-name",
            "printer:allow-custom-print-pdf",
            "printer:allow-get-printers-by-name",
            "printer:allow-get-printers"
        ]
    }
  ```

```rust

fn generate_temp_file(content: &[u8]) -> Result<String, String> {
    let temp_dir = std::env::temp_dir().join("stocki_temp");
    fs::create_dir_all(&temp_dir)
        .map_err(|e| format!("Failed to create temp directory: {}", e))?;

    // Create a temp file with .pdf extension that won't be automatically deleted
    let temp_file = Builder::new()
        .prefix("receipt-")
        .suffix(".pdf")
        .rand_bytes(5)
        .tempfile_in(&temp_dir)
        .map_err(|e| format!("Failed to create temp file: {}", e))?;

    let temp_path = temp_file.path().to_path_buf();

    // Persist the file (prevent automatic deletion)
    let (mut file, path) = temp_file.keep()
        .map_err(|e| format!("Failed to persist temp file: {}", e))?;

    file.write_all(content)
        .map_err(|e| format!("Failed to write content: {}", e))?;

    Ok(path.to_str()
        .ok_or("Invalid temp file path")?
        .to_string())
}

fn windows_print_file(app_handle: tauri::AppHandle, temp_path: &str, request: PrintRequest) -> Result<(), String> {
    let preferred_printer = get_saved_printer_preference(&app_handle)
        .ok_or_else(|| "No preferred printer set".to_string())?;

    println!("Printing to printer: {}", preferred_printer);
    tauri_plugin_printer::custom_print_pdf(preferred_printer, temp_path.to_string(), String::new(), false);

    Ok(())
}

async fn print_receipt(app_handle: tauri::AppHandle, request: PrintRequest) -> Result<(), String> {
    let temp_path = generate_temp_file(&request.content)?;
    windows_print_file(app_handle, &temp_path, request)
    if let Err(e) = fs::remove_file(&temp_path) {
        println!("Warning: Failed to remove temporary file {}: {}", temp_path, e);
    }

    result
}
```

On frontend (using react/ts)
```tsx
  const printSilentlyWithTauri = async (pdfBytes: Uint8Array) => {
    try {
      await invoke('print_receipt', { 
        request: {
          content: Array.from(pdfBytes),
          is_pdf: true
        }
      });
      return true;
    } catch (error) {
      console.error('Failed to print silently:', error);
      return false;
    }
  };

  export const openPrintWindow = async (props: props) => {
    // I generate pdfBlob using .toBlob() from react pdf with my props.
    const pdfBytes = new Uint8Array(await pdfBlob.arrayBuffer());
    const printed = await printSilentlyWithTauri(pdfBytes);
  }
```