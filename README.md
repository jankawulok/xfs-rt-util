# xfs-rt-util

A utility for managing file usage on XFS filesystems, specifically targeting files related to the realtime (`FS_XFLAG_REALTIME`) attribute, often used in tiered SSD/HDD storage environments.

## Background: XFS Realtime and SSD/HDD Tiering

**XFS Realtime Devices:** XFS filesystems can optionally include a "realtime subvolume" or "realtime device". This is often a separate physical device (like an HDD) formatted specifically for storing large, sequential data files (e.g., video/audio streams, large logs). Key characteristics include:
    *   **Pre-allocation:** Space is typically pre-allocated in large, contiguous "extents".
    *   **Fixed Extent Size:** Helps avoid fragmentation for large files.
    *   **Performance:** Optimized for high-throughput sequential I/O, often sacrificing some random I/O performance or metadata operation speed compared to the primary XFS data device.
    *   **`FS_XFLAG_REALTIME`:** Files intended for the realtime device are marked with this flag. The filesystem directs their data blocks to be allocated on the configured realtime device.

**Tiered Storage (SSD + HDD):** A common storage setup involves using:
    *   **SSDs:** For the operating system, XFS metadata, XFS logs (for speed and responsiveness), and potentially frequently accessed or performance-sensitive regular data.
    *   **HDDs:** For bulk data storage capacity, often configured as the XFS realtime device due to its suitability for large files and lower cost per GB.

**The Problem This Tool Addresses:** In such tiered setups, regular files (those *without* the `FS_XFLAG_REALTIME` flag) can accumulate on the primary XFS data partition, which might reside on the valuable, faster SSD storage. If these non-realtime files grow large or numerous, they can consume SSD space needed for metadata, logs, or other critical data, even if the large HDD realtime device has ample free space. There's often a need to identify and manage these non-realtime files on the SSD partition.

**How `xfs-rt-util` Helps:** This utility scans a specified path (expected to be on the *SSD* XFS partition) and identifies files *without* the `FS_XFLAG_REALTIME` flag. If the total size of these files exceeds a configured threshold, it selects candidates (oldest modification time, largest size first) for action.
    *   **Identification:** It lists these candidates, helping administrators understand which non-realtime files are consuming significant space on the SSD partition.
    *   **Marking (`--move-files`):** The `--move-files` option performs an *in-place replacement* of the selected files on the *same filesystem* (the SSD partition) and sets the `FS_XFLAG_REALTIME` flag on the new file.
        *   **Important:** This action **does not physically move the file to the separate HDD realtime device.**
        *   It marks the file according to policy. This marking can be used by other scripts or manual processes to later migrate the file to the actual realtime HDD device if desired.
        *   It may change how blocks are allocated for this file *on the SSD* going forward.
        *   The copy-replace *might* help defragment space *on the SSD* if the original file was fragmented.
    *   **Reclaiming Space (`fstrim`):** After files are potentially replaced (and the old blocks deallocated), `fstrim` is called (if `--move-files` was used) to signal the underlying SSD to reclaim the now-unused blocks.


## Purpose

This tool helps manage storage space on XFS filesystems where the `FS_XFLAG_REALTIME` attribute is significant (often used for performance-critical files on dedicated subvolumes or devices). It identifies regular files (those *without* the `FS_XFLAG_REALTIME` flag set) and helps determine which of these files are candidates for migration or archival to keep the usage of non-realtime files below a specified threshold.

The primary use case is to prevent non-realtime files from consuming excessive space on potentially limited or performance-sensitive storage designated for realtime data.

## Features

*   **Scan:** Recursively scans a specified directory path on an XFS filesystem.
*   **Filter:** Identifies regular files (excluding directories, symlinks, etc.) that do **not** have the `FS_XFLAG_REALTIME` attribute set. Ignores empty files.
*   **Threshold Check:** Calculates the total size of these non-realtime files.
*   **Candidate Selection:** If the total size exceeds a configurable threshold (in Megabytes), it selects candidate files to move based on a scoring system prioritizing older (by modification time) and larger files.
*   **Reporting:**
    *   Lists the non-realtime files found.
    *   Lists the files selected as candidates for moving, including their size, modification time, and calculated score.
    *   Provides a summary of total non-realtime usage and the potential space freed by moving candidates.
    *   Verbose mode (`-v`, `--verbose`) provides more detailed output for selected files.
