# ME 7.x parser
Based on the amazing [me_cleaner](https://github.com/corna/me_cleaner), parses
 the required signed partition from an ME update file to generate a valid
 flashable ME binary.

 This was written for [Heads ROM](https://github.com/osresearch/heads)
  to allow continuous integration reproducible builds for Lenovo xx20 models
  (X220, T420, T520, [etc](https://download.lenovo.com/ibmdl/pub/pc/pccbbs/mobiles/83rf46ww.txt)).

## Usage
```bash
usage: me7_update_parser.py [-h] [-O output_file] file

positional arguments:
  file                  ME/TXE image or full dump

optional arguments:
  -h, --help            show this help message and exit
  -O output_file, --output output_file
                        save save file name other than the default
                        'flashregion_2_intel_me.bin'

```

## How to use
1. Download an intel ME update from Lenovo, the latest version as of November 2020 is `7.1.91.3272` contained in `83rf46ww.exe`:
```bash
wget https://download.lenovo.com/ibmdl/pub/pc/pccbbs/mobiles/83rf46ww.exe
```

2. Using [innoextract](https://constexpr.org/innoextract/), unpack `83rf46ww.exe`
```bash
innoextract -I app/ME7_5M_UPD_Production.bin 83RF46WW.exe
```

3. Run the script:
```bash
python me7_update_parser.py app/ME7_5M_UPD_Production.bin
```

The result is a file `flashregion_2_intel_me.bin`

## Resources
* [me_cleaner](https://github.com/corna/me_cleaner)
* [Igor Skochinsky - Rootkit in your laptop - Breakpoint 2012](http://me.bios.io/images/c/ca/Rootkit_in_your_laptop.pdf)
* [Igor Skochinsky - Intel ME Secrets - RECON 2014](https://recon.cx/2014/slides/Recon%202014%20Skochinsky.pdf)
* [MEAnalyzer](https://github.com/platomav/MEAnalyzer)
* [dump_me](https://github.com/zamaudio/dump_me)
* [me.bios.io ME_blob_format (Wayback Machine link)](https://web.archive.org/web/20200225121609/http://me.bios.io/ME_blob_format)

## Why not use me_cleaner.py

[me_cleaner.py](https://github.com/corna/me_cleaner) works by using the
 partition table to determine what information exists and what can be removed.
 Unfortunately, the ME 7.x update file only contains the data required to
 update; which does not include a partition table.  Additionally, it modified
 the file directly.  The locations of the partitions in the update file do not
 correspond to the stock file so add a partition table will not be enough.

 We must work backwards.  Using a valid ME partition from a stock BIOS,
  `me_cleaner.py` outputs a minified cleaned blob.  Inspecting this blob, we see
  there is only one remaining partition, `FTPR` or "Factory Partition". In the
  update binary, this partition is found at offset `0x00000` and has the same
  length as the `FTPR` in the ME partition from a stock BIOS, `0x76000`.  While
  the offset in the update file is `0x00000`, the offsets within this are
  relative to the location in the stock file.  The same functions that
  `me_cleaner.py` uses to strip the unnecessary parts and relocate required
  remainder from a full ME blob have been copied and modified to work with an in
  memory copy of the `FTPR` partition and hard coded offsets to mimic a full ME.


  Only the partitions are signed.The partition table is not signed and can
   be generated. Between information found on [me.bios.io ME_blob_format (Wayback Machine link)](https://web.archive.org/web/20200225121609/http://me.bios.io/ME_blob_format), [MEAnalyzer](https://github.com/platomav/MEAnalyzer) source code and Igor Skochinsky's presentations ([Rootkit in your laptop - Breakpoint 2012](http://me.bios.io/images/c/ca/Rootkit_in_your_laptop.pdf)
   and [Intel ME Secrets - RECON 2014](https://recon.cx/2014/slides/Recon%202014%20Skochinsky.pdf)) the values of the `me_cleaner.py` output are
   associated with fields.  The values of fields, in combination with the output
   from the modified me_cleaner functions, are written to a file.  The value at
   address `0x00039`is set to `0x00`, at which point the file now can pass the
   verification function that was copied from me_cleaner.

## Structure of output

<style type="text/css" rel="stylesheet">
table {
  text-align: center;
}

td > p {
  font-size:0.9rem;
  text-orientation: upright;
  white-space: nowrap;
  writing-mode: vertical-rl;
}
</style>


#### $FPT Partition table header
<table>
  <thead>
    <tr>
      <th>Offset</th>
      <th>0</th>
      <th>1</th>
      <th>2</th>
      <th>3</th>
      <th>4</th>
      <th>5</th>
      <th>6</th>
      <th>7</th>
      <th>8</th>
      <th>9</th>
      <th>A</th>
      <th>B</th>
      <th>C</th>
      <th>D</th>
      <th>E</th>
      <th>F</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0x00</td>
      <td colspan=4>ROM BYPASS INSTR 0</td>
      <td colspan=4>ROM BYPASS INSTR 1</td>
      <td colspan=4>ROM BYPASS INSTR 2</td>
      <td colspan=4>ROM BYPASS INSTR 3</td>
    </tr>
    <tr>
      <td>0x10</td>
      <td colspan="4">"$FPT"</td>
      <td colspan="4"> # of partitions</td>
      <td colspan="1"><p>Version</p></td>
      <td colspan="1"><p>Entry Type</p></td>
      <td colspan="1"><p>Length</p></td>
      <td colspan="1"><p>Checksum</p></td>
      <td colspan="2">Flash Cycle Life</td>
      <td colspan="2">Flash Cycle Limit</td>
    </tr>
    <tr>
      <td>0x20</td>
      <td colspan="2">UMA size</td>
      <td colspan="6">Flags</td>
      <td colspan="2">FIT Minor</td>
      <td colspan="2">FIT Major</td>
      <td colspan="2">FIT Hotfix</td>
      <td colspan="2">FIT build</td>
    </tr>
  </tbody>
</table>

#### Partition table entry
<table>
  <thead>
    <tr>
      <th>Offset</th>
      <th>0</th>
      <th>1</th>
      <th>2</th>
      <th>3</th>
      <th>4</th>
      <th>5</th>
      <th>6</th>
      <th>7</th>
      <th>8</th>
      <th>9</th>
      <th>A</th>
      <th>B</th>
      <th>C</th>
      <th>D</th>
      <th>E</th>
      <th>F</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0x30</td>
      <td colspan="4">Entry Name</td>
      <td colspan="4">Entry Owner</td>
      <td colspan="4">Entry Offset</td>
      <td colspan="4">Entry Length</td>
    </tr>
    <tr>
      <td>0x40</td>
      <td colspan="4">Start tokens</td>
      <td colspan="4">Max tokens</td>
      <td colspan="4">Scratch sectors</td>
      <td colspan="4">Entry Flags</td>
    </tr>
    <tr>
      <td>0x50 - 0x400</td>
      <td colspan=16>(blank filled with 0xFF)</td>
    </tr>
  </tbody>
</table>

The remainder of the file is the cleaned signed and minified `FTPR` partition.


## Things to investigate
* `ROM BYPASS INSTR 0`-`ROM BYPASS INSTR 4`: unsure what these values do or if
 they are required.  May try replacing with `0x00` or `0xFF`.
* `Header flags`: unsure of what the correspond to.
* `Entry Flags`: correlate to ExclBlockUse, Execute, Write, Read, DirectAccess.
  Not sure if any of these can be changed/disabled.
* Why does the value at `0x00039` need to be changed to `0x00`?  That has been
  tested with multiple ME update versions.
