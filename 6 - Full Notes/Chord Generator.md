[[Twitter Posting]]
Right then, been a while since I made something that I can actually post about. 

Thanks to `@_streetdogg 's` lectures, where he makes a sine wave generator, I got the idea to extend it into a Chord Generator! Github - https://github.com/dubistweltmeister05/Chord_Generator

So, I basically generate integer samples that are encoded as 16-bit PCM values into a raw file. These samples correspond to a sine wave that's sampled at a a rate of 441Khz, and played out for 2 seconds. If you know a little bit about music, you'll know that a chord, is nothing but a combination of multiple sine waves. 3, to be exact. The root note, the third, and the fifth note. A simple progression of 2 chords, involves generating samples of the first chord for a fixed duration, and moving on to the next chord's samples. 

Now, the prime source of a rather unpleasant listen to the progression in the output file, is an abrupt change in the chords that are being played. It feels forced, and evidently lacks cohesion. So, a better way, is to introduce Cross-Fade between the two! Here is how I implemented it. 

First, I determined when to start the fade in for the next and fade out for the current chord. The last 10% - 0.2 seconds of the allocated time (2 seconds) for each chord seemed fair. The plan was to add weights to the samples of the current and the next chord, while I was in the cross-fade zone of the sample generation process. 

A linear cross-fade seemed like a fair enough first-implementation. Basically, the current chord's samples carry the max weight at the start of the cross-fade zone, and the next chord's weight shall carry max weight as we approach the end of the zone. This repeats till we are at the last chord, which plays out with no fades. I wanted to add loops, meaning the first chord shall fade in as the last chord fades out, but that was something best left for another day. 

As of now, I have hard coded the frequencies that make up the chords - C, F, Am, and G. I wish to eventually make it user-selectable, where the user can input the number of chords they want, the duration of each chord, and the chords that comprise the progression. But first, a wav header for the output file, so that you guys can try it out on your laptops without any special software that can play audio files would be of a more urgent order.

Once I am happy with it, who knows, maybe I'll try and implement this on a microcontroller too? Either way, if you are still reading, well, thank you so much! Do take a look at the code (it isn't that big, lol), please feel free to suggest improvements, and follow for more such shenanigans!