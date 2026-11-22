# Mac Contacts Archive (ABBU file) to JSON and VCF

This python script converts an Apple "Contacts Archive" (eg, `My Contacts.abbu`) into:

- a nice JSON file of your contacts,
- the contact images stored in the .abbu file, and
- alphabetized, de-duplicated `.vcf` (vCard) files with the photos embedded (shrunk to a sensible size) and phone numbers given a `+1` country code, ready to import into Contacts or your phone.

It can also write a copy of the `.abbu` itself with smaller photos.

# Motivation

In Apple's "Contacts" app, File > Export lets you export your contacts as a nice readable vCard format (`.vcf`) or as a strange "Contacts Archive" (`.abbu`) file format. The .vcf file doesn't contain images, so you'll lose the pictures of each contact. In contrast, the .abbu file contains the images (right click, "Show Package Contents", find "Images" directory), but there's no way to view the abbu file in Contacts App without wiping out your existing contacts!:

```
Are you sure you want to replace your Contacts data
with the contacts in "Contacts - 2012-MACBOOK.abbu"?

If you click Replace All, the information in this Contacts archive
completely replaces your Contacts data. You can’t undo this action.
```

![](assets/import-abbu-warning.png)

[Searching Apple StackExchange](https://apple.stackexchange.com/search?q=abbu+file) and [GitHub](https://github.com/search?q=%22abbu%22&type=repositories) shows that lots of folks have this problem:

- Github user `jonpacker` wrote an abbu converter ([link](https://github.com/jonpacker/abbu-to-adr/blob/master/ocontacts.coffee)), but it's from 2011, it's written in CoffeeScript, it refers to `abcdmr` files (which my abbu file doesn't have), and most importantly, it misses important contact data: It imports "person" files `.abcdp`, but in my experience, the SQLite db (`.abcddb`) has more contact info than what's in the `abcdp` files: I had one abbu file with 100 contacts in the SQLite db, but only 20 `.abcdp` files.

- Annoyingly, on StackExchange, a [common solution](https://apple.stackexchange.com/a/49172/145895) has you enable iCloud, upload your current contacts, disable your network connection, then import the abbu file (thereby wiping out your contacts) to view it or export it, then reverse the process to recover your "real" contacts from iCloud. Ugh! Maybe that works, but enabling iCloud has privacy issues, and also it seems risky to depend on iCloud's file-sync logic.

- [Another solution](https://apple.stackexchange.com/questions/30544/is-a-partial-restore-from-an-address-book-abbu-file-possible) has you create a guest account on your Mac and import the Contacts there. That solution didn't work for me: Contacts claimed to import it, but no contacts appeared in the app. [One commenter](https://apple.stackexchange.com/questions/30544/is-a-partial-restore-from-an-address-book-abbu-file-possible#comment135098_30545) with the same issue blamed this on Apple changing the .abbu file format between MacOS Lion and Mavericks.

- [The best solution](https://apple.stackexchange.com/a/209942/145895) uses a SQLite database browser to view the SQLite db `AddressBook-v22.abcddb` contained in the .abbu file (right click, "Show Package Contents"). This solution gives a short SQL query to extract some basic contact info from the abbu file. But you lose the contact images.

I wrote this Python program that extends ^-- that solution:

- It queries the SQLite db to get the contact info, automatically finding and joining relevant tables.
- It copies the images out of various Images/ directories, giving them appropriate file extensions.
- It renames the images to include the names of the contacts, not just the UIDs.


# Usage

**Requirements:** Python 3 (standard library only) on a Mac. The script needs the `file` command, and uses macOS's built-in `sips` to shrink photos (without `sips` everything else works, but photos aren't shrunk).

**Quick start:** put exactly one `.abbu` file in the `in/` directory and run:

    python3 main.py

Your original `.abbu` is only ever read, never modified.

**Options** (all optional; details in the sections below):

| Flag | Default | What it does |
|---|---|---|
| `--photo-max-px N` | 512 | Shrink `.vcf` photos so neither side exceeds N pixels. `0` = keep full size. |
| `--photo-quality Q` | 80 | JPEG quality (1-100) for shrunken photos. |
| `--phone-country-code CC` | 1 | Add `+CC` to phone numbers that lack a country code. `none` = leave numbers as stored. |
| `--contacts-per-file N` | 500 | Maximum contacts in each `.vcf` in `out/vcf/`. |
| `--dedupe-only` | off | Write one sorted, de-duplicated `out/dedupe/contacts-all.vcf` instead of everything else. |
| `--shrink-abbu` | off | Also write `out/<name>-small.abbu`, a copy of the archive with smaller photos. |

`python3 main.py --help` lists them too.

The script parses the contacts and images in the .abbu file,
and creates these kinds of outputs in the `out/` directory:

1. `contacts.json` contains the contacts, formatted nicely:
    ```
    {
        "uid": "C13384AC-D081-4190-B5CB-DAEEE889A64D",
        "organization": "Apple Inc.",
        "phone": [
            [
                "Main",
                "1-800-MY-APPLE"
            ],
            [
                "Office",
                "123-123-1234"
            ]
        ],
        "url": [
            [
                "HomePage",
                "http://www.apple.com"
            ]
        ],
        "address": [
            [
                "Work",
                {
                    "street": "1 Infinite Loop",
                    "city": "Cupertino",
                    "state": "CA",
                    "zip": "95014",
                    "country": "United States",
                    "country code": "us"
                }
            ]
        ],
        "ims": [
            {
                "path": "/foo/in/My-Contacts.abbu/Images/C13384AC-D081-4190-B5CB-DAEEE889A64D",
                "info": "TIFF image data, big-endian, direntries=14, height=320, bps=6, compression=none, PhotometricIntepretation=RGB, orientation=upper-left, width=320\n",
                "image type": "tiff",
                "base name": "C13384AC-D081-4190-B5CB-DAEEE889A64D",
                "dst": "/foo/out/ims/Apple-Inc__C13384AC-D081-4190-B5CB-DAEEE889A64D.tiff"
            }
        ]
    }, ...
    ```
    - Note: the email / address / url / phone fields may have multiple "types", eg, home, work, etc.
    - Note: I preserve the UID in case you need it, like for matching up images with contacts.
    - Note: This json format supports the case where 1 contact has multiple images in the .abbu file. I don't know why an .abbu file has multiple images for some contacts, but it does.

2. The `ims/` directory contains copies of images that were found in the abbu file.
    To make the image filenames easier to use, I prepend the first name, last name, and organization.
    Also I add a file extension.

3. The `ims/orphans/` directory contains copies of images that were found in the abbu file,
    but whose filenames (UIDs) don't map to any UID of any contact.

4. The `vcf/` directory contains the contacts as vCard (`.vcf`) files: alphabetized by last name, then first name; de-duplicated (cards with the same name and a shared email or phone are merged); at most 500 contacts per file (`--contacts-per-file`); with photos embedded and `+1` added to phone numbers. Contacts with no name and no organization are skipped. Besides names, phones, emails, URLs and addresses, the cards include nickname, job title, department, birthday, notes, social profiles, instant-messaging addresses, related names and other dates.

5. With `--dedupe-only` you get `dedupe/contacts-all.vcf` instead of items 1-4, and with `--shrink-abbu` you also get `<name>-small.abbu` (see below).

## Smaller photos

Some contacts' photos are huge (a few are multi-megabyte, 3000+ pixels wide), which bloats the `.vcf` files.
By default the photos embedded in the `.vcf` files are shrunk so neither side is more than 512 pixels,
and re-encoded as JPEG (PNG if they have transparency). This typically cuts the `.vcf` size roughly in half or better,
with the largest photos going from ~2.6 MB to ~200 KB. A photo is only replaced if the result is smaller, so a few
already-tiny photos stay a bit over 512px. This uses macOS's built-in `sips`; on other systems photos are left as is (with a warning).

    python3 main.py                        # default: photos shrunk to 512px, JPEG quality 80
    python3 main.py --photo-max-px 800     # keep a bit more detail
    python3 main.py --photo-max-px 0       # don't shrink the photos in the .vcf files

(`--photo-quality Q` sets the JPEG quality, 1-100, default 80.)

## Phone numbers and country codes

Most numbers in an archive are stored without a country code (`(503) 318-2547`, `4064589401`, `720.480.5263`), so the phone may not match them against incoming calls. In the `.vcf` files, plain 10-digit US/Canada numbers (and 11-digit ones starting with `1`) are written as `+1XXXXXXXXXX`. A trailing extension (`x225`, `;702`) is kept. Left unchanged rather than guessed at: numbers already starting with `+`, 7-digit local numbers, short codes like `911`, and anything that isn't a valid 10-digit US/Canada number (e.g. foreign numbers stored without a `+`).

- `--phone-country-code CC` uses a different country code instead of `1`. For codes other than `1` only numbers that already start with that code get a `+` added.
- `--phone-country-code none` leaves numbers exactly as stored.
- `contacts.json` and the `--shrink-abbu` copy keep the original numbers; only the `.vcf` output is changed.

## Contacts per file

The `.vcf` files in `vcf/` hold at most 500 contacts each. Change that with `--contacts-per-file N`, e.g. `python3 main.py --contacts-per-file 200`. (`--dedupe-only` always writes a single file.)

## Options for a clean archive

- `--dedupe-only` skips `contacts.json` and the image copies and writes a single sorted, de-duplicated, photo-shrunk
  `out/dedupe/contacts-all.vcf` (same merging and ordering as the split files in `vcf/`).
- `--shrink-abbu` also writes `out/<name>-small.abbu`: a copy of your archive with every contact photo (and thumbnail)
  shrunk. Nothing else changes: no contacts are merged or removed, and the original in `in/` is never touched.
  The photo crop rectangles are scaled to match and each database is checked with `PRAGMA integrity_check`.
  (Whether Contacts accepts the result is untested on my end beyond that; see the warning below.)

**Why `--dedupe-only` makes a `.vcf` and not an `.abbu`:** most duplicates are the same person stored in different
accounts (the archive's `Sources/*` databases), so merging them *inside* an `.abbu` would mean rewriting Apple's internal
Core Data databases by hand, and the only way to test the result is "Import" in Contacts. To get a clean `.abbu`, let Apple write it:

1. Run `python3 main.py --dedupe-only`.
2. In a **separate macOS user (e.g. a Guest or spare user) or a Mac you don't mind changing**, open Contacts and import `out/dedupe/contacts-all.vcf` (File > Import).
3. File > Export > Contacts Archive. That `.abbu` is sorted by Contacts, de-duplicated, and has the shrunken photos.

> Importing any `.abbu` into Contacts **replaces all your Contacts data** (see the warning screenshot above). Never do this in your real account.
> The `.vcf` route doesn't carry over contact groups or photo crop rectangles.




# Details

An Apple "Contacts Archive" is a directory like `foo.abbu`. I think the `ab` stands for "Address Book". It stores contacts and their images in 3 main ways:

1. Contacts (first name, last name, phone number, etc) are stored in a SQLite3 db named `foo.abbu/AddressBook-v22.abcddb`.
    - The main table is `ZABCDRECORD` and other tables like `ZABCDEMAILADDRESS` have a column `ZOWNER` or `ZCONTACT` that's a foreign key to `ZABCDRECORD.Z_PK`. 
        - My script automatically joins to `ZABCDRECORD` any table with a `ZOWNER` or `ZCONTACT` column, to try to extract all relevant info from the SQLite db.
    - For my initial exploration, I used [DB Browser for SQLite.app](https://sqlitebrowser.org/) to browse this db, and I had to copy the db into a safe location in my home folder, not in `~/Library/`, to avoid file-permissions errors with `~/Library`.

2. Contact info also appears in Apple binary plist files with extension `.abcdp`, e.g., `foo.abbu/Metadata/95693224-BE9F-4E3C-8D13-86CDF31FF941:ABPerson.abcdp`.
    - See the contents of the `.abcdp` file with `plutil -p 'foo.abbu/Metadata/C34E458D-9B42-4818-8E27-64B4D0B41540:ABPerson.abcdp'`
    - Each `.abcdp` file contains one person's info.
    - In my experience, every `.abcdp` file seems to corresponds to 1 person in the db, and the .abcdp contains a subset of what's in the SQLite db for that person.
        - So you could probably safely ignore the `.abcdp` files. But to be safe, my script imports them and verifies all their info is already stored in the db-based contacts.
    - Importantly, not all contacts in the db have a .abcdp file. So if you simply use `plutil` to extract the `.abcdp` files, like [this answer](https://apple.stackexchange.com/a/223875/145895) says, you'll miss many contacts!


3. Images. In older archives they are files in `foo.abbu/Images/`, with or without file extensions, named after their UID in the SQLite db. In newer archives (the ones with `Sources/`) the photos live in the SQLite dbs themselves: columns `ZIMAGEDATA` (full image, with a crop rectangle in `ZCROPRECT`) and `ZTHUMBNAILIMAGEDATA` (the small display copy) of `ZABCDRECORD`. Each holds either the image bytes inline (first byte `0x01`) or the UUID of a file in `.AddressBook-v22_SUPPORT/_EXTERNAL_DATA/` next to the db (first byte `0x02`). Those external files are where the multi-megabyte photos come from. The `.vcf` export and `--shrink-abbu` read these; the `contacts.json` / `ims/` export only uses the `Images/` files.
    - Image files may not have file extensions.
        - When my script copies an image into `out/`, it appends the file extension for easy viewing.
    - Image files are not just jpgs, but also tiffs.
    - The `foo.abbu` Contact Archive may not have an `Images/` directory.
    - Not all contacts in the SQLite db have images.
    - For some reason, there seem to be a lot of image files with different UIDs, but it's all the same the picture (it's the same person). Sometimes the image is cropped slightly differently. Weird.
    - Lots of images' UIDs don't actually appear in the DB. It's like a contact was deleted out of the DB but its corresponding image wasn't deleted.


- Lastly, `foo.abbu` sometimes contains a `Sources/` dir that seems to contain other `.abcddb` sqlite dbs, other `.abcdp` files, and other `Images/` directories. My program finds and parses them all. So there may be duplicate contacts.


# Warnings / Future Work

- `in/` must contain exactly one `.abbu` file. Combining several archives isn't supported (the same contact appearing in two archives would need merging rules).
- Running the script overwrites `out/contacts.json`, `out/vcf/` (old `contacts-*.vcf` are removed first), `out/dedupe/contacts-all.vcf` and `out/<name>-small.abbu`.
- Phone numbers are only normalized in the `.vcf` files, and only for US/Canada-style numbers by default (see "Phone numbers and country codes").
- There's some sort of "Group" concept in a .abbu file, which I ignore.
- If you run this program multiple times, you should delete the images out of `ims/` and `ims/orphans/`,
or else you'll end up with image copies like `foo__2.jpg`, `foo__3.jpg`, etc. This is because my program
tries not to overwrite image files as it copies them out of the .abbu file.

# Credits

- Justin Pearson
- https://justinppearson.com
- Nov 22, 2022

- Upgrades for python 3, vcf support, normailization, etc
- Russell Castagnaro @cryptowampum
- https://russell.castagnaro.com
- Nov 22, 2026