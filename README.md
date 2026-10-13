# SimpleFS

A compact educational file system implemented in **C** that stores files inside a fixed-size binary disk image. SimpleFS demonstrates core operating-system file system concepts such as superblocks, inodes, bitmaps, directory entries, block allocation, metadata updates, and binary file I/O.

> Developed as a team project. My primary contribution focused on the **file-adder search and allocation logic**, including first-fit inode/data-block allocation, duplicate detection, directory-slot discovery, required-block calculation, and allocation-state management.

---

## Overview

SimpleFS uses a **256 KiB virtual disk image** and a fixed on-disk layout. The project contains two main programs:

- `simplefs_builder` — creates and initializes a fresh SimpleFS image.
- `simplefs_adder` — adds regular files into an existing SimpleFS image.

The implementation intentionally keeps the design small and predictable so that the underlying file system structures can be inspected directly using tools such as `xxd`.

---

## Key Features

- Creates a fixed-size **256 KiB** file system image
- Uses a **4096-byte block size**
- Maintains separate **inode and data-block bitmaps**
- Supports **32 inodes**
- Uses **first-fit allocation**
- Stores file metadata in fixed-size inodes
- Supports up to **3 direct data-block pointers per file**
- Supports files up to **12,288 bytes**
- Stores filenames in root-directory entries
- Prevents duplicate filenames
- Handles zero-byte files correctly
- Detects oversized files and resource exhaustion
- Validates the SimpleFS magic number
- Provides safe error handling for missing/invalid files and images
- Supports binary-level verification with `xxd`

---

## File System Layout

SimpleFS contains **64 blocks**, each **4096 bytes**.

| Block | Purpose |
|---:|---|
| `0` | Superblock |
| `1` | Inode bitmap |
| `2` | Data bitmap |
| `3` | Inode table |
| `4` | Root directory |
| `5-63` | File data blocks |

### Core Parameters

| Property | Value |
|---|---:|
| Disk image size | 262,144 bytes |
| Block size | 4096 bytes |
| Total blocks | 64 |
| Total inodes | 32 |
| Root inode | 1 |
| Root data block | 4 |
| Inode size | 128 bytes |
| Directory entry size | 64 bytes |
| Maximum filename length | 58 characters |
| Direct pointers per inode | 3 |
| Maximum file size | 12,288 bytes |

---

## How It Works

### 1. Building the File System

`simplefs_builder` initializes a clean disk image by:

1. Creating a zero-filled 256 KiB image.
2. Writing the superblock.
3. Marking the root inode as allocated.
4. Marking the root data block as allocated.
5. Creating root inode `1`.
6. Creating the `.` and `..` root-directory entries.

The initial state contains only the root directory.

### 2. Adding a File

`simplefs_adder` performs the following workflow:

1. Validates command-line arguments.
2. Opens and validates the SimpleFS image.
3. Opens the source file and determines its size.
4. Rejects files larger than the supported maximum.
5. Checks whether the filename already exists.
6. Finds the first free inode.
7. Calculates the required number of data blocks.
8. Allocates data blocks using first-fit.
9. Finds a free root-directory entry.
10. Copies the source file into the allocated blocks.
11. Creates and writes the file inode.
12. Updates inode and data bitmaps.
13. Creates the directory entry.
14. Updates the root-directory size.

---


### Implemented

- Free inode discovery using first-fit allocation
- Free data-block discovery using first-fit allocation
- Inode-number and bitmap-index conversion
- Data-block and bitmap-index conversion
- Duplicate filename detection
- Free root-directory entry search
- Required-block calculation using ceiling division
- Zero-byte file handling
- In-memory data-block reservation before metadata commit
- Insufficient-data-block detection

### Allocation Logic

For an inode:

```text
inode number = bitmap index + 1
```

For a data block:

```text
absolute block = DATA_REGION_BLOCK + bitmap index
```

Required blocks are calculated as:

```text
(size + BLOCK_SIZE - 1) / BLOCK_SIZE
```

with a special case for zero-byte files.

This ensures that partial blocks are counted correctly and that repeated first-fit searches do not allocate the same block twice.

