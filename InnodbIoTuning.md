# Introduction #

InnoDB uses background threads to perform some IO operations including:
  * the purge thread physically removes deleted rows
  * the insert buffer thread updates secondary indexes
  * the log thread performs transaction log IO
  * the writer thread writes dirty buffer cache pages to disk
  * the reader thread prefetches blocks

Alas, one thread is not enough in many cases for the reader and writer, so we have added to configuration variables to set the number of threads for each of those:
  * innodb\_read\_io\_threads - number of threads to handle prefetch reads
  * innodb\_write\_io\_threads - number of threads to perform dirty page writes, see InnodbAsyncIo. For buffered IO, 1 thread may be sufficient

Rate limiting is used to prevent IO done by background threads from using all of the capacity of the server. The limit is based on the assumption that the server can do 100 IOPs. That is rarely true today, so we added a variable to specify the IOPs provided by the server:
  * innodb\_io\_capacity - number of disk IOPs the server can do

Finally, one more parameter has been added that you can try:
  * innodb\_extra\_dirty\_writes - flush dirty buffer pages when dirty pct is less than max dirty pct