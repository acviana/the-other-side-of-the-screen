Title: Writing a FITS File Bigger Than Your Memory
Date: 2014-04-08
Tags: fits, python, code
Slug: writing-a-fits-file-bigger-than-your-memory
Author: Alex C. Viana
Category: Work

<center>_I need to start by thanking [Erik Bray](XXX) for taking the time to explain this to me and then proof reading this post._</center>


> "I’ve always thought that one of the the great things about physics is that you can add more digits to any number and see what happens and nobody can stop you."  
> - Randall Munroe, [What If?](https://what-if.xkcd.com/20/)


I've been working a lot lately on my HST WFC3 [PSF project](http://acviana.github.io/tag/psf.html) and recently had to solve a challenging scaling problem that forced me to deal with the hardware limits of my machine. I needed to create a FITS file containing a 4,000,000 x 11 x 11 data cube [^1]. This is more than even my 16GB machine can handle and it resulted in [paging](http://en.wikipedia.org/wiki/Paging) to the virtual memory which killed performance. As I was trying to find a solution to this problem a play on Randall Munroe's quote from the beginning of this post kept popping up in my head:

_"The annoying thing about writing software is that people can just add zeros and break everything and you can't stop them."_ 

So as exciting as it was that my dataset had grown to the point that it couldn't all fit in memory at once, the question now was _"how do you create a FITS file from a NumPy array that's too big to fit in memory?"_

### Chunking the Data

I store my entire dataset in a MySQL database so my first step was to break up the stellar images in the database into "chunks" that fit easily into memory. To do this I first queried the database to find the total number of records I was going to retrieve, then divided that by the size of my chunks and rounded up. Because all my records have a monotonically increasing integer primary key I can use the value of my primary key (`id`) to essentially numerically index the records and select all the records between two id numbers like slices in a Python list. Something like this:

```sql
SELECT * FROM psf_table WHERE id >= 0 AND id < 300,000;
```

This takes advantage of the fact that the `id` values I'm querying are contiguous and all the chunks contain the same number of records. If the records were more scattered each chunk would return a different number of records (up to the maximum chunk size) depending on how many records fell between those two `id` values. But for my purposes that's fine, my concern is keeping the memory use under control. If I cared about performance at that level I would learn how to implement this at the SQLAlchemy or SQL level, but that's beyond my needs for this project.

### Reading the Docs

Now that I have my data in memory in manageable chunks I can start writing to my FITS file. To do this I use the AstroPy `io.fits` [module](http://astropy.readthedocs.org/en/latest/io/fits/index.html). For those of you familiar with PyFITS you'll find that the code in AstroPyhas been wholly migrated over from PyFITS so the functionality is currently identical. So much so that you can still use the PyFITS docs to understand AstroPy's `io.fits`, which is great because the PyFITS FAQ explicitly answers this question: [How can I create a very large fits file from scratch?](http://pyfits.readthedocs.org/en/latest/appendix/faq.html#how-can-i-create-a-very-large-fits-file-from-scratch). Go ahead and read that section. 

With that FAQ as a starting point I ended up with this code snippet:

```python
if chunk == 0:
    data = np.zeros((1, 1, 1), dtype=np.float32)
    hdu = fits.PrimaryHDU(data=data)
    header = hdu.header
    header['NAXIS1'] = 11
    header['NAXIS2'] = 11
    header['NAXIS3'] = record_count
    header.tofile(fits_file_name, clobber=True)
    with open(fits_file_name, 'rb+') as fobj:
        fobj.seek(len(header.tostring()) + (11 * 11 * record_count * 4) - 1)
        fobj.write('\0')
bottom_index = chunk * chunk_size
top_index = (chunk + 1) * chunk_size
hdul = fits.open(fits_file_name, mode='update')
hdul[0].data[bottom_index:top_index,:,:] = numpy_data_cube 
hdul.close() 
```

The FAQ got me 80% of the way there and Erik helped me connect the dots. It's worth walking though this for this last 20% as well as an explanation of what exactly is going on.

### Hacking the Header

```python
data = np.zeros((1, 1, 1), dtype=np.float32)
hdu = fits.PrimaryHDU(data=data)
header = hdu.header
header['NAXIS1'] = 11
header['NAXIS2'] = 11
header['NAXIS3'] = record_count
header.tofile(fits_file_name, clobber=True)
```

If you were able to follow the FAQ the first 8 lines should make sense. I create a dummy NumPy array just to get the dimensionality right. Note that I explicitly create a single precision data type by specifying `dtype=np.float32`. Then I create a `PrimaryHDU` instance with that NumPy array. Under the hood the `fits` module is using this to set up some basic elements of the FITS file format. I then immediately hack those by changing the `NAXIS` keywords required by the FITS standard to match those of our expected output and not those of our dummy NumPy array. Changing this keyword doesn't do anything to actually change the dimensions or size of the file, those were set by our initial NumPy array. But it does update our header to match the data we'll be putting in. To wrap this section up I write the basic template of the file using the `tofile` method with the clobber option so that it overwrites the file if it already exists. As we keep going you'll see why we didn't just create the HDU with an array the size of our expected output in the first place.

### Getting Close to the Metal

```python
with open(fits_file_name, 'rb+') as fobj:
    fobj.seek(len(header.tostring()) + (11 * 11 * record_count * 4) - 1)
    fobj.write('\0')
```

Now comes the interesting part. One specific advantage Python offers scientists without a programming background is that Python is a "high level" programming language. The term "high" is subjective but the point is that Python takes care of many of the "low-level" aspects of programming such as memory allocation, pointers, and garbage collection. However, as you get further into the language you'll find that you have to learn how these concepts are implemented to take solve more complicated problems. This was one of those times for me.

I start by using the "new" standard Python convention of using `with` to open a file object. In this case I open it with the `rb+` setting which means I'm going to be read (`r`) and update (`+`) the file in binary mode (`b`). Binary mode means that rather than trying to encode strings in something like UTF-8 the file object will expect raw byte code. 

When you open a file object in binary update mode your current position is the beginning of the file. Meaning if you tell python to start reading or writing to the file it will start right a the beginning of the file. In my case I don't want that so I first use the `seek` function to tell Python how far ahead on the disk to skip in units of bytes. 

I can use the `tostring` method like I did in my last post about [writing NumPy arrays to MySQL]() and find the length of the header in bytes. Then I figure out how many bytes my data is going be by multiplying the number of elements in my array times the number of bytes required to store each element. This is interesting and something I haven't really thought about before; I can tell Python exactly how big my file will be before I write anything to it. I'm using a single precision, or 32bit, floating point, which can be represented by 4 bytes. So my data will take `11 x 11 x 4,000,000 x 4 byes` or a little over 1.8 GB. 

Now you can start to see why we needed to go through all this trouble. If we used a double precision float this would have been 3.6 GB of data. That would have to be read in from the database and then passed into an array and then written to the disk which would have required 7+ GB of memory. Plus the SQL query returns other fields and some of the memory is being used by system and suddenly it's clear why we couldn't do this all in memory. 

Also notice that I never had to tell Python _what_ was going to be in those bytes. An array of complex decimals takes up just as much space as an array of zeros if they're both stored as the same data type. _This_ is the reason we couldn't just create our original HDU with a 4,000,000 x 11 x 11 NumPy array of zeros - that would take up just as much room as a NumPy array of the real data!

But what about the `- 1` at the end? Well we go back just one spot from the end in this case to write the final character of a FITS file, the `\0` byte. In this case it acts as kind of like a place holder staking out how big the file is going to be. So what was the point of all that? Well, without even needing to create anything in memory anywhere near the size of our dataset we've now created a file exactly big enough to hold all our data. 

### (Finally) Writing the Data

```python
bottom_index = chunk * chunk_size
top_index = (chunk + 1) * chunk_size
hdul = fits.open(fits_file_name, mode='update')
hdul[0].data[bottom_index:top_index,:,:] = numpy_data_cube 
hdul.close() 
```

Now that we've created our output file of the correct size and dimensionality we can start writing our data chunks to the file. As you can see the 3rd line here uses `fits.open` to open our FITS file. But wait, what's going on here? Won't opening this file just read all the data into memory - the exact thing we're trying to avoid?

What's happening is that the `fits` module is by default opening the file with [mmap](http://en.wikipedia.org/wiki/Mmap). What this means is that the file is read in a "lazy" or "on-demand" mode, data is only read into memory as needed. You can find more info in both the [AstroPy](http://astropy.readthedocs.org/en/latest/io/fits/index.html#working-with-large-files
) and [PyFITS](http://pyfits.readthedocs.org/en/latest/appendix/faq.html#how-do-i-open-a-very-large-image-that-won-t-fit-in-memory) docs.

Look back at my code snippet you can see that I index the FITS data just like a NumPy array and then update it with the chunk from my current data cube. Iterating over this eventually write the entire file without ever needing to store the entire FITS data in memory at once.

[^1]: [FITS](http://en.wikipedia.org/wiki/FITS) is a standard data file format in astronomy.