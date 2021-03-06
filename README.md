## Illustrate
> tools for my learning basic principle of Git, only python3 supported

Tool Name | Annotation
--- | ---
indexcat | parse .git/index to human readable
fkgit | fake version of git, implement hash-object/ls-files/add/diff etc

## Contents
- [parse index object](#indexcat)
    - [first glance of objects](#glance)
    - [contents stored in objects](#contents_store)
    - [format of index file](#format)
    - [tree and commit objects](#tree)
- [fake git version](#fkgit)
- [MIT license](#license)

<a id = 'indexcat'></a>
## indexcat.py - parse index object
- git object was referenced by sha1 hashcode
- every time you use git add, the file content was hashed as a whole
- hashcode was stored in .git/object/'hashcode'
- git has three types of data structures, blob/commit/tree
- usage
    - python3 indexcat.py [index_file]
    - python3 indexcat.py .git/index

<a id = 'glance'></a>
### first glance of ./git/objects
> take main.cpp as example, see how the objects/ files was generated

``` bash
cat main.cpp | wc
      5      12      77
```
> Namely 77 characters total.<br>
the blob data structure is just like `"blob ${chars_total}\0${Contents}"`
we suppose this main.cpp will be stored as `"blob 77\0${Contents}`"
in a object file.

```bash
echo -ne "blob 77\0" | cat - main.cpp
```
```c
blob 77#include <stdio.h>

int main(int argc, const char *argv[]) {
        return 0;
}
```
```bash
echo -ne "blob 77\0" | cat - main.cpp | shasum -a 1
bee80fe26e979b11a5ed10f4802c6aa9fbee3375  -

git add main.cpp

find .git/objects/ -type f
.git/objects/be/e80fe26e979b11a5ed10f4802c6aa9fbee3375  <= the same
```

<a id = 'contents_store'></a>
### contents stored in ./git/objects/
> contents stored in the object file was compressed by zlib.<br>
In linux we can use gunzip to simulate compress & decompress.

```bash
printf "\x1f\x8b\x08\x00\x00\x00\x00\x00" |
    cat - .git/objects/be/e80fe26e979b11a5ed10f4802c6aa9fbee3375 | gzip -d 2>/dev/null
```
```c
blob 77#include <stdio.h>

int main(int argc, const char *argv[]) {
        return 0;
}

<=== the same content
```

<a id = 'format'></a>
### format of ./git/index
```
  | 0           | 4            | 8           | C              |
  |-------------|--------------|-------------|----------------|
0 | DIRC        | Version      | File count  | Ctime          | 0
  | Nano-Sec    | Mtime        | Nano-Sec    | Device         |
2 | Inode       | Mode         | UID         | GID            | 2
  | File size   | Entry SHA-1    ...           ...            |
4 | ...           ...          | Flags  | File Name(variant)  | 4
  | Index SHA-1   ...           ...            ...            |
6 | ...                                                       |


2 | Mode - 32 bit     |      4 | Flags - 16 bit
  |-------------------|        |-------------------------|
  | 16-bit unknown    |        | 1-bit assume-valid flag |
  | 4-bit object type |        | 1-bit extended flag     |
  | 3-bit unused      |        | 2-bit stage             |
  | 9-bit unix perm   |        | 12-bit name length      |
```

> index itself is a binary file, directly *`cat`* make no sense

```bash
cd ./git
cat index
KY�����m�
         �K����M���n�����,j���3main.cpp�'m~�3 (���/�λ

xxd index

0000000: 4449 5243 0000 0002 0000 0001 59ac c102  DIRC........Y...
0000010: 190d 104b 59ac c0fd 0fbc d36d 0000 fc00  ...KY......m....
0000020: 0ba2 024b 0000 81a4 0000 03e8 0000 03e8  ...K............
0000030: 0000 004d bee8 0fe2 6e97 9b11 a5ed 10f4  ...M....n.......
0000040: 802c 6aa9 fbee 3375 0008 7361 6d70 6c65  .,j...3u..sample
0000050: 2e63 0000 da27 6d06 7e90 3320 ceac 7f28  .c...'m.~.3 ...(
0000060: 88ac d72f e28b cebb                      .../....
```
> Or `vim -b index` and then `:% !xxd`

```bash
git add main.cpp indexcat.py

# use self-written script to parse current index file, multiple files supported.
./indexcat.py .git/index

-------------------- Index File --------------------
Head: DIRC
Version: 2
File Count: 2
Ctime: 2017-09-12 05:12:54
Mtime: 2017-09-12 05:12:54
Device: 64512
-------------------- File No. 1 --------------------
Inode : 194643294
File Type: Regular File
Unix Permission: 493
UID: 1000
GID: 1000
File Size: 8730 [Char]
SHA-1: 6dd1382a4dcc9ef465515885865f41f89623873c
Valid Flag: 0
Extended Flag: 0
Stage : 0
Length: 14
File Name: indexcat.py
CheckSum: 59b7a3142df6e7b959b7a3142df6e7b90000fc00
-------------------- File No. 2 --------------------
Inode : 194643356
File Type: Regular File
Unix Permission: 420
UID: 1000
GID: 1000
File Size: 77 [Char]
SHA-1: bee80fe26e979b11a5ed10f4802c6aa9fbee3375
Valid Flag: 0
Extended Flag: 0
Stage : 0
Length: 8
File Name: main.cpp
CheckSum: f3bae40e88e41eee3d78b2ae3c7ab4f2963d8f19
-------------------- End of Parse --------------------
```
> Notice above two SHA-1 fields

```bash
find .git/objects/ -type f
.git/objects/6d/d1382a4dcc9ef465515885865f41f89623873c
.git/objects/be/e80fe26e979b11a5ed10f4802c6aa9fbee3375

git cat-file -p bee80fe26e979b11a5ed10f4802c6aa9fbee3375
```
```c
#include <stdio.h>

int main(int argc, const char *argv[]) {
    return 0;
}
```

<a id = 'tree'></a>
### tree and commit objects
```bash
git commit -m "try #1"
[master (root-commit) 69e6377] try #1
 2 files changed, 242 insertions(+)
 create mode 100755 indexcat.py
 create mode 100644 main.cpp

find .git/objects/ -type f
.git/objects/6d/d1382a4dcc9ef465515885865f41f89623873c
.git/objects/69/e6377db0916d2b76efbbfcdf6b919400dbdf10
.git/objects/89/f329a6a91ccdf6646edd513b1ccbf6616020bf
.git/objects/be/e80fe26e979b11a5ed10f4802c6aa9fbee3375

# there are two extra files:
.git/objects/69/e6377db0916d2b76efbbfcdf6b919400dbdf10
.git/objects/89/f329a6a91ccdf6646edd513b1ccbf6616020bf

git cat-file -p 89f329a6a91ccdf6646edd513b1ccbf6616020bf
100755 blob 6dd1382a4dcc9ef465515885865f41f89623873c    indexcat.py
100644 blob bee80fe26e979b11a5ed10f4802c6aa9fbee3375    main.cpp

git cat-file -t 89f329a6a91ccdf6646edd513b1ccbf6616020bf
tree

gitt cat-file -p 69e6377db0916d2b76efbbfcdf6b919400dbdf10
tree 89f329a6a91ccdf6646edd513b1ccbf6616020bf
author corsair <xiangp126@126.com> 1505217357 -0400
committer corsair <xiangp126@126.com> 1505217357 -0400

try #1

git cat-file -t 69e6377db0916d2b76efbbfcdf6b919400dbdf10
commit

# tree object is just like Directory in Linux OS
```

<a id = 'fkgit'></a>
## Usage of fkgit.py
> add fkgit.py to PATH, and `chmod +x` to make it excutable.

``` bash
fkgit -h
```
```
usage: fkgit [-h]
             {init,add,hash-object,ls-files,cat-file,commit,diff,status} ...

positional arguments:
  {init,add,hash-object,ls-files,cat-file,commit,diff,status}
    init                initialize a new repo
    add                 Add file contents to the index
    hash-object         hash contents of given file(optionally write to object
                        store)
    ls-files            list files in index
    cat-file            display contents of object
    commit              commit current state of index to master branch
    diff                show diff of files changed (between index and working
                        tree)
    status              show status of working copy

optional arguments:
  -h, --help            show this help message and exit
```
```bash
pwd
./fkgit/sample

fkgit add .
./pygit.py
./main.cpp
./README.md
./indexcat.py
./fkgit.py
./haha/god/new.txt
./deer/data.txt
./deer/raw.txt

find .git/objects/ -type f
.git/objects/d6/8e191653c2d298e0ddfc57daeb3dbd4fa78ef0
.git/objects/6d/2ca835f03fd3ea8c764552fedae78d42cd364f
.git/objects/e6/9de29bb2d1d6434b8b29ae775ad8c2e48c5391
.git/objects/49/26eacb21fb33b96f58435e2390c99331a446b8
.git/objects/85/66ef07fbfa6160dc7d07cca46aebad11802196
.git/objects/11/7f5333dfb990da66d3726ab70e9819aa639b5b
.git/objects/46/b8220f29283650c14bdf262d13c75532b0a78a
```
---
```bash
fkgit hash-object main.cpp
117f5333dfb990da66d3726ab70e9819aa639b5b

fkgit cat-file -h
```
```
usage: fkgit cat-file [-h] [-t] [-s] [-p] [-r] sha1

positional arguments:
  sha1          SHA1 of the object

optional arguments:
  -h, --help    show this help message and exit
  -t, --type    show object type
  -s, --size    show object size
  -p, --pretty  pretty-print object's content
  -r, --raw     show raw data of object after decompressed
```
---
```bash
fkgit cat-file -p 117f5333dfb990da66d3726ab70e9819aa639b5b
```
```c
#include <stdio.h>

int main(int argc, const char *argv[]) {
    printf("Hello World!");
    return 0;
}

```

<a id = 'license'></a>
## License
The [MIT](./LICENSE.txt) License (MIT)