# Optician

Optician is a bash script which creates optical backups of directories.

Optical backups are a useful supplement to any backup strategy. Once created,
they are read-only, decreasing the likelihood of accidental deletions and the
ability of corruption to propagate through the backups. Their physical
properties allow them to persist in environments where [spinning
rust](https://en.wikipedia.org/wiki/Hard_disk_drive) or [piles of
sand](https://en.wikipedia.org/wiki/Flash_memory) may fail. Products such as
Blu-ray [M-DISC](https://en.wikipedia.org/wiki/M-DISC) offer attractive
longevity claims, but even cheaper and more common optical media can be a good
choice if you repeatedly archive the same directories on a schedule (such as
annually).

## Usage

    Usage: optician [OPTION...]
    Create an archive and burn it.

    Required:
        -d      the directory to archive
        -n      the name of the archive

    Optional:
        -e      the key ID to encrypt the archive to
        -c      encrypt the archive with a symmetric passphrase
        -l      the label of the disc image"

## Process

The specified directory is first archived via `tar`. To simplify later
recovery, no compression is applied. The resulting archive is optionally
encrypted via `gpg` to the given key.

The integrity of the archive is next recorded via three different means:
    
1. A detached cryptographic signature of the archive is created via `gpg`.
2. Parity of the archive and the signature is recorded with 30% redundancy via `par2`.
3. Hashes of the archive, detached signature, and parity files are created via `hashdeep`.

A filesystem image containing the archive and all integrity files is then
created via `mkisofs`.

Finally, the image is burned to disc via `cdrecord` and temporary files are
deleted.

## Examples

I use Optician to create monthly optical backups of my password database. These
files are already encrypted, so I do not encrypt the archive itself.

    $ optician -d ~/.password-store -n password-store

I use Optician to create yearly optical backups of my financial data. This
consists of two different directories, so I first copy them to a single
directory to archive. The data is a mix of encrypted and plain text, so I tell
Optician to encrypt the entire archive.

    $ mkdir ~/tmp/finance
    $ rsync -a ~/library/ledger ~/tmp/finance
    $ rsync -a ~/library/finance ~/tmp/finance
    $ optician -d ~/tmp/finance -n finance -e pm@pig-monkey.com

I use Optician to create yearly optical backups of my e-book library. This is a
large directory. Optician defaults to using `XDG_RUNTIME_DIR` for its temporary
files. If that environment variable is unset, it falls back to `/tmp/$USER`. On
my machine, these are both temporary filesystems stored in memory that are too
small to contain the archive. So I tell Optician to use a directory in my home
directory.

    $ XDG_RUNTIME_DIR=~/tmp optician -d ~/library/books -n books

## Verification

Each of the three integrity methods can later be used to verify the archive.
After mounting the optical disc and changing into the directory, the hashes of
each file can be verified against those stored in the `hashes` file.

    $ hashdeep -m -k hashes *

The cryptographic signature of the archive may also be verified.

    $ gpg --verify archive-name.tar.sig

And finally the parity files can be checked.

    $ par2verify archive-name.par2

If verification fails, recovery of the archive may be attempted.

    $ par2repair archive-name.par2

Note that all of these processes will likely go more quickly if the contents
are first copied off of the optical disc.

## Requirements

* [GnuPg](https://gnupg.org/) (with a configured key for signing and, optionally, encryption)
* [Parchive](https://parchive.github.io/)
* [hashdeep](http://md5deep.sourceforge.net/)
* [Cdrtools](http://cdrtools.sourceforge.net/private/cdrecord.html)
