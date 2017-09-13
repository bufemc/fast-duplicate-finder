# fast-duplicate-finder
A python program to locate duplicate files - and do it fast

A bit of history...
===================
If you're anything like me, you'll have hundreds of photos, spread over different directories, but you don't know if you've got a file repeated a number of times (e.g. you've copied the contents of an SD card to your hard drive for 'safe keeping').

In my case, this totalled around 300Gb+ of photos and videos etc. I'd tried other duplicate finding programs, and while these worked, they were
a) horrendously slow (e.g. fslint took three days)
b) weren't that helpful in the output that they gave (e.g. fslint just gave a list of duplicate file names - not even the directory where they were.

I'd created a duplicate file finding program in GAMBAS a while ago, but I thought it's time to re-code this, and do this in Python 3. After some experimentation, the resulting program worked through the 300Gb in around 20 minutes.


FEATURES
========

The initial version of the program is a simple command line python program - I'm intending to put a GUI version here, as well as a compiled windows EXE in time.

Features of the commandline version:

* Its fast: in tests, it takes around 20 mins to scan about 300Gb of files (thats around 30,000 files).
* You can specify one or more directories to scan (e.g. -i /my/directory -i /my/second/directory -i /my/third/directory)
* You can 'preserve' one or more directories (see below) (e.g. -p /my/directory/savefiles -p /my/second/directory/save)
* The program can save the results of your duplicate finding run in a XML database file - and it can use the data on subsequent runs
* You can save all the input parameters into a configuration file, so you don't have to keep typing them in ;-)
* Pattern matching of files is performed by using SHA256, not MD5.
* The program outputs a runnable script that contains the necessary file delete commands, so you can review/ammend before you run.

Key Concepts
============

Input-directories
-----------------
In my situation, I have a single 'main' directory holding all my pictures, then I go and dump all my new pictures into another directory - which could be on another part of the disk. So, I've made it so you can scan multiple input directories e.g.:
-i /My/Master/PhotosDirectory   -i "/My/Dump of SDCard"

(note the quotes if you have spaces in your directory names!)

Preserve-directories
--------------------
If you have all your 'sorted' files in a master directory, and all your new files in another directory, the last thing you want is for the duplicate finder to delete files from your master directory and keep them in the new-files directory.
The 'preserve' directive says "if you find duplicates, don't delete files found in this directory or below"

e.g.
If you have:
/My/Master/PhotosDirectory
    |
    >  Summer 2017
        |
        > smiles.jpg

/My/SDCardDump
    |
    > dc12345.jpg
    
Using the SHA256 comparison, the program knows that 'smiles.jpg' and 'dc12345.jpg' are the same, however, adding the command:
    -p /My/Master/PhotosDirectory

instructs the program to not delete anything in the PhotosDirectory, or below - if duplicates are found, it will delete them from the 'other' directory - in this case, it'll delete 'dc12345.jpg'.

So, using the 'preserve' command, it means that you can dump all your pictures from your SD card into a directory, and once the program has finished, you'll only be left with 'new' files.

Note, you can have multiple 'preserve' directories - just like multiple input directories.


Output file
-----------
The program is not intended to perform the deletions of duplicate files.
What I mean by this is that with the best of intentions, its not possible to create a bug-free program, and so, the program should never immediately perform file deletions. Instead, it'll create an output file which contains all the deletions - so that you, the user, can check it, and be happy with it before its run.

To help you check, the program outputs comments in the file so you can see if its correct - or alternatively ammend so that you switch which file is deleted and which is preserved.

for example - the output file will look something like:
#    (7f8d294589939739ff779b1ac971a07006c54443dce941d7a1dff026839272b3) /home/carl/Downloads/lib/source-map/array-set.js SAVE Size:2718
# KEEP "/home/carl/Downloads/lib/source-map/array-set.js"
#    (7f8d294589939739ff779b1ac971a07006c54443dce941d7a1dff026839272b3) /home/carl/Downloads/source-map/lib/source-map/array-set.js Size:2718
rm "/home/carl/Downloads/source-map/lib/source-map/array-set.js"

So, for each set of duplicates found, you see the SHA256 check (showing the files are the same), the location of the file and its name, and also its size.
You should also see '# KEEP "....    this is one of the duplicates, which the program has elected to keep, and
rm "...    this is the other (duplicate) file(s), which the program has elected to delete.

Note: If you used the 'preserve' option, you would see  # PRESERVE "....   against the file, rather than # KEEP "...

As the output is a text file of commands, you should have no difficulty in altering it if you decide to switch around which to keep and which to delete.


OK, you say its fast - how's that done? What trick has been employed?
=====================================================================
as mentioned above, in tests the program found duplicates in around 300Gb (approx 30,000 files), in around 20 minutes. There's a couple of 'tricks' being employed here:

Don't calculate SHA256 hashes for every file
--------------------------------------------
There's no point in calculating a hash value for a file if there's no other files exactly the same size - by definition, there can be no 'exactly the same file' if there are no other files of the same size. This alone makes a massive speed increase (compared with some other duplicate finders that calculate hashes for every file).  In the 30,000 file example, it meant hashes were created for around 1000 files.

Use an XML database
-------------------
By using the -d <databasefile> option, when the program finishes, it saves all the data its gathered to a file - including the important SHA256 values - these calculations are what take the time when running.
The next time the program is run, and you specify the database file, it initially re-reads the values in from the database file, and instead of re-calculating the hash for a file, it uses the saved value.
Note: It'll only used the saved SHA256 hash value if the file name is the same, the size is the same and the modification date time is the same.
Note2: There's nothing stopping you _not_ specifying the database file, and the program will then re-calculate the SHA256 value for the files fresh each time.

Oh, by the way, the 20 minutes example was _before_ the XML database save was used - when the database file was used the time came down to around 6 minutes :-)


Configuration file
==================
There's quite a number of parameters that can be entered, and if you run this on a regular basis, it can be painful to keep typing in.
By using the '-c <configfile' option, you can have these pre-defined in INI format e.g.:

[InputDirectories]
InputDirectory=/home/carl/Dropbox
InputDirectory2=/home/carl/Documents
InputDirectory3=/home/carl/AnotherDirectory

[OutputFile]
OutputFileName=/home/carl/Dropbox/CARL/Development

[DatabaseFile]
DatabaseFileName=/home/carl/Dropbox/CARL/Development/scandb.xml

[PreserveDirectories]
PreserveDirectory=/home/carl/Dropbox/Development
PreserveDirectory2=/home/carl/Dropbox/ADirectoryToPreserve

These INI entries correspond to the input parameters of the program.

NOTE: If you use the -c option, you can still use the other -i and -p options - the extra ones on the command line are just added to the ones defined in the configuration file...


USE
===

fdf_scanner.py -i <inputdir> [-i <inputdir>] [-p <preservedir>] [-p preservedir] [-d <databaseSavefile>] [-o <outputfile>]


Happy De-duplicating!

Carl.
