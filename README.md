# Android Media mtime Restorer

Restore incorrect Android photo and video **filesystem modification dates (`mtime`)** from dates encoded in their filenames.

This tool was created after an Android/WhatsApp backup and restore caused thousands of media files to receive the date of the restore operation instead of their original dates.

For example:

```text
IMG_20190723_143512.jpg
```

contains enough information to restore:

```text
2019-07-23 14:35:12
```

while a WhatsApp filename such as:

```text
IMG-20220213-WA0006.jpg
```

contains the original date but not the original time. In that case the script uses an explicitly chosen placeholder time.

The script runs on your computer and accesses the Android device through **ADB (Android Debug Bridge)**.

## What this script does

The script:

* scans one or more directories on an Android device;
* reads the current filesystem modification time of each media file;
* extracts a date/time from supported filename formats;
* optionally restricts processing to files whose current timestamps fall inside a specified corruption window;
* can first perform a completely read-only audit;
* can test the operation on only a few files;
* can copy corrected files to a new directory;
* can move corrected files to a new directory;
* can correct files in place;
* writes a CSV log of every run;
* verifies modified timestamps after a test run;
* requires a backup to the host computer before files are moved.

It **does not modify EXIF metadata**.

## Why this exists

After some Android backup/restore operations, the contents of a photo may remain perfectly intact while the filesystem timestamp is changed to the date on which the backup was restored.

This can make thousands of old photos suddenly appear as if they were created or modified on the same day.

Fortunately, many Android camera applications and WhatsApp encode at least part of the original date in the filename.

This script uses that information to reconstruct the filesystem modification date.

## Supported filename formats

Full date and time can currently be extracted from filenames resembling:

```text
IMG_20190723_143512.jpg
IMG-20190723-143512.jpg
VID_20190723_143512.mp4
PANO_20190723_143512.jpg
MVIMG_20190723_143512.jpg
BURST_20190723_143512.jpg
Screenshot_20190723_143512.png
20190723_143512.jpg
```

WhatsApp-style filenames such as:

```text
IMG-20220213-WA0006.jpg
VID-20220213-WA0006.mp4
```

contain a date but no time.

For these files the script uses a placeholder time, by default:

```text
12:00:00
```

This is intentional. The program does not attempt to invent an original time that cannot be recovered from the filename.

Files whose names do not match a known pattern are skipped.

## Requirements

You need:

* Python 3
* ADB / Android Platform Tools
* an Android phone or tablet
* a USB cable
* USB debugging enabled on the Android device

No additional Python packages are required. The script only uses the Python standard library.

## 1. Install ADB

Install Google's Android Platform Tools for your operating system and make sure `adb` is available from your terminal.

Check with:

```bash
adb version
```

## 2. Enable USB debugging

On the Android device:

1. Enable **Developer options**.
2. Enable **USB debugging**.
3. Connect the device to the computer by USB.
4. Accept the debugging authorization prompt on the phone.

Then run:

```bash
adb devices
```

You should see the device listed with the status:

```text
device
```

If it says:

```text
unauthorized
```

unlock the phone and accept the authorization prompt.

## 3. Download the script

Clone this repository:

```bash
git clone https://github.com/YOUR-USERNAME/android-media-mtime-restorer.git
cd android-media-mtime-restorer
```

Or download `restore_mtime_from_filename.py` directly.

## 4. Find the affected Android directory

For recent WhatsApp versions, images are commonly stored somewhere similar to:

```text
/storage/emulated/0/Android/media/com.whatsapp/WhatsApp/Media/WhatsApp Images
```

The exact location can vary by Android version, application, and configuration.

You can inspect directories with ADB, for example:

```bash
adb shell ls "/storage/emulated/0/Android/media/com.whatsapp/WhatsApp/Media"
```

## 5. Determine which files were corrupted

The safest way to use the program is to identify the time at which the incorrect restore occurred.

Suppose your old photographs were accidentally assigned modification dates around:

```text
2024-04-20 07:34:50
```

You can use `--corrupted-before` and/or `--corrupted-after` to restrict the files eligible for modification.

This is an important safety mechanism: files outside the specified current-mtime window are left alone.

## Recommended workflow

Do **not** begin with `--mode full`.

Use the following sequence:

```text
AUDIT → TEST → inspect results → FULL
```

### Step 1 — Audit

Audit mode is the default and does not modify anything.

Example:

