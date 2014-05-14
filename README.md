logstash-multiline
==================

An alterative version of Multiline filter which implements LRU cache with TTL and Max Size settings.

The multiline filter is for combining multiple events from a single source
into the same event.  The multiline filter in master implements a hash table to store messages and then applies regex to determine whether a message should be joined with the "previous" or "next" in a stream.

I found that the original multiline.rb filter was problematic in certain scenarios where multiline messages are "transaction-oriented" and the stream_identity field is effectively unique for a given series of messages.  This can result in 1-or-more messages being received and the stream_identity never being used again.  In logstash pre-1.4.1 the filter flusher is disabled, so these messages could end up "stuck" in the filter and never output back to the pipeline.  Re-enabling the filter flusher mostly fixed that problem but there are cases where part of a message is received, then filter flusher runs, then the remainder message is received and this resulted in fragmented messages.

I re-wrote the multiline filter to implement a Least Recently Used cache, based on a modified version of the excellent cache.rb.  The library implements a maximum size (count) of messages in the cache, and a max TTL (seconds).  The filter flusher triggers a TTL check, and the max size only comes into play if the cache fills up and the filter tries to add more.   Evicted messages go back to the pipeline.

Configure your shipper.conf like:

  multiline {
  
    pattern => "."
    
    negate => false
    
    what => "streamcache"
    
    stream_identity => "%{field1}.%{field2}"
    
    cache_ttl => 2
    
    cache_size => 50000
    
  }