---

## Build

### Requirements

- Linux
- GCC
- Standard C11 toolchain

Compile both programs:

```bash
gcc -Wall -Wextra -std=c11 simplefs_builder.c -o simplefs_builder
gcc -Wall -Wextra -std=c11 simplefs_adder.c -o simplefs_adder
```

---

## Usage

### Create a new file system image

```bash
./simplefs_builder --image disk.img
```

Example output:

```text
SimpleFS image created successfully: disk.img
```

### Add a file

```bash
./simplefs_adder --input disk.img --file test1.txt
```

Example output:

```text
test1.txt added successfully to disk.img
```

### Add multiple files

```bash
./simplefs_adder --input disk.img --file test1.txt
./simplefs_adder --input disk.img --file test2.txt
./simplefs_adder --input disk.img --file test3.txt
```

---

## Example Allocation

On a fresh image:

```text
Root inode        -> inode 1
Root directory    -> block 4
First added file  -> inode 2
First file block  -> block 5
```

After adding three one-block files:

```text
Inodes:
1 -> root
2 -> test1.txt
3 -> test2.txt
4 -> test3.txt

Data blocks:
4 -> root directory
5 -> test1.txt
6 -> test2.txt
7 -> test3.txt
```

The inode and data bitmaps both begin with:

```text
0F
```

which represents the first four allocation bits being set.

---

## Testing

The implementation was tested with:

- Fresh image creation
- Exact image-size verification
- Root inode initialization
- Root `.` and `..` entries
- One-block files
- Two-block files
- Three-block files
- Multiple files
- Zero-byte files
- Maximum-size files
- Oversized-file rejection
- Duplicate filename rejection
- Long-filename rejection
- Missing source file handling
- Missing image handling
- Invalid magic-number handling
- No-free-inode condition
- Insufficient-data-block condition
- Binary inspection of inodes, bitmaps, directory entries, and file data

Example verification commands:

```bash
xxd -s 4096 -l 8 disk.img
xxd -s 8192 -l 8 disk.img
xxd -s 16384 -l 320 disk.img
```

---

## Repository Structure

```text
.
├── README.md
├── simplefs.h
├── simplefs_builder.c
├── simplefs_adder.c
├── test1.txt
├── test2.txt
├── test3.txt
└── .gitignore
```

Generated binaries, backup source files, and `disk.img` are intentionally excluded from version control.

---

## Concepts Demonstrated

This project strengthened my understanding of:

- File system layout design
- Superblocks
- Inodes
- Allocation bitmaps
- Root-directory structures
- Direct block pointers
- First-fit allocation
- Binary file I/O in C
- Byte offsets and on-disk structures
- Metadata consistency
- Error handling
- Low-level debugging with `xxd`

---

## Current Limitations

SimpleFS is intentionally minimal and does not support:

- Subdirectories
- Indirect block pointers
- File deletion
- File renaming
- Hard links
- Symbolic links
- Permissions
- Journaling
- Checksums
- Mounting through the operating system
- Dynamic directory expansion

Because each file inode contains only three direct pointers, the maximum supported file size is **12,288 bytes**.

---

## Possible Future Improvements

Potential extensions include:

- File deletion
- File renaming
- Nested directories
- Indirect block pointers
- Free-space reclamation
- Larger file support
- Filesystem consistency checking
- A command-line file listing tool
- File extraction/read-back support
- Automated test scripts

---

## Author

**Mejbah Uddin Bhuiyan**  
Full-Stack Laravel Developer • Backend Developer • Student Researcher • AI & ML Enthusiast

- GitHub: [MejbahUddinBhuiyan](https://github.com/MejbahUddinBhuiyan)
- Portfolio: [portfolio.mejbahuddin.xyz](https://portfolio.mejbahuddin.xyz/)
- LinkedIn: [Mejbah Uddin Bhuiyan](https://www.linkedin.com/in/mejbah-uddin-bhuiyan-79b9b6249/)

---

## Note

This repository is intended to showcase low-level file system implementation concepts and practical operating-system programming in C.