*   **Dry Run:** By default, only reports which files *would* be moved.
*   **Move Operation (`--move-files`):** Optionally performs the actual file moving process:
    1.  Safely locks the original file.
    2.  Creates a new temporary file.
    3.  Pre-allocates space (`fallocate`).
    4.  Copies the content.
    5.  Atomically replaces the original file with the new one (`rename`).
    6.  Sets the `FS_XFLAG_REALTIME` attribute on the newly placed file.
*   **Filesystem Trim (`fstrim`):** If files were successfully moved (using `--move-files`), it attempts to run `fstrim` on the filesystem containing the starting path to potentially reclaim freed blocks on SSDs.

## Prerequisites

*   **Rust Toolchain:** Ensure you have Rust and Cargo installed (see [rustup.rs](https://rustup.rs/)).
*   **Linux System:** This tool relies on Linux-specific features (`ioctl`s like `FS_IOC_FSGETXATTR`, `FS_IOC_FSSETXATTR`, `FITRIM`, file locking via `fcntl`).
*   **XFS Filesystem:** The target path must reside on an XFS filesystem for the realtime attribute checks to be meaningful.
*   **Permissions:**
    *   Read access to the scan directory and its contents.
    *   If using `--move-files`:
        *   Write/delete permissions in the directories containing the files to be moved.
        *   Permissions to set extended attributes (`ioctl(FS_IOC_FSSETXATTR)`).
        *   Permissions to run `fstrim` (`ioctl(FITRIM)`), which typically requires **root privileges**.

## Building

Clone the repository and build the release executable:

```bash
git clone <your-repo-url> xfs-rt-util
cd xfs-rt-util
cargo build --release
```

The executable will be located at `./target/release/xfs-rt-util`.

## Usage

```bash
./target/release/xfs-rt-util [OPTIONS] <START_PATH>
```

**Arguments:**

*   `<START_PATH>`: The directory path on the XFS filesystem to start scanning.

**Options:**

*   `-t`, `--threshold-mb <THRESHOLD_MB>`: The usage threshold in Megabytes for non-realtime files. If the total size exceeds this, the tool will select files to move. [default: 100000]
*   `-v`, `--verbose`: Enable verbose output, showing more details for files selected to move.
*   `--move-files`: **Actually perform the file moving operations and run `fstrim`**. Without this flag, the tool performs a dry run.
*   `-h`, `--help`: Print help information.
*   `-V`, `--version`: Print version information.

**Examples:**

1.  **Dry Run:** Check usage on `/mnt/my-xfs-rt` with a 50 GB threshold and list candidates:
    ```bash
    ./target/release/xfs-rt-util --threshold-mb 50000 /mnt/my-xfs-rt
    ```

2.  **Dry Run (Verbose):** Same as above, but with more details about candidate files:
    ```bash
    ./target/release/xfs-rt-util -v --threshold-mb 50000 /mnt/my-xfs-rt
    ```

3.  **Actual Move & Trim:** Scan, move files if needed to meet a 200 GB threshold, and run `fstrim`:
    ```bash
    sudo ./target/release/xfs-rt-util --threshold-mb 200000 --move-files /mnt/my-xfs-rt
    # Note: sudo is likely required for fstrim and potentially file operations depending on ownership.
    ```

## :warning: Disclaimer :warning:

*   **Use with Caution:** The `--move-files` option modifies your filesystem by replacing files and setting attributes.
*   **Test Thoroughly:** **ALWAYS test this tool in a non-production environment with test data before running it on critical systems.**
*   **Permissions:** Ensure the tool is run with the correct permissions. Running with insufficient permissions might lead to partial operations or errors. Running `fstrim` almost always requires root privileges (`sudo`).
*   **Data Loss Risk:** While care has been taken (locking, atomic rename), bugs or unexpected system interactions could potentially lead to data loss. Backups are strongly recommended.
*   **No Guarantees:** This software is provided "as is" without warranty of any kind.

## License

(Optional: Add your chosen license here, e.g., MIT, Apache 2.0)