```bash
python3 restore_mtime_from_filename.py \
    --dir "/storage/emulated/0/Android/media/com.whatsapp/WhatsApp/Media/WhatsApp Images" \
    --corrupted-before "2024-04-20 07:34:50"
```

On Windows, depending on your Python installation, use:

```text
python
```

instead of:

```text
python3
```

For example:

```powershell
python restore_mtime_from_filename.py --dir "/storage/emulated/0/Android/media/com.whatsapp/WhatsApp/Media/WhatsApp Images" --corrupted-before "2024-04-20 07:34:50"
```

Review the output carefully.

The script also creates a CSV audit log containing the source path, current timestamp, reconstructed timestamp, scope decision, action and status for every scanned file.

### Step 2 — Test on a few files

The default action is `copy`, which leaves the originals untouched.

Create corrected copies in a separate Android directory:

```bash
python3 restore_mtime_from_filename.py \
    --dir "/storage/emulated/0/Android/media/com.whatsapp/WhatsApp/Media/WhatsApp Images" \
    --corrupted-before "2024-04-20 07:34:50" \
    --dest-dir "/storage/emulated/0/Pictures/WhatsAppRestored" \
    --mode test \
    --test-count 5
```

This processes only five eligible files.

After writing them, the program reads their timestamps back from the Android device and reports:

```text
[PASS]
```

or:

```text
[FAIL]
```

for each file.

Inspect those files on the device before proceeding.

### Step 3 — Run the full copy

Once the test is correct:

```bash
python3 restore_mtime_from_filename.py \
    --dir "/storage/emulated/0/Android/media/com.whatsapp/WhatsApp/Media/WhatsApp Images" \
    --corrupted-before "2024-04-20 07:34:50" \
    --dest-dir "/storage/emulated/0/Pictures/WhatsAppRestored" \
    --mode full
```

The original WhatsApp files remain untouched.

## Actions

Three actions are available.

### `copy`

Default and recommended for the first full recovery.

```text
--action copy
```

The original remains where it is and a corrected copy is created in `--dest-dir`.

Example:

```bash
python3 restore_mtime_from_filename.py \
    --dir "/storage/emulated/0/Android/media/com.whatsapp/WhatsApp/Media/WhatsApp Images" \
    --dest-dir "/storage/emulated/0/Pictures/WhatsAppRestored" \
    --action copy \
    --mode full
```

### `move`

Moves files into the destination directory instead of keeping the originals in their old Android location.

Because this removes the original from its source directory, **`move` requires `--backup-dir`**.

Before any selected file is moved on the Android device, the script first downloads it to the computer.

Example:

```bash
python3 restore_mtime_from_filename.py \
    --dir "/storage/emulated/0/Android/media/com.whatsapp/WhatsApp/Media/WhatsApp Images" \
    --dest-dir "/storage/emulated/0/Pictures/WhatsAppRestored" \
    --backup-dir "./WhatsAppHostBackup" \
    --action move \
    --mode test \
    --test-count 5
```

If the host backup fails, the move is not performed.

The backup routine also refuses to overwrite an existing local backup file.

After verifying the test, change:

```text
--mode test
```

to:

```text
--mode full
```

for the complete operation.

### `fix-in-place`

Changes the filesystem timestamp of the original files without copying or moving them:

```text
--action fix-in-place
```

Example:

```bash
python3 restore_mtime_from_filename.py \
    --dir "/storage/emulated/0/Android/media/com.whatsapp/WhatsApp/Media/WhatsApp Images" \
    --corrupted-before "2024-04-20 07:34:50" \
    --action fix-in-place \
    --mode test
```

This is less reversible than `copy`. Make an independent backup before using it.

## Corruption window

You can specify either boundary:

```text
--corrupted-before
--corrupted-after
```

or both.

For example:

```bash
--corrupted-after "2024-04-20 07:30:00" \
--corrupted-before "2024-04-20 07:40:00"
```

Only files whose **current filesystem mtime** falls within that interval are eligible.

The values may also be supplied as Unix timestamps.

## Time zones

By default:

```text
--tz local
```

uses the local timezone of the computer running the script.

You can specify an explicit UTC offset instead:

```bash
--tz +02:00
```

or:

```bash
--tz -05:00
```

If the photos were taken in a different timezone from the computer currently running the script, specify the appropriate offset explicitly.

## WhatsApp files without a time

For WhatsApp-style filenames that contain only a date, the default reconstructed time is noon:

```text
12:00:00
```

You can change this:

```bash
--placeholder-time "00:00:00"
```

For example:

```bash
python3 restore_mtime_from_filename.py \
    --dir "/path/on/android" \
    --placeholder-time "12:00:00"
```

Remember that this time is only a placeholder. The original time cannot be recovered from a filename that does not contain it.

## Multiple directories

`--dir` can be repeated:

```bash
python3 restore_mtime_from_filename.py \
    --dir "/storage/emulated/0/DCIM/Camera" \
    --dir "/storage/emulated/0/Pictures" \
    --mode audit
```

## File extensions

By default the script scans:

```text
jpg
jpeg
png
heic
heif
mp4
mov
3gp
```

Specify extensions manually by repeating `--ext`:

```bash
--ext jpg --ext mp4
```

## CSV logs

Every run produces a CSV log similar to:

```text
log_restore_audit_copy_XXXXXXXXXX.csv
```

You can choose the filename yourself:

```bash
--log-csv my_restore_log.csv
```

The log contains:

* source path
* destination path
* filename parsing precision
* whether the file was in scope
* current mtime
* reconstructed target mtime
* action taken
* result/status

Keep this log until you are satisfied that the recovery was successful.

## Safety notes

This program modifies filesystem timestamps and, depending on the selected action, can copy or move files.

Before processing an important photo collection:

1. Keep an independent backup.
2. Run `audit` first.
3. Examine the proposed timestamps.
4. Run `test` on a small number of files.
5. Verify the results on both the command line and the Android device.
6. Only then use `full`.

Unrecognized filenames are deliberately skipped rather than guessed.

Files beginning with Android's `.trashed-...` naming convention are deliberately ignored.

The program never reads or modifies EXIF metadata.

## Troubleshooting

### `adb` is not found

ADB is either not installed or is not in your system `PATH`.

Verify:

```bash
adb version
```

### Device is `unauthorized`

Run:

```bash
adb devices
```

Unlock the phone and accept the USB debugging authorization dialog.

### Device is `offline`

Disconnect and reconnect the USB cable and restart ADB if necessary:

```bash
adb kill-server
adb start-server
adb devices
```

### `No such file or directory` for files that definitely exist

An earlier development version of this script contained a newline-handling bug while reading the output of `adb shell`.

That bug caused an invisible newline character to become part of generated source paths.

The current version strips both CR and LF characters from streamed ADB output.

If you are debugging a similar problem, the generated Android shell script can be inspected with:

```bash
adb shell cat -e -v /data/local/tmp/restore_mtime_apply.sh
```

Each command should appear entirely on one line.

### Scan takes a long time

The script calls Android `stat` for each media file.

On directories containing many thousands of files this can take some time. A progress count is printed every 1,000 discovered files.

This slower approach is intentional: batched `find -exec ... +` operations can exceed Android command-line argument limits on very large directories.

## Command reference

Display the complete command-line help:

```bash
python3 restore_mtime_from_filename.py --help
```

Important options:

```text
--dir DIR
    Android directory to scan. May be repeated.

--dest-dir DIR
    Android destination for copy/move.

--backup-dir DIR
    Directory on the host computer used to back up files before move.

--action {copy,move,fix-in-place}
    Operation to perform. Default: copy.

--mode {audit,test,full}
    audit = no changes
    test  = process a small sample
    full  = process all eligible files

--test-count N
    Number of files processed in test mode.

--corrupted-before TIME
    Only process files whose current mtime is at or before TIME.

--corrupted-after TIME
    Only process files whose current mtime is at or after TIME.

--placeholder-time HH:MM:SS
    Time assigned to date-only filenames.

--tz local|±HH:MM
    Timezone used when interpreting dates encoded in filenames.

--ext EXT
    File extension to scan. May be repeated.

--log-csv FILE
    Custom path for the audit CSV.
```

## Important distinction: filesystem dates vs EXIF

This utility repairs the **filesystem modification time (`mtime`)**.

It does not change metadata embedded inside the photo or video itself.

Depending on the Android gallery application, sorting behavior may use:

* filesystem timestamps;
* EXIF `DateTimeOriginal`;
* media database metadata;
* application-specific metadata.

This tool is specifically for cases where repairing the filesystem timestamp solves the problem.

## License

This project is available under the MIT License.

## Disclaimer

This utility was created to recover a real media collection after incorrect timestamps were introduced during a backup/restore process.

It has been successfully used for that purpose, but Android devices, filesystems, applications and backup systems differ.

Review what the script intends to change before using it on your own files and keep an independent backup of important data.
