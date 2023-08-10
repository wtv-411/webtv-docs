# WebTV/MSN TV PTL Format Writeup

Author: wtv-411

Refined and converted to Markdown on August 10, 2023. Originally written as a text file that was last updated on June 17, 2023

Last updated: August 10, 2023

----

.PTL is a database file format used to store TV listings on set top boxes that make use of technology developed by WebTV Networks. It is known that PTL databases are used on WebTV Plus and DishPlayer boxes, both of which have an electronic programming guide (EPG) feature that lets the user see what channels and programs from a given broadcast source (cable, antenna, satellite) are airing for the week the listings were downloaded on. They are usually stored in the box's storage medium's proprietary VFAT partition under the path `file://Disk/Browser/TV/Listings.db`. It's believed that WebTV Plus boxes started downloading PTLs from the WebTV service for listings by the time the Summer 2000 update (build 2.5) was rolled out, and up until then, were only generated locally as a cache for plaintext .TVL listings downloaded periodically by older WebTV builds. A single PTL database typically stores a day's worth of listings within a certain time frame (usually from 5 AM at the user's local time on the day to 5 AM at the user's local time the following day, although programs can report to be airing past the cutoff period).

PTL files use a binary format to store TV listing data. All integer values in the .PTL databases are packed in big endian.

## File Format

PTL databases use the following format, which makes use of a header and footer to identify the file:

| Offset | Size | Description |
|---|---|---|
| 0x00 | 4 bytes | Magic bytes - `A9 B8 C7 D6` |
| 0x04 | 1 byte | Version number. Listings generated client side by builds prior to 2.5 (before late 2000) are version 2 (0x02). Listings generated server-side for use by clients 2.5 and up (late 2000 onward) are version 3 (0x03) |
| 0x05 | 1 byte | Always appears to be 0x06 |
| 0x06 | 2 bytes | No. of blocks present in the database file |
| 0x08 | Variable | Block data |
| | 8 bytes | Database footer - `A1 23 B4 56 00 00 00 00` |

## Blocks

Blocks are array-like structures that store all data in a PTL database. Blocks store information in segments, which have a variable size as specified by the block, and blocks can contain multiple of these segments to store multiple items. Each block has its own unique ID number that appears to determine the type of data that will be stored inside the block.

All blocks use this format:

| Size | Description |
|---|---|
| 4 bytes | Magic bytes - `19 28 73 64` |
| 2 bytes | Block ID. 16-bit integer |
| 2 bytes | Unknown. Possible type value? |
| 4 bytes | Length of individual segments in block |
| 4 bytes | No. of segments in block |
| Variable | Data segments |

### ID 0x01 (Channel data)
* Stores a list of definitions for channel numbers and their corresponding names, among other things(?)
* Type?: 0x04 (Version 2), 0x01 (Version 3)
* Data segment length: 0x0E

Segment format:

| Size | Description |
|---|---|
| 2 bytes | Channel number |
| 1 byte | Possibly a type value. 0x01 if the segment is for channel identification data. 0x05 for something else. |
| 3 bytes | Location of channel station and name in block 0x07 (`WCBS\x00CBS\x00`) |
| 4 bytes | In version 3 PTLs, this is always `20 02 D0 7F`. In version 2 PTLs, though, the data here will vary. The purpose of these bytes are unknown |
| 4 bytes | Unknown |

### ID 0x02 (Program data)
* Stores information for individual TV programs. This includes: the program name and description, the content rating, what type of program it is, the production year if it is a movie, duration, extended rating (not yet documented), and category information for the program.
* Type?: 0x05 (Version 2), 0x01 (Version 3)
* Data segment length: 0x10 (Version 2), 0x0E (Version 3)

Segment format:
* Version 2:

| Size | Description |
|---|---|
| 2 bytes | Some sort of counter value. Starts at 0 and increments on each segment. |
| 1 byte | ??? |
| 3 bytes | Pointer to program title and description in string table (block 0x07) |
| 1 byte | Possible flag? Observed values include "B", "+", and 0x03 |
| 9 bytes | ??? |
| 1 byte | Possibly another flag value. Observed values include 0x00, 0x89, 0x84 |

* Version 3:

| Size | Description |
|---|---|
| 3 bytes | Pointer to program title and description in string table (block 0x07) |
| 2 bytes | A [bitfield](#program-bitfield) describing characteristics of a program |
| 2 bytes | An (average) duration for a show or movie |
| 1 byte | Unknown. Usually null, although this has been observed to carry other values |
| 1 byte | Value representing the year a movie was released. It is an integer representing the number of years after the year 1850. This value is 0 for any program that isn't a movie. |
| 4 bytes | ??? |
| 1 byte | [Category information bitfield](#category-bitfield). Puts the program into a category and corresponding subcategory. If the byte is 0xFF, then it likely has no category |

### ID 0x03 (Schedule)
* Stores schedule data for EPG.
* Type?: 0x01 (Version 2), 0x02 (Version 3)
* Data segment length: 0x09 (Version 2), 0x0A (Version 3)

Segment format:

| Size | Description |
|---|---|
| 2 bytes | Zero-based index for a channel in block 0x01 |
| 2 bytes | Zero-based index for a program in block 0x02 |
| 2 bytes | Current position of the program on the channel in minutes. This is relative to the start date specified in the PTL starting at midnight at the user's local time, and this value increases in the next segment (if any) by an amount specified in the following 2 bytes. |
| 2 bytes | Duration for a program in minutes |
| 1 byte | (Version 3 only) Unknown. Usually null |
| 1 byte | A [bitfield for scheduled programs](#schedule-bitfield) that toggles certain characteristics of the specific airing of a program. Values observed for this byte include 0x44, 0x04, 0x0c, and 0x40 |

### ID 0x04 (TV Sites)
* Stores segments identifying TV Sites for participating programs.
* Type?: 0x01
* Data segment length: 0x0E

Segment format:

| Size | Description |
|---|---|
| 1 byte | Type? Value is usually 0x02 |
| 2 bytes | Index for program in block 0x02 |
| 3 bytes | Pointer to link for TV site in string table (block 0x07) |
| 5 bytes | Unknown. Usually all 0x00's. |
| 3 bytes | Pointer to name for TV site in string table (block 0x07) |

### ID 0x05 (String data)
* Block that acts as a container for all strings.
* Type?: 0x01
* Data segment length: Variable (length of string data)
* No. of segments: 1

Format:
* Simple sequence of NUL-terminated strings. First string is the headend ID (i.e., `B12345`), and the strings that follow are a sequence of NUL-terminated strings that follow a format like this:
    * `Station name ("WWTV") - [0x00] - Channel name - [0x00]`
    * Official WebTV/MSN TV firmware do not show more than 5 characters for the channel name, but the channel name can be any length.
* Afterwards, a simple structure of text for program data is stored. This consists of two strings: the program name and its description. These structures are referenced in the segments in block 0x02 to assign names and descriptions for programs.
    * In version 2 PTLs, this is immediately followed by groups of string sequences consisting of a URL (HTTP or a wtv-url) followed by a name. This is likely meant to be the data for TV Sites.
    * On version 3 listings, the programming data will be followed by a locale string (i.e., "enUS"), and if applicable, will be immediately followed by the TV Sites data. Following this is a list of NUL-terminated names for subcategories. These strings are used for category mapping in block 0x08.

### ID 0x06 (Unknown)
* Type?: 0x01
* Data segment length: 0x02

Segment format: Unknown.

### ID 0x07 (Timestamps?)
* Contains values mostly for UNIX epoch timestamps.
* Type?: 0x02 (Version 2), 0x03 (Version 3)
* Data segment length: 0x10 (Version 2), 0x20 (Version 3)
* No. of segments: 1

Format:

| Size | Description |
|---|---|
| 4 bytes | Timestamp. This tends to match the last modified date on the .ptl file (or for version 2 PTLs, the last modified time on the original .TVLs), and the time tends to be in the early morning. Might be used to indicate when the listing was generated. If no listing data is present, then this value will be 0xFFFFFFFF |
| 4 bytes | Another timestamp. This will tend to have a date 1-2 days after the date in the previous timestamp. The purpose of this timestamp is unknown. If no listing data is present, this this value will be 0x00000000 |
| 4 bytes | In version 2 listings, this seems to be another timestamp value that has an unclear use. In version 3 listings, this value is always 0x367711F8 |
| 4 bytes | Always 0x00000000(?) |
| 3 bytes | (Version 3 only) Pointer to locale string |
| 1 byte | (Version 3 only) Unknown |
| 4 bytes | (Version 3 only) Another possible timestamp. Unknown use |
| 4 bytes | (Version 3 only) Another timestamp. Appears to indicate when listings start. This tends to be a date at 10:00 AM UTC, but in EST, the time maps to 5 AM. There's a chance the UTC time changes for different time zones to be 5 AM for a user's local time, but this has not been properly confirmed as of writing |
| 4 bytes | (Version 3 only) Another timestamp. Probably indicates the end of listings. This is usually a day right after the start time |

### ID 0x08 (Categories)
* Only appears on version 3 listings. This block contains an array that maps pointers to subcategory names to the main WebTV categories for programs in the listing.
* Type?: 0x01
* Data segment length: Variable
* No. of segments: 1

Format:

| Size | Description |
|---|---|
| 2 bytes | Count of items in the following array. Always 8 |
| Variable | An array of 16-bit integers, the number of integers present being determined by the first two bytes in the data segment. The sum of the integers in the array match up with the number of category names in the string table block. It's likely that each integer reads a certain number of category name pointers that appear later in the structure, and maps them into the 8 main categories used by WebTV/MSN TV |
| Variable | An *N*th-sized array of pointers (3 bytes) to the string table that are mapped to a specific subcategory name. The previous array sequentially tracks the values these pointers represent in separate integers, and maps the amount specified by each one for each array position (main category) |

### ID 0x09 (Unknown)
* Only appears on version 3 listings.
* Type?: 0x01
* Data segment length: 0x0A

Segment format: Unknown. This block is usually empty, but is populated in DISHPlayer listings(?)

## Bit fields
### Program bitfield

| Bits | Description |
|---|---|
| 1 | ??? |
| 2 | Seems to always be on if the program is a movie, off if it isn't. Purpose unknown |
| 3 - 6 | The actual rating for the program |
| 7 - 9 | Value for the star rating used in movies |
| 10 | ??? |
| 11 - 16 | The type the program belongs to |

#### Rating values
* 0x00: Not rated
* 0x01: G
* 0x02: PG
* 0x03: PG-13
* 0x04: R
* 0x05: NC-17
* 0x06: X
* 0x07: TV-Y
* 0x08: TV-G
* 0x09: TV-Y7
* 0x0a: TV-PG
* 0x0b: TV-14
* 0x0c: TV-MA

#### Program type value
* 0x00: Arts
* 0x01: Cartoon
* 0x02: Children's Show
* 0x03: Cinema
* 0x04: Children's Special
* 0x05: Filler
* 0x06: Finance
* 0x07: First-run syndicated
* 0x08: Game Show
* 0x09: Hobbies & Crafts
* 0x0a: Health
* 0x0b: Instructional
* 0x0c: Mini-series
* 0x0d: Music Special
* 0x0e: Music
* 0x0f: Movie
* 0x10: News
* 0x11: Network Series
* 0x12: Other
* 0x13: Pelicula
* 0x14: Playoff Sports (unconfirmed)
* 0x15: Psuedo-sports (unconfirmed)
* 0x16: Public Affairs (unconfirmed)
* 0x17: Religious (unconfirmed)
* 0x18: Sports Anthology (unconfirmed)
* 0x19: Sporting Event (unconfirmed)
* 0x1a: Daytime Soap
* 0x1b: Special
* 0x1c: Sports-related (unconfirmed)
* 0x1d: Syndicated (unconfirmed)
* 0x1e: Team vs. Team (unconfirmed)
* 0x1f: Talk Show
* 0x20: TV Movie

### Category bitfield

| Bits | Description |
|---|---|
| 1 - 4 | Zero-based index for the category the program falls into. This maps to an index in the array of 16-bit integers in block 0x08 |
| 4 - 8 | Index for the subcategory. Unlike the category index, though, this starts at 1 instead of 0. This is mapped to an index to a subcategory string at the specified category index in the second block 0x08 array |

#### Category values

* 0x00: Movies
* 0x01: Sports
* 0x02: Instructional
* 0x03: Series
* 0x04: News
* 0x05: Specials

### Schedule bitfield

| Bits | Description |
|---|---|
| 1 - 4 | ??? |
| 5 | PPV? |
| 6 | Program has closed captions. 1 = yes, 0 = no |
| 7 | Program is a repeat airing |
| 8 | Program is in stereo |
