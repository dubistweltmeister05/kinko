[[RhyGen]]
[[Blog Topics]]

# The Problem

So, an interesting issue came up at work, and it involves SD cards and files. Me and my team is working on a data logger, where the log files are stored to an SD card. We have a bunch of other features that are involved with this, but I want to talk about the methods that me and my teammate used to manage the number of files that have been stored to the SD card. We have a highly space optimized logging struct, that gets written to these binary files, all titled LOG_00x.bin - x being the number for the file. We have stipulated that each file should be no more than 50MB, and 100 such files should amount to 5GB of logs - on an 8GB SD card.

The problem statement to be implemented is quite simple, implement a clean rollover, once we have hit the 100 files threshold. The oldest log file - LOG_001.BIN in this case, shall be deleted and, and over-written with new data, while retaining the same name. The trick was to intelligently track the latest file that is being written to, and implement secure and save overwrites.

What follows is a detailing of 2 implementations of solving the same issues -

## Method 1 - Writing the current log file index to an Index.txt file on the SD card

The first approach was fairly straightforward - maintain a separate file on the SD card, `INDEX.TXT`, whose sole purpose was to remember which log file was last written to.

Every time the SD card is inserted and successfully mounted, the logger first reads this index file. If the file exists and contains a valid index, that value is incremented by one to determine the next log file that should be written to. If the incremented value goes beyond the maximum number of log files, the index wraps back around to 1.

For example, if the logger had previously written to `LOG_042.BIN`, the next time the card is mounted it would read `42` from `INDEX.TXT`, increment it to `43`, and begin writing to `LOG_0043.BIN`. Once it reaches `LOG_0100.BIN`, the next file becomes `LOG_0001.BIN` again.

Now, here is one important detail here. Before opening the selected file for writing, the existing file is explicitly deleted. This is necessary because, when we rollover, the target filename already exists on the SD card. Completely deleting it and recreating a new file ensures that we don't end up with a mixture of old logs and overwritten, new logs - all in the same file. That......is an absolute disaster!

Once the new index has been determined, the logger immediately updates `INDEX.TXT` with that value. This means that the index stored on the card always represents the file that the logger intends to use for the current logging session.

The rest of the logic is concerned with the SD card's physical state. The insertion signal is de-bounced by waiting 100 ms after the card is detected before attempting to mount it. If mounting fails, the timer is restarted and another attempt is made. Once the card is successfully mounted, the index is recovered, the next filename is generated, the old file is removed if necessary, and the binary stream is opened. 

On removal, the stream is closed and the filesystem is unmounted before the SD card state is cleared. The nice thing about this approach - finding the next file is effectively an O(1) operation. Literally - READ A DAMN FILE!

But this approach introduces another file whose integrity now matters. `INDEX.TXT` becomes a piece of persistent state that the logging system depends upon. If that file becomes corrupted, is deleted by the user, or somehow gets out of sync with the actual log files, the logger can make an incorrect decision about which file should be written next.

So while this method is simple and efficient, it effectively creates a second source of truth for the state of the logger. For me - that was an issue. So....I did the following - 

## Method 2 - Find the gap

The crux of the approach -  **don't maintain the state separately; derive it from the state of the filesystem itself.**

Instead of having an `INDEX.TXT` file telling us where the logger left off, the logger simply looks at which log files already exist on the SD card. On mounting the card, the logger starts at `LOG_0001.BIN` and checks each possible filename sequentially. The first file that does not exist is treated as the next file to be written.

This creates what we can think of as a deliberate "gap" in the sequence.  On an empty card, the first file checked is `LOG_0001.BIN`. Since it doesn't exist, that becomes the next file to write.  If files `LOG_0001.BIN` through `LOG_0045.BIN` already exist, the first missing file is `LOG_0046.BIN`, so that becomes the next logging file.

The interesting case is rollover. Suppose the card contains files `LOG_0002.BIN` through `LOG_0100.BIN`, while `LOG_0001.BIN` has been removed. The search wraps around the sequence, finds the missing `LOG_0001.BIN`, and uses that as the next file. After identifying the missing file, the logger opens it for streaming and then deliberately deletes the _next_ file in the sequence.

This is what maintains the gap.

For example, if `LOG_0046.BIN` is the missing file, the logger opens `LOG_0046.BIN` and deletes `LOG_0047.BIN`. Once `LOG_0046.BIN` starts being written, the sequence now contains a single gap at `LOG_0047.BIN`. On the next mount, the search will eventually find that gap and select `LOG_0047.BIN` as the next file.

At the rollover boundary, the same mechanism wraps around. If `LOG_0100.BIN` is the file being created, the "next" file becomes `LOG_0001.BIN`. If `LOG_0001.BIN` is subsequently missing, it becomes the next target.

There is also a useful edge case here: a card that has been completely populated with 100 files. In that situation, the search finds no gaps at all, so the logger falls back to `LOG_0001.BIN`. It then opens that file and deletes `LOG_0002.BIN`, effectively creating the gap that the algorithm needs for the next cycle.

The major difference compared to Method 1 is that there is no additional state to maintain. The filesystem itself is the source of truth. If the user removes files manually, copies old log files onto the card, or deletes `INDEX.TXT` - well, there is no `INDEX.TXT` to worry about. The algorithm simply looks at what actually exists and works from there. 

The tradeoff is that we have exchanged storage efficiency for filesystem operations. Instead of immediately knowing which file to use, we potentially perform up to 100 `file_exists()` checks every time the SD card is mounted. With only 100 possible files, however, this is a very manageable cost. More importantly, the resulting state is self-healing: the logger does not depend on a separate piece of metadata remaining synchronized with the actual files on the card.

And this is ultimately where the two approaches differ.

**Method 1 stores the state. Method 2 derives the state.**

The first approach is faster and requires fewer filesystem operations, but introduces an additional piece of persistent state that can become inconsistent. The second approach performs more work when the card is mounted, but the filesystem itself remains the single source of truth.

For a small, bounded number of files like our 100-file limit, I find the second approach particularly interesting because the additional complexity of maintaining `INDEX.TXT` simply isn't necessary. The filesystem already contains all the information - we need we just have to interpret it correctly.