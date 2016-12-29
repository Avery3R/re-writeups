Inside of the `white_data\sys` folder there are a couple of large `.bin` files. `filelist_*` and `white_*`. Searching the binary for strings containing `filelist` and then xrefing on that brings us to the game's main init function and game loop, located at `0x004054D0`. The first references we see to filelist strings are determining what language the game should be using by checking the name of the filelist file. 

![Language determination disasm](http://i.imgur.com/ETygoY4.png)

After performing some other initializations we don't really care about the game loads the entire contents of `sys/filelist%LANGCODE.%PLATFORM.bin` into memory, and then passes the contents of the file along with the name of it's corresponding data file, `sys/white_img%LANGCODE.%PLATFORM.bin` to a function I have called `FileController::AddBinFile` which is located at `0x00827C00`.

![hex-rays is cheating](http://i.imgur.com/EZQkFPu.png)

The file controller is an object that handles all file-related operations for the game and calls the underlying platform specific code. The AddBinFile function adds an entry to the file controller that causes it to look inside of that bin file when trying to open a file, and fixes some offsets inside of the filelist header.

![hex-rays is like heroin, try it once and you're hooked for life](http://i.imgur.com/3XiZoE0.png)

That function gives us a good idea of what the header of the filelist looks like:

![Why do things manually when you have a debugger that can make the app do things for you?](http://i.imgur.com/r2Nwodx.png)

The first two DWORDs are pointers to somewhere further in the file. The third DWORD is the number of files contained in the archive. For each file there are four WORDs. At this point I don't know what the first two are, but the third is the subpackage number, and the fourth is the location of the file info within the subpackage info.

That very first DWORD is a pointer to info used to retrieve subpackage info. For each subpackage there are three DWORDs:

![subpackage compression info](http://i.imgur.com/yje9Eca.png)

The first DWORD is the size of the subpackage info, the second is the size of the compressed subpackage info, and the third is a pointer to the zlib-compressed subpackage info.

After creating a couple of structs to reflect what we've learned so far  
![filelist structs](http://i.imgur.com/gXBL230.png)  
that offset fixing function that I mentioned earlier now looks like:

![offset fixing function part 2: electric-boogaloo](http://i.imgur.com/Dzil5mF.png)

The zlib-decompressed subpackage data for the first subpackage looks like:

![subpackage data](http://i.imgur.com/97dkoSO.jpg)

It's a series of NULL-terminated strings deliminated by colons. Each string is split into four parts:

1. Index into the data archive file(`white_*.bin`) multiply by `0x800` to get an offset to the zlib data.
2. Uncompressed size
3. Compressed size
4. File name

If the uncompressed size and compressed size are the same, then it is raw data and zlib decompression should not be applied.

I wrote some terrible hacked-together code to extract archives. If you'd like to play around with it it's available here: http://pastebin.com/XbyAsFZ1.