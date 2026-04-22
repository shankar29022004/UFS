# UFS


# UNIONFS Implementation - Line by Line Explanation

## Overview
This is a FUSE (Filesystem in Userspace) implementation of a **UnionFS** filesystem. UnionFS allows overlaying two directories: an upper (read-write) layer and a lower (read-only) layer. Changes are written to the upper layer while preserving the lower layer.

---

## Header and Includes

```c
#define FUSE_USE_VERSION 31
```
- Specifies which FUSE API version to use (version 3.1)

```c
#include <fuse3/fuse.h>
```
- Main FUSE library header for creating custom filesystems

```c
#include <stdio.h>        // Standard I/O
#include <string.h>       // String operations (strcpy, strcmp, etc.)
#include <errno.h>        // Error number definitions
#include <fcntl.h>        // File control (open flags like O_RDONLY)
#include <unistd.h>       // POSIX API (read, write, close)
#include <sys/stat.h>     // File statistics and mkdir/rmdir
#include <dirent.h>       // Directory operations (opendir, readdir)
#include <stdlib.h>       // General utilities (malloc, realpath)
```

---

## Data Structure

```c
struct unionfs_state {
    char *lower_dir;      // Path to read-only lower directory
    char *upper_dir;      // Path to read-write upper directory
};
```
- Stores the two directory paths that make up the union filesystem

```c
#define UNIONFS_DATA ((struct unionfs_state *) fuse_get_context()->private_data)
```
- Macro to easily access the unionfs_state structure from any FUSE callback
- `fuse_get_context()` returns FUSE context with private_data pointer
- Casts it to our state structure

---

## COPY FILE Function

```c
int copy_file(const char *src, const char *dest)
{
    int in = open(src, O_RDONLY);
    if (in == -1) return -errno;
```
- Opens source file for reading only
- Returns error if open fails

```c
    int out = open(dest, O_CREAT | O_WRONLY | O_TRUNC, 0644);
    if (out == -1) {
        close(in);
        return -errno;
    }
```
- Opens destination file for writing with create flag
- `O_CREAT` - create if doesn't exist
- `O_WRONLY` - write only
- `O_TRUNC` - truncate (empty) if exists
- `0644` - permissions (rw-r--r--)
- Closes input file if open fails

```c
    char buf[4096];
    ssize_t n;
    while ((n = read(in, buf, sizeof(buf))) > 0)
        write(out, buf, n);
```
- Reads 4KB chunks and writes them to destination
- Loop continues until read returns 0 (EOF)

```c
    close(in);
    close(out);
    return 0;
}
```
- Closes both files and returns success

**Purpose**: Implements Copy-on-Write (CoW) - copies a file from lower directory to upper directory when it's modified.

---

## RESOLVE PATH Function

```c
int resolve_path(const char *path, char *resolved)
{
    char upper[512], lower[512], whiteout[512];
    
    snprintf(upper, sizeof(upper), "%s%s", UNIONFS_DATA->upper_dir, path);
    snprintf(lower, sizeof(lower), "%s%s", UNIONFS_DATA->lower_dir, path);
```
- Constructs full paths by concatenating base directories with the relative path

```c
    snprintf(whiteout, sizeof(whiteout), "%s/.wh.%s",
             UNIONFS_DATA->upper_dir,
             path[0] == '/' ? path + 1 : path);
```
- Creates whiteout file path (`.wh.` prefix marks deleted files)
- Skips leading `/` if present

```c
    if (access(whiteout, F_OK) == 0)
        return -ENOENT;
```
- If whiteout file exists, the file is marked as deleted
- Returns "no such file or directory" error

```c
    if (access(upper, F_OK) == 0) {
        strcpy(resolved, upper);
        return 0;
    }

    if (access(lower, F_OK) == 0) {
        strcpy(resolved, lower);
        return 0;
    }

    return -ENOENT;
}
```
- Checks upper layer first (priority)
- If found, copies path to resolved and returns success
- Then checks lower layer
- If neither exists, returns error

