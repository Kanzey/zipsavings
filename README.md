# zipsavings
`zipsavings` is a simple Python script/module that uses `subprocess.Popen`
to invoke `7z l` on each given archive and print stats about it.

```
$ zipsavings ./test/dracula.7z ./test/dracula.zip
archive           |size      |unpacked  |saved     |saved_percent|file_count|type
------------------|----------|----------|----------|-------------|----------|----
./test/dracula.7z |268.38 KiB|846.86 KiB|578.48 KiB|68.31%       |1         |7z
./test/dracula.zip|310.59 KiB|846.86 KiB|536.27 KiB|63.32%       |1         |zip
```

It will also invoke (for files with extensions `.cso` and `.zso`) `csoinfo`, which
is another small tool I made, to print information about `cso` and `zso` files. If
it's missing you'll get errors but before `csoinfo` was made and added to
`zipsavings` they were also errors since `7z` can't parse `cso`/`zso` files.

You can get `csoinfo` from releases here: [FRex/csoinfo](https://github.com/FRex/csoinfo).

See `Exes` below for how to specify what `7z` and `csoinfo` exe to run.

See `Example usage` below for more complex examples of invoking and output.


# Further info

Made on `Python 3.7.1 (v3.7.1:260ec2c36a, Oct 20 2018, 14:57:15) [MSC v.1915 64 bit (AMD64)] on win32`,
might work on older versions too. If you need it for older version of Python 3 feel free to open an issue for it.

Run with `--help` or `-h` to get a help/usage message generated by argparse.

If you plan on using it or use it then please let me know and/or star this project so
that I know my work on making it usable by others is not in vain.

It should never hang (for example, on `7z` files with encrypted headers
which make `7z l` prompt for a password), crash or print garbage. On files
that are mangled, encrypted `7z`, files with not size info in the
headers `bz2`, directories, non-archive files, etc. it should print an error
and go on with processing and printing all the other files. The only was for it
to print garbage is if `7z l` itself got confused (see below
in `File formats` section for an example of where I found such a file).

Note: field 'size' means physical size of file on disk (the size listed in
'Scanning the drive for archives:' part of `7z l` output, as far as I know
it's the same as the 'Physical Size = ' line but that line isn't present in
outputs for some formats like `gzip`). It'll be the same as or slightly
larger than (due to format headers, padding, filenames and so on) the
sum of 'Compressed' column in `7z l` output. Field 'unpacked' means the sum
of 'Size' column in `7z l` output and is sum of byte sizes of all files
in the archive (loose files on disk might take up more or less space due to
how filesystem allocates disk space exactly). Due to this uncompressed formats
(`iso`, `tar`, etc.) and small archives or archives with incompressible data
where padding, headers and filenames add more bytes than compression saves
will show small negative savings.


# Exes

`zipsavings` will look through `PATH` environment variable to find `7z` and
`csoinfo` (both without any extension and with `.exe` extension, on all OSes).

When looking for `7z` it'll also look for `7za`, and fallback to it if it's
found, if both are found then `7z` will be used.

To make `zipsavings` use other exes than ones found in `PATH` the environment
variables `ZIPSAVINGS_7ZEXE` and `ZIPSAVINGS_CSOINFOEXE` or command line
parameters `--exe-7z=` and `--exe-csoinfo=` can be used.

If both command line parameters and environment variables are used to provide
a path for a given exe then command line parameters take precedence.

It's okay to mix, e.g. let one exe be found in `PATH` but use environment
variable or command line parameter for the other, or use command line paremeter
for one of the exes and environment variable for the other one.


# Options
Use `-h` or `--help` to see the full arguments list help generated by `argparse`.

Use `--total` or `-t` to print another entry at the end that is sum of all others,
`--sort=field` or `-s field` to sort by a field (pass in wrong field name to get list of field names),
add `--reverse` or `-r` to reverse the sort. Total is not sorted and always last.

Use `--stdin-filelist` to pass list of files from stdin (like from running `find -type f` or similar)
due to bash/shell saying argument list is too long to contain all your files.

Use `--walk-dir` or `--list-dir` to walk a dir tree for files or use all files in a dir (but not it's subdirs).


# File formats

It works fully (both compression stats and file
count) for `rar`, `7z`, `zip`, some `exe` (NSIS installers), etc.

In case of `iso` and `tar` (without additional compression around it) the file
count is accurate but compression will be slightly negative (see 'Note' about
output field meanings above).

In case of `cso` and `zso` the file count is set to 1 since they compress a
single `iso` file each.

In case of `xz` and `gzip` the file count will be 1 (since these aren't archives
but simple compression layers around single file, just usually used with `tar`)
but compression will be accurate (except for `gzip` where the original size
is reported modulo 2^32 due to format limitation so for files that were bigger
than 4 GiB before compression it'll be inaccurate and report high negative
savings due to 'unpacked' field being too small).

You can use the option `--guess-gzip-unpacked` to make `zipsavings` try see if
any other file has the same unpacked size modulo 2^32 as the `gzip` files being
analyzed, and use that unpacked size for the `gzip` too. This option can be
useful e.g. when comparing how well did `gzip`, `xz` and `7z` compress the same
big file. Only `gzip` will have its unpacked size modulo 2^32, so with this
option all three will have correct unpacked sizes, correct saved amount and
saved percent, etc. See the example use of it below in examples section. Beware
of the (very slim) chance of false positive if unrelated file has such
unpacked size that is same modulo 2^32 to one of the `gzip` files.

In case of `bzip2` (another compression often used with `tar`) an error will be
printed as `Size` column in `7z l` output is empty (`bz2` file format
has no header field saying how big the original uncompressed file was).

Some files give **very** unusual results if they confuse `7z` or `csoinfo`
itself (or if they were crafted with intention of confusing/misleading them).
For example, running `zipsavings` on an entire tree of files using `--walk-dir` I
ran across a `.o` file made by `GHC` (Glasgow Haskell Compiler) on `Windows 10`
that `7z l` reports as being a bit over `50 TiB` (`54975648497664` bytes,
exactly `50 TiB` and `64 MiB`) unpacked. It even broke a certain fragile part
of `7z l` output parsing code due to how wide that number is in the output (and
I never ran across an archive that had such crazy sizes in them and thus don't
have such an archive in files I test zipsavings on each commit with).

It doesn't unpack the archive nor looks at filenames to warn about possible
archive-in-archive scenarios that will make savings look really small (because
the real savings are in inner archives, not the outer one that this tool analyzes).

In case of a split archive the file contains the first part will work, with the
type listed as `Split` and all other fields except `size` (which will be the size
of just the first file) being accurate. Other parts will error out
with `Can not open the file as archive.` or (sometimes) `Headers error, unconfirmed start of archive.`


# Example usage

I use `bash` and run `zipsavings` using a bash script named `zipsavings` in my
`PATH` that does `python /path/to/my/dir/zipsavings/__main__.py "$@"` (`python`
is Python 3). If you don't intend to tinker with the code you can instead pack
it with `python -m zipapp -c zipsavings` and then run the resulting `.pyz` file
with `python /some/path/zipsavings.pyz` in some batch/bash/sh script in your
`PATH`, or pack it into a standalone executable file using some other tool.
Adjust the first part of the examples accordingly to your set up.

```
$ python zipsavings.pyz zipsavings.pyz
archive       |size    |unpacked |saved   |saved_percent|file_count|type
--------------|--------|---------|--------|-------------|----------|----
zipsavings.pyz|5.34 KiB|13.71 KiB|8.37 KiB|61.07%       |6         |zip
```

```
$ zipsavings ./test/snek.7z
archive       |size      |unpacked|saved     |saved_percent|file_count|type
--------------|----------|--------|----------|-------------|----------|----
./test/snek.7z|484.75 KiB|1.4 MiB |946.87 KiB|66.14%       |6         |7z
```

```
$ zipsavings zero.data.xz zero.data.gz
archive     |size     |unpacked |saved     |saved_percent|file_count|type
------------|---------|---------|----------|-------------|----------|----
zero.data.xz|2.21 MiB |8.79 GiB |8.79 GiB  |99.98%       |1         |xz
zero.data.gz|39.26 MiB|808.0 MiB|768.74 MiB|95.14%       |1         |gzip

$ zipsavings zero.data.xz zero.data.gz --guess-gzip-unpacked
archive     |size     |unpacked|saved   |saved_percent|file_count|type
------------|---------|--------|--------|-------------|----------|----
zero.data.xz|2.21 MiB |8.79 GiB|8.79 GiB|99.98%       |1         |xz
zero.data.gz|39.26 MiB|8.79 GiB|8.75 GiB|99.56%       |1         |gzip
```

```
$ zipsavings --list-dir test --total --sort file_count --reverse --time
ERROR: test/a.bz2 : No size data in bzip2 format.
ERROR: test/b.notarchive : Can not open the file as archive.
ERROR: test/dracula-encrypted.7z : Encrypted filenames.
ERROR: test/dracula.txt : Can not open the file as archive.
ERROR: test/fake-bad-file.cso: no CISO or ZISO 4 magic bytes.
ERROR: test/fake-short-file.cso: fread failed = 4.
ERROR: test/FreeDOS-FD12CD.7z.002 : Headers error, unconfirmed start of archive.
ERROR: test/FreeDOS-FD12CD.7z.003 : Headers error, unconfirmed start of archive.
ERROR: test/FreeDOS-FD12CD.7z.004 : Headers error, unconfirmed start of archive.
ERROR: test/FreeDOS-FD12CD.7z.005 : Headers error, unconfirmed start of archive.
ERROR: test/FreeDOS-FD12CD.zip.002 : Headers error, unconfirmed start of archive.
ERROR: test/FreeDOS-FD12CD.zip.003 : Headers error, unconfirmed start of archive.
ERROR: test/FreeDOS-FD12CD.zip.004 : Headers error, unconfirmed start of archive.
ERROR: test/FreeDOS-FD12CD.zip.005 : Headers error, unconfirmed start of archive.
ERROR: test/random10megs.7z.002 : Can not open the file as archive.
ERROR: test/random10megs.7z.003 : Can not open the file as archive.
ERROR: test/random10megs.binary : Can not open the file as archive.
ERROR: test/random10megs.zip.002 : Can not open the file as archive.
ERROR: test/random10megs.zip.003 : Can not open the file as archive.
ERROR: test/wat.txt.bz2 : No size data in bzip2 format.
There were 20 errors.
END OF ERRORS.

archive                                |size      |unpacked  |saved      |saved_percent|file_count|type
---------------------------------------|----------|----------|-----------|-------------|----------|-----
test/million-files.7z                  |6.4 MiB   |5.72 MiB  |-699.06 KiB|-11.93%      |1000000   |7z
test/d8krhj4kasdu3~.swf                |11.38 MiB |11.36 MiB |-15.36 KiB |-0.13%       |2628      |SWF
test/FreeDOS-FD12CD.iso                |418.45 MiB|417.5 MiB |-971.99 KiB|-0.23%       |553       |Iso
test/NorthBuryGrove.rar                |966.21 MiB|2.17 GiB  |1.23 GiB   |56.55%       |198       |Rar5
test/Fedora-Xfce-Live-x86_64-28-1.1.iso|1.29 GiB  |1.37 GiB  |84.26 MiB  |6.0%         |39        |Iso
test/windirstat1_1_2_setup.exe         |630.59 KiB|2.16 MiB  |1.54 MiB   |71.47%       |23        |Nsis
test/snek.7z                           |484.75 KiB|1.4 MiB   |946.87 KiB |66.14%       |6         |7z
test/showframe.cab                     |1.33 KiB  |2.29 KiB  |990 Bytes  |42.15%       |2         |Cab
test/x.tar                             |10.0 KiB  |54 Bytes  |-9.95 KiB  |-18862.96%   |2         |tar
test/d.gz                              |22 Bytes  |0 Bytes   |-22 Bytes  |0%           |1         |gzip
test/d8krhj4kasdu3.swf                 |9.87 MiB  |11.38 MiB |1.51 MiB   |13.29%       |1         |SWFc
test/dracula.7z                        |268.38 KiB|846.86 KiB|578.48 KiB |68.31%       |1         |7z
test/dracula.zip                       |310.59 KiB|846.86 KiB|536.27 KiB |63.32%       |1         |zip
test/dracula.zip.7z                    |310.74 KiB|310.59 KiB|-149 Bytes |-0.05%       |1         |7z
test/fixpdfmag.tar.lzma                |1.29 KiB  |10.0 KiB  |8.71 KiB   |87.14%       |1         |lzma
test/FreeDOS-FD12CD.7z.001             |100.0 MiB |418.45 MiB|318.45 MiB |76.1%        |1         |Split
test/FreeDOS-FD12CD.cso                |414.6 MiB |418.45 MiB|3.85 MiB   |0.92%        |1         |cso
test/FreeDOS-FD12CD.zip.001            |100.0 MiB |418.45 MiB|318.45 MiB |76.1%        |1         |Split
test/FreeDOS-FD12CD.zso                |415.23 MiB|418.45 MiB|3.22 MiB   |0.77%        |1         |zso
test/random10megs.7z.001               |4.0 MiB   |10.0 MiB  |6.0 MiB    |60.0%        |1         |Split
test/random10megs.zip.001              |4.0 MiB   |10.0 MiB  |6.0 MiB    |60.0%        |1         |Split
test/wat.txt.gz                        |1.0 MiB   |1.0 MiB   |-186 Bytes |-0.02%       |1         |gzip
test/xz.xz                             |492.93 KiB|4.79 MiB  |4.31 MiB   |89.96%       |1         |xz
---------------------------------------|----------|----------|-----------|-------------|----------|-----
TOTAL(23)                              |3.69 GiB  |5.64 GiB  |1.96 GiB   |34.7%        |1003465   |SUM
Processed 23 files (7.98/s) out of 43 given (14.92/s) in 2.883 seconds.
```


# Efficiency

This tool's main bottleneck is actual disk access and starting and running `7z` processes themselves (one for each archive given).

All of the timings below are from repeated runs on a quad core (8 thread) Intel CPU on a laptop
with plenty free RAM (for OS to cache into) with `7z.exe` and Python 3 on an SSD and archives and this tool's code on an HDD.

Running with more cores or without OS caching the exes, code and archives into RAM will give different results
but the point of `7z` being the bottleneck and Python code being irrelevant still stands.

The above 23 analyzable archives among 43 files takes literally no time to run:

```
$ zipsavings test/* -t -s file_count -r --time 2>&1 | tail -n 1
Processed 23 files (8.69/s) out of 43 given (16.25/s) in 2.646 seconds.
```

Running it on a very large list of files (MiKTex local package repo) show that `7z` is the real bottleneck:

```
$ find D:/MiKTexDownloadFiles -type f | zipsavings --stdin-filelist -t -s file_count -r --time 2>&1 | tail -n 1
Processed 3463 files (246.79/s) out of 3530 given (251.57/s) in 14.032 seconds.
```

Changing code to run and wait for 1 `7z` process at a time (by
changing `for file_group in split_into_portions(all_files, 8):` to `for file_group in split_into_portions(all_files, 1):`)
causes the tool to take predictably longer:

```
$ find D:/MiKTexDownloadFiles -type f | zipsavings --stdin-filelist -t -s file_count -r --time 2>&1 | tail -n 1
Processed 3463 files (69.61/s) out of 3530 given (70.96/s) in 49.747 seconds.
```
