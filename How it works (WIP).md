### Rationale.
Akumuli is trying to solve real-world problem - time series data storage. Time-series data is very specific, because of that existing storage solutions doesn’t suits well for this problem. Specific properties of time-series data includes:
* Very fast ingestion speed, millions of inserts per second or greater. Writes have priority over reads.
* But this data arrives almost ordered, disorder is slight and caused by network delays.
* Writes to distant past usually means error in data source (broken sensor).
* Old data is immutable, data doesn’t updated, only outdated data need to be deleted.
* Range read queries must be fast, not point queries.
* This data can be efficiently compressed.

Generic databases (K/V stores or RDBMS) have very different set of properties, they must implement random reads, writes and deletes efficiently. This is very difficult to achieve but doesn’t needed by time-series database. 
This project tries to fill this gap. Akumuli designed specifically for time-series data. It provides hight speed and good compression ratio.

### Big picture.
Akumuli implemented as dynamic C library. It’s written in C++ and have simple C-style interface. Disk space is allocated in 4Gb chunks (volumes), this volumes organaized as a ring buffer. When you write data to akumuli your data goes to one of the volumes, when this volume fills - next volume will be selected and become active. Because of that, lifetime of your data depends on two things - write rate and size fo the storage.

Akumuli doesn’t use any of the B-tree incarnations(TSDB-tree, HV-tree) instead volumes is just a sorted files. Volumes organaized very similar to pages in relational databases. Each volume have inderection vector in the beginning - array that contains offsets of the data elements. This design allows to store data elements of variable size and sort elements by sorting only inderection vector. This design allows very fast writes, and simplifies other things like data recovery. All data are written in reverse order from bigger to smaller addresses. Inderection vector grows in ascending order. The page becomes full when they mets somewhere in a middle of the volume.

Akumuli disallows late writes, there is sliding window and everything that doesn’t fit to this sliding window can’t be written to akumuli. You can set up sliding window size depending on your conditions.

Akumuli does have cache. All the data-points that fits inside sliding window are cached. Cache are used to search recent data and to sort data before it will be written to disk. Cache is based on patience sort algorithm, it’s just a series of sorted runs that can be binary searched.

All volumes in akumuli does have index to speed up searching. Index in akumuli is approximate and constant size - 1Mb. Approximate means that it doesn’t always guide search algorithms directly to searched element, instead of that, it can guide search algorithm to the region when searched element is placed. This index is an approximate histogram of the data, all writes is sampled to this histogram with some probability.

### Searching.
Akumuli is powered by the stack of search algorithms. When you perform some search, histogram is used first, it narrows the search drastically. After that interpolation search is used, it stops when data-element is found or number of interpolation search steps is greater than some small value or when search range is narrowed to single page. After that binary search is taking place.

### Performacne.
Write performacne: more than one million writes per second is possible on commodity hardware (this isn’t a batch writes).
Read performance: akumuli is good at retrieving large ranges of time-series data, more than one million sequential reads on commondity hardware is what you can excpect. Akumuli is not very good at random reads - on my machine I can’t get more than 30 000 req/sec when reading data elements by one in random order (all the testing is done on 32Gb database with one billion elements that doesn't fits in RAM).