**Purpose**: Finds the actual file path considering both layers and whiteout markers.

---

## GETATTR Function

```c
static int unionfs_getattr(const char *path, struct stat *stbuf,
                           struct fuse_file_info *fi)
{
    (void) fi;
    char resolved[512];
```
- `(void) fi;` - unused parameter warning suppression

```c
    if (resolve_path(path, resolved) != 0)
        return -ENOENT;

    if (lstat(resolved, stbuf) == -1)
        return -errno;

    return 0;
}
```
- Gets file metadata (size, permissions, timestamps, etc.)
- Uses `lstat` (doesn't follow symlinks)
- Returns error if file not found or lstat fails

**Purpose**: Implements the getattr callback for FUSE - returns file metadata.

---

## READDIR Function

```c
static int unionfs_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                            off_t offset, struct fuse_file_info *fi,
                            enum fuse_readdir_flags flags)
{
    (void) offset; (void) fi; (void) flags;
    
    DIR *dp;
    struct dirent *de;
    
    char upper[512], lower[512];
    char seen[100][256];
    int count = 0;
```
- `DIR *dp` - directory pointer (type from `<dirent.h>`)
- `seen[]` - tracks entries already added to avoid duplicates
- `count` - number of seen entries

```c
    snprintf(upper, sizeof(upper), "%s%s", UNIONFS_DATA->upper_dir, path);
    snprintf(lower, sizeof(lower), "%s%s", UNIONFS_DATA->lower_dir, path);
    
    dp = opendir(upper);
    if (dp) {
        while ((de = readdir(dp))) {
            if (strncmp(de->d_name, ".wh.", 4) == 0) continue;
            filler(buf, de->d_name, NULL, 0, 0);
            strcpy(seen[count++], de->d_name);
        }
        closedir(dp);
    }
```
- Opens upper directory
- Reads all entries
- Skips whiteout files (`.wh.` prefix)
- Adds entries to buffer with `filler` callback
- Records entry name in `seen[]`
- Closes directory

```c
    dp = opendir(lower);
    if (dp) {
        while ((de = readdir(dp))) {
            int found = 0;

            for (int i = 0; i < count; i++)
                if (strcmp(seen[i], de->d_name) == 0)
                    found = 1;

            char whiteout[512];
            snprintf(whiteout, sizeof(whiteout), "%s/.wh.%s",
                     UNIONFS_DATA->upper_dir, de->d_name);

            if (access(whiteout, F_OK) == 0)
                found = 1;

            if (!found)
                filler(buf, de->d_name, NULL, 0, 0);
        }
        closedir(dp);
    }
```
- Opens lower directory
- For each entry, checks if already seen (upper layer priority)
- Checks if whiteout file exists (marked as deleted)
- Only adds if not found in upper layer and not whiteout'd
- Closes directory

```c
    return 0;
}
```

**Purpose**: Lists directory contents, merging upper and lower layers without duplicates.

---

## OPEN Function

```c
static int unionfs_open(const char *path, struct fuse_file_info *fi)
{
    char upper[512], lower[512], resolved[512];
    
    snprintf(upper, sizeof(upper), "%s%s", UNIONFS_DATA->upper_dir, path);
    snprintf(lower, sizeof(lower), "%s%s", UNIONFS_DATA->lower_dir, path);
```

```c
    // Handle CoW
    if ((fi->flags & (O_WRONLY | O_RDWR)) &&
        access(upper, F_OK) != 0 &&
        access(lower, F_OK) == 0) {
        
        if (copy_file(lower, upper) != 0)
            return -errno;
    }
```
- If opening for writing and file doesn't exist in upper but exists in lower
- Copy file from lower to upper (Copy-on-Write)

```c
    // Handle create (safe)
    if ((fi->flags & O_CREAT) && access(upper, F_OK) != 0) {
        int fd = open(upper, O_CREAT | O_WRONLY, 0644);
        if (fd == -1) return -errno;
        close(fd);
    }
```
- If creating a new file, create it in upper layer

```c
    if (resolve_path(path, resolved) != 0)
        return -ENOENT;

    int fd = open(resolved, fi->flags);
    if (fd == -1)
        return -errno;

    close(fd);
    return 0;
}
```
- Finds actual file path
- Opens it with requested flags
- Closes immediately (FUSE handles actual file handles)

**Purpose**: Opens files, implementing Copy-on-Write for modifications.

---

## READ Function

```c
static int unionfs_read(const char *path, char *buf, size_t size,
                        off_t offset, struct fuse_file_info *fi)
{
    (void) fi;
    char resolved[512];

    if (resolve_path(path, resolved) != 0)
        return -ENOENT;

    int fd = open(resolved, O_RDONLY);
    if (fd == -1)
        return -errno;

    int res = pread(fd, buf, size, offset);
    close(fd);
    return res;
}
```
- Finds actual file location
- Opens for reading
- Uses `pread` to read from specific offset without changing file position
- Returns number of bytes read (or error)

**Purpose**: Reads file data.

---

## WRITE Function

```c
static int unionfs_write(const char *path, const char *buf, size_t size,
                         off_t offset, struct fuse_file_info *fi)
{
    char upper[512], lower[512], resolved[512];

    snprintf(upper, sizeof(upper), "%s%s", UNIONFS_DATA->upper_dir, path);
    snprintf(lower, sizeof(lower), "%s%s", UNIONFS_DATA->lower_dir, path);

    if (access(upper, F_OK) != 0 && access(lower, F_OK) == 0)
        copy_file(lower, upper);
```
- Checks if file exists in lower but not upper
- If so, copies to upper before writing (CoW)

```c
    if (resolve_path(path, resolved) != 0)
        return -ENOENT;

    int fd = open(resolved, O_WRONLY);
    if (fd == -1)
        return -errno;

    int res = pwrite(fd, buf, size, offset);
    close(fd);
    return res;
}
```
- Finds actual file path
- Opens for writing
- Uses `pwrite` to write at specific offset
- Returns bytes written

**Purpose**: Writes data to files.

---

## CREATE Function

```c
static int unionfs_create(const char *path, mode_t mode,
                          struct fuse_file_info *fi)
{
    char upper[512];

    snprintf(upper, sizeof(upper), "%s%s", UNIONFS_DATA->upper_dir, path);

    int fd = open(upper, O_CREAT | O_WRONLY, mode);
    if (fd == -1)
        return -errno;

    close(fd);
    return 0;
}
```
- Creates new file in upper layer only
- Uses provided permissions (mode)

**Purpose**: Creates new files.

---

## UNLINK Function (Delete File)

```c
static int unionfs_unlink(const char *path)
{
    char upper[512], lower[512], whiteout[512];

    snprintf(upper, sizeof(upper), "%s%s", UNIONFS_DATA->upper_dir, path);
    snprintf(lower, sizeof(lower), "%s%s", UNIONFS_DATA->lower_dir, path);

    snprintf(whiteout, sizeof(whiteout), "%s/.wh.%s",
             UNIONFS_DATA->upper_dir,
             path[0] == '/' ? path + 1 : path);
```

```c
    if (access(upper, F_OK) == 0)
        return unlink(upper);
```
- If file exists in upper layer, delete it

```c
    if (access(lower, F_OK) == 0) {
        int fd = open(whiteout, O_CREAT | O_WRONLY, 0644);
        if (fd == -1)
            return -errno;
        close(fd);
        return 0;
    }

    return -ENOENT;
}
```
- If file exists only in lower layer, create a whiteout marker in upper
- Whiteout file hides the lower file
- Returns error if file doesn't exist

**Purpose**: Deletes files (physically or with whiteout marker).

---

## MKDIR Function

```c
static int unionfs_mkdir(const char *path, mode_t mode)
{
    char upper[512];

    snprintf(upper, sizeof(upper), "%s%s", UNIONFS_DATA->upper_dir, path);

    if (mkdir(upper, mode) == -1)
        return -errno;

    return 0;
}
```
- Creates directory in upper layer only
- Uses provided permissions

**Purpose**: Creates directories.

---

## RMDIR Function (Remove Directory)

```c
static int unionfs_rmdir(const char *path)
{
    char upper[512], lower[512], whiteout[512];

    snprintf(upper, sizeof(upper), "%s%s", UNIONFS_DATA->upper_dir, path);
    snprintf(lower, sizeof(lower), "%s%s", UNIONFS_DATA->lower_dir, path);

    snprintf(whiteout, sizeof(whiteout), "%s/.wh.%s",
             UNIONFS_DATA->upper_dir,
             path[0] == '/' ? path + 1 : path);

    if (access(upper, F_OK) == 0) {
        if (rmdir(upper) == -1)
            return -errno;
        return 0;
    }

    if (access(lower, F_OK) == 0) {
        int fd = open(whiteout, O_CREAT | O_WRONLY, 0644);
        if (fd == -1)
            return -errno;
        close(fd);
        return 0;
    }

    return -ENOENT;
}
```
- Similar to unlink but for directories
- Removes from upper if exists
- Creates whiteout marker if only in lower
- Returns error if doesn't exist

**Purpose**: Removes directories.

---

## OPERATIONS Structure

```c
static struct fuse_operations unionfs_oper = {
    .getattr = unionfs_getattr,
    .readdir = unionfs_readdir,
    .open    = unionfs_open,
    .read    = unionfs_read,
    .write   = unionfs_write,
    .create  = unionfs_create,
    .unlink  = unionfs_unlink,
    .mkdir = unionfs_mkdir, 
    .rmdir = unionfs_rmdir,
};
```
- Maps filesystem operations to callback functions
- FUSE calls these functions when users interact with the filesystem

---

## MAIN Function

```c
int main(int argc, char *argv[])
{
    if (argc != 4) {
        fprintf(stderr, "Usage: %s <lower> <upper> <mountpoint>\n", argv[0]);
        return 1;
    }
```
- Checks for exactly 3 arguments: lower directory, upper directory, mountpoint
- Exits with error if wrong number of arguments

```c
    struct unionfs_state *state = malloc(sizeof(struct unionfs_state));
    if (!state) {
        perror("malloc");
        return 1;
    }
```
- Allocates memory for unionfs_state structure
- Exits if malloc fails

```c
    state->lower_dir = realpath(argv[1], NULL);
    state->upper_dir = realpath(argv[2], NULL);

    if (!state->lower_dir || !state->upper_dir) {
        perror("realpath");
        return 1;
    }
```
- Converts relative paths to absolute paths
- `realpath` also validates that directories exist
- Exits if conversion fails

```c
    char *fuse_argv[3];
    fuse_argv[0] = argv[0];
    fuse_argv[1] = "-f";
    fuse_argv[2] = argv[3];

    return fuse_main(3, fuse_argv, &unionfs_oper, state);
}
```
- Builds FUSE argument array
- `"-f"` means run in foreground (not as daemon)
- Calls `fuse_main` which starts the filesystem
- Passes state structure as private data (accessible in callbacks)
- Never returns if successful (FUSE runs until unmounted)

**Purpose**: Initializes the filesystem and starts FUSE.

---

## Key Concepts

### Copy-on-Write (CoW)
When a file is modified, it's copied from lower to upper layer first, then modified. This preserves the lower layer.

### Whiteout Files
Files starting with `.wh.` mark deletions in the lower layer. They hide corresponding files below.

### Layer Priority
Upper layer is checked first. If a file/directory exists in both layers, the upper version is used.

### Merge Semantics
Readdir shows all unique entries from both layers, excluding whiteout'd files from lower layer.
