Infinispan recently added support for streams to be used in a performant way on a distributed cache.  Unfortunately some methods such as sort were not implemented in a distributed way.  Instead the entire contents of the cache was copied to the originating node and a final sort was done on all of the contents in memory.  This while valid can be an extremely inefficient usage of memory and in some cases even run the node out of memory resulting in an OutOfMemoryException.

This however will change when Distributed Sorting is introduced.  The general idea is a [merge sort](https://en.wikipedia.org/wiki/Merge_sort) where each node contains a sub set of data that is independently sorted first.  Then the data is finally sorted at the originator.  This way we can also utilize the CPUs across the cluster to do initial sorting on each node so the coordinator only has to sort sets of already sorted data which is faster.

Even doing just a merge sort will not help with memory.  In actually it can be a bit worse because now each node could possibly run into OutOfMemory errors if they have cache stores that have more data than they could hold in memory.  Thus we need some way of using batched sizes to prevent this as well.  The distributedBatchSize is used to control how many sorted elements are retained at a time on each node before sending those batches to the originator.  This then requires each node to iterated upon a number of times equal to N / b times where N is the number of primary owned elements on that node and b is the batch size.

## Non Rehash

Below is the pseudo code for the remote and coordinator side sorting for use when doing a sort when rehash is not supported.

### Remote Node Sort

    array = new Array[batchSize * 2]
    offset = 0
    foreach element in cache {
        if offset < batchSize || array[batchSize -1] < element
            array[offset++] = element
            if offset == array.length
                sort array
                offset = batchSize
    }

Then when the entries are used for a stream the array is sorted and only the elements up to batchSize are returned.  Subsequent iterations only add an element to the array if the element is larger than the last element from the last batch.

The local coordinator holds all the responses from the various nodes including itself with a BoundedQueue which blocks that thread until it has sent all of its responses.

### Coordinator Sort

    array = new Array[nodeCount]
    nodeReturnedLastValue = null
    completedNodes = 0
    
    method getNext
    if nodeReturnedLastValue == null
        offset = 0
        foreach node
            array[offset++] = node.queue.next
        sort array
    else
        value = nodeReturnedLastValue.queue.next
        if value != null
            array[completedNodes] = nodeReturnedLastValue.queue.next
            sort array
        else
            completedNodes++

    nodeReturnedLastValue = node where value originated
    return array[completedNodes]

## Rehash sort

Rehash sort is quite a bit similar to the non rehash.  If a node finds that a rehash occurs while iterating upon the cache entries it will send a message to the originator to say what the new view id is.  The originator upon receiving said message will wait for all nodes to send the same message or a completion message (in the case the node completed iteration without the rehash).  The originator after this state has occurred will only allow elements received until 1 node has run out but wasn't completed.  At this point any additional elements retrieved must be ignored.  The originator will then send a new batch of requests but will also send along the highest value it has seen to filter out any values that are smaller than it in the iteration process.