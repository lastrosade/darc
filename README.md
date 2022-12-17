# Deduplicating Archiver

Keep in mind that this is my first rust project.

If it sucks you know why.

## Files

1. .da deduplicating archive
2. .di deduplicating index

## Parameters

1. Compress/Decompress

2. Compression presets (per chunk)

   1. None
   2. Lizard/lzturbo for speed.
   3. Zstd for speed/efficiency.
   4. Test multiple methods, keep the best.

3. Fast CDC chunking settings (per archive)

    Only works when creating an archive for the first time.
    When updating an archive read Fast CDC settings in index.

   - Presets
        1. Bigger chunks, for speed.
        2. Smaller chunks for tradeoff
        3. Very Small chunks for better dedup but worse compression
   - Manual settings to select optimal based on content type

4. Input/Output Files/Dirs
5. Decompression path (defaults to working dir)
6. Archive name
7. Long help containing info on compression methods and such.
8. Verbosity
9. Version

## Example

Create an archive
`darc MyArc a -m3 -c2 -i File.png -i /home/user/somedir/ -i Song.mp3:/Music/`
Creates archive MyArc
    select compression method 3
    chunking method 2
    add a file
    add a directory
    add a file with a custom archive path

The archive tree should look like

```dir
File.png
somedir/[...]
Music/Song.mp3
```

Extract File.png from the archive.
`darc MyArc d -i File.png -o ExtractDir`

`darc MyArc u -i Code.c`
Update the archive by adding a new file

`darc MyArc u -i Newconf.conf:Somefile.conf -r Somefile.conf`
Update the arhive

`darc MyArc l`
List contents

`darc MyArc l somedir/*.iso`
List contents using glob

## Process

### Archive Creation, Compression and Additive Updates

A arc_filelist dict saved in the index

```struct
file
    [hashes]
```

A fcdc_chunklist dict used during compression.

```struct
hash
    filepath
    chunk_start
    chunk_length
```

A arc_chunklist dict saved in the index

```struct
hash
    compression_method
    chunk_start
    chunk_length
```

If these object do not exist, create them. if they do, read them.

Iterate over the files using fast CDC to fill in arc_filelist and fcdc_chunklist
    if chunk exists in fcdc_chunklist do not add it.

Once done and no pending removal, pack arc_filelist and write to index.

Iterate over fcdc_chunklist, read chunks from source files, filter through compression function, write an arc_chunklist entry and write to arc.

Once done and no pending removal, pack arc_filelist and write to index.

### Removal updates

Create temporary_array
Create arc_chunklist_new

Parse arc_filelist for filenames given, add chunks to temporary_array
Parse arc_filelist ommiting files that are to be removed and remove from temporary_array the chunks that are part of files.
Walk through arc_chunklist and add chunks that are not part of temporary_array to arc_chunklist_new

Pack and write data.

### Extraction

Create extract dict

```struct
chunk_start
    compression_method
    chunk_length
    [files]
```

Parse arc_filelist and arc_chunklis
For each filenames, for each hash, add entry to extract dict
if the same chunk_start already exists, add filename to files array.

Open file descriptor for every filename
Parse extract dict in order, Seek to from chunk_start to chunk_length, decompress and write to the relevant files so as to only decrompress once.
Maybe if files only contain one entry then write directly to the fd otherwise write to a buffer then to the appropriate fds

### Crates

Things that might be useful

https://docs.rs/crate/fastcdc/latest

https://docs.rs/lzma-rs/0.2.0/lzma_rs/
https://docs.rs/zstd/0.12.1+zstd.1.5.2/zstd/
https://docs.rs/crate/brotli/3.3.4
https://docs.rs/crate/lz4_flex/0.9.5

https://docs.rs/crate/evmap/10.0.2